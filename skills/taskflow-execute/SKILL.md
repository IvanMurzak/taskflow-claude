---
name: "taskflow-execute"
description: "Orchestrate execution of a completed Taskflow from its ROADMAP status board: compute ready tasks, dispatch within dependency and conflict limits, verify repository and CI evidence, and update the board as its sole writer."
argument-hint: "[<taskflow slug>] [scope: wave, group, or task subset — default: whole board]"
---

# taskflow-execute — ROADMAP-driven orchestration

## Select and validate the taskflow

- Default root: `.taskflow/`; operate in one `YYYY-MM-DD-<slug>/` folder. Honor a
  supplied slug; resolve ambiguity with the owner. Never use legacy
  workflow artifacts as input or fallback.
- Stop unless the frame is locked and reviewed, `tasks/` is populated, and
  ROADMAP has a status board. Direct the owner to `/taskflow-tasks` if needed.
- Before dispatching, reconcile every non-pending board row with repository
  evidence. Prefer available forge/CI integrations; otherwise inspect branches,
  commits, merged revisions, test evidence, and executor locks.

## State and ownership contract

- ROADMAP's board is the only mutable task-state record. You are its sole
  writer; implementers never edit ROADMAP or immutable task specs.
- Change a row only after evidence of dispatch, review, merge, and/or passing
  CI supports that state. Make every board update a surgical `ROADMAP.md`
  commit with a progress-log entry where appropriate.
- If a workspace planning store exists, ensure it has just one thin pointer to
  this ROADMAP. Do not duplicate per-task state.

## Dispatch rules

1. Honor an owner-requested scope; otherwise resolve the whole board. Work is
   sequential within a conflict group by `sequence` and parallel only across
   independent groups, subject to owner concurrency limits.
2. The ready set is pending rows whose `needs` are completed and whose approval
   gates are cleared. For production, money, secrets, or irreversible effects,
   request a distinct owner GO through the available input mechanism; present a
   safe option first and record the decision before dispatch.
3. Check for existing branches, changes, reviews, runs, or locks for the same
   task before dispatching. Reconcile rather than collide.
4. Use the consumer project's execution workflow and its approved mapping for
   the board's `top`/`mid`/`fast` tier when available. Otherwise dispatch one
   isolated worktree worker per task. Give the worker only its immutable spec
   (Goal, Scope & seams, DoD, and taskflow references), never permission to
   edit ROADMAP.

## Loop

1. Dispatch a ready task and change `⬜ pending` to `🔵 running`, recording run
   reference and date; commit ROADMAP.
2. Track via the available forge/CI evidence. If tooling is unavailable, use
   local branch, commit, test, and review evidence; do not busy-wait.
3. On an opened review, record `🟣 review`. Merge only when repository policy
   and explicit owner authorization permit it. Verify cleanup only after the
   merge result is known.
4. Verify every DoD item. On success record `✅`, the verified change reference,
   date, and a progress-log line; commit and recompute the ready set. On failure
   record `⛔` with a concise reason, then retry, rescope, or escalate—never
   silently continue.
5. At each wave boundary report landed/running/blocked work, risk, and any
   resource concerns; audit worktrees and branches before the next wave.

## Finish and anti-patterns

When all scoped rows are complete, update the taskflow README status and
ROADMAP counter, remove any thin pointer created for this taskflow, commit, and
report verified results and gates. Do not mark work complete from a worker
report alone, run same-group tasks concurrently, edit task specs, implement
inline, or leave board changes uncommitted.
