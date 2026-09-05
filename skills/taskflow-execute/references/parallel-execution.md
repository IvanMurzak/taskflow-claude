# Parallel execution on Claude Code

Use a separate `taskflow-implementer` agent for every task. Its
`isolation: worktree` provides native root-repository isolation. `repo: "."` is
valid and means the root repository; never run that worker in the shared
checkout.

## Capacity and dispatch

Select at most `min(--parallel, available worker slots, ready conflict groups)`.
Launch all agents concurrently before waiting for any one result. Each worker
prompt is only the absolute task-file path. Native workers start in their host
worktree; toolkit workers locate `worktree-<task-id>` in the repository named by
the spec.

Use toolkit for assigned ports, a dedicated submodule worktree, or a base the
host cannot place. Require `status: created`; reconcile a reused slot before
dispatch. If toolkit is absent, keep unsupported tasks pending and continue
native-safe work.

```text
pipeline worktree create --name <task-id> --base <base> --submodules <paths> --ports <n> --json
pipeline worktree finalize --name <task-id> --base <base> --submodules <paths> --json
pipeline worktree destroy --name <task-id> --outcome completed --json
pipeline worktree list --json
pipeline ci-wait --pr <n> --repo <owner/repo> --timeout <seconds> --json
pipeline submodule bump --no-admin
pipeline gc --project <root> --clean --json
```

Track every launched worker before ending the round. On resume, reconcile the
board against worktrees, branches, PRs, and CI. Remove a slot only after its
worker ended and its work is committed or merged; never delete unknown, dirty,
locked, or unmerged work.

`on-green` merges only when DoD, required review, and required CI are green.
`ask` and `never` leave the row `🟣`. Workers and reviewers never merge. Never
bypass protection.

## Integration branch

With `--integration-branch`, create or adopt the ref in each repository on the
board before dispatch, cut each task worktree from it, and verify the worktree
is actually on it before the worker starts. `isolation: worktree` places the
worker itself; it does not guarantee the base you asked for reached a submodule
or a reused worktree, so check rather than assume. Task PRs target the
integration ref; one final PR merges it into `base_branch`.
