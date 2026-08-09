# Taskflow for Claude Code

Taskflow is a four-stage lifecycle that a user or agent can invoke to turn a fuzzy change
into repository-evidenced architecture, adversarial verification, immutable task
specifications, and ROADMAP-driven execution.

```text
/taskflow-frame → /taskflow-review → /taskflow-tasks → /taskflow-execute
```

## Compatibility with 0.5.1

A bare `/taskflow-execute <slug>` needs no new flags, no migration, and nothing to
configure. Every capability this version adds — parallel dispatch, code review,
execution tiers, submodule sync — is opt-in, and none of it activates unless you
type the flag for it.

One default did change, deliberately. Dispatch is now explicitly serial
(`--parallel=1`): one task at a time. 0.5.1 had no `--parallel` flag at all — its
own prose promised "parallel only across independent groups," and run for real
that meant a round could start more than one ready task at once (two, in a
side-by-side comparison against this version). Pinning the new default to `1` is
strictly more conservative than that — nobody loses isolation or safety by
upgrading — but it is a genuine behavior change, not an identical rerun of
0.5.1's own scheduling: a workflow that relied on 0.5.1 starting several tasks in
round one now sees one, and needs `--parallel=N` (or `--parallel=auto`) to get
that throughput back.

## Package

This repository is the Claude Code package. Its plugin manifest is
`.claude-plugin/plugin.json`, and its four public skills live under `skills/`.
Any user or agent may invoke each stage; no skill pins a model.

## `/taskflow-execute` arguments

Every flag is optional and every default is the one shown below — this table is
kept in lockstep with `skills/taskflow-execute/SKILL.md` §3, which is the
authoritative source if the two ever disagree.

| Flag | Values | Default | Notes |
|---|---|---|---|
| `<slug>` (positional) | a taskflow folder under `.taskflow/` | the only folder there | ambiguity is resolved with you, never guessed |
| `--scope=` | `all` · `wave:N` · `group:B` · comma-separated id list | `all` | free-text scope after the slug is still accepted, unchanged from before |
| `--parallel=` | `1` · `N` · `auto` | `1` | `>1` (or `auto`) enables concurrent dispatch |
| `--engine=` | `auto` · `native` · `toolkit` · `pipeline` | `auto` | picks the execution tier — see below |
| `--pipeline=` | a pipeline name | — | only valid together with `--engine=pipeline` |
| `--review=` | `off` · `low` · `medium` · `high` · `xhigh` | `off` | anything but `off` dispatches a reviewer subagent that is never the implementer |
| `--merge=` | `ask` · `on-green` · `never` | `ask` | `native`/`toolkit` only — see "Why `--merge` defaults to `ask`" below |
| `--submodules=` | `auto` · `off` | `auto` | `auto` means "run the sync only when `git submodule status` is non-empty" |
| `--solo=` | comma-separated id list | empty | forces single-slot dispatch — the authoritative channel for a task that needs an exclusive resource |
| `--on-fail=` | `continue` · `stop` | `continue` | `stop` drains the in-flight slots and halts the run |
| `--dry-run` | flag | off | prints the resolved plan — flags, ready set, slot count, execution tier, withheld tasks — and dispatches nothing |

An unrecognized `--flag`, or a value outside a flag's listed vocabulary, stops
the run rather than silently falling back to a default. Every resolved value —
including defaults nobody typed — is printed before dispatch begins.

Two things are fixed and not configurable by any flag: the review fix-round
budget (`K = 2`) and the concurrency ceiling (`8`).

## Execution tiers — and the Pipeline CLI is optional

`--engine` picks the substrate that provisions a task's working directory.
Default `auto`.

| Tier | Needs | Provides |
|---|---|---|
| `native` | Nothing beyond Claude Code's own worker isolation — no CLI required | One worktree per task, with an enforced main-checkout boundary. Cannot allocate a port, cannot give a task its own submodule worktree, and cannot cut a worktree from any branch but the repository's default. |
| `toolkit` | The `pipeline` CLI, at or above the minimum version that ships `pipeline worktree` — **0.16.0**, published. `--engine=auto` resolves to `toolkit` once the installed CLI is at or above that version (below it, absent, or unparseable ⇒ `native`, and the run states why) | Everything `native` provides, plus a port block, per-submodule worktrees, an arbitrary base branch, `ci-wait`, `submodule bump` and `gc` |
| `pipeline` | Explicit `--engine=pipeline --pipeline=<name>` | Each task becomes one `pipeline drive` run, which owns implement → review → PR → CI → merge → sync itself. `--merge` does not apply here |

Because `native` needs nothing beyond Claude Code itself, **parallel dispatch
works with no CLI installed at all** — worker isolation is a host feature, not a
CLI feature. What the CLI adds, once installed, is the three things native
worker isolation structurally cannot provide. Without it, a run withholds
exactly three classes of task, and only those three:

- a task that needs a bound **port** — the host places a worktree, it does not
  hand out a port block;
- a task whose `repo:` is a **submodule** — host isolation covers the
  superproject checkout, not a worktree per submodule;
- a task that must integrate on a **base branch other than the repository
  default** — the host's own worktree placement accepts only "fresh" or "head,"
  never a named branch. (Both plugin repositories in this workspace are exactly
  this case: they are gated, and every PR targets `next`, not `main`.)

A withheld task's row stays pending, not failed. The reason is reported once per
run — naming the tier, the gap, and the affected task ids — and every other
ready task still dispatches at full concurrency.

Each task runs inside its own git worktree, never the shared checkout; on
Claude Code the host blocks edits and git commands redirected at the main
checkout for that worker and every subagent it spawns. (That boundary is
partial rather than absolute — see `agents/taskflow-implementer.md` for the
measured enforcement matrix before relying on it for anything you haven't
verified yourself.)

## Why `--merge` defaults to `ask`

`--merge=ask` holds each finished task at "verified, merge held" and waits for
you — nothing merges unattended. That is the default because an orchestrator
that merges pull requests by itself, by default, is a large blast radius for a
plugin anyone can install.

`--merge=on-green` (merge once CI is green, no blocking review finding is open,
and the row is behind no approval gate) and `--merge=never` (stop at "verified,
merge held" permanently, and never ask) both exist for when you want something
else — but they're opt-in, and neither bypasses branch protection or elevates
under any circumstance. `--merge` applies to the `native` and `toolkit` tiers
only; the `pipeline` tier merges by its own run definition instead.

## Workflow contract

- New artifacts live only in `.taskflow/YYYY-MM-DD-<slug>/`. Prefix every new
  Taskflow folder with its local creation date; do not rename existing folders.
- `ROADMAP.md` is the sole mutable task-state record. Task specs are immutable
  and never contain `status`.
- Task groups run sequentially by `sequence`; independent groups may run in
  parallel when their `needs` dependencies allow it and `--parallel` is raised
  above its default.
- `security_critical` and `production_touching` raise the model-routing tier.
- Production, money, secrets, irreversible effects, and product decisions
  require an explicit owner gate.

Legacy `.claude/design/` and `.claude/taskflow/` folders are archives and are
not read or migrated.

## License

MIT
