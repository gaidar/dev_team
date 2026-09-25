# RUNBOOK.md – how the dev team works

This is the single rulebook for the AI dev team in this repository. Humans read it. The lead and every teammate load it through CLAUDE.md. To change how the team behaves, edit this file. Re-run BUILD.md only when roles, skills, models or permission settings change (see section 14).

Vocabulary: "you" is the human team lead. "The lead" is the main Claude Code session you talk to. "Teammates" are the Claude Code sessions the lead spawns.

## 1. Principles

- You talk to the lead only. The lead talks to everyone else.
- Nothing starts without an approved plan. Nothing merges without passing the merge gate.
- Every task leaves a trace: a plan, decisions, status, and a branch or PR.
- One owner per path. A teammate never edits outside the paths the plan assigns to it.
- The lead never writes application code or tests. It plans, delegates, checks, reports.
- Cost is a design input. Every plan names a model per role and says why.

## 2. Roles

| Role | Runs as | Owns | Delivers | Model |
|---|---|---|---|---|
| lead | main session, started by `./team` | `docs/team/STATUS.md`, the task list, branches, merges (lead mode) | plan summaries at gates, spawn and shutdown of teammates, final report | Fable |
| planner | teammate | `docs/team/REQUIREMENTS.md`, `docs/team/DECISIONS.md`, `docs/team/plans/` | the plan file; keeps project memory current through the task | section 13 |
| pm | teammate | the acceptance criteria section of the plan | acceptance criteria, user-facing behavior, edge cases, out of scope | section 13 |
| architect | teammate | `docs/ARCHITECTURE.md`, `docs/design/<id>.md` | design: modules, interfaces, data model, file ownership map, ordered implementation steps | section 13 |
| developer (1 to n) | teammates named `developer-1`, `developer-2`, ... | the paths the plan assigns | code on the task branch, building and self-tested | section 13 |
| reviewer | teammate | `docs/team/reviews/<id>.md` | findings with severity (blocker, major, minor), verdict approve or block | section 13 |
| qa | teammate | the test directory, `docs/team/qa/<id>.md` | test plan, tests, run results, defects with reproduction steps | section 13 |

Models per role and tier are in section 13; the build interview can change the defaults, and every plan restates them per task. Only the lead spawns teammates; teammates cannot spawn.

## 3. Tiers

| Tier | When | Roster |
|---|---|---|
| fix | behavior unchanged or trivial bug, a few files, no design impact | planner, developer, reviewer |
| change | behavior changes inside the existing design | planner, developer, reviewer, qa |
| feature | new capability that needs design | planner, pm, architect, developer(s), reviewer, qa |
| build | a whole application or major subsystem from scratch | all roles; planner splits the work into milestones, each run as a feature |

Decision rule: the lead assigns the tier by scope and risk, not by file count. When in doubt, the higher tier. You can override at any time with `tier: <fix|change|feature|build>`.

## 4. Task lifecycle

1. Intake. You run `/team <text | #issue | path/to/spec.md>`. The lead assigns a task id (the issue number when there is one, otherwise `T-YYYYMMDD-NN`), picks the branch name `team/<id>-<slug>`, proposes a tier, and writes the intake line to `STATUS.md`.
2. Planning. The lead spawns the planner (and the pm for feature and build) with the request, the proposed tier, and the repo context. The planner reads `REQUIREMENTS.md` and `DECISIONS.md` first, then writes `plans/<id>.md` using the template in section 5. For feature and build the pm writes the acceptance criteria into the plan before it is presented. Fix plans are ten lines or fewer.
3. Plan gate. The lead posts a plan summary of twelve lines or fewer (tier, roster with model and cost note, sequence, merge mode, risks) and stops. This gate is never skipped.
4. Recruitment. On approval the lead spawns exactly the roster in the plan. Each spawn prompt contains: role, task id, plan path, owned paths, deliverable, done criteria, and the instruction to read CLAUDE.md, RUNBOOK.md and the plan before working.
5. Execution, by tier:
   - fix: developer → reviewer → merge gate.
   - change: developer → reviewer and qa in parallel → merge gate.
   - feature: architect → design gate → developers in parallel by file ownership, qa writes tests from the design meanwhile → reviewer and qa → merge gate.
   - build: the planner writes a milestone map in `plans/<id>.md` and one plan per milestone in `plans/<id>-m<n>.md`; milestones run in order, each with its own design and merge gates; `STATUS.md` carries milestone progress across sessions.
6. Design gate (feature and build). The lead posts the architect's design summary and stops.
7. Review and QA. The reviewer approves or blocks with findings. QA reports pass or fail with defects. Defects go to the owning developer through the lead. The loop repeats until the reviewer approves and QA passes, at most three rounds; after that the lead escalates to you with options.
8. Merge gate. The lead posts: what changed, tests run and results, review verdict, PR link or branch. In pr mode you merge. In lead mode the lead merges on `approved`.
9. Close. The planner fills the outcome section of the plan, updates `REQUIREMENTS.md`, and appends decisions. The lead updates `STATUS.md`, shuts down every teammate by name, and posts a final summary of eight lines or fewer.

## 5. Plan file template

```markdown
# Plan <id> – <title>
Status: draft | approved | in progress | done | abandoned
Task: <request verbatim, or issue link, or spec path>
Tier: fix | change | feature | build
Merge mode: pr | lead
Permissions: prompts | auto | unattended

## Roster and models
| role | instances | model | why this model |

## Acceptance criteria (pm; feature and build only)
1. <verifiable statement>

## Sequence
1. <step> – owner – depends on: <step numbers> – deliverable: <path or artifact>

## File ownership
| path or glob | owner |

## Risks and open questions

## Outcome (filled at close)
Branch or PR:
Tests run and results:
Review verdict:
Changed versus plan, and why:
Follow-ups:
```

## 6. Gates and your commands

At any gate you reply with one of these. The lead accepts nothing else as approval.

| You say | Effect |
|---|---|
| `approved` | proceed to the next phase |
| `changes: <text>` | the owning role revises, the lead re-presents |
| `tier: <x>` | re-plan at that tier |
| `merge: pr` or `merge: lead` | change the merge mode for this task |
| `unattended` | the lead approves the design gate itself; in lead mode it merges when the reviewer approves and QA passes; in pr mode you still merge |
| `auto` | run in auto mode: a classifier reviews each tool call and blocks destructive ones instead of asking you, see below |
| `stop` | graceful shutdown, status recorded |

The plan gate is never skipped, even when unattended.

Auto mode, mechanics. Teammates inherit the permission mode of the lead session at the moment they are spawned, and the lead cannot change that mode itself. So either start the session with `./team --auto`, or switch the mode in the terminal (Shift+Tab to auto) before you say `approved` at the plan gate. If you say `auto` after the roster is running, the lead tells you which of the two to do. The deny list in `.claude/settings.json` (destructive git and shell commands, secrets) applies in every mode. The team never runs with permission checks switched off: the launcher refuses `--dangerously-skip-permissions`, and auto mode still prompts you when the classifier keeps blocking an action.

## 7. Merge modes

- pr: the lead pushes the task branch and opens a pull request with the host CLI (`gh` for GitHub, `glab` for GitLab). On Bitbucket or any host without a CLI it pushes the branch and posts the URL that opens the pull request form. You merge.
- lead: after the merge gate the lead merges into the default branch with the strategy chosen at build (squash by default).
- Always: no force pushes, no direct commits to the default branch, no branch deletion without your approval, no rewriting of history that has been pushed.

## 8. Project memory (`docs/team/`)

| File | Owner | Format |
|---|---|---|
| `REQUIREMENTS.md` | planner | living document; each requirement has an id `R-n`, the task that introduced it, and a status (proposed, accepted, implemented, dropped) |
| `DECISIONS.md` | planner | append-only log; one line per decision: `YYYY-MM-DD | <task id> | decision | why | role` |
| `plans/<id>.md` | planner | one plan per task, template in section 5 |
| `STATUS.md` | lead | current task, phase, waiting on, blockers, last updated; a milestone table for build tasks |
| `reviews/<id>.md` | reviewer | findings and verdict |
| `qa/<id>.md` | qa | test plan, results, defects |

Traceability rule: a decision made by any role (architect, developer, reviewer, qa) is sent to the planner as a message `decision: <what> because <why>`; the planner records it the same day. Anyone can read these files; only the owner writes them.

## 9. Definition of done

Fix: developer's change on the task branch, build and existing tests pass, reviewer approved, merge gate passed, plan outcome filled.

Change: fix plus qa test plan executed with results in `qa/<id>.md`, new tests for the changed behavior.

Feature: change plus acceptance criteria in the plan, design file approved, every acceptance criterion mapped to a passing test or an explicit waiver from you, `ARCHITECTURE.md` updated, `REQUIREMENTS.md` updated.

Build: every milestone meets the feature definition; the application runs from a clean checkout following the README; `STATUS.md` shows all milestones done.

## 10. Commands and monitoring

- `./team` (Windows: `team.cmd`) starts the lead session. Flags: `--auto` (auto mode, classifier-reviewed tool calls instead of prompts), `--resume` (pass through to Claude Code). The launcher never starts Claude Code with permission checks off.
- `/team <task>` runs a task through the lifecycle in section 4.
- `/check-status` makes the lead read `STATUS.md`, the task list and teammate states and report in twelve lines or fewer.
- `/stop-team` shuts every teammate down gracefully and records what was not finished.

In the terminal, teammates appear in the agent panel below the prompt: up and down arrows select one, Enter opens its transcript and lets you message it directly, Escape returns to the lead (and interrupts the teammate's current turn while its transcript is open), `x` stops the selected teammate, Ctrl+T toggles the shared task list. You may talk to a teammate directly; tell the lead afterwards if you changed its direction.

## 11. Rules for every teammate

- Read CLAUDE.md, RUNBOOK.md and your plan before doing anything.
- Edit only the paths assigned to you. If you need a change elsewhere, message the owner or the lead; do not make it yourself.
- Work on the task branch. Commit messages: `<id>: <what changed>`.
- A task is complete only when its deliverable exists in the repository and the commands you ran are listed in your report.
- Finish with a report of eight lines or fewer: Done, Not done, Changed versus plan, Needs decision.
- Send decisions to the planner (section 8). Send blockers to the lead immediately, not at the end.
- Never approve anything on behalf of the human. Never mark someone else's task complete.
- Never hardcode secrets; read them from the environment. Never touch CI, deployment or release configuration unless the plan assigns it to you.

## 12. When things go wrong

| Symptom | What the lead does |
|---|---|
| A teammate stops on an error | opens its transcript, redirects it, or spawns a replacement with the same role and the suffix `-b` |
| A task shows in progress but the work is done | updates the task status itself and notes it in `STATUS.md` |
| The session was resumed and teammates are gone | reads `STATUS.md` and the plan, then respawns only the roles needed for the remaining steps |
| The lead starts implementing itself | you say `wait for your teammates`; the lead stops and delegates |
| Review or QA loop passes three rounds | escalates to you with the options: accept with known issues, re-plan, abandon |
| Permission prompts pile up | you add narrow rules to the allow list in `.claude/settings.json`, or restart with `./team --auto` |
| Two teammates need the same file | the lead reassigns ownership in the plan and records the decision |

## 13. Models and cost rules

### 13.1 Model per role and tier

Set at build time as the agent file default (the feature and build column), restated in every plan, and named by the lead in each spawn prompt. A model named in the spawn prompt overrides the agent file.

| Role | fix, change | feature, build | Why |
|---|---|---|---|
| lead | Fable 5.1 | Fable 5.1 | long-lived session whose cost is mostly cache reads, where Fable's premium is small; its triage, gate summaries and recovery decisions set the quality of everything else |
| planner | Sonnet 5 | Fable 5.1 | a fix plan is ten lines; a feature or build plan decides the roster, sequence and ownership for every other role |
| pm | not in roster | Fable 5.1 | acceptance criteria are cheap in tokens and expensive to get wrong; every test and review traces back to them |
| architect | not in roster | Fable 5.1 | design errors cascade to every developer; reads a lot, writes little, so the premium per task is a few dollars |
| developer | Opus 5.5 (Sonnet 5 when the plan marks the work mechanical) | Opus 5.5 | highest token volume of any role; Anthropic positions Opus 5.5 for long-running agentic coding at 2.5 times less than Fable per token |
| reviewer | Opus 5.5 | Fable 5.1 | input-heavy, output-light, so Fable costs little here and gives a stronger second opinion than the model that wrote the code |
| qa | Sonnet 5 | Opus 5.5 | test writing is agentic coding; straightforward changes do not need Opus, features do |

Mechanical work, for the developer rule: renames, boilerplate, config changes, dependency bumps, copy edits, generated code from a template. Anything touching data models, concurrency, security, public interfaces or error handling is not mechanical.

### 13.2 Escalation and downgrade rules

- A developer that fails twice on the same defect is respawned for that step on Fable 5.1; the plan records why.
- A build milestone that is mostly mechanical (scaffolding, CRUD, wiring) may run its developers on Sonnet 5 if the architect says so in the design.
- Haiku 4.5 is not used for any role: teammates load full project context, and the saving over Sonnet 5 is small next to the cost of a wrong edit.
- The lead questions the defaults in every plan and states the reason for any change in the "why this model" column.
- Fable 5.1 ships with safeguards for dual-use content. For work that touches offensive security, malware handling or similar, expect refusals or limits and name a fallback model in the plan.

### 13.3 Team size and behavior

- At most six teammates per task unless the plan justifies more. Spawn only what the current phase needs; shut down roles whose work is done before the next phase starts.
- The lead delegates reading and searching to teammates instead of loading large files into its own context.
- Human gates create idle time. Teammate prompt caches expire after five minutes by default, so a slow approval means the next turn rewrites the cache. Set `subagentPromptCacheTtl` to `1h` in settings when gate waits routinely exceed five minutes; writes cost more, re-reads cost far less.

### 13.4 Price reference

Claude API list prices, USD per million tokens, checked 2026-09-25 at platform.claude.com. On a Claude subscription the same choices consume rate limits instead of dollars, in the same proportions.

| Model | Input | Output | Cache read | Positioning per Anthropic |
|---|---|---|---|---|
| Fable 5.1 | 10 | 50 | 0.25 | demanding reasoning, long-horizon agentic work |
| Opus 5.5 | 4 | 20 | 0.20 | long-running agentic coding and knowledge work |
| Sonnet 5 | 2 | 10 | 0.20 | speed and intelligence balance |
| Haiku 4.5 | 1 | 5 | 0.10 | fastest, high volume |

Use Opus 5.5, not Opus 5 (5 / 25) and not Fable 5 (cache read 1.00): same tier, worse price. Rough cost per task with the defaults above, before caching discounts (estimate, not measured): fix about 3 to 5 USD, feature with one developer about 15 to 25 USD, dominated by the developer and qa token volume; the judgment roles on Fable add a few dollars in total.

## 14. Changing this runbook

Edit this file and commit. New sessions pick it up immediately because CLAUDE.md points here. Re-run BUILD.md when you add or remove a role, rename a skill, or change models, permissions or hooks, because those live in generated files under `.claude/`. On re-run BUILD.md reuses your previous answers from `docs/team/BUILD_CONFIG.md` and reports what changed.
