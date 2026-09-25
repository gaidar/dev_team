# Claude Code development team

A template for running a software project with Claude Code as a team: a lead you talk to, and planner, pm, architect, developer, reviewer and qa agents the lead recruits per task. You give the lead a task, approve its plan, approve at the gates, and merge. Two documents carry the whole system:

| File | Purpose |
|---|---|
| `BUILD.md` | Installer. Claude Code reads it, interviews you about the repo once, and generates the team assets. Re-runnable. |
| `RUNBOOK.md` | Rulebook. Roles, tiers, lifecycle, gates, project memory, models and cost rules. The lead and every teammate load it through `CLAUDE.md`. |

Built on Claude Code agent teams, which are experimental and off by default; `BUILD.md` turns them on for the repo. Verified against the Claude Code docs on September 25, 2026.

## How it works

- You work with the lead only. The lead is the main Claude Code session, started by `./team`.
- Every task starts with a plan. The lead spawns a planner, which writes `docs/team/plans/<id>.md`; you approve it, then the lead spawns exactly the roles the plan names.
- Tiers decide the roster: fix (developer, reviewer), change (adds qa), feature (pm, architect, developers, reviewer, qa), build (a whole application, all roles, split into milestones).
- Gates: plan approval on every task; feature and build add a design gate; every task ends at a merge gate. You merge via PR, or the lead merges, chosen per task.
- Each role is `.claude/agents/<role>.md` with a tool allowlist and a path ownership rule. No two roles edit the same paths.
- The planner keeps project memory in `docs/team/`: requirements, an append-only decision log, one plan per task. Every task is traceable afterwards.
- Models are chosen per role and tier for cost: Fable 5.1 for the lead and for planning, product, design and review on features; Opus 5.5 for developers; Sonnet 5 where the work is small. The lead states the model per role in every plan. Details in `RUNBOOK.md` section 13.

## Set up a project

Prerequisites: a git repository, Claude Code installed, `gh` or `glab` if you want the lead to open pull requests.

1. Copy `BUILD.md` and `RUNBOOK.md` into the repo root. Works on a new or an existing repo; nothing is overwritten.
2. Run `claude` once in the repo to trust the folder and `/login`.
3. Run `claude "Read BUILD.md and follow it exactly."` and answer the interview (stack, commands, merge mode, models, permissions). Type `defaults` to accept every detected value.
4. Claude generates `.claude/agents/`, `.claude/skills/`, `.claude/settings.json`, a `Dev team` section in `CLAUDE.md`, `docs/team/`, and the `team` launcher, runs its verification checks, and prints a report with anything it could not verify.
5. Commit (the interview offers to do this on a branch `team/setup`).

## Run the team

```
./team                          # start the lead (add --auto for classifier-reviewed permissions instead of prompts)
/team <task | #issue | spec.md> # run a task; the lead returns with a plan
approved                        # or: changes: <text>, tier: <x>, merge: pr|lead, unattended
/check-status                   # where every teammate stands
/stop-team                      # graceful shutdown, status recorded
```

Teammates appear in the panel below the prompt; arrow keys select, Enter opens a transcript, Ctrl+T shows the task list. Full lifecycle, gate words and recovery table: `RUNBOOK.md` sections 4, 6 and 12.

Start with a fix-tier task, for example a typo or a `--version` flag, to see the whole loop in ten minutes before giving the team a feature.

## Adapt

- To change how the team behaves, edit `RUNBOOK.md` and commit. New sessions pick it up immediately.
- To add or remove roles, rename skills, or change models, permissions or hooks, edit `RUNBOOK.md` and re-run `BUILD.md`; it reuses your previous answers from `docs/team/BUILD_CONFIG.md` and reports what changed.
- Keep both files in the project repo so the setup can be reproduced.

## Limits to know

- One team per session; teammates cannot recruit their own teammates.
- Teammates do not survive `/resume`; the lead re-reads `docs/team/STATUS.md` and respawns what a task still needs.
- Agent teams cost more tokens than a single session. Fix-tier tasks are cheap; features are dominated by developer and qa volume. Cost rules and a price reference are in `RUNBOOK.md` section 13.
