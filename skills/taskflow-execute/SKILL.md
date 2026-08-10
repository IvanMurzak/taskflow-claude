---
name: "taskflow-execute"
description: "Orchestrate execution of a completed Taskflow from its ROADMAP status board: compute ready tasks, dispatch within dependency and conflict limits, verify repository and CI evidence, and update the board as its sole writer."
argument-hint: "[<slug>] [scope] [--parallel=N] [--review=<depth>] [--merge=ask|on-green|never] [--dry-run]"
allowed-tools:
  - Bash(git --version)
  - Bash(git status:*)
  - Bash(git rev-parse:*)
  - Bash(git submodule status)
  - Bash(git worktree list:*)
  - Bash(git branch --list:*)
  - Bash(git ls-tree:*)
  - Bash(gh auth status)
  - Bash(pipeline --version)
---

# taskflow-execute — the scheduler contract

This skill **schedules; it does not execute**. It computes what is ready,
dispatches inside dependency and conflict limits, verifies against the
repository rather than against a worker's report, and writes the board as its
sole writer.

Everything below the dispatch line — how a worker gets a working directory, how
a diff is reviewed, how submodule pointers move — lives in the reference modules
named in §1 and is read **only** when a flag asks for it.

---

## 1. Load first — the conditional loading table

**This is the file's first instruction. Resolve it before anything else.**
Parse the arguments (§3), take the preflight snapshot (§2), then load exactly
the modules whose condition holds — no more, no fewer:

| Condition | Load |
|---|---|
| `--parallel > 1` | `references/parallel-execution.md` |
| `--review != off` | `references/code-review.md` |
| `git submodule status` non-empty **and** `--submodules != off` | `references/submodules.md` |

- A condition that does not hold means the module is **not read**. Do not read
  one "for context", and do not summarise one you did not read.
- **The default invocation loads none of them.** `--parallel=1 --review=off` in
  a repository with no submodules leaves this file as the whole skill, and it
  behaves exactly as it did before the modules existed. Parallelism, review,
  execution tiers and submodule sync are opt-in without exception.
- Composition is deterministic on purpose: the table is decided by flag values
  and one command's output, never by judgment. The same invocation therefore
  loads the same files on every host — including hosts where §2's injection
  never runs, and including hosts that are not Claude Code — and the plugin can
  be audited by reading this one file.
- All three files ship in this plugin. If one a condition selects is missing,
  say so and stop; do not proceed on a partial contract.

---

## 2. Preflight snapshot

Claude Code only, and a fast path rather than a dependency. It arrives
pre-rendered:

```!
git --version
git status --porcelain
git rev-parse --abbrev-ref HEAD
git submodule status
git worktree list
git branch --list worktree-*
git ls-tree -d --name-only HEAD .taskflow
pipeline --version
gh auth status
git --version
```

`git --version` appears twice, deliberately, and is the only command here whose
output is neither a decision input nor ever empty. It is the **sentinel frame**
(§2.2): the first occurrence proves the block fired, the last proves it ran to
the end instead of dying partway.

### 2.1 What each command decides

| Command | Decision it closes |
|---|---|
| `git --version` (first **and** last line) | whether the block fired at all (§2.2) — this and nothing else |
| `git status --porcelain` | the D6 preflight gate (§7) |
| `git rev-parse --abbrev-ref HEAD` | the base branch |
| `git submodule status` | empty ⇒ `references/submodules.md` is never loaded |
| `git worktree list` | orphaned slots left by a previous run |
| `git branch --list worktree-*` | candidate already-dispatched work to reconcile (§12) |
| `git ls-tree -d --name-only HEAD .taskflow` | which taskflows exist |
| `pipeline --version` | execution-tier resolution, against §8.1's constant |
| `gh auth status` | whether a PR path exists at all |

The branch glob is `worktree-*` — the namespace the substrate actually produces
and the one `pipeline gc` reaps. This system has **no other branch namespace
that this skill creates**; do not create branches under a different prefix and
do not look for them under one. The raw glob is broader than that, though: it
also catches the host's own `worktree-agent-*` isolation branches, which this
skill neither creates nor is entitled to reap — see §12 for how the two are
told apart.

### 2.2 Reading the block — first, did it fire?

The outputs arrive **concatenated in the order above, with no separators and no
command echoed**, and several are routinely empty — a clean tree, no submodules,
no dispatched branches. A block that fired into a quiet repository therefore
looks very much like a block that never ran.

**That question is settled by the sentinel frame and by nothing else.**

| What the block shows | What it means | What to do |
|---|---|---|
| A `git version …` line **first and last** | It **fired and ran to completion** | Read it. Everything between the two sentinels is real output |
| Neither sentinel | It did not fire — no shell, or suppressed (§2.3) | §2.4: obtain the facts directly |
| The policy marker where output belongs | Suppressed by policy (§2.3) | §2.4 |
| A first sentinel but no last one | It started and did not finish | Treat every fact in it as unattributable; §2.4 |

**Make that determination by this check alone.** Do **not** conclude the block
was degraded, absent or unavailable because much of it is empty, and do not
re-run all eight commands "to be safe" on a block that carries its frame. That
inference is precisely what failed — twice, in the same run, against a block that
had in fact fired with real output — and it is why the frame exists rather than
another paragraph asking for care.

**Inside a complete frame, empty means empty.** A clean tree, an absence of
submodules and no dispatched branches all render as nothing at all; between two
sentinels that nothing is a *result*, not a missing one.

Attribution of an individual line is a separate question, and there the block is
still merged and unlabelled. Attribute conservatively:

- one bare branch name on its own line is `rev-parse`;
- `<path> <sha> [<branch>]` lines are `worktree list`;
- a bare `.taskflow` is `ls-tree`;
- a bare semver is `pipeline --version`;
- a multi-line block naming a host is `gh auth status`.

**Whenever a decision depends on a fact you cannot attribute to its command with
certainty, run that one command yourself and use its answer** — that one command,
not the block again.

Two decisions re-run their own command directly **whatever the frame says**,
because each is cheap and each is load-bearing: the D6 gate (§7), which stops the
run, and the third loading condition (§1), which decides whether a reference
module is read at all. Those two are belt-and-braces on purpose, and they keep
§7 and §1 correct on a host where the block never fires.

### 2.3 Degradation is a normal path, not an exceptional one

Three ways the block yields nothing, all expected. **In all three the sentinel
frame is missing, and that — not the emptiness of any particular output — is how
each is recognised (§2.2):**

1. **No shell.** The block runs under bash and `shell:` is deliberately left
   unset, so a Windows host without Git Bash — or any non-Claude host — produces
   no injection at all. The block is simply absent or left unexpanded.
2. **Disabled by policy.** `disableSkillShellExecution` replaces every injection
   with a marker string rather than output. On Claude Code the marker reads:

   ```
   [shell command execution disabled by policy]
   ```

   Treat any other text standing where output was expected — a permission error,
   or the command lines echoed back verbatim — the same way.
3. **Permission.** The block is permission-checked as a **single multi-part
   command**: one part the grant does not cover fails the whole block, not just
   that line. The frontmatter grant above exists for exactly this, and it is
   read-only in every entry (§14). This is also why the sentinel carries its own
   grant entry, `Bash(git --version)`: an ungranted sentinel would fail the very
   block it exists to vouch for.

In all three cases the run continues, because nothing here depends on the
injection having happened.

### 2.4 The same eight facts, as instructions

| Fact | Obtain it directly |
|---|---|
| Main checkout clean | `git status --porcelain` |
| Base branch | `git rev-parse --abbrev-ref HEAD` |
| Submodules present | `git submodule status` |
| Orphaned slots | `git worktree list` |
| Candidate dispatched branches (§12) | `git branch --list worktree-*` |
| Taskflows available | `git ls-tree -d --name-only HEAD .taskflow`, or list `.taskflow/` |
| CLI version | `pipeline --version` |
| PR path available | `gh auth status` |

### 2.5 What may never enter the block

Every injected command is a **fixed literal**. None is composed from runtime
data — not the invocation's arguments, not board text, not task text, not a
slug, not a path read from a file. Argument substitution happens *before* an
injected command runs, so argument text placed inside the block would be handed
to a shell; the block therefore takes no input at all, and the argument
placeholder token is deliberately used nowhere in this file. Commands are bare
invocations with no pipes, redirects or subshells, and the branch glob is
written unquoted so there is no quoting surface either.

**The sentinel is bound by every rule in this section**, which is why it is a
second copy of an existing read-only command rather than an echoed marker string:
a marker would either need a new grant broad enough to cover arbitrary text, or a
literal chosen for this file and matched nowhere else. `git --version` takes no
input, writes nothing, cannot fail where the other git commands succeed, and its
output collides with no other line in the block.

---

## 3. Arguments

| Argument | Values | Default | Notes |
|---|---|---|---|
| `<slug>` (positional) | a taskflow folder under `.taskflow/` | the only folder there | ambiguity is resolved with the owner, never guessed |
| `--scope=` | `all` · `wave:N` · `group:B` · comma-separated id list | `all` | free-text scope is still accepted (§3.1) |
| `--parallel=` | `1` · `N` · `auto` | `1` | `>1` loads `references/parallel-execution.md` |
| `--engine=` | `auto` · `native` · `toolkit` · `pipeline` | `auto` | §8 |
| `--pipeline=` | a pipeline name | — | only with `--engine=pipeline` |
| `--review=` | `off` · `low` · `medium` · `high` · `xhigh` | `off` | anything but `off` loads `references/code-review.md` |
| `--merge=` | `ask` · `on-green` · `never` | `ask` | `native`/`toolkit` only; the `pipeline` tier merges by its own definition |
| `--submodules=` | `auto` · `off` | `auto` | `auto` means "only when `git submodule status` is non-empty" |
| `--solo=` | comma-separated id list | empty | forces single-slot dispatch; the authoritative channel for exclusive resources (§6.2) |
| `--on-fail=` | `continue` · `stop` | `continue` | `stop` drains the in-flight slots and halts |
| `--dry-run` | flag | off | print the resolved plan, dispatch nothing (§6.3) |

### 3.1 Parsing

- **Print every flag's resolved value** — including the defaults nobody typed —
  before anything else happens.
- **An unrecognized `--flag` stops the run.** Name the offending token and print
  the accepted list. A mistyped `--paralel=4` must never silently mean `1`.
- **An out-of-vocabulary value stops the run too**, by the same rule:
  `--review=max` and `--review=inherit` do not exist here.
- **A token that does not begin with `--` is not a flag.** The first such token
  is the slug; further free text is read as scope, exactly as it always has
  been. This is what keeps a bare `/taskflow-execute <slug> <some scope>`
  working unchanged.
- `--pipeline` without `--engine=pipeline`, or `--engine=pipeline` without
  `--pipeline`, stops the run.

### 3.2 Deliberately absent

`--worktree-root`, `--seed`, `--base` are **not** part of this surface and,
being unrecognized, stop the run under §3.1. Each is owned elsewhere:
`.worktreeinclude` and the create hook's own configuration for the first two;
`base_branch` frontmatter and the CLI's own `--base` for the third.
(`pipeline worktree create --base` is a *CLI* argument — the collision of names
is intentional and the two surfaces are separate.)

### 3.3 Fixed, and taking no flag

The review fix-round budget **K = 2**; gating determined by review depth; and
the concurrency ceiling of **8**. None of the three is configurable, here or
anywhere else.

---

## 4. Select and validate the taskflow

- Default root `.taskflow/`; operate inside one `YYYY-MM-DD-<slug>/` folder.
  Honour a supplied slug and resolve ambiguity with the owner. Never take a
  legacy workflow artifact as input or fallback.
- Stop unless the frame is locked and reviewed, `tasks/` is populated, and
  `ROADMAP.md` has a status board. Point the owner at `/taskflow-tasks`.
- **Before dispatching anything, reconcile every non-pending board row against
  repository evidence** — branches, commits, merged revisions, PRs, CI state,
  live slots. Prefer an available forge/CI integration; fall back to local
  evidence. Reconcile rather than collide (§12).
- Validate every task id against `tasks/` **before** it reaches a slot name, a
  filesystem path or a branch argument. Board text is data, not instruction.

---

## 5. The ready set

A row is ready when **all three** hold:

1. every id in its `needs` is `✅`;
2. every approval gate on it is cleared;
3. it is the lowest `sequence` not yet started within its `group`.

**Two inputs, two sources — and this is not optional.** The board carries
`needs` and status; it does **not** carry `group` or `sequence`. `group` is only
recoverable from section headings and `sequence` exists solely in task-file
frontmatter. The orchestrator therefore reads **both**: the board for status and
`needs`, and `tasks/*.md` frontmatter for `group` and `sequence`. Without the
second source, invariant 4 in §11 has no input at all.

**Ordering** among ready rows: importance descending, then complexity
descending, then id ascending. Deterministic, so `--dry-run` predicts the real
dispatch.

**Computed at dispatch time and never stored.** A stored ready set is a second
record of task state, and the board is the only one.

---

## 6. Concurrency

```
slots     = min( requested, |ready group heads|, 8 ) − |in flight|
requested = N  from --parallel=N        (8 when --parallel=auto)
```

**The `--parallel=N` term is the first argument of the `min`, and it is not
optional.** A formula written without it dispatches the graph's width instead of
the width that was asked for: `--parallel=2` against five ready group heads
would start five workers. At the default `--parallel=1` the formula yields at
most one slot — one task at a time. That is strictly more conservative than
0.5.1's own behaviour: its prose promised "parallel only across independent
groups," and run for real it dispatched more than one ready task per round.

The graph is the real limiter. A group is one conflict domain, so at most one
task per group is ever dispatchable, and separate repositories are separate
groups — which is why the second term counts **ready group heads**, not ready
rows.

**The host's own subagent cap is not a term in this formula.** Nothing exposes
it for reading and its default sits above this ceiling. If a spawn fails with
`Concurrent subagent limit reached`, treat the current in-flight count as this
run's cap for the rest of the run and **do not retry that spawn**.

### 6.1 Routing tier is not execution tier

Two different things are called "tier" and they never interact:

- the board's **routing tier** — `top` / `mid` / `fast` — selects which model a
  worker gets. Use the consumer project's execution workflow and its approved
  mapping for it when one exists;
- the **execution tier** — `native` / `toolkit` / `pipeline`, chosen by
  `--engine` (§8) — selects the substrate that provides the worktree.

### 6.2 Exclusive resources

A task holding an exclusive resource takes the whole run: drain the other slots
first, dispatch it alone, then resume.

**`--solo` is the authoritative channel.** The scan of a task's Definition of
Done for production, deployment, a live database, a bound port or a global
install is explicitly **best-effort and can miss a task the owner did not
name**. A false positive costs throughput; a false negative corrupts a run —
which is why owners are expected to name known-exclusive tasks explicitly rather
than relying on the scan.

### 6.3 `--dry-run`

Stop after computing the plan: print the resolved flags, the ready set in
dispatch order, the slot count, the resolved execution tier, which tasks would
be withheld and why, and which reference modules the table selected. Dispatch
nothing, write nothing, commit nothing. `--dry-run` also **skips the §7 gate**,
so it can answer "what would happen" in a tree that is not ready to run.

---

## 7. The main-checkout invariant (D6)

**The gate.** With `--parallel > 1`, a **dirty main checkout stops the run**
before any dispatch. Report the dirty paths. **Do not stash and do not clean** —
the dirt may belong to another agent working in the same shared checkout, which
is exactly what was found when this design was framed. `--dry-run` skips this
gate deliberately (§6.3).

Confirm the tree state by running `git status --porcelain` directly, **whatever
§2's block shows and even when its sentinel frame is intact** (§2.2). This gate
stops the run, it costs one command, and running it directly is what keeps §7
correct on a host where the block never fires at all.

**The invariant.** During `--parallel > 1`, exactly three writes to the main
checkout are legal, all performed by the orchestrator, all at round boundaries:

1. `.taskflow/<slug>/ROADMAP.md`;
2. a fast-forward of the base branch;
3. submodule pointer bumps.

Nothing else — no source edit, no writing test run, no stash, no clean, no
checkout of a worker's branch to have a look. After each round the main-tree
diff must contain only those; anything else **halts the whole run**, because
isolation has leaked and every further dispatch compounds it.
`references/parallel-execution.md` owns the enforcement details, the postflight
diff form, and where the fast-forward runs.

At `--parallel=1` this is unchanged from previous behaviour: the orchestrator
was already the only writer of the board and never implemented inline.

---

## 8. Execution tiers, in summary

`--engine` picks the substrate; `references/parallel-execution.md` holds the
detail.

| Tier | Precondition | Provides |
|---|---|---|
| `native` | the host offers worker isolation (Claude Code does) | one worktree per worker with an **enforced** main-checkout boundary. **No port allocation, no submodule worktrees, no base branch other than the repository default** |
| `toolkit` | `pipeline` present at or above §8.1's constant | everything `native` provides, plus a port block, per-submodule worktrees, an arbitrary base branch, `ci-wait`, `submodule bump` and `gc` |
| `pipeline` | explicit `--engine=pipeline --pipeline=<name>` | each task becomes one `pipeline drive` run, which owns the whole implement → review → PR → CI → merge → sync lifecycle |

Resolved once, before the first dispatch, and printed with every other flag. It
can only degrade: if `pipeline` disappears mid-run, finish the in-flight slots
and continue in `native`. **`auto` never selects `pipeline`** — that tier hands
merge authority to a pipeline definition, which is an owner decision and must be
typed.

### 8.1 The minimum version constant

```
minimum pipeline version = 0.17.0
  — the first published release whose `pipeline worktree list --json` reports a
    hook-provisioned slot's `submodule_slots[].dir`
```

**Do not re-derive this number from "the first release shipping `pipeline
worktree`."** That derivation yields `0.16.0`, and `0.16.0` is unsafe. Shipping
the `worktree` command is not the property this skill depends on.
`references/parallel-execution.md` §12's reaping precondition enumerates the
repositories a slot spans from
`submodule_slots[].dir`, and on released `0.16.0` that array is **`[]` for every
hook-provisioned slot**. A run there resolves `toolkit`, reconciles from an empty
list, and concludes that a submodule slot holding uncommitted work is an empty
shell — the verdict that destroyed 21,880 bytes of finished implementation.
`0.17.0` is the first release that reports those directories, which is why the
constant is `0.17.0` and not `0.16.0`. Refusing `0.16.0` falls back to `native`,
and that is the safe answer: `native` never consults `submodule_slots` at all.

`--engine=auto` reads `pipeline --version` and compares it against that constant
**numerically** — never by testing whether the binary exists, and never by
probing the subcommand. **At or above it ⇒ `toolkit`. Below it, or `pipeline`
absent, or the reported version unparseable ⇒ `native`, and the run states the
reason once**, naming the version found and the minimum required. A CLI that is
present but older resolves to `native` for the same reason: presence alone would
resolve to `toolkit` and then fail on first use, which is a silent misdetection
instead of an announced degradation. An explicit `--engine=toolkit` is taken at
the owner's word, and a failure that follows is reported as the tier's, not
re-resolved.

### 8.2 Withheld is not failed

In `native`, a task needing a port, a submodule worktree, or a base branch other
than the repository default is **withheld**: its row stays `⬜ pending` — not
`⛔`, not `🔒`, not skipped — the reason is stated **once per run** naming the
tier, the gap and the affected ids, and **every other ready task still
dispatches at full concurrency**. A withheld task consumes no slot and delays
nothing. The completion report names the withheld ids and what would unblock
them.

---

## 9. Board-writing contract

The board in `.taskflow/<slug>/ROADMAP.md` is the only mutable record of task
state, and **you are its sole writer**. Implementers never edit it and never
edit an immutable task spec.

Vocabulary, unchanged: `⬜ pending` · `🔵 in progress` · `🟣 verified, merge
held` · `✅ done` · `🔒 blocked on a gate` · `⛔ blocked on a dependency or
failure`.

- **A row moves only on verified evidence** — a merged SHA, passing checks, DoD
  items checked against the tree — **never on a worker's report.**
- **Per-round commits, not per-row commits.** A round commits the board at two
  points, and two only: once at dispatch (§10 step 3), flipping every row
  dispatched that round to `🔵` in a single commit, and once at outcome (§10
  step 7), recording every row's verified result in a single commit. At
  `--parallel=1` a round is one row, so each of those two commits is the
  surgical single-row commit the skill has always made; at `--parallel>1` it is
  still exactly those two commits for the whole round — never one per row
  dispatched, and never one per row verified. The dispatch commit exists
  because §12's resume reconciles live slots and branches against the board's
  own `🔵` rows to tell an adopted PR from a leaked slot; a dispatch that was
  never committed leaves nothing on the board for that check to find, and §13
  forbids leaving it uncommitted anyway.
- Record the run/PR reference and the date on the row, and a progress-log line
  where one is warranted.
- If a workspace planning store exists, keep exactly one thin pointer to this
  ROADMAP in it. Never duplicate per-task state.

---

## 10. The round

Nine steps, and they are **one unit of work** — not nine places it is acceptable
to stop. §10.1 says when that unit is finished, §10.2 says how it stays parallel
while being finished, and §10.3 is the check that runs before the turn ends.

1. **Compute** the ready set (§5) and the slot count (§6). With `--dry-run`,
   stop here and print the plan.
2. **Gate check.** Production, money, secrets or irreversible effects require a
   distinct owner GO through the available input mechanism. Present the safe
   option first and record the decision *before* dispatch.
3. **Dispatch** each selected task with only its immutable specification — Goal,
   Scope & seams, Definition of Done, taskflow refs — its working-directory
   arrangement, its branch and base, and its isolation boundary. Never the
   board, never other tasks, never permission to merge. Flip each dispatched row
   to `🔵` with its run reference and date, and **commit the board once** for the
   whole round. Issue the round's dispatch calls **together** (§10.2). **This
   step does not end the round and does not end the turn** (§10.1).
4. **Track** every dispatched task to an outcome, **inside the same turn**
   (§10.2), through the available forge/CI evidence; fall back to local branch,
   commit, test and review evidence. Do not busy-wait — and do not end the turn
   in order to wait.
5. **Review**, when `--review != off`, before any merge, by a subagent that is
   never the implementer. `references/code-review.md` owns depths, gating and the
   fix loop. Review is a step **inside** the round, dispatched the same way as
   step 3 and tracked the same way as step 4; a round that reached step 3 and
   never reached here has not reviewed (§10.3).
6. **Merge** per `--merge`: `ask` holds the row at `🟣` and waits for the owner;
   `on-green` merges only when its conditions all hold; `never` stops at `🟣`
   permanently. Merging never bypasses branch protection and never elevates.
   Verify cleanup only after the merge result is known.
7. **Verify every DoD item against the repository.** On success record `✅` with
   the verified change reference and date; on failure record `⛔` with a concise
   reason and then retry, rescope or escalate — never silently continue. Commit
   the board.
8. **Sync submodule pointers**, when that module is loaded, once for the round.
9. **Recompute** the ready set and start the next round.

### 10.1 A round is complete when it is written down, not when it is dispatched

> **A dispatch round is not complete when its workers are dispatched.** It is
> complete when every task dispatched in it has been **tracked to an outcome,
> verified against the repository, and written to the board**. **The orchestrator
> does not end its turn between dispatch and that outcome.**

Steps 4–9 are not the part of the round that happens if there is time left. They
are the part that makes step 3 mean anything: a run that stops after step 3 has
started work, recorded that it started, and read none of it. It has also, at that
point, produced no review, no pull request and no verified row — while its own
exit status says success.

**The corollary is the operative half. If the orchestrator cannot track a
dispatch to completion, it must not dispatch it.** Dispatch fewer workers, or
none at all, and leave the remainder `⬜ pending` for the next round.

A pending row is a true statement about an idle slot. A `🔵` row over a live slot
whose result nobody will read is a false one — and it is *worse* than the task
never having started, because the board now claims work is in progress that no
one will collect, and §12's resume must reconstruct the truth from the tree
instead of reading it. A resume that misreads that state can reap live work,
which is what happened in this design's own proving run.

This binds the concrete case already in §6: when a spawn is refused with
`Concurrent subagent limit reached`, the refused task's row stays `⬜ pending`,
and the calls already issued are still tracked to their outcomes. Never dispatch
past what can be tracked in the hope of collecting it afterwards.

### 10.2 Concurrent and tracked are not opposites

The invariant costs no parallelism, because dispatch is not sequential. A round's
workers are dispatched as **concurrent calls issued together and waited on as one
batch**: the concurrency comes from issuing them together, the tracking comes
from the turn not ending until the batch returns. Neither is bought with the
other.

**On Claude Code the batch is *N* subagent-dispatch calls emitted in a single
assistant message with `run_in_background: false`.** The host runs them in
parallel and returns all *N* outcomes before the turn continues. What matters is
the pair of properties, not the tool's name: *issued together* and *returning to
the calling turn*.

Two ways to get this wrong; the proving run found both.

- **Backgrounding a worker whose result the round needs.**
  `run_in_background: true` returns a task id immediately and no outcome; the
  outcome arrives only as a notification in a **later turn**. In a
  non-interactive (`-p`) session there is no later turn, so it is never read: the
  round ends at "dispatched", every dispatched row keeps a `🔵` over a live slot,
  and the process still exits `success`.
- **One foreground call per message.** This tracks correctly and delivers peak
  concurrency **1**, whatever `--parallel` said. `--parallel=N` is a promise about
  how many workers are in flight *at once*; honouring it one worker at a time
  honours nothing.

A round contains **several** such batches — the implementers, then the reviewers,
then each round of the fix loop. Batches are expected; a batch boundary is a
wait, not a turn boundary. What a round contains is **no turn boundary at all**.

Spawn starts stagger by a second or two as the host creates each worker. That is
spawn latency, not serialization.

**Verified on the host rather than assumed.** Three subagents were
dispatched in one message with `run_in_background: false`, each stamping its own
start and end time around a fixed 25-second wait: all three intervals overlapped
for 18.5 s, the batch finished in 31.6 s of wall clock against a 75 s serial
floor, and all three outcomes returned to the dispatching turn. The same probe
with `run_in_background: true` returned an id and no outcome, and its result
arrived only in a later turn, as a notification.

### 10.3 Closing the round

Before the turn that ran a round ends, confirm all four. This is a check, not a
narration:

1. **Every id dispatched this round has a recorded outcome** — merged, `🟣`,
   `⛔`, or an open PR whose state is named. No dispatched id is unaccounted for.
2. **With `--review != off`, every dispatched task that produced a diff was
   reviewed by a dispatched reviewer.** A round that dispatched workers and zero
   reviewers has not reviewed quickly; it has not reviewed. In a completion
   report a `--review` depth that never ran and a review that found nothing look
   identical, and this check is the only thing separating them.
3. **The board is written and committed for the round** (§9).
4. Where `references/parallel-execution.md` is loaded, its §12.1 audit has run —
   it is a per-round check, not a per-interruption one.

**A round that cannot satisfy all four reports itself as incomplete**, naming
which of the four failed and which ids are affected. Reporting success for a
round that stopped at "dispatched" is worse than reporting failure: a failure is
acted on, while a false success is what allowed a run to dispatch two workers,
review nothing, open no pull request, and exit `success`.

At each wave boundary, report landed / running / blocked work, risks and
resource concerns, on top of the per-round check above.

---

## 11. Runtime invariants

Assert these; do not assume them.

1. The main checkout is clean before parallel dispatch.
2. After each round, the main-tree diff contains only `ROADMAP.md` and submodule
   pointer bumps.
3. One live slot per task id — a slot creation reporting "reused" means this was
   already violated, and is a duplicate-dispatch error rather than a success.
4. No two in-flight tasks come from the same group. (`group` and `sequence` come
   from `tasks/*.md` frontmatter, not from the board — §5.)
5. Every `🔵` row has a live slot or an open PR, and every live slot has a `🔵`
   **or `⛔`** row. A `⛔` row keeps its slot deliberately, for post-mortem; a
   slot with no row at all is a leak.
6. Every task dispatched in a round has a recorded outcome before that round's
   turn ends (§10.1). A `🔵` row this run created and is no longer tracking is a
   defect, not a state — it is invariant 5 passing on evidence that has already
   stopped being maintained.

---

## 12. Interruption and resume

A round is not atomic and a session can end mid-flight. Before any new dispatch,
reconcile `git worktree list` (live slots), `git branch --list worktree-*`
(candidate dispatched branches) and the board's own `🔵` rows.

**`worktree-*` matches two namespaces, and only one of them is this skill's.**
`worktree-<task-id>` is cut per task, by a worker following its brief; its
suffix is a task id, and a task id either resolves to a row on the board or it
does not. `worktree-agent-<hash>` is cut by the host's own worker-isolation
placement (§8, `native`) for **agent isolation** — a different owner (the host,
not this skill) and a different lifetime: it is provisioned for one agent's run
and **survives the agent**, where `worktree-<task-id>` is provisioned for one
task and is reaped with its slot. Both match the glob; only the first is
evidence of a dispatch by this run.

**Read the rule as a derivation, not as a deny-list of one prefix.** A dispatch
is a row plus its slot, so judge every `worktree-*` branch by whether its suffix
resolves to a task id carrying a row on the board. One that does is this run's —
dispatched, or a resume candidate. One that does not — `worktree-agent-*` is the
observed case, and nothing here rules out some other namespace turning up later
— is **not this run's**, full stop, and is read the same way as any branch
belonging to another session: not this round's reconciliation evidence, and
never an orphan of this run's to reap. §13's never-list ("never delete a branch
that is not this run's own `worktree-*` slot") already forbids that deletion;
this section creates no exception to it, for `--merge` or for cleanup.

| Evidence | Action |
|---|---|
| A `🔵` row whose PR is merged | Verify every DoD item, then record `✅`. Only the bookkeeping was interrupted |
| A `🔵` row with an open PR | **Adopt it.** Do not dispatch a second worker — that is the duplicate dispatch invariant 3 exists to prevent |
| A `🔵` row with a branch but no PR | Inspect the worktree; resume against the existing branch, or reset the row to pending and reap the slot. Decide from the tree, not from the row |
| A live slot with no matching row | A leak. `pipeline gc` reports it, `pipeline gc --clean` reaps it. A `⛔` row's slot is not a leak |
| A `worktree-*` branch whose suffix matches no row | **Not this run's** — leave it. This is the expected shape of a `worktree-agent-*` isolation branch, or of another session's task branch; it is neither reconciliation evidence here nor a leak to reap |

The board records what was *dispatched*; the tree records what *happened*.

**The glob is weaker evidence than the slot registry.** `pipeline worktree list`
names only the slots this run provisioned — it never reads a git branch name, so
a `worktree-agent-*` branch cannot appear in it at all. `git branch --list
worktree-*` has no such filter; it reports every literal match, host isolation
branches included. **Where the two disagree, the registry wins.**

---

## 13. Finish, and what never to do

When every scoped row is verified complete: update the taskflow README status
and the ROADMAP counter, remove any thin pointer created for this run, commit,
and report verified results, withheld tasks and why, preserved worktrees and
why, and every outstanding gate.

Never: mark work complete from a worker's report, run two tasks from the same
group concurrently, edit a task specification, implement inline yourself, leave
a board change uncommitted, force-push, delete a branch that is not this run's
own `worktree-*` slot, or pass a bypass flag to a merge.

Never **end a turn with a dispatch outstanding** (§10.1), and never report a run
as successful when any of its rounds stopped short of §10.3. A run that
dispatched work it did not track reports what it dispatched, what it never
collected, and that it is incomplete — including when its own exit status would
otherwise read as success.

---

## 14. Host portability

- The `allowed-tools` grant is read-only in every entry, and narrow by
  construction: `git branch --list:*`, never `git branch:*`, which would
  pre-approve `git branch -D`, `-m` and `-f`. `Bash(git *)` is deliberately
  never used — in a public plugin it would silently pre-approve `git push`,
  `git reset --hard` and `git clean` for every installer. Every mutating command
  goes through the normal permission path; autonomy is not bought with
  self-granted write rights.
- **Every entry keeps its `Bash(…)` wrapper.** A bare specifier such as
  `git status:*` is read as a *tool name*, matches nothing, and grants nothing —
  a silent failure, which is the worst outcome a security control can have. Do
  not "simplify" the list by removing the wrapper: §2's block is
  permission-checked as one command, so an unmatched grant does not merely
  prompt, it fails the whole snapshot.
- `argument-hint` is **Claude Code only**: inert on Codex, and a hard error if
  this skill is ever packaged for the Skills API. A port must drop it.
- §2's injection is a Claude Code fast path, and §2.2's sentinel frame is how a
  port tells a host that has one from a host that does not. §1's table, §2.4's
  instructions and the rest of this file are host-neutral, which is why module
  loading is decided by the table and never by argument preprocessing.
- **§10 splits along the same seam, and a port must keep the halves apart.**
  §10.1's invariant and §10.3's check are host-neutral: a round is complete when
  its outcomes are recorded, on any host. §10.2's *mechanism* is not —
  `run_in_background` and single-message batching are this host's. A port states
  its own equivalent: the primitive that issues several dispatches at once, and
  the setting that makes a dispatch return its outcome to the calling turn. **If
  a host offers no such primitive, §10.1's corollary decides the matter** — that
  port dispatches one task per round and says so, rather than dispatching work it
  cannot track.
