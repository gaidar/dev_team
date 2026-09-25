# BUILD.md – install the dev team into this repository

## Task

Read `RUNBOOK.md` next to this file. Interview the user once. Then generate the assets in section 3 so the team described in RUNBOOK.md can be started with `./team` and given work with `/team <task>`.

## Setting

You are Claude Code, running interactively from the repository root. RUNBOOK.md is the specification; this file is the procedure. You may ask questions only in the interview (section 2). After the interview, do not ask anything: assume, proceed, and log every assumption in the final report (section 7).

## 1. Preflight (read only, write nothing)

Detect and note:

- Git: is this a repository; default branch; remote host (GitHub, GitLab, Bitbucket, other, none); whether `gh` or `glab` is installed and authenticated.
- Existing assets: `CLAUDE.md`, `.claude/agents/`, `.claude/skills/`, `.claude/commands/`, `.claude/settings.json`, `docs/team/`, `README.md`.
- Stack: language, runtime, package manager; commands for install, build, test, lint, taken from `package.json`, `pyproject.toml`, `Makefile`, `go.mod`, `Cargo.toml` or equivalents; the test directory.
- Claude Code: `claude --version`; whether `claude --help` lists `--agent`, `--permission-mode` and `--teammate-mode`; whether `--permission-mode auto` is accepted on this account (auto mode is a research preview and not available on every plan).
- Models: whether the alias `fable` is accepted in an agent file `model:` field. Test it on a throwaway agent file if the docs are unclear; if it is not accepted, use the full ID from the table in section 8.
- OS, for the launcher script.

## 2. Interview

Ask all questions in one message. Show the detected value in brackets as the default. Accept the single word `defaults` as an answer to everything. If `docs/team/BUILD_CONFIG.md` exists, show its answers as the defaults instead and offer `reuse`.

1. Repository: new or existing. [detected]
2. Commands: install, build, test, lint. [detected]
3. Default branch and task branch prefix. [detected, `team/`]
4. Merge mode default: `pr` or `lead`. [`pr`] Merge strategy in lead mode. [`squash`]
5. Git host and PR tool. [detected]
6. Roles to include from RUNBOOK section 2. [all seven] Maximum developers per task. [3]
7. Model per role. [the feature and build column of RUNBOOK section 13.1: Fable 5.1 for lead, planner, pm, architect, reviewer; Opus 5.5 for developer and qa] The user may change any of these now. The fix and change column stays a plan-time choice, not an agent file setting.
8. Permission allow list. [proposed from the detected commands: the install, build, test and lint commands, `git status/diff/log/add/commit/checkout/branch/push` on `team/*` branches, `gh pr *` or `glab mr *`, `Read`, `Edit`, `Write`] Deny list. [`rm -rf`, `git push --force`, `git reset --hard`, anything under `.env*`, `secrets/`, `*.pem`, `*.key`]
9. Display mode. [`in-process`] Teammate prompt cache TTL: `1h` if approvals at gates usually take more than five minutes, otherwise default. [`1h`]
10. Hooks: install a `TaskCompleted` gate that blocks completion when the deliverable path named in the task is missing. [yes] Install a `TeammateIdle` nudge. [no]
11. Task id scheme. [issue number when present, otherwise `T-YYYYMMDD-NN`]
12. Existing assets with the same name: `skip`, `replace` or `merge`. [`skip`]
13. Commit the generated files on a branch `team/setup`. [yes]
14. Anything in RUNBOOK.md to adjust for this repository. [none]

Write the answers to `docs/team/BUILD_CONFIG.md` before generating anything else. Every later step reads from that file, not from memory.

## 3. Assets to build

### 3.1 Agent definitions, `.claude/agents/`

One file per included role: `lead.md`, `planner.md`, `pm.md`, `architect.md`, `developer.md`, `reviewer.md`, `qa.md`. Frontmatter: `name` (equals the file stem), `description` (one line: what the role does and when the lead spawns it), `tools`, `model` (the interview answer to question 7; alias if accepted, otherwise the full ID; if neither works, `inherit`). The lead names the model in every spawn prompt regardless, because the fix and change tiers use cheaper models than the agent file default (RUNBOOK section 13.1) and a model named in the spawn prompt overrides the file.

Body, at most 60 lines, in this order: role in two sentences; files to read first (CLAUDE.md, RUNBOOK.md, the plan path given in the spawn prompt); owned paths; deliverables; done criteria; report format (RUNBOOK section 11); forbidden actions; who to message for what (lead for blockers and completion, planner for decisions). Reference RUNBOOK sections by number instead of copying them.

Tools per role. Tool lists are allowlists and are not path-restricted, so path limits go in the body as rules.

| Role | Tools |
|---|---|
| lead | all tools; body rule: writes only `docs/team/STATUS.md` and git operations; never edits application code or tests; delegates reading of large files |
| planner | Read, Grep, Glob, Write, Edit; body rule: `docs/team/` only |
| pm | Read, Grep, Glob, Edit; body rule: the acceptance criteria section of the plan only |
| architect | Read, Grep, Glob, Write, Edit; body rule: `docs/` only |
| developer | Read, Grep, Glob, Edit, Write, Bash; body rule: assigned paths only, task branch only |
| reviewer | Read, Grep, Glob, Bash, Write; body rule: read-only commands (`git diff`, `git log`, test and lint runs); writes `docs/team/reviews/` only |
| qa | Read, Grep, Glob, Edit, Write, Bash; body rule: test directory and `docs/team/qa/` only |

Claude Code adds messaging and task tools to teammates itself; do not list them.

The lead file is special: it is the system prompt of the main session when started with `claude --agent lead`. Its body must contain the full task procedure from RUNBOOK section 4 in imperative form, the gate vocabulary from section 6, the spawn prompt template from section 4 step 4, and the rule that it spawns exactly the roster in the approved plan and nothing else. It must also contain the recovery table from RUNBOOK section 12.

### 3.2 Skills, `.claude/skills/`

- `team/SKILL.md`, name `team`. Body: parse `$ARGUMENTS` as free text, `#<number>` (fetch the issue with `gh` or `glab`), or a path to a spec file (read it). Then run RUNBOOK section 4 steps 1 to 3 and stop at the plan gate. If `$ARGUMENTS` is not substituted, use the text after the command.
- `check-status/SKILL.md`, name `check-status`. Body: read `docs/team/STATUS.md` and the shared task list, list teammates and their states, report in twelve lines or fewer. Do not name it `status`; that name is taken by a built-in command.
- `stop-team/SKILL.md`, name `stop-team`. Body: ask every teammate by name to shut down, wait for confirmations, write unfinished work to `STATUS.md`, report.

Each SKILL.md has frontmatter `name` and `description`. Bodies under 40 lines.

### 3.3 Settings, `.claude/settings.json`

- `env.CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` = `"1"`.
- `permissions.allow` and `permissions.deny` from the interview, in the `Tool(pattern:*)` rule syntax used by Claude Code.
- `teammateMode` only if the interview chose something other than in-process.
- `subagentPromptCacheTtl` = `"1h"` if the interview chose it.
- Hooks per the interview. Before writing any hook, read the current hooks documentation (section 8) for the event names, the JSON delivered on stdin, and the exit code convention (exit 2 blocks and returns feedback). Hook scripts go in `.claude/hooks/` and must be executable.

Merge into an existing settings file key by key; never drop an existing key; never change existing permission rules, only add.

### 3.4 CLAUDE.md

If a `## Dev team` section is absent, append one of at most 15 lines: RUNBOOK.md is the rulebook and wins over anything else in CLAUDE.md on conflict; the three commands; the one-owner-per-path rule; where project memory lives. If CLAUDE.md does not exist, create it with that section only.

### 3.5 Project memory, `docs/team/`

Create if absent: `REQUIREMENTS.md`, `DECISIONS.md`, `STATUS.md`, `plans/README.md` (holds the plan template from RUNBOOK section 5), `reviews/.gitkeep`, `qa/.gitkeep`, and `BUILD_CONFIG.md` from step 2. Each memory file starts with a two-line header saying who owns it and its format, taken from RUNBOOK section 8.

### 3.6 Launcher

`team` (bash, executable) and `team.cmd` (Windows). Behavior:

- Export the agent-teams variable, then run `claude --agent lead` plus `--teammate-mode` from the config. Pass any other arguments through, with the exception below.
- `--auto` maps to `--permission-mode auto`: Claude Code's auto mode, where a background classifier reviews each tool call and blocks destructive ones instead of asking you. If preflight found that auto mode is not accepted on this account, `--auto` prints one line saying so and starts in the default mode.
- The launcher never passes `--dangerously-skip-permissions` or `--permission-mode bypassPermissions`. If either is given as an argument, it prints `This team never runs without permission checks. Use --auto.` and exits without starting Claude Code.
- If preflight found no `--agent` flag, run `claude` with the initial prompt `Act as the lead defined in .claude/agents/lead.md. Read RUNBOOK.md. Wait for a task.`

Auto mode drops broad allow rules such as a blanket `Bash(*)` or wildcarded interpreters when it starts, and keeps narrow ones. Write the allow list from question 8 as narrow rules (`Bash(npm test)`, `Bash(git commit:*)`) so it survives in both modes.

### 3.7 README

Append a `## Dev team` section of at most 10 lines: the three commands and a link to RUNBOOK.md. Skip if a section with that heading exists.

## 4. Rules

- Never overwrite an existing file. Follow the interview answer for same-name assets; report every skip.
- Idempotent: a second run with the same answers changes nothing and says so.
- Touch nothing outside the paths in section 3. No changes to application code, tests, CI, dependencies or `.gitignore`.
- No secrets in any generated file.
- Generated files use sentence-case headings and plain language. No marketing tone.
- Commit only if the interview said yes, on the branch it named, with the message `team: install dev team assets`. Never push.

## 5. Verification (run before the report)

1. Every agent file parses: frontmatter present, `name` equals the file stem, `model` is a value Claude Code accepts (test one file if unsure).
2. `.claude/settings.json` is valid JSON and every permission rule follows the documented syntax.
3. Every flag used by the launcher appears in `claude --help`.
4. The launcher starts (`./team --help` or equivalent dry run) and exits cleanly.
5. Walk RUNBOOK section 4 step by step and confirm the lead agent body and the `team` skill contain an instruction for each step and each gate word from section 6.
6. Hooks, if installed, run against a sample payload and exit 0 on a present deliverable and 2 on a missing one.

## 6. Definition of done

1. Every file in section 3 exists for every included role, or is listed as skipped with the reason.
2. `docs/team/BUILD_CONFIG.md` holds the interview answers.
3. All six verification checks pass, or the failing check is in the could-not-verify list with what was tried.
4. Nothing outside section 3 paths changed (`git status` shows only those paths).
5. The final report in section 7 was printed.

## 7. Final report

Print, in this order:

1. Files created, modified, skipped, each with one line.
2. Interview answers used.
3. Assumptions made after the interview.
4. Could not verify: items and what was tried (for example the `fable` alias, the hooks payload schema, host CLI authentication).
5. Next steps: `./team`, then `/team <task>`. Suggest one smoke test at fix tier, for example adding a `--version` flag or fixing a typo in the README, so the user sees the full loop within ten minutes.

## 8. Reference

Model IDs, in case aliases are not accepted:

| Name | ID |
|---|---|
| Fable | `claude-fable-5-1` |
| Opus | `claude-opus-5-5` |
| Sonnet | `claude-sonnet-5` |
| Haiku | `claude-haiku-4-5-20251001` |

If the account's model allowlist blocks a model, Claude Code substitutes another; report which roles were affected. Fallback order when a model is unavailable: Fable 5.1 → Opus 5.5 → Sonnet 5. Never fall back to Opus 5 or Fable 5; they cost more than their successors.

Documentation to read when a detail is unclear (do not guess):

- Agent teams: https://code.claude.com/docs/en/agent-teams
- Subagents and agent file format: https://code.claude.com/docs/en/sub-agents
- Skills: https://code.claude.com/docs/en/skills
- Hooks: https://code.claude.com/docs/en/hooks
- Settings and permission rules: https://code.claude.com/docs/en/settings
- CLI flags: https://code.claude.com/docs/en/cli-reference

Known constraints of agent teams that the generated assets must respect: the main session is the lead for its lifetime; one team per session; teammates cannot spawn teammates; in-process teammates do not survive `/resume`; teammates load CLAUDE.md and the spawn prompt but not the lead's conversation; plan-mode approvals are granted by the lead session without human review, so human gates live in the task list, not in plan mode; teams form only in interactive sessions, not with `-p`.

## 9. Re-running

When `docs/team/BUILD_CONFIG.md` exists: offer `reuse`, regenerate agent and skill files from the current RUNBOOK.md and config, diff against the existing files, apply only where content changed, and print the diff summary. Ask nothing else.
