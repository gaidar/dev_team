# Development team setup

Everything to create so PLAYBOOK.md runs from a clean clone. Verified against the Claude Code docs on September 25, 2026.

How to use with Claude Code: from the repo root run `claude --permission-mode plan`, then:

> Build the package described in BUILD.md. Follow each spec line. Check every command and frontmatter field against code.claude.com/docs before writing it. Ask before deviating. Run the acceptance tests in section H before reporting done.

## Target layout

```
repo/
├── README.md
├── PLAYBOOK.md
├── BUILD.md
├── CLAUDE.md
├── .gitignore
├── .worktreeinclude
├── .claude/
│   ├── settings.json
│   ├── settings.local.json.example      
│   ├── agents/
│   │   ├── qa.md
│   │   ├── reviewer.md
│   │   ├── developer.md                 
│   │   └── sensitive-path-reviewer.md   
│   └── skills/                          
│       ├── brief/SKILL.md
│       ├── handoff/SKILL.md
│       └── standup/SKILL.md
├── prompts/
│   ├── build-brief.md
│   ├── qa-brief.md
│   ├── review-brief.md
│   └── handoff-message.md
└── scripts/
    └── check.sh                         
```

## A. Policy and configuration

- [ ] **CLAUDE.md** – team policy. Loaded by every session and subagent. Sections, in order:
  1. One-paragraph project summary and stack.
  2. Commands: run, test, lint, build. Exact invocations.
  3. Git: branch pattern (`feat/<ticket>-<slug>`, `fix/…`), commit format (imperative, under 72 chars, ticket ref). Then, verbatim: "Do not commit or push. The lead commits and opens PRs." Without this line a background session commits without asking, pushes when a remote exists, and may open a draft PR.
  4. Do-not-touch list: directories and files no agent edits without a human instruction.
  5. Sensitive paths: money, auth, persistence. Rule: stop and ask before editing. Sessions on these paths run in manual mode (`--permission-mode default`).
  6. Messaging rule: when you change a shared interface, schema or contract, message every session working on an affected area with what changed, why, and what they must do.
  7. Definition of done default: tests pass, lint passes, diff summary written, no unrelated refactors.
  8. Review rules for `/code-review` (it reads `CLAUDE.md`, not `REVIEW.md`).
  Acceptance: under 150 lines; every line is a rule, not an essay. Prose shapes behavior; only settings, hooks and tool allowlists enforce it.

- [ ] **.claude/settings.json** – limits and guardrails:
  ```json
  {
    "env": {
      "CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH": "1",
      "CLAUDE_CODE_MAX_CONCURRENT_SUBAGENTS": "20"
    },
    "permissions": {
      "deny": [
        "Bash(git push *)",
        "Bash(rm -rf *)",
        "Bash(git reset --hard *)"
      ]
    },
    "worktree": { "baseRef": "head" }
  }
  ```
  - `Bash(git push *)` blocks every push, including from the lead's own Claude session; the lead pushes from the shell. A narrower `Bash(git push --force *)` only matches commands that start with those words and misses `git push origin --force`.
  - `baseRef`: `head` so worktrees carry the branch's committed work; `fresh` (default) starts every worktree from the remote default branch. Judgment call; document the choice in README.
  - Manual mode is not set here. Project settings can set `"permissions": {"defaultMode": "default"}` but can't set `auto` or `bypassPermissions`. Leave auto as default; start sensitive sessions with `--permission-mode default`.

- [ ] **.gitignore** – `.claude/worktrees/`, `.claude/settings.local.json`, `.claude/agent-memory-local/`.

- [ ] **.worktreeinclude** – `.gitignore` syntax. `.env`, `.env.local`, project-specific local config. Only gitignored matches are copied. Applies to `--worktree` and subagent worktrees; not processed when a `WorktreeCreate` hook replaces git.

- [ ] **.claude/settings.local.json.example** – template for personal overrides (model, effort). Real file is gitignored.

## B. Agents – `.claude/agents/`

Rules: `name` and `description` required; `name` unique across the tree; `description` says when to use it; multi-word fields camelCase (`maxTurns`, `disallowedTools`); unknown fields are ignored without error. Validate with `claude plugin validate .claude/agents` (checks that frontmatter parses; doesn't flag a missing `name`). Edits apply live; the first file in a new `agents` directory needs a restart.

Read-only in practice: `Edit` and `Write` absent from `tools`. `Bash` can still write through the shell. Acceptance is `git status` clean after a run. For enforcement add a `PreToolUse` hook on `Bash`.

| File | Job | tools | model | effort | isolation | maxTurns |
|---|---|---|---|---|---|---|
| `qa.md` | Run the app, break the feature, report defects. Never fix. | Read, Glob, Grep, Bash | sonnet | high | none | 30 |
| `reviewer.md` | Strict review of a diff; findings only, no refactors | Read, Glob, Grep, Bash | opus | xhigh | none | 8 |
| `developer.md` | Build from a brief; plan first; message QA when ready | omit (inherits all) | inherit | high | none | none |
| `sensitive-path-reviewer.md` | BLOCK or ALLOW verdict on diffs touching money, auth, persistence | Read, Glob, Grep | opus | max | none | 6 |

No `isolation: worktree` on any role. A subagent worktree branches from the base branch, so it never holds the uncommitted diff it is asked to review or test. Background sessions get their own worktree without the field.

- [ ] **qa.md**
  - Frontmatter as in the table. Add inline `mcpServers` for the test harness if the project needs one (Playwright for web, simulator tooling for iOS). Inline definitions load only after the repo folder is trusted.
  - Body: role; how to launch the app (reference CLAUDE.md commands); flows to exercise; what counts as a defect; output contract: numbered list with steps to reproduce, expected vs actual, severity (blocker / major / minor), file or screen reference; final one-line summary for `SendMessage` to the developer session.
  - Runs against the developer's branch: dispatch from agent view as `qa @<developer-worktree> <brief>` (typing `@` lists worktrees under `.claude/worktrees/`), or `cd .claude/worktrees/<name> && claude --bg --name qa --agent qa "<brief>"`. A session started inside a worktree stays there.
  - Acceptance: report matches the contract; `git status` in the worktree is unchanged.

- [ ] **reviewer.md**
  - Body: assume concurrent access everywhere; flag any change to a public API surface without a migration note; no unrequested refactors; numbered findings with file and line; priority order critical / warning / suggestion.
  - `memory: project` plus a body line to record recurring issues in memory. Needs auto memory on; writes to `.claude/agent-memory/reviewer/`, committed.
  - Acceptance: `@"reviewer (agent)" review the uncommitted changes` returns numbered findings; `git status` unchanged.

- [ ] **developer.md** – runs as the session's main agent: `developer <brief>` in the agent view dispatch input, or `claude --bg --name developer --agent developer "<brief>"`. `--agent` replaces the default system prompt entirely (CLAUDE.md still loads), so the body must be complete: read the brief, produce a plan and wait for approval, follow CLAUDE.md git rules, run tests before reporting, message `@qa` when the feature is ready and again whenever a shared interface changes, stop and ask before touching sensitive paths.
  - Plan mode comes from the dispatch, not the file: open the board with `claude agents --permission-mode plan`.
  - Acceptance: session header shows `@developer`; it asks for plan approval before editing.

- [ ] **sensitive-path-reviewer.md** – invoked by the lead on any diff touching CLAUDE.md section 5 paths. Output: `BLOCK` or `ALLOW`, then numbered reasons with file and line. Nothing else.

## C. Prompt templates – `prompts/`

Plain markdown the lead pastes or references. Each under 40 lines.

- [ ] **build-brief.md** – Ticket / Scope / Out of scope / Constraints (performance, compatibility, dependencies, sensitive paths touched yes/no) / Definition of done / Plan first, wait for approval.
- [ ] **qa-brief.md** – Feature under test / Worktree to test in / How to launch / Flows to break (happy path, edge cases, error paths, concurrency) / What not to do (no fixes, no refactors) / Report format / Report to (session name).
- [ ] **review-brief.md** – for `/code-review` or `@reviewer`: What changed / Risk areas / Public surfaces touched / What not to flag (style, unrequested refactors) / Output format.
- [ ] **handoff-message.md** – for cross-session messages: What changed / Why / Who is affected / Action needed / Deadline or blocker status. One screen max.

## D. Skills – `.claude/skills/`

Format: `.claude/skills/<name>/SKILL.md`; the directory name is the command. Frontmatter: `description` (recommended), `disable-model-invocation: true` (lead-only), `argument-hint`. Fields are lowercase-hyphenated; unknown fields ignored. `$ARGUMENTS`, `$0`, `$1` substitute arguments; a line starting `` !`cmd` `` injects command output before Claude reads the skill. Validate with `claude plugin validate .claude/skills`.

- [ ] **brief/SKILL.md** – `/brief <ticket or text>`: fill `${CLAUDE_PROJECT_DIR}/prompts/build-brief.md` from `$ARGUMENTS`, then ask only the two questions it cannot infer (sensitive paths, definition of done). Runs inline so the brief lands in the conversation.
- [ ] **handoff/SKILL.md** – `/handoff <session> <change>`: `arguments: [session, change]`; draft from `prompts/handoff-message.md`; send to `@$session` with `SendMessage`.
- [ ] **standup/SKILL.md** – `/standup`: inject `` !`claude agents --json --all` ``; `allowed-tools: Bash(claude agents *)` so the injection isn't blocked outside auto mode; answer in three lines: blocked on me, ready for review, should be stopped.

## E. Guardrails

- [ ] Depth 1 in settings. Verify: a subagent that tries to spawn a subagent has no `Agent` tool.
- [ ] Deny list in settings. Verify: `git push` is refused in a session.
- [ ] Sensitive-path rule in CLAUDE.md section 5; README instructs starting those sessions with `--permission-mode default`. Subagents inherit the main session's mode; a `permissionMode` in a role file is ignored under auto.
- [ ] Review-capacity number in README ("at most N build sessions per lead per day").
- [ ] Reviewer and QA: `Edit` and `Write` removed by allowlist; Bash writes only caught by the `git status` acceptance check unless the hook is added.

## F. Documentation

- [ ] **README.md** – one paragraph on the package; ten-line quickstart (clone, `claude --version` check, trust the folder with one interactive `claude` run, `/login`, `claude agents`, dispatch developer and qa); link to PLAYBOOK.md; the `baseRef` decision; the capacity number; tested-with version.
- [ ] **PLAYBOOK.md** – copy as is.
- [ ] **BUILD.md** – this file; keep it so the package can be rebuilt.
- [ ] **scripts/check.sh** – `claude --version`, `claude plugin validate .claude/agents`, `claude plugin validate .claude/skills`, `.gitignore` contains `.claude/worktrees/`; prints pass/fail.

## G. Prerequisites to confirm at build time

- [ ] `claude --version` is 2.1.247 or later. Covers `/subtask`, session `@`-mentions, default fork mode, messaging on native Windows, `claude plugin validate`, message previews.
- [ ] macOS, Linux, WSL 2 or native Windows. A WSL session and a native Windows session on one machine can't message each other.
- [ ] Pro, Max or Team plan if auto permission mode is expected by default.
- [ ] claude.ai login (`/login`) for `/code-review ultra` and Remote Control. Neither works on Bedrock, Google Cloud's Agent Platform, Foundry, or under zero data retention. Ultrareview bills usage credits after 3 free runs (Pro, Max); none free on Team, Enterprise.
- [ ] Git repository with a remote named `origin` (needed for `--worktree "#<pr>"` and `fresh` base).
- [ ] Folder trusted once (interactive `claude` run) so `--worktree`, inline `mcpServers` and agent frontmatter hooks work.

## H. Acceptance tests – run before calling it done

1. `claude plugin validate .claude/agents` and `claude plugin validate .claude/skills` pass.
2. Two terminals: `claude --name developer` and `claude --name qa`. `/list-agents` in each lists the other. A message sent from one shows a `Message from @…` line in the other; `Ctrl+O` shows the full text.
3. `@"reviewer (agent)" review the uncommitted changes` returns numbered findings; `git status` shows no new changes.
4. `@"qa (agent)" edit README.md` is declined; no file changes.
5. `claude --bg "list the files in this repo"` appears as a row in `claude agents`; a background session that edits does so under `.claude/worktrees/`.
6. `/subtask summarize the conversation so far` shows a row in the panel and returns a result.
7. `/code-review` runs in the background; `/tasks` lists it; findings arrive in the conversation.
8. A subagent asked to spawn a subagent reports it has no `Agent` tool.
9. `git push` from inside a session is refused.
10. Dispatch `qa @<worktree> smoke test the app` from agent view; `/status` in that session shows the worktree as working directory.
11. Fresh clone on a second machine: tests 1 to 4 pass with no setup beyond trust and login.

## I. Not in scope

- Agent teams (experimental, separate coordination model).
- CI integration of ultrareview (research preview).
- Language- or framework-specific tooling beyond the QA harness placeholder in `qa.md`.
