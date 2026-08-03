---
name: design-implement
description: Orchestrate the EXECUTION of a finished design's task list - read the design's ROADMAP status board (the single source of task state), compute the ready set from dependency edges, dispatch each task through the project's pipeline system (model tier per the board), verify ground truth (merged PR, green CI), and flip board rows + progress log as the single writer, wave after wave until the board is resolved. Operates on a per-design sub-folder under .claude/design/ (or a caller-specified path). Use when the user asks to "start the tasks", "implement the design", "drive the roadmap", "запусти выполнение задач", or gives GO on a designed feature.
argument-hint: [<design slug or path>] [scope: wave/group/task subset — default: whole board]
---

# design-implement — orchestrate a design's task-list execution

**Model:** inherits the session model — orchestration is procedural (reconcile, dispatch,
verify, flip); it does not need a top reasoning tier. The IMPLEMENTERS' models come from the
board's `model` column, not from this session.

## Which design folder

- **Default root:** `.claude/design/`. Drive the per-design **sub-folder** `<root>/<slug>/`.
- **Caller override:** honor an explicitly named path/slug; if the root holds one design, use
  it, else ask which. Confirm the resolved path before dispatching anything.

## Preconditions (STOP if unmet)

1. Design status LOCKED + reviewed + **tasks designed** — `tasks/` populated and
   `ROADMAP.md` has a filled status board. If not → point the user at `/design-tasks`.
2. **Reconcile the board with ground truth FIRST.** The board is a cache: check `gh pr list`
   / merged SHAs / CI for every non-⬜ row and fix drift (a prior session may have died
   mid-flight). Never dispatch on top of an unreconciled board.

## The contract (single source of truth — you are the single writer)

- The ROADMAP **status board is the ONLY mutable record of task state**. You — the orchestrator
  — are the only agent that edits it. Implementers report; they never touch ROADMAP or the task
  spec files (immutable).
- Flip a row ONLY after verifying ground truth (run started / PR opened / PR merged + CI green).
- **Every board change = a surgical single-file commit** of ROADMAP.md (plus a progress-log line
  for wave events). Never `git add -A`, never branch-switch a shared checkout. A restarted
  session must be able to trust the committed board.
- If the consumer project keeps a global plan store (e.g. `.claude/plans/tasks/`), ensure it
  holds the design's **one thin pointer task** (linking to this ROADMAP); create it at start if
  missing. Never mirror per-task state there.

## Scope & concurrency directives

- Honor caller scope ("only wave 1", "only group A", a named task subset). Default: resolve the
  whole board.
- **Sequential within a group, parallel across groups** (groups = merge-conflict domains) —
  never two tasks of one group in flight. Honor standing owner concurrency caps (e.g.
  single-lane directives) over theoretical parallelism.
- **Model routing:** dispatch each task on the tier in its board `model` column
  (top=Fable-class, mid=Sonnet-class, fast=Haiku-class), unless the owner has ruled out
  per-task overrides for this project — then note the deviation in the progress log.

## The loop

1. **Ready set** = rows ⬜ pending whose `needs` are all ✅ AND whose human-approval gate (if
   listed in ROADMAP) is cleared. Gates (production, money, secrets, irreversible) →
   AskUserQuestion BEFORE dispatch, safe option first; record the GO in the progress log.
2. **Pre-dispatch duplicate check** (per task): any in-flight issue/PR/branch or executor lock
   for the same work → reconcile before dispatching, don't collide.
3. **Dispatch through the project's pipeline system** where one exists (e.g. an
   `implement-task` workflow via `/pipeline:dispatch` or `/pipeline:find` + `/pipeline:run`,
   following the project's routing and parallel-wave rules). If the project has NO pipeline
   system, dispatch each task to an isolated worker agent in its own git worktree — one worker
   per task, never implementation inline in this session, never two workers of one group. The
   dispatch brief = the task file's Goal + Scope & seams + DoD, plus its `design_refs` docs —
   the implementer gets the spec, never this conversation.
4. **Flip on dispatch:** ⬜ → 🔵 + run id + date → commit.
5. **Track:** pipeline runs may park at CI-wait and NOT auto-resume — poll ground truth
   (CI-wait tooling, `gh pr checks`; never sleep-poll loops) and drive the
   merge / pointer-bump / worktree-teardown yourself. PR opened → 🟣 review.
6. **Verify DoD, then close:** every DoD box checked against ground truth → ✅ + PR link +
   date + progress-log line → commit → recompute the ready set (downstream tasks may have just
   unblocked) → next dispatch. Failure → ⛔ + one-line reason in Run/PR → decide retry /
   re-scope / escalate to the owner — never silently.
7. **Wave boundary:** tight owner report (landed / running / blocked / spend / risks) + leak
   hygiene (worktree/branch GC audit) before opening the next wave.

## Finish

When every scoped row is ✅: update the design README status line + ROADMAP implementation
counter, append the closing progress-log entry, delete the plan-store pointer task (if one was
created), surgical commit, and report: tasks landed (with PRs), gates cleared, coefficients vs
actual difficulty (feed surprises back via `/design-review` if the design drifted from reality).

## Anti-patterns

- Marking ✅ from a worker's report without `gh`/`git` verification ("status=completed ≠ done").
- Two tasks of the same group in flight; batching dependent dispatches in one turn.
- Editing task spec files, or letting status exist anywhere outside ROADMAP.
- Implementing inline or via unscoped general-purpose agents when a pipeline system exists.
- Skipping an approval gate because the wave "was already approved" — each gated task gets its
  own GO.
- Leaving the board uncommitted between flips — a crashed session then re-dispatches live work.
