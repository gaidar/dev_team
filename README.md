# Claude Code development team

A template for running a software project with Claude Code as a small team: developer, QA and reviewer agents under one human lead. Roles are files, each session works in its own git worktree, sessions message each other, and the lead reads every diff and commits. Two documents carry the whole system:

| File | Purpose |
|---|---|
| `BUILD.md` | What to create in a project repo, with acceptance tests. Claude Code builds it. |
| `PLAYBOOK.md` | How to run the team day to day: commands, settings, decision rules. |

Verified against the Claude Code docs on September 25, 2026. Requires Claude Code v2.1.247 or later.

## How it works

- The lead dispatches sessions from one board, `claude agents`, and reviews at most as many diffs per day as they can read.
- Each role is `.claude/agents/<role>.md` with a tool allowlist. QA and reviewer can't edit; the developer builds from a brief and waits for plan approval.
- Every session works in its own worktree under `.claude/worktrees/`. QA tests in the developer's worktree.
- Sessions message each other when a shared interface changes.
- Agents never push: `settings.json` denies `git push`. The lead commits and opens PRs.

## Set up a new project

Prerequisites: a git repository with an `origin` remote.

1. Copy `BUILD.md` and `PLAYBOOK.md` into the repo root.
2. Run `claude` once in the repo to trust the folder, then `/login` with a claude.ai account.
3. Run `claude --permission-mode plan` and paste the build prompt from the top of `BUILD.md`.
4. Approve the plan. Claude creates `CLAUDE.md`, `.claude/settings.json`, the agents, skills, prompt templates, `README.md` and `scripts/check.sh`, then runs the acceptance tests in `BUILD.md` section H.
5. Fill in what Claude can't infer: do-not-touch directories and sensitive paths in `CLAUDE.md`, your daily review capacity in the project README.
6. Commit.

## Run the team

```
claude agents --permission-mode plan      # open the board
developer <brief>                         # dispatch a build session; approve its plan
qa @<developer-worktree> <brief>          # test the developer's branch
/code-review                              # background review before you read the diff
```

Read the diff, commit, push. Full loop, decision rules and command reference: `PLAYBOOK.md` Part 2.

## Adapt

Edit `BUILD.md` to change roles, guardrails or templates, then rebuild with the same prompt. Keep both files in the project repo so the setup can be reproduced.
