# Claude Code development team playbook

Commands and settings for running a development team in Claude Code. Verified against the Claude Code docs on September 25, 2026.

## Operating model

The lead assigns, unblocks, and reviews. No hand-written code, no repeated context. Few sessions, each with a written role and its own worktree. The lead reads every diff before committing.

## Five rules

1. One board: `claude agents` is the standup.
2. Information moves sideways: sessions message each other.
3. Fork for context, spawn for isolation. Task needs the current conversation: `/subtask`. Independent: new session in its own worktree.
4. Documented roles: each role is a subagent file with a tool allowlist.
5. Team size based on team lead capacity to review.

## Part 1 – one-time setup per repository

### Step 1. `CLAUDE.md` as team policy

Loaded by every session and subagent. Contents:

- Branch naming and commit message format.
- When to open a draft PR.
- Test and lint commands.
- Directories not to touch.
- Sensitive paths: money, auth, persistence.
- Who commits. Without this line a background session commits without asking, pushes when a remote exists, and may open a draft PR. If the lead commits, write: "Do not commit or push. The lead commits."

### Step 2. Worktrees

- Add `.claude/worktrees/` to `.gitignore`.
- Create `.worktreeinclude` (`.gitignore` syntax) listing gitignored files every worktree needs: `.env`, local config.
- Base branch: new worktrees branch from the remote default branch (`"fresh"`, the default). Set `"worktree": {"baseRef": "head"}` in `.claude/settings.json` to branch from the current local HEAD instead. Applies to `--worktree` sessions and subagent worktrees. A new worktree never contains uncommitted changes.
- Background sessions move into their own worktree under `.claude/worktrees/` before their first edit. No flag needed.

### Step 3. Roles in `.claude/agents/`

Frontmatter per file:

- `name`, `description` – required. `description` states when to use the role; Claude routes on it.
- `tools` – allowlist. `Read, Glob, Grep` is read-only. `Bash` adds test runs but can also write through the shell; block writes with a `PreToolUse` hook if that matters. To block pushes, add `Bash(git push *)` to `permissions.deny` in settings. A `disallowedTools` entry with a specifier removes the whole tool.
- `isolation: worktree` – own checkout, base branch per Step 2.
- `permissionMode` – ignored when the main session is in auto, acceptEdits or bypass; the subagent takes the main session's mode.
- `effort`, `model` – matched to task difficulty. Effort: `low`, `medium`, `high`, `xhigh`, `max`; available levels depend on the model.
- `maxTurns` – hard stop. Output returns marked partial; Claude can resume it.

Validate: `claude plugin validate .claude/agents`. Checks that frontmatter parses; does not flag a missing `name`.

Files are watched. Edits apply within seconds, except the first file in a new `agents` directory, which needs a restart.

### Step 4. Limits in `.claude/settings.json`

```json
{
  "env": {
    "CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH": "1",
    "CLAUDE_CODE_MAX_CONCURRENT_SUBAGENTS": "20"
  },
  "permissions": {
    "deny": ["Bash(git push --force *)", "Bash(rm -rf *)"]
  }
}
```

- `CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH`: `1` turns nesting off (default 3). Keep `1` on production code.
- `CLAUDE_CODE_MAX_CONCURRENT_SUBAGENTS`: default 20. Raise only on `Concurrent subagent limit reached`. `/subtask` takes a slot but is never blocked.
- Permission mode: auto is the default on Pro, Max and Team (a classifier gates each action). For a repository that must run in manual mode, add `"permissions": {"defaultMode": "default"}` here. Project settings can restrict the mode; they can't set `auto` or `bypassPermissions`. For one session: `--permission-mode default`.

### Step 5. Commit

`CLAUDE.md`, `.claude/agents/`, `.claude/settings.json`, `.worktreeinclude` go into version control.

## Part 2 – daily loop

1. Board: `claude agents`. Groups: Pinned, Ready for review, Needs input, Working, Completed. Clear blocked rows first. `Space` peeks and replies, `Enter` attaches, `←` detaches. Terminal sessions appear only after `/bg` or `←`.
2. Build session: in the dispatch input, `developer <brief>`. A first word matching a subagent name runs it as the session's main agent; `@developer` anywhere in the prompt does the same. Brief = scope, constraints, definition of done. Open the board with `claude agents --permission-mode plan` so dispatched sessions start in plan mode; peek or attach to approve the plan before code. Dispatched sessions get auto-generated names: rename with `Ctrl+R`, or dispatch from the shell with `claude --bg --name developer --agent developer "<brief>"`, so `@developer` resolves.
3. QA session: `qa <brief>` or `claude --bg --name qa --agent qa "<brief>"`. Role: run the app, break the feature. Does not build.
4. Side tasks that need the conversation: `/subtask <task>`, e.g. `/subtask draft unit tests for the parser changes so far`. Inherits the full conversation and prompt cache, runs in the background, returns the result to the conversation. Use for tests of the current change, a decision summary, a second opinion.
5. Independent tracks: `/fork <prompt>` (copy of this session as a new background session, own worktree), `claude --bg "<prompt>"`, or `claude --worktree "#128"` (session branched from PR or MR 128; GitHub and GitLab URLs accepted).
6. Cross-session updates: `Tell @qa that users.name is now users.display_name`. Claude uses `ListAgents` and `SendMessage`, and sends unprompted when its change affects another session. `/list-agents` lists reachable sessions. Names come from `--name` or `/rename`; duplicates get a variant. The receiver shows a `Message from @<name>` line; `Ctrl+O` expands it. A message never approves a permission prompt or changes configuration.
7. Review before reading: `/code-review` (alias `/review`) runs as a background forked subagent; findings arrive in the conversation. `/code-review high` widens coverage; `--fix` applies findings. For sensitive changes: `/code-review ultra [pr]` (alias `/ultrareview`), a cloud reviewer fleet; every finding is reproduced before reporting. 5–10 minutes. Research preview; claude.ai login; 3 free runs on Pro and Max, none on Team and Enterprise, then $5–25 in usage credits per run; not on Bedrock, Google Cloud's Agent Platform, Foundry or under zero data retention; limit 500 files or 8,000 lines.
8. Read the diff. Stop at the capacity number.
9. Wrong result: `/rewind` to a checkpoint, correct the brief, resend. Do not iterate with the agent. `/rewind` doesn't undo edits by subagents or background reviews (including `/code-review --fix`); revert those with git.
10. Commit under the lead's name.
11. Cleanup: `Ctrl+X` twice in agent view deletes the session and its worktree, uncommitted changes included. Commit first.
12. Remote: `claude remote-control` (server mode) or `/remote-control` in a session. Steer from claude.ai/code or the Claude app's Code tab: diff pane, `/model`, `/effort`, permission prompts. Pro, Max, Team, Enterprise; an Owner must enable it on Team and Enterprise.

## Decision rules

| Question | Rule |
|---|---|
| Fork or new session | Needs current conversation: `/subtask`. Independent: new session, own worktree. |
| Effort | Standard for routine work. `xhigh` for hard bugs. `max` for one-off hard problems (session-only). Never the whole team at max. |
| Permission mode | Auto by default. Manual (`default`) for money, auth, persistence, set on the main session; subagents inherit it. |
| Subagent nesting depth | 1 on production code. |
| Sessions per day | Not more than diffs the lead can read. |
| Commit author | The lead. Say so in `CLAUDE.md`. |

## Command reference

| Command | Function |
|---|---|
| `claude agents` | Agent view: background sessions, state, needs input. Research preview |
| `claude agents --permission-mode plan` | Dispatch defaults for the view (`--model`, `--effort`, `--agent` too) |
| `claude --bg "<prompt>"` | Dispatch a background session from the shell |
| `claude --bg --name <name> --agent <role> "<prompt>"` | Named background session running a role |
| `claude attach <id>` / `logs` / `stop` / `respawn` / `rm` | Manage a background session |
| `claude --worktree <name>` | Interactive session in a worktree under `.claude/worktrees/<name>` |
| `claude --worktree "#<pr>"` | Session branched from a pull or merge request |
| `claude --agent <name>` | Run the session as a named role |
| `claude --name <name>` / `/rename` | Name the session for messaging |
| `/bg` | Move this terminal session to the board |
| `/subtask <task>` | Forked subagent: side task with full conversation, result returns here |
| `/fork [prompt]` | Copy this session into a new background session with its own worktree |
| `/list-agents` (`/peers`) | Reachable sessions and agents |
| `@<session>` in a prompt | Address a session directly |
| `@"<role> (agent)"` | Force a specific subagent for one task |
| `/tasks` | Background work in this session, including finished subagents and reviews |
| `/code-review [effort] [--fix] [pr#]` (`/review`) | Background review of current changes or a PR |
| `/code-review ultra [pr]` (`/ultrareview`) | Cloud review fleet with verified findings |
| `/rewind` | Return the session to a checkpoint |
| `claude remote-control` / `/remote-control` | Expose a session to claude.ai/code and the Claude app |
| `claude plugin validate .claude/agents` | Validate agent files |

## Sources

- Parallel agents overview: https://code.claude.com/docs/en/agents
- Agent view: https://code.claude.com/docs/en/agent-view
- Subagents: https://code.claude.com/docs/en/sub-agents
- Worktrees: https://code.claude.com/docs/en/worktrees
- Cross-session messaging: https://code.claude.com/docs/en/cross-session-messaging
- Code review: https://code.claude.com/docs/en/code-review
- Ultrareview: https://code.claude.com/docs/en/ultrareview
- Remote Control: https://code.claude.com/docs/en/remote-control
- Commands: https://code.claude.com/docs/en/commands
