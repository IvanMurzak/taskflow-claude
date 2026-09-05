---
name: "taskflow-execute"
description: "Execute ready Taskflow tasks through isolated workers, verify evidence, merge per policy, and update ROADMAP as its sole writer."
---

# Taskflow execute

You are the scheduler, never the implementer. Delegate every task. `ROADMAP.md`
is the only live state and only you may edit it.

## Defaults

`--parallel=1`, `--review=off`, `--merge=on-green`, `--engine=auto`,
`--submodules=auto`, `--integration-branch=auto`. Reject unknown or conflicting values. `auto` parallelism is
bounded by ready work, one task per conflict group, and available worker slots.

Accepted options: `--scope=all|wave:N|group:X|<ids>`, `--parallel=N|auto`,
`--review=off|low|medium|high|xhigh`, `--merge=ask|on-green|never`,
`--engine=auto|native|toolkit|pipeline`, `--pipeline=<name>` (pipeline engine
only), `--submodules=auto|off`, `--solo=<ids>`,
`--on-fail=continue|stop`, `--integration-branch=auto|off|<name>`, and
`--dry-run`.

Load references only when needed:

- parallelism, isolation, merge, or cleanup: `references/parallel-execution.md`;
- review enabled: `references/code-review.md`;
- detected submodules and sync enabled: `references/submodules.md`.

## Preflight and scheduling

1. Resolve one `.taskflow/YYYY-MM-DD-<slug>/`; read its README and ROADMAP, not
   task bodies. Reconcile `🔵`/`🟣` rows with branches, worktrees, PRs, and CI.
2. A task is ready when all `needs` are `✅`, its group has no earlier unfinished
   sequence, its owner gates are satisfied, and its repository does not conflict
   with another selected task.
3. Obtain `repo`, `base_branch`, and `id` from ROADMAP. For legacy boards, parse
   only task frontmatter mechanically; never load the task body into scheduler
   context.
4. `--dry-run` prints resolved options, ready/withheld tasks, planned slots,
   and the integration ref per repository, then exits without writes, workers,
   or refs.

## Isolation on Claude Code

With `--engine=native`, dispatch the `taskflow-implementer` agent whose
`isolation: worktree` creates one host worktree per task.
`repo: "."` means the root git repository and is fully supported. Use `toolkit` when a task needs a dedicated
submodule worktree, assigned ports, or a non-default base the host cannot place.
`auto` prefers a compatible Pipeline CLI for those needs and otherwise uses
native isolation. `pipeline` delegates the lifecycle to a named pipeline. Never
run a worker in the shared checkout.

See `references/parallel-execution.md` for lifecycle commands and constraints.

## Dispatch contract

For each selected row:

1. Provision any toolkit slot required by the task.
2. Mark all selected rows `🔵` and commit the ROADMAP once for the batch.
3. Launch all implementers concurrently before waiting for any one result.
4. The worker prompt must contain **exactly one value: the absolute path to the
   immutable task file**. Do not paste, summarize, or pre-read its body; do not
   include the board, sibling tasks, worktree path, or merge permission. The
   worker reads the file and resolves its worktree from the task id/repository.
5. Wait for every worker in the batch. A round does not end until each outcome
   is verified and recorded. If dispatch is refused, leave that task pending and
   continue tracking already-started workers.

Verify each DoD item from repository/PR/CI evidence, never from the worker's
claim. If review is enabled, use a different reviewer and follow
`references/code-review.md`.

## Integration branch

`--integration-branch=auto|off|<name>` (default `auto`). `auto` derives
`taskflow/<slug>` from the Taskflow folder name. Task PRs then target that ref
instead of `base_branch`, and the whole Taskflow lands as ONE reviewed merge —
so concurrent Taskflows never share a branch, and the default branch sees one
coherent change with one combined CI run.

**It is one ref per repository, not one per Taskflow.** A board spanning several
repositories needs the same-named ref in each. For every distinct `repo` on the
board, create the ref in THAT repository, cut from THAT repository's own
`base_branch`. Never assume a repository's default branch from another's.

1. **Preflight, before any dispatch.** For each repository on the board, create
   the ref if absent, or adopt it unchanged if present — a resumed run must
   reuse the ref it already created, never recreate or reset it.
2. **Verify the host honoured the base.** Provisioning may accept a requested
   base and silently place the worker somewhere else — in a submodule, or on a
   resumed worktree that replays an earlier create. After provisioning, confirm
   each worker's worktree is actually positioned on the integration ref, and
   treat a mismatch as a failed dispatch: leave the row pending and report it.
   An unverified base is the failure this option exists to prevent, so never
   infer it from the absence of an error.
3. **Refresh at round boundaries.** Merge `base_branch` into the integration ref
   between rounds so the final merge does not accumulate drift. On conflict,
   stop and report; never force, never rebase a shared ref.
4. **Land once.** When every row for a repository is `✅`, open ONE PR from the
   integration ref to `base_branch` and apply `--merge`. That PR is the
   combined CI run; two independently green task PRs are not evidence that
   their union is green.
5. **Never delete a ref that holds unmerged work**, and keep the board row
   pointing at it so host cleanup spares it.

With `off`, task PRs target `base_branch` directly and none of the above runs.

## Merge and state

- `on-green` (default): merge only after DoD verification, required review, and
  required CI pass; never bypass protection. With an integration branch, this
  governs both the task PRs into that ref and the single PR out of it.
- `ask`: hold verified work at `🟣` for owner approval.
- `never`: leave verified PRs open at `🟣`.

Record `✅` only after the merge/change is verified; record `⛔` with a concise
cause when execution fails. Commit board outcomes once per round, sync touched
submodules once, clean finished worktrees, recompute readiness, and continue.
Never edit task specs, implement inline, run conflicting group tasks together,
or treat an unverified worker report as completion.

When every row is `✅`, move the Taskflow folder to `.taskflow/archive/` only if
that destination is free, then commit that move.
