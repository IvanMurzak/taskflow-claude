# Parallel execution — tiers, slots, worker briefs, merge, cleanup

Reference module for `taskflow-execute`. It is loaded **only** when
`--parallel > 1`. A `--parallel=1` run never reads it, which is what keeps the
default invocation as cheap as it has always been.

Read this as the **orchestrator**. Everything below the dispatch line is here:
which execution tier is in force, how a worker gets a working directory, what a
worker is told, what the orchestrator is still allowed to write, when a pull
request may merge, what happens when any of it fails, and how the run is torn
down. Workers never read this file.

**What lives elsewhere, and is not repeated here.**

| Subject | Home |
|---|---|
| Arguments, defaults, the ready set, the concurrency formula, the conditional loading table, the board vocabulary | `SKILL.md` |
| Review depths, gating, the K = 2 fix loop, the reviewer-is-not-the-implementer rule | `references/code-review.md` |
| Submodule detection, the per-round sync, the fetch constraint, the pointer bump | `references/submodules.md` |

Where this file must lean on one of those, it points at it rather than restating
it as a second source of truth. Two places are deliberate exceptions and are
marked as such: the merge section states the one command in the whole run that
can elevate (§9.4), and §8 states the base-branch fast-forward constraint in one
sentence so this file reads standalone.

---

## 1. The three tiers, and how `--engine=auto` resolves

`--engine` picks the substrate. Default `auto`.

| Tier | Precondition | Provides |
|---|---|---|
| `native` | The host offers worker isolation (Claude Code does) | One worktree per worker with an **enforced** main-checkout boundary, gitignored config carried in via `.worktreeinclude`, slot locking while the agent runs, and a sweep afterwards. **No port allocation, no submodule worktrees, no base branch other than the repository default.** |
| `toolkit` *(default when available)* | `pipeline` present **at or above the minimum version that ships `worktree`** | Everything `native` provides, plus the three things it cannot: a port block, per-submodule worktrees, and a slot cut from an arbitrary base branch. Also `ci-wait`, `submodule bump` and `gc`. |
| `pipeline` | explicit `--engine=pipeline --pipeline=<name>` | Each task becomes one `pipeline drive` run; the pipeline owns the whole implement → review → PR → CI → merge → sync lifecycle |

### 1.1 The resolution rule for `auto`

```
pipeline --version        →  bare semver on one line, e.g. 0.15.0
```

- **At or above the minimum version that ships `pipeline worktree` ⇒ `toolkit`.**
- **Below it, or `pipeline` absent, or the check failed ⇒ `native`,** and the run
  states the reason once: *"pipeline X.Y.Z is below the minimum M.N.P that ships
  `worktree` — running in the `native` tier; tasks needing a port, a submodule
  worktree, or a non-default base branch will be withheld."*

**Presence alone is explicitly insufficient, and this is not a stylistic
preference.** A CLI has been on PATH throughout this design that has no
`worktree` command at all (0.15.0). A presence check would resolve that install
to `toolkit` and then fail on first use — a silent misdetection that surfaces as
a broken dispatch instead of an announced degradation. The minimum-version
constant lives in `SKILL.md`; compare against it numerically, never by testing
whether a binary exists and never by probing a subcommand.

**`auto` never selects `pipeline`.** That tier hands the entire task lifecycle to
a pipeline definition, and such a pipeline merges by its own `merge-and-sync`
step rather than by `--merge`. Handing merge authority to a definition the run
did not choose is an owner decision, so it must be typed explicitly.

### 1.2 The tier is resolved once, and it can only degrade

Resolution happens at F1, before the first dispatch, and the resolved value is
printed with every other flag. If `pipeline` disappears mid-run — uninstalled,
PATH changed, a `worktree` call that starts failing on environment — finish the
in-flight slots, then continue in the `native` tier, withholding the tasks that
need what is now gone. The run does not stop and does not upgrade back.

### 1.3 What `--merge` means per tier

`--merge` applies to `native` and `toolkit` only. In the `pipeline` tier the
run's own definition decides, F4–F6 below are skipped entirely, and the run
report says so rather than implying `--merge` was honoured (O5).

---

## 2. What `native` cannot do — three gaps, and *withheld, not failed*

| Gap | Why | Consequence for a task that needs it |
|---|---|---|
| **No port allocation** | The host places a worktree; it does not hand out a free port block | A task whose verification binds a port is withheld, or dispatched solo via `--solo`, or deferred to a `toolkit` run |
| **No submodule worktrees** | Host isolation covers the superproject checkout, not a worktree per submodule | A task whose `repo:` frontmatter names a submodule is withheld |
| **Base branch is the repository default only** — the isolation config's `worktree.baseRef` accepts `"fresh"` or `"head"`, never a branch name | There is no field in which to say "cut this from `next`" | A task whose repository integrates on another branch is withheld. **Both plugin repositories in this workspace are exactly that case:** they are gated and every PR targets `next` |

### 2.1 Withheld is not failed

This distinction is the whole point of the section. When a ready task needs one
of the three gaps in `native`:

1. **The row stays `⬜ pending`.** It is not `⛔`, not `🔒`, not skipped, not
   marked blocked. Nothing about the task is wrong.
2. **The reason is stated once per run, not once per task and not once per
   round.** One line naming the tier, the gap, and the affected ids.
3. **Every other ready task still dispatches**, in parallel, at full concurrency.
   A withheld task consumes no slot and delays nothing else.
4. The completion report names the withheld ids and says what would unblock them
   — normally "install `pipeline` at or above the minimum version and re-run".

Announcing a degradation and continuing is the required behaviour; stopping the
run because one task in eight needs a port is not.

### 2.2 How the orchestrator knows a task needs one of the three

- **Submodule** — the task's `repo:` frontmatter names a path that appears in
  `git submodule status`.
- **Non-default base** — the task's declared base branch (its `base_branch`
  frontmatter, or the repository's known integration branch) is not the base the
  main checkout is on.
- **Port** — `--solo` naming the task is the **authoritative** channel. The scan
  of DoD text for a bound port is best-effort and can miss a task the owner did
  not name; a false negative here corrupts a run, which is why owners are
  expected to name known-exclusive tasks explicitly.

---

## 3. Slot provisioning, per tier (F3)

A **slot** is the triple a worker needs: a working directory, a branch, and a
resolved environment.

### 3.1 `native` — nothing is provisioned, and no path is passed

Dispatch the implementer agent. Its definition carries `isolation: worktree`, and
**that field is the mechanism**: the host creates the worktree, enforces the
main-checkout boundary in code, applies `.worktreeinclude` so gitignored config
such as `.env` reaches the worker, locks the slot for the agent's lifetime, and
sweeps it afterwards. The orchestrator provisions nothing, runs no `git worktree`
command, and creates no branch.

> **The orchestrator passes no path.** Not in the brief, not as a `cd`, not as a
> working-directory argument, not as a "your worktree is at …" line.

**The reason, and it is the reason the design chose native placement at all.**
Handing a worker an externally provisioned directory means the worker must enter
a directory outside `.claude/worktrees/`, and that **costs an approval prompt
that no permission rule can suppress**. Not a prompt that a broader
`allowed-tools` grant removes, not one a settings entry pre-approves — one
unavoidable interaction per worker. At eight parallel workers that is eight
prompts per round, which is not unattended parallelism at all. Native placement
costs zero prompts because the host put the worker there itself.

So in `native`, brief item 2 (§7) reads *"your agent definition's
`isolation: worktree` has already placed you in your own worktree; that worktree
is your only writable root"* — a statement of fact, not an instruction to
navigate.

### 3.2 `toolkit` — native placement still places; the CLI adds what it cannot

The implementer agent still carries `isolation: worktree`, so the worker is still
placed by the host and the prompt cost is still zero for the ordinary case. The
CLI is called for the gaps in §2:

```
pipeline worktree create --name <task-id> --base <base> \
                         [--submodules <declared>] [--ports <n>] --json
```

Run it **from the project root** — the command uses the current directory as
`PIPELINE_WT_PROJECT_ROOT`, exactly as the pipeline run path does.

`--json` returns one object; the fields that matter to dispatch are:

| Field | Meaning |
|---|---|
| `status` | `created` · `reused` · `failed` — see §5 |
| `worktree_path` | absolute path to the provisioned **parent** directory. For a submodule task this is *not* where the work goes — see the next row |
| `branch` | the branch that was cut, `worktree-<name>` by contract |
| `env_file` | absolute path to the dotenv file — the input to §6 |
| **`submodule_slots`** | **one entry per declared submodule — `{path, name, dir, base, source, exists}` — and `dir` is the directory a submodule task's worker actually works in.** `[]` when none were declared: never `null`, never absent |
| `ports` / `port_base` / `ports_source` | the allocated block, its base, and which side allocated it (`builtin` or `hook`) |
| `provisioner` | `builtin` or `hook` — which side cut this slot |
| `reused` / `reused_evidence` | `true` plus `registry` or `git-worktree` when the slot already existed |
| `base_branch`, `submodules`, `hook_dir` | echoed back as resolved |
| `detail` | the failure reason when `status` is `failed` |

Exit **0** success · **1** the provisioning hook failed (soft-fail or hard-fail;
`detail` says which) · **2** usage, an invalid `--name`, or an invalid
`--outcome`.

**`submodule_slots` is in this table because a resume depends on it.**
`pipeline worktree list --json` reports the same field per slot, derived at list
time for slots the current process did not create — which is the only channel a
resumed run has (§12). Each entry's `source` says which channel named `dir`, so a
derived guess is never dressed up as a reported fact:

| `source` | where `dir` came from |
|---|---|
| `record` | the built-in provisioner reported it as it cut it, and the slot record kept it |
| `env-file` | the slot's env file publishes it — a **hook's** own answer, on the channel the frozen contract leaves for it |
| `derived` | neither channel named it, so this is the layout convention both provisioners follow (`<parent slot dir>--<submodule basename>`) — check `exists` |

A `derived` entry carries `base: ""` rather than a guessed branch; the key is
always present either way.

**The env file names the same directories, and named them first.** Beside
`WORKTREE_PATH` and the ports, it carries `SUBMODULE_COUNT` and, per submodule,
`SUBMODULE_<n>_PATH` · `SUBMODULE_<n>_NAME` · **`SUBMODULE_<n>_DIR`** ·
`SUBMODULE_<n>_BASE`, plus `SUBMODULE_DIR_<NAME>` / `SUBMODULE_BASE_<NAME>`
aliases. `SUBMODULE_<n>_DIR` is the submodule worktree — the value §7 item 5
inlines into a submodule task's brief. Two channels now name it; before `a11` the
env file was the only one, and nothing on the resume path read it. §12 is where
that cost is paid.

**Which of the three gaps a slot is closing decides where the worker actually
works**, and the brief must be explicit about it:

- **Ports only.** The worker stays in its host-placed worktree. The orchestrator
  inlines the port values (§6) and the worker never learns the CLI slot exists.
- **A submodule task.** The submodule's own CLI-provisioned worktree — branched
  from *that submodule's* integration branch, not from the superproject's pin —
  is where the submodule work happens. Its directory and branch are inlined.
- **A non-default base branch.** Host placement cannot cut from `next`, so the
  CLI slot itself is the working directory. **This is the case that pays the
  approval prompt from §3.1**, once for that worker. That is expected and
  correct: `toolkit` provisioning exists for the tasks `native` would otherwise
  withhold, and a prompt is a better outcome than a withheld row. It is not a
  reason to provision a CLI slot for every dispatch — do not.

**Two properties of the shipped command worth knowing before relying on it.**

1. **`create` runs the consumer's `worktree-create.*` hook where one exists**,
   resolved from `<project>/.pipeline/.hooks` (override with `--hook-dir <path>`).
   `--base` and `--submodules` reach that hook as `PIPELINE_WT_BASE_BRANCH` and
   `PIPELINE_WT_SUBMODULES`, and the hook is then what cuts the branch. **A
   project with no such hook is not a failed create:** the CLI's built-in
   provisioner cuts the slot itself — the worktree outside the repository, one
   worktree per `--submodules` entry from *that submodule's* own integration
   branch, a free port block, and the env file — and reports
   `provisioner: "builtin"`. `toolkit` is reachable in a hookless project, so a
   task is not withheld for lack of a hook.
2. **`--ports <n>` is allocated, not merely recorded.** `--json` returns the block
   in `ports` and `port_base` and names the allocator in `ports_source`. The
   allocation is resolved **per field**, so a hook that returns no ports still
   receives the provisioner's: a hook-provisioned slot reports
   `provisioner: "hook"` alongside `ports_source: "builtin"`. §6.1 owns that
   precedence and explains why it is per-field.

### 3.3 `pipeline` — the run provisions itself

The orchestrator launches one run per task:

```
pipeline drive --root <pipeline_root> --run-id <id> --start <step-name> \
               [--effort code-review=<depth>] --json
```

The run creates its own slot, and **F4 through F6 are skipped entirely** — no
worker brief from §7, no orchestrator-side merge from §9. Mint the run id with
`pipeline id`; never invent one. `--merge` does not apply (§1.3).

### 3.4 A slot is made by the substrate or it is not made at all

**Never reproduce `pipeline worktree create` with raw git.** Not
`git worktree add -b worktree-<task-id> <base>` in the superproject, not
`git -C <submodule> worktree add …`, not "just this once, because `create` is
failing and the round is waiting".

When a slot cannot be provisioned — the CLI is below the minimum version, the
tier degraded mid-run, `create` exits non-zero, the hook refuses — the correct
response is the one §2.1 already defines for the `native` gaps: **withhold the
task and say why.** A withheld row stays `⬜ pending`, costs one stated line in
the run report, and delays nothing else. It is exactly how `native` withholds a
task that needs a port, and a `toolkit` slot that could not be cut is the same
situation, not a worse one.

**What a hand-rolled worktree costs, concretely.** This is why the rule is a
correctness rule and not a preference:

- **no slot record** — nothing under `.pipeline/.runtime/worktrees/` knows it
  exists;
- **no env file** — §6's inlining has nothing to read, and §7 item 4 cannot be
  satisfied;
- **no port allocation** — a task that needed a port still does not have one,
  which was the whole reason to be in `toolkit` rather than `native`;
- **invisible to `pipeline worktree list`**, which answers *"no provisioned
  worktree slots"* while the directories are live on disk;
- **invisible to the registry side of `pipeline gc`** — and a worktree hand-rolled
  inside a *submodule* is invisible to the superproject's `git worktree list`
  too, so §12.1's audit does not see it either;
- and therefore **the run-completion `gc` report is false**: it says the ground is
  clear because it cannot see what is standing on it.

That last consequence is the one that matters. The orchestrator's own closing
evidence becomes untrue, and nothing downstream can tell that it is.

**Observed, not hypothetical.** In this design's own proving run, a resumed round
believed the CLI path was broken and hand-rolled two worktrees with
`git -C <submodule> worktree add … -b … origin/main`. Branch and base were
correct, so nothing looked wrong from the outside; there were simply no slots.
`pipeline worktree list` reported none while both were live, no env file existed
for either, no ports were allocated, and the run's `gc` reported clean ground. A
CLI defect is a reason to withhold and report — never a licence to route around
the substrate, because the substrate is also the bookkeeping.

---

## 4. Slot naming

| Item | Form | Why |
|---|---|---|
| **Slot name** | the task id, **validated against `tasks/` first** | The id reaches a filesystem path, a branch name, and a user-authored hook's environment. Validating that the id names a real file under `tasks/` before it reaches any of those is gate SG6 |
| **Branch** | the substrate's own — `worktree-<name>` | Not invented by this skill. It is the namespace `pipeline gc` already scans and reaps, and the namespace the preflight's `git branch --list worktree-*` already reports |
| **Location** | `native`: `.claude/worktrees/`, host-managed. `toolkit`: the substrate's own root, outside the repository | A worker's build artifacts never land in the project folder |

**The skill invents no branch namespace of its own.** `worktree-<name>` is the
substrate's, and it is the only one *this skill* deliberately creates or looks
for — in the preflight snapshot, in reconciliation (§11), and in `gc`'s reaping
(§12). Do not create branches under any other prefix and do not look for them
under one. The raw `worktree-*` glob is not this exclusive, though: it also
catches the host's own `worktree-agent-*` isolation branches (§11), which this
skill neither creates nor is entitled to reap.

The CLI additionally enforces its own name rules, which a validated task id
already satisfies: it must match `[A-Za-z0-9][A-Za-z0-9._-]*`, be at most 64
characters, contain no `..`, not end in `.`, and not be a Windows reserved device
name. A refusal is exit 2 with the reason — treat it as a bug in the task id, not
as something to work around by rewriting the name.

---

## 5. Collision safety is not git's job

**Slot creation is idempotent per name by frozen contract.** A second `create`
for a name that already has a slot does **not** fail: it **reuses** the existing
slot and re-reports it, with `status: "reused"` and `reused_evidence` naming what
proved it (`registry` — the CLI's own slot record, or `git-worktree` — a git
registration that predated the hook call). Exit code **0**.

That is correct behaviour for a contract that has to be re-runnable. It also
means **git will not catch a duplicate dispatch for you.**

Therefore:

1. **Uniqueness is the orchestrator's in-flight table.** One live slot per task
   id, asserted by the orchestrator before it calls `create`, not discovered
   afterwards.
2. **A `create` that reports `reused` is a duplicate-dispatch error.** It means
   invariant "one live slot per task id" was already violated. Do not proceed
   with the worker. Do not treat the exit code 0 as success. Stop that dispatch
   and reconcile through §11.
3. **The git refusal to check one branch out twice is a backstop only** — a
   second line of defence that happens to exist, never the control being relied
   on.
4. **`--force` is never passed.** `pipeline worktree` exposes no force flag on
   any of its four verbs, and `git worktree add --force` — which would override
   exactly the backstop in point 3 — is never used by the orchestrator or by a
   worker. If a slot cannot be created without forcing, that is information, not
   an obstacle.

---

## 6. Environment — precedence, and inlining rather than sourcing

### 6.1 Precedence, later wins

For any value visible to a worker:

1. **The substrate's defaults.**
2. **The built-in provisioner's values.**
3. **The consumer hook's values — resolved *per field*, not wholesale.**
4. **Values the orchestrator inlines into the brief** (task-specific).

**Point 3 is per-field on purpose, and getting it wrong makes ports unreachable.**
A hook that returns `ports`/`port_base` **empty or absent does not suppress the
provisioner's ports** — the provisioner fills those fields, and only a hook
returning *non-empty* ports overrides them. Hook-wins-wholesale would have made
port allocation unreachable in every repository that already has a create hook,
including this one, whose hook returns `{}` for both fields. The feature would
have shipped unreachable in exactly the project that proves it.

### 6.2 The env file's grammar is constrained, and not by accident

The env file is dotenv. Values must be **unquoted and free of spaces and shell
metacharacters**. The tolerance of the parser is not the constraint that matters:
the file has a second consumer that reads it with `set -a && source`, and that
consumer will happily execute what a lenient parser would merely tolerate.

### 6.3 Inlining, not sourcing

**The orchestrator reads the env file and inlines the values as text in the
brief.** It never tells a worker to source it.

- **Why not source:** sourcing is shell-specific. `set -a && source` is not a
  thing on PowerShell, and the worker's shell is not something the orchestrator
  chose or can rely on. An inlined value works everywhere, including on a host
  with no bash at all.
- **Why only some keys:** the orchestrator inlines **only the keys the brief
  declares it needs** — the ports, the worktree path where one applies, the
  submodule directories. **Unknown keys are never echoed.**

**The privacy reason for that last rule, stated plainly.** The create hook is
**user-authored**, and the frozen contract does not forbid it writing a secret
into `env_file`. The file is *intended* to carry non-secret configuration; it is
not *guaranteed* to. An orchestrator that dumped the whole file into a brief
would put whatever a third party's hook chose to write there into an agent
transcript, a PR comment, and a run report. Declared keys only. Anything else a
worker genuinely needs is passed **by reference to the env file**, not by value.

### 6.4 Ports

Allocated per slot as a contiguous free block, probed from a deterministic base
derived from the slot name — so re-provisioning the same slot is stable, and a
port already in use on the machine is skipped rather than assumed free. The block
is written into the env file and inlined into the brief. **A worker never infers
a port and never picks a default one.**

---

## 7. The worker brief — the contract

Seven items. All seven are required; the brief is short, but it is not partial.

1. **The immutable specification** — Goal, Scope & seams, Definition of Done,
   taskflow refs. Verbatim, not summarised. The worker may not edit it.
2. **How the working directory is established.** In `native`: the agent
   definition's `isolation: worktree` has already placed the worker, and no path
   is passed (§3.1). In `toolkit`, for a task working in a CLI-provisioned slot:
   the slot path the worker is entering. **This is the item whose absence leaves
   a worker implementing in the main checkout.** A brief missing item 2 is not a
   terse brief — it is a broken one.
3. **Branch and base branch.** The branch is `worktree-<task-id>`. The base is
   the repository's real integration branch — `next` for both plugin
   repositories in this workspace, not `main`. Never "the default"; name it.
4. **Resolved environment values, ports included — declared keys only** (§6.3).
5. **For a submodule task:** which submodule, which directory, and which
   integration branch. All three; two of the three is a worker guessing the
   third.
6. **The isolation boundary.** The worktree is the only writable root; the main
   checkout is read-only; the board is not the worker's to edit; `git worktree`
   and `git submodule` are never run in the parent. On Claude Code the host
   **enforces** the first two — the brief states them so a worker *understands a
   refusal it will receive*, not because the brief is the control (§8).
7. **What to report:** outcome, evidence, PR reference — and that **the report is
   not proof of completion**. The orchestrator verifies every DoD item against
   the repository itself.

**Never included in a brief, under any flag:**

- **The board.** Not its contents, not its path, not a summary of it. A worker
  that can see the board is a worker that can be tempted to update it.
- **Other tasks.** Not the wave, not the dependency graph, not what else is in
  flight. A task's spec is complete by construction; context about siblings only
  invites scope creep across a conflict boundary.
- **Permission to merge.** Merge authority is not delegated to a worker under any
  flag, in any tier. §9 is the orchestrator's, entirely.

---

## 8. The main-checkout invariant (D6)

During `--parallel > 1`, **exactly three writes to the main checkout are legal**,
all performed by the orchestrator, all at round boundaries:

1. `.taskflow/<slug>/ROADMAP.md`;
2. a fast-forward of the base branch;
3. submodule pointer bumps.

Nothing else. No source edit, no test run that writes, no stash, no clean, no
`git checkout` of a worker's branch to "have a look".

### 8.1 On Claude Code the boundary is enforced, not requested

A session isolated in a worktree has, **in code**:

- `Edit` / `Write` / `NotebookEdit` into the main checkout blocked;
- Bash and PowerShell commands whose working directory resolves there blocked;
- `git -C` / `--git-dir` / `GIT_DIR` / `cd`-then-git redirects into it blocked;
- **the same checks applied to every subagent spawned from that session.**

The design relies on this rather than on a worker's good behaviour. Write the
rules into the brief anyway (§7 item 6) — but write them so a future editor
cannot mistake the prose for the enforcement and delete one believing the other
is doing the work.

### 8.2 Orchestrator-side residue

Three things the host does not do for you:

**Preflight — the main tree must be clean.** With `--parallel > 1`, a dirty main
checkout **stops the run** before any dispatch. Report the dirty paths. **Do not
stash and do not clean** — the dirt may belong to another agent working in the
same shared checkout, which is exactly what was found in this workspace at
framing time. (`--dry-run` skips this gate deliberately, so it can answer "what
would happen" in a tree that is not ready to run.)

**Per-repo slots.** A task whose `repo:` is a submodule works in that submodule's
own worktree, on that submodule's integration branch. It does not work in the
superproject's worktree with a `cd` into a submodule directory.

**Postflight — after each round, the main-tree diff must contain only
`ROADMAP.md` and pointer bumps.** Anything else halts the whole run (§10).

### 8.3 The postflight diff uses the merge base — three dots, not two

```
git -C <main-checkout> diff --name-only <base>...HEAD        # ✔ three dots
git -C <main-checkout> diff --name-only <base>..HEAD         # ✘ two dots
```

**Two-dot compares the two tips.** If the base branch advanced after the round
started — someone else merged, or an earlier round's own pointer bump landed —
every file that moved on the base since then shows up in the diff as though this
line of work had *reverted* it. That reads exactly like foreign changes in the
main tree, and the response to foreign changes in the main tree is **halting the
entire run**. A false isolation-leak alarm is therefore not a cosmetic bug; it is
an outage of the orchestrator.

**Three-dot compares against the merge base** (`git merge-base <base> HEAD`),
which is what "what did this line of work change" actually means. Whatever moved
on the base independently is excluded, because it is on the base's side of the
fork point.

This is not theoretical: **it fired during round 1 of this taskflow's own
execution** and produced exactly that false alarm.

The general rule: **any diff that answers "what did this branch/line of work
change" uses three dots against the base.** Only a diff between two recorded
points on the *same* line of history — the round-start SHA and the round-end SHA
of the main checkout, for example — may use two dots, because there is no fork
to be confused by.

### 8.4 Where the fast-forward runs

Stated in one sentence for standalone reading, and owned in full by
`references/submodules.md` §4: git refuses to move a branch that any worktree has
checked out, so the base-branch fast-forward runs as
`git -C <main-checkout> pull --ff-only` while the main checkout is on the base
branch — never as a `<src>:<dst>` refspec fetch from a worktree. When you only
need to *know* where a branch is, fetch and read `origin/<base>` without moving
the local ref.

### 8.5 The host's isolation root is the *primary* checkout, not the orchestrator's cwd

An orchestrator can itself be running inside a **linked** worktree. The host does
not follow it there: worker worktrees are still created under the **primary**
checkout's `.claude/worktrees/`. Observed directly in this design's proving run —
the orchestrator's cwd was a linked worktree under `C:/tmp/…`, and its three
worker slots were created under the primary checkout's `.claude/worktrees/`.

Two consequences follow, and neither is cosmetic.

1. **D6 survives this only because `.claude/worktrees/` is gitignored.** Worker
   directories land *inside* the main checkout's path; §8.3's postflight diff does
   not report them only because git never sees them. In this workspace the ignore
   is a **tracked** `.gitignore` entry, so it travels with the repository. A
   consumer repository that lacks one gets worker worktrees appearing as untracked
   paths in the main tree, which §10 reads as a leaked isolation boundary and
   **halts the whole run** for. Before the first `--parallel > 1` run in an
   unfamiliar repository, confirm the entry exists.
2. **F-9's own remedy is unavailable to a run scoped to a worktree — and that is
   the arrangement this design prescribes.** §10's host-refusal row says to clear
   the leftover registration and retry once, but the leftover is in the *primary*
   checkout, and a worktree-scoped run may not write there (§8, enforced in code
   on Claude Code). Such a run cannot clear its own leftovers. **This is not an
   edge case.** D6 and D13 put the orchestrator in a linked worktree with the
   primary checkout read-only, so the sanctioned remedy is unreachable by
   default — unavailable exactly when it is needed. It blocked an entire proving
   run: two spawns refused, nothing dispatched, both rows withheld. Report the
   refusal, name the primary checkout's `.claude/worktrees/` path so the leftover
   can be found, and withhold the affected dispatch; clearing it is an owner
   action, run from the primary checkout.

**The standalone-clone workaround, and what it costs.** The remedy *is* reachable
from a **standalone clone**, because a clone is its own primary checkout and the
orchestrator may write its `.claude/worktrees/`. That is the only reason the
second proving run dispatched at all after the first was blocked. It is a real
workaround and it is not free — take it knowingly, and record in the run report
that the run is executing from a clone and which one:

- **It does not prevent the refusal.** The clone's very first isolation attempt
  was refused too, on a clean slate. What the clone restores is the *reachability*
  of the remedy, nothing about the isolation itself.
- **It is no longer the D13 arrangement.** §8's three legal writes — the board,
  the base fast-forward, the pointer bumps — now land in a tree the shared
  checkout cannot see until they are pushed and pulled, and two trees hold the
  same taskflow. Exactly one of them may be the board's writer.
- **The submodule pointers are the clone's.** They are uninitialized until
  `git submodule update --init`, and then they are whatever the cloned commit
  pins — never the shared checkout's *local* pointer state, so anything bumped,
  staged or checked out there but not pushed is simply absent.
- **Local state a clone lacks is genuinely gone**: gitignored configuration such
  as `.env` (which is what `.worktreeinclude` exists to carry into a worker — in
  a fresh clone there is nothing to carry), `.git/info/exclude` entries,
  local-only branches, installed dependencies and build outputs, and the
  `.pipeline/.runtime/` slot records of earlier rounds. §11's reconciliation
  therefore correctly finds nothing to adopt, and a run resumed in a clone starts
  against an empty slot registry.
- **The leftovers stay where they were.** Relocating abandons the locked
  registrations the refusals already left in the shared checkout. They remain an
  owner action there, and nothing in the clone will ever reap them.

---

## 9. Merge (F6)

| `--merge` | Behaviour |
|---|---|
| `ask` *(default)* | Hold the row at `🟣`, present the PR and its CI state, and wait for the owner. Nothing merges unattended |
| `on-green` | Merge only when **all four** conditions in §9.2 hold |
| `never` | Stop at `🟣` permanently. This run does not merge, and does not ask |

`--merge` applies to `native` and `toolkit`. The `pipeline` tier merges by its
own definition (§1.3).

Before the first branch push and the first PR **per remote**, obtain
authorization once for that remote, for that run. Outward actions are not covered
by having been told to execute a taskflow.

### 9.1 Required-check presence is read from the branch-protection API

**Read once per repository per run, from the branch-protection / rulesets API:**

```
gh api repos/<owner>/<name>/branches/<base>/protection
gh api repos/<owner>/<name>/rulesets
```

Cache the answer for the run. What you are determining is a single fact: **does
this repository's base branch require any status check to pass before merge?**

**Never infer it from a `gh pr checks` exit code.** That exit code **cannot
distinguish "no required checks are configured" from "checks failed"** — both are
non-zero, and treating the first as the second (or worse, the second as the
first) is how an auto-merge fires on a repository where "green" was never
defined. This is the specific misuse the rule exists to forbid; the exit code is
useful for many things and is not usable for this one.

**No required checks ⇒ "green" is undefined ⇒ fall back to `ask`, and say so.**
Print the fallback: *"`<owner>/<name>` has no required checks on `<base>`; green
is undefined there, so `--merge=on-green` falls back to `ask` for its rows."*
Silently merging because nothing could fail is precisely backwards.

Note that `pipeline ci-wait`'s own exit **4** — *no checks appeared within
`--grace`* — is a fact about **this PR at this moment**, not about the
repository's configuration. It is not a substitute for the API read either.

### 9.2 The four conditions for `on-green`

All four, every time:

1. **CI reached a terminal pass.**

   ```
   pipeline ci-wait --pr <n> --repo <path|owner/name> [--timeout <sec>] [--json]
   ```

   Exit **0** every check passed · **1** a check failed · **2** usage or `gh`
   missing · **3** timeout · **4** no checks appeared within `--grace`. Only exit
   0 is green. It fails fast by default, so the first failed check ends the wait
   rather than burning the timeout.
2. **The repository actually has required checks on the base branch** (§9.1).
3. **No blocking review finding is open.** Whether a finding blocks is decided by
   `references/code-review.md`, not here.
4. **The row is behind no approval gate.** `on-green` never applies to a gated
   row — a gate is an owner decision and a green check is not one.

### 9.3 `on-green` never elevates

- It **never bypasses branch protection**.
- It **never passes `--admin`**, and never any other bypass flag or
  rule-bypass API call, to `gh pr merge` or to anything else.
- A merge GitHub **refuses is reported, not retried with elevation.** Report the
  PR, the refusal, and the reason; leave the row at `🟣`. Do not re-run the merge
  with a flag that makes the refusal go away — the refusal *is* the repository's
  answer.
- It never force-pushes, and it never deletes a branch that is not its own
  `worktree-*` slot.

### 9.4 The one place elevation can occur — `pipeline submodule bump`

There is exactly one command in a taskflow run that can elevate, and the
orchestrator is responsible for stopping it.

**`pipeline submodule bump` lands its pointer commit through its own pull
request, and by default its land step retries a refused merge with
`gh pr merge --admin`**, reporting `merged_via_admin: true` — equivalently
`merge_outcome: "admin"` — when it did. That is the command's default, not the
orchestrator's policy; but a run that invokes it unguarded has elevated,
whatever the policy says.

Therefore:

- **The orchestrator suppresses that elevation by passing `--no-admin`, on every
  `submodule bump` invocation.** It is a bare flag, no value. Every invocation —
  including a `--dry-run` rehearsal. Not conditional on whether the repository
  looks protected: the orchestrator does not decide that per run.
- **The flag is what makes §9.3 true of this command.** With it, the plain merge
  is attempted once, a refusal is reported and terminal — `status: "halted"`,
  `merge_outcome: "refused"`, `halt_reason` naming the PR, GitHub's own text in
  `stderr`, exit **1** — and `gh` is never invoked with `--admin` at all. Without
  it, the CLI's default fallback is live and the promise is only prose.
- **A refused bump is reported and retried, never routed around.** Nothing is
  lost: the commit is on `origin` and the PR is open, so the bump lands by
  satisfying the gate that refused it, exactly like a task PR. Leave the pointers
  unbumped and let the next round retry. Do not re-run the command without the
  flag, and do not hand-merge past the gate.
- **`merged_via_admin: true` in a bump report is a defect, not a note.** The flag
  makes that outcome unreachable, so seeing it means the invocation omitted
  `--no-admin`. Fix the invocation; report the elevation that already happened as
  a finding against the run.
- The bump procedure itself — when it runs, which submodules it names, how its
  `skipped[]` entries are read — belongs to `references/submodules.md` §6 and is
  not repeated here.

The CLI's default is still the fallback, deliberately: flipping it would change
every existing caller's behaviour without their asking. The burden of opting out
sits with the automated caller that made the promise, and here that caller is
this orchestrator.

This is the only command that has to be told not to elevate. Task PRs never
elevate, under any flag, in any tier.

---

## 10. Failure paths

Every response is defined, announced, and non-forcing. Exactly one halts the run.

| Failure | Response |
|---|---|
| **Dirty main checkout at preflight** | Stop before any dispatch. Report the dirty paths. No stash, no clean |
| **Slot creation fails** | The row stays pending. Reap the partial slot with `pipeline worktree destroy --name <task-id> --outcome completed` — `completed` reaps, and `halted` would *preserve*, which is wrong for a slot with nothing in it. **Do not retry blindly, and do not hand-roll the slot with raw git** (§3.4): withhold the task and say why |
| **Host refuses to create the isolation worktree** — `native` tier, before any `pipeline worktree` slot exists at all: *"Refusing to use … as an isolation worktree: … git metadata could not be resolved … Isolation is refused rather than assumed"*. **The trigger is the protected checkout being itself a linked worktree** — its `.git` is a *file* pointing at the primary checkout's git dir, not a directory, so the host cannot verify its git identity. It is **not** a property of the host OS, and not specific to Windows | **Fails closed, correctly** — no silent fall-back to the main checkout, so this is a retry situation, not a corruption one. But the refused attempt leaves the worktree **locked**, and that locked leftover is what makes the very next attempt fail the same way. **Do not retry blindly, and do not force** — retrying straight against the same lock only compounds the leftovers rather than clearing them. **First decide whether the lock is stale or live (§10.1): they present identically and their responses are opposite.** *Stale* — no holder is still running: clear the leftover, then retry the dispatch once. The clearing runs in the *primary* checkout, whose `.claude/worktrees/` holds it, **so a run that is itself scoped to a worktree cannot perform it and must withhold instead (§8.5)**. *Live* — the holder is still running: there is nothing to clear, `prune` is a **no-op**, every retry re-fails identically, and forcing the removal **destroys a running worker**. Wait for the holder to exit or withhold the dispatch, and never force |
| **`create` reports `reused`** | Duplicate dispatch (§5). Do not proceed with the worker. Reconcile via §11 |
| **`gh` absent or unauthenticated** | There is no PR path. Workers push branches only, rows stop at `🟣`, and the run says so **once** — not per task |
| **Base-branch fast-forward fails** | Do not force. Report and continue the round; the next round retries |
| **Pointer bump or parent push rejected** | Report. Leave pointers unbumped. No force-push, no elevation flag added by the orchestrator |
| **Port allocation exhausted** | Withhold the task exactly as in the `native` tier (§2.1), and state the range that was tried |
| **Reviewer subagent fails** | Treat as a blocking finding for that round. A second failure escalates the row to `⛔` rather than merging unreviewed |
| **Reviewer subagent cannot be *dispatched* at all** — host isolation refuses it, typically against a live lock held by the task's own still-running implementer (§10.1) | **Do not substitute another agent type for it.** `general-purpose` in particular carries **neither** property gate SG5 exists to assert: no `isolation: worktree`, so the review is not placed in its own worktree and §8.1's enforced boundary covers neither it nor anything it spawns; and none of `taskflow-reviewer`'s own definition — its read-only tool set, and the never-review-a-diff-you-produced rule (`references/code-review.md`). Once posted to the PR its output is indistinguishable from a contract review, and it is not one. Hold the row unreviewed and report why: **a failed review round is the better outcome than a review that silently dropped both guarantees.** Observed — the second proving run made exactly this substitution and posted the result to a PR |
| **Worker fails or times out** | `⛔` with the reason. **The worktree is preserved** and its path recorded. `--on-fail=continue` keeps the other slots running; `--on-fail=stop` drains the in-flight slots and halts |
| **Review still blocking after K rounds** | `⛔`. Worktree preserved. (K and the loop are `references/code-review.md`'s) |
| **CI red** | The row stays `🟣` with the failing check named. No merge |
| **Merge conflict** | `⛔`. The branch and the worktree are preserved. **The orchestrator does not resolve conflicts on a task's behalf** |
| **Postflight finds foreign changes in the main tree** | **⚠ HALT THE WHOLE RUN.** Report the paths. Isolation has leaked, and every further dispatch compounds it. Check §8.3 first — a two-dot diff manufactures this alarm out of an advanced base branch |
| **`pipeline` disappears mid-run** | Finish the in-flight slots, then continue in the `native` tier, withholding the tasks that need it (§1.2) |
| **A spawn fails with `Concurrent subagent limit reached`** | Treat the current in-flight count as this run's cap for the rest of the run. Do not retry the spawn |

**The halting row is the only one.** Everything else degrades, reports, and
continues — because a run that stops on the first imperfection finishes nothing,
and a run that continues past a leaked isolation boundary corrupts a shared
checkout.

### 10.1 A stale lock and a live lock look identical — separate them before acting

Both appear as `locked` in `git worktree list`, and **that annotation is not the
discriminator**: git suppresses the `prunable` marker on a locked entry even when
its directory is already gone, so the listing alone cannot tell them apart. Ask
two questions, in this order. `git worktree list --porcelain` sets them up — it
gives each entry's path, a `locked <reason>` line wherever a lock is held, and a
`prunable <reason>` line wherever git considers the registration dead.

Everything below concerns the **host's** leftovers under `.claude/worktrees/` in
the primary checkout. A CLI-provisioned slot is never reaped with raw git — that
is `pipeline worktree destroy` and `pipeline gc`, §12, and §3.4 is unaffected.

**1. Does the registered directory still exist on disk?**

- **Gone ⇒ a stale registration.** `git worktree prune` **on its own will not
  clear it** — prune skips a locked registration entirely and does not even
  report it as prunable. `git worktree unlock <path>`, then
  `git worktree prune`. Retry the dispatch once afterwards.
- **Present ⇒ question 2**, because who owns that directory decides everything.

**2. Is the lock's holder still running?** The reason string is the host's own
text and names its holder — observed as `claude agent agent-<id> (pid <n>)`. Ask
the operating system whether that pid is alive: `Get-Process -Id <n>` on Windows,
`ps -p <n>` elsewhere.

- **Alive ⇒ a LIVE lock, and there is nothing to remedy.** Prune is a **no-op**,
  every retry re-fails identically and mints one more locked leftover, and the
  only thing that would remove the directory destroys a running worker. Wait for
  the holder to exit, or withhold the dispatch and say why. This is the case that
  blocked a reviewer dispatch in the second proving run — against the
  *implementer's own* worktree, still running its test suite.
- **Not running ⇒ an abandoned leftover**, the ordinary product of a refused
  spawn: the directory and its registration both outlived an agent that never
  started. Ask git about it before judging it, exactly as §12's reaping
  precondition requires — `git log --oneline <base>..<branch>` and
  `git status --porcelain` in that directory. Such a worktree is empty by
  construction, but that is a claim to check, not one to assume. Then
  `git worktree unlock <path>`, `git worktree remove <path>`,
  `git worktree prune`, and retry the dispatch once.

**`git worktree prune --dry-run --verbose` is the cheap check, and its exit status
is not the answer.** It names exactly what a real prune would remove and exits
**0** whether it found something or nothing, so read the output, not the code. An
empty result says only that prune would do nothing — true of a live worktree, and
equally true of a locked registration whose directory is already gone. That is
why question 1 comes first.

**Every escalation past a plain `remove` is the thing to refuse.**
`git worktree remove` refuses a **locked** worktree — exit **128**, quoting the
lock reason back at you — and refuses an **unlocked** one holding modified or
untracked files: exit **128**, *"contains modified or untracked files, use
--force to delete it"*. Both refusals are answers, not obstacles (§5): a leftover
that will not remove cleanly is one that still has a holder or still holds work.
Only `git worktree remove -f -f` overrides a lock, and it deletes the directory
with everything uncommitted in it — which, by §12's own argument, is exactly what
a live worker's output is. **`-f -f` is not on this runbook's command list and is
never run**; *"prune didn't clear it, so force it"* is §12's catastrophe reached
from the other side. A removal that fails instead on a Windows file lock is the
F-12 case §12.2 already covers: record the path and let run-completion `gc`
re-scan it.

---

## 11. Interruption and resume

A dispatch round is not atomic, and a session can end mid-flight. On the next
invocation the preflight snapshot exposes the wreckage before anything is
computed:

```
git worktree list                    # live slots
git branch --list worktree-*         # candidate dispatched branches
```

plus the board's own `🔵` rows.

**`worktree-*` matches two namespaces that share nothing but the prefix.**
`worktree-<task-id>` is cut per task (§4), and its suffix is a task id — one
that either carries a row on the board or does not. `worktree-agent-<hash>` is
cut by the host's own worker-isolation placement (§3.1) for **agent
isolation**: a different owner than this skill, a different lifetime than a
task slot, and it survives the agent that produced it. Both match the glob.
Only a branch whose suffix resolves to a task id with a board row is evidence
of a dispatch by *this run* — the durable rule is that a dispatch is a row plus
its slot, and any `worktree-*` branch without a matching row belongs to
somebody else's session, `worktree-agent-*` being the observed case and not the
only one the rule has to cover. Such a branch is read the same way a foreign
session's branch always is: not reconciliation evidence, and never an orphan of
this run's to reap — §9.3's rule ("never deletes a branch that is not its own
`worktree-*` slot") already covers it, and cleanup below (§12) creates no
exception to it.

**Reconcile all of it before any new dispatch.** Five cases:

| Evidence | Action |
|---|---|
| **1. A `🔵` row whose PR is merged** | Verify every DoD item against the repository, then record `✅` with the merged reference. The work is done; only the bookkeeping was interrupted |
| **2. A `🔵` row with an open PR** | **Adopt it.** Do not dispatch a second worker for that task — that is the duplicate dispatch §5 exists to prevent. Pick the row up at review or merge, wherever it actually is |
| **3. A `🔵` row with a branch but no PR** | **Run §12's reaping precondition before you form an opinion** — every repository the slot spans, branch commits *and* working-tree status, in that order. Where it finds work: resume the worker against the existing branch. Only where **every** repository the slot spans is both commitless and clean may the row reset to pending and the slot be reaped. Decide from the tree, not from the row — and the branch and the submodule slot are both part of the tree |
| **4. A live slot with no matching row** | A leak. `pipeline gc` reports it; `pipeline gc --clean` reaps it. A `⛔` row keeps its slot deliberately — that is not a leak, and reaping it destroys the post-mortem |
| **5. A `worktree-*` branch whose suffix matches no row** | **Not this run's.** Leave it — this is the ordinary shape of a `worktree-agent-*` isolation branch or of another session's task branch, not a leak and not something this reconciliation acts on |

Reconciliation is the reason the in-flight table (§5) is rebuilt from repository
evidence at the start of every invocation rather than trusted from the board
alone. The board records what was *dispatched*; the tree records what *happened*.

**The glob is weaker evidence than the slot registry.** `pipeline worktree list`
(§3.2) names only the slots this run provisioned through `pipeline worktree
create` — it has no way to report a branch it did not cut, host isolation
branches included. `git branch --list worktree-*` has no such filter and reports
every literal match. Where the two disagree about whether a branch is this
run's, **the registry wins.**

---

## 12. Cleanup

**The reaping precondition — nothing below reaps on emptiness. Read this first.**

Every destructive step in this section reaps a directory *and a branch*: §12.2's
`destroy --outcome completed`, §12.4's `pipeline gc --clean`, and any "reset the
row and reap the slot" decision reached through §11. Each of them is preceded by
the same three checks, **in this order** — **the branch is checked, in every
repository the slot spans, before any directory is reaped.**

**And every one of them reaps only a branch this run's own board already
accounts for.** §11's row-derivation — a `worktree-*` branch whose suffix
resolves to a task id with a board row — is what proves a branch is this run's
to reap, not the raw glob by itself. `pipeline gc [--clean]` and
`--force-worktree-branches` (§12.4) act on whatever locally matches
`worktree-*`; a literal match is not, on its own, a licence to delete — a
`worktree-agent-*` host-isolation branch or another session's
`worktree-<other-task-id>` branch matches the same pattern and belongs to
neither this run nor this task. The rule §9.3 states for `on-green` ("never
deletes a branch that is not its own `worktree-*` slot") applies here exactly
the same way; this section adds no cleanup-specific exception to it.

**1. A submodule task's work is not in the parent slot, and never was.** A task
whose `repo:` frontmatter names a submodule works in the **submodule** slot, on
that submodule's own integration branch (§8.2). The parent superproject slot is
empty, clean, and has uninitialized submodules **by design** — that is what a
correctly provisioned slot for such a task looks like. `worktree_path` is the
parent; **`submodule_slots[].dir` is where the work is** (§3.2). An empty parent
slot is the *expected* state and **proves nothing whatever** about whether the
task produced anything.

**2. Ask git before you judge a directory — in every repository the slot spans.**
Enumerate them first from `pipeline worktree list --json`: the parent
`worktree_path`, plus every `submodule_slots[].dir`. Then, in each:

```
git -C <dir> branch --list worktree-<task-id>            # does the branch exist
git -C <dir> log --oneline <base>..worktree-<task-id>    # does it carry commits
git -C <dir> status --porcelain                          # is there uncommitted work
```

**All three, in every one of them.** `<base>` is that repository's own
integration branch, not the superproject's — a submodule slot is cut from the
submodule's branch, and asking the superproject's question of it returns a
confidently wrong answer. Any non-empty output, from any of the three, in any of
those repositories, means the slot holds work.

**3. Only then judge what a directory contains** — and judge it at
`submodule_slots[].dir`, never at `worktree_path` alone.

**Why the third command is not optional.** A slot that needs reconciliation at
all is usually one whose worker was *killed*, and a killed worker's output is by
definition uncommitted. "The branch carries zero commits" is the expected answer
in that case and is **not** a finding of emptiness. Commits survive a removed
worktree; uncommitted work does not, and `--outcome completed` and `--clean`
delete the branch as well as the directory. There is nothing to recover from
afterwards.

**This ordering is the rule that was missing, and its absence is the most
expensive defect this design has recorded.** A resumed run in this design's own
proving run inspected two **parent** slots, ran its branch check in the
**superproject** only, and concluded — verbatim — *"Both abandoned branches carry
zero commits and both worktrees are clean — the killed workers produced
nothing,"* and *"Both slots are empty shells … the target code was never even
present in them."* Every clause was true of the directory it looked at and false
about the work. The work was uncommitted, in the two **submodule** slots, on
same-named branches in different repositories. All four worktrees were reaped;
**21,880 bytes of finished implementation were destroyed**, and the run reported
that it had never existed.

`a11` fixed the CLI half — `submodule_slots` is populated on the hook path now,
so `list --json` names those directories. But a tool can only answer the question
it is asked, and the question asked was about the wrong directory in the wrong
repository. This precondition is the question being asked correctly.

### 12.1 The wave-boundary audit — every round, not only on interruption

Run §11's reconciliation — `git worktree list`, `git branch --list worktree-*`,
the board's `🔵` rows — **after every round**, not only when a session resumes
from an interruption. This is invariant 5 (`04-subsystem-rules.md` §9), stated
there as a property; here it is the instruction to *check* it: every `🔵` row
has a live slot or an open PR, every live slot has a `🔵` **or `⛔`** row, and a
slot with no row at all is a leak.

**This is the audit that caught F-12, and nothing else would have.** A leaked
local branch does not fail a task, does not fail CI, and does not block a merge
— it is invisible to every other check this runbook runs. The same is true of a
slot directory that survived its own removal behind a file lock: the removal
already reported failure and the round already moved on, correctly, and nothing
came back to look. Only a deliberate round-boundary comparison of slots against
branches against rows surfaces either one before it compounds. A run that only
reconciles on interruption finds this after nine merges instead of after one.

**`git worktree list` in the superproject does not show a submodule slot.** Take
those from `pipeline worktree list --json`'s `submodule_slots[].dir` (§3.2), or
the audit reconciles only the half of each submodule task that never held the
work.

The audit itself changes nothing — it is a read, not a `--clean`. Flag what it
finds in the round report; §12.2's per-task cleanup and §12.4's run-completion
`pipeline gc` are what act on it — and each of those first runs the reaping
precondition above.

### 12.2 Per task, on success

```
pipeline worktree finalize --name <task-id> [--json]      # where a terminal hook exists
pipeline worktree destroy  --name <task-id> --outcome completed [--json]
```

- **`destroy --outcome completed` reaps**: `PIPELINE_WT_DELETE_BRANCHES=1` — the
  worktree, every submodule worktree it provisioned, and the local
  `worktree-<task-id>` branch **in each of those repositories**, not only the
  slot directory — and the slot record is dropped.
- **`finalize` is strict must-succeed** — only an explicit `{"ok":true}` from the
  consumer's terminal hook passes, and a **missing** hook fails too. Call it only
  where the project defines one; a failure is reported and the slot preserved,
  never swallowed.
- **In the `native` tier there is nothing to destroy, and that is exactly where
  the local branch leaks.** The host locks the slot for the agent's lifetime and
  sweeps the worktree afterwards, but the sweep removes the working directory,
  not the branch: `worktree-<task-id>` stays a real local branch of the main
  checkout's repository after the sweep, merged or not, remote-deleted or not —
  `gh pr merge --delete-branch` only ever reaches the *remote* copy. Nothing in
  this subsection reaps that local branch; §12.4's `pipeline gc [--clean]` at run
  completion is the step that does (F-12).
- **A removal that fails is not final.** The most common cause on Windows is a
  file lock held at the moment of removal — `destroy` (or the host's sweep)
  reports the failure and the round continues, correctly; forcing it inline is
  not this subsection's job. Record the path and move on. `pipeline gc [--clean]`
  at run completion re-scans the same ground — including the built-in slot root
  outside the repository, which it now covers (`a8`) — and reaps what it finds
  there through the same teardown, by which point a transient lock has usually
  cleared. Report anything that still survives after that as a residual for the
  run report, not as something to keep retrying by hand mid-run.

### 12.3 Per task, on failure — the worktree is kept

**A failed task keeps its worktree.** `⛔` rows, rows that exhausted the review
fix loop, rows with a merge conflict: the slot stays, and **its path is recorded
on the board row and in the report**. This is the whole point of preserving it —
somebody is going to look.

Where a slot must be explicitly preserved through the CLI, that is
`--outcome halted` (`PIPELINE_WT_DELETE_BRANCHES=0`, record kept). The one
inversion to remember is the failed *create* in §10: a slot with nothing in it is
reaped with `completed`, because there is nothing to preserve.

`pipeline worktree list [--json]` enumerates the slots this command group
provisioned and whether each is still on disk — use it to build the "what was
preserved and why" section of the report.

### 12.4 At run completion

When every scoped row is verified complete: update the taskflow README status and
the ROADMAP counter, remove any thin pointer created for this run, then

```
pipeline gc [--project <path>] [--json] [--no-submodules]
```

and **report what it found** — registered worktrees and their merged state, stale
unregistered directories, prunable worktree records, and orphaned `worktree-*`
branches, per initialized submodule as well as in the superproject. Reporting is
the deliverable; a `gc` whose output nobody reads has collected nothing.

- **`--clean`** prunes records, removes fully-merged worktrees, deletes stale
  directories and safe-deletes merged branches (`git branch -d`, never `-D`) —
  per submodule too, and, since `a8`, across the built-in slot root outside the
  repository as well as `.claude/worktrees`. **This is the step that closes
  F-12**: it is the only point in the run that reaps a `native`-tier local
  branch §12.2 could not touch, and the only point that retries a slot directory
  an earlier `destroy` failed to remove. Run it — do not treat it as merely
  optional housekeeping — but never while a `⛔` row's slot is still wanted for
  inspection, and never before the reaping precondition at the head of this
  section has been satisfied for every row whose slot it would touch. `--clean`
  deletes branches, and this is the last point at which that is reversible.
- **A safe `-d` delete does not reap a squash-merged branch.** Git never sees the
  original commits as ancestors of a squash commit, so it reads as unmerged
  forever. If task PRs in a given repository squash-merge, plain `--clean`
  leaves those local branches behind even after this step; only
  **`--force-worktree-branches`** (below) reaches them.
- **`--force-worktree-branches`** (requires `--clean`) additionally hard-deletes
  **unmerged** `worktree-*` branches — squash-merged branches read as unmerged
  forever, which is the case it exists for. It destroys unmerged work by
  definition, so it is **never run unattended**: it requires an explicit owner
  decision. The owner making that decision needs to know it works from the same
  literal `worktree-*` match §12's reaping precondition warns about: nothing in
  the flag distinguishes this run's own unmerged task branches from a
  `worktree-agent-*` host-isolation branch or another session's unmerged task
  branch. Confirm against the board (§11) before naming what it will touch, not
  after.

Finish by reporting verified results, withheld tasks and why (§2.1), preserved
worktrees and why (§12.3), any bump that reported `merged_via_admin: true` —
as the §9.4 defect it is, not as a routine line — and every outstanding gate.

---

## 13. Every command this runbook names

Each was verified against the CLI's own `--help` / usage output rather than
assumed. A command or flag that is not in this table does not belong in a brief
either.

| Command | Flags named here |
|---|---|
| `pipeline --version` | — (bare invocation; tier resolution, §1.1) |
| `pipeline id` | — (mints the UUIDv7 used as a run id in the `pipeline` tier) |
| `pipeline worktree create` | `--name` `--base` `--submodules` `--hook-dir` `--ports` `--json` |
| `pipeline worktree finalize` | `--name` `--base` `--submodules` `--hook-dir` `--json` |
| `pipeline worktree destroy` | `--name` `--outcome completed\|halted` `--hook-dir` `--json` |
| `pipeline worktree list` | `--json` |
| `pipeline ci-wait` | `--pr` `--repo` `--timeout` `--interval` `--grace` `--fail-fast` / `--no-fail-fast` `--json` `--verbose` |
| `pipeline gc` | `--project` `--clean` `--json` `--no-submodules` `--force-worktree-branches` |
| `pipeline drive` | `--root` `--run-id` `--start` `--effort <step_id>=<level>` `--json` |
| `pipeline submodule bump` | `--no-admin`, on every invocation (§9.4). The remaining flags are owned by `references/submodules.md` §6 |
| `git worktree list`, `git worktree list --porcelain`, `git worktree prune`, `git worktree prune --dry-run --verbose`, `git worktree unlock <path>`, `git worktree remove <path>` — **host leftovers only** (§10.1), never a `pipeline`-provisioned slot, `git branch --list worktree-*`, `git log --oneline <base>..<branch>`, `git status --porcelain`, `git diff --name-only <base>...HEAD`, `git merge-base`, `git submodule status`, `git submodule update --init`, `git -C <checkout> pull --ff-only` | git |
| `gh api repos/…/branches/…/protection`, `gh api repos/…/rulesets`, `gh pr merge` | GitHub CLI |
| `Get-Process -Id <n>` (Windows) · `ps -p <n>` (elsewhere) | the operating system — the live-lock check, §10.1, and nothing else |

**Named only to forbid them:** `--force` on slot creation (the CLI exposes none
on any `worktree` verb), `git worktree add` in **any** form as a substitute for
`pipeline worktree create` (§3.4) — `--force` least of all —
`git worktree remove -f -f` against a locked worktree, which is the one command
that overrides a live lock and destroys the worker behind it (§10.1), and
`gh pr merge --admin` or any other bypass flag. The orchestrator runs none of
these, and no brief asks a worker to. Likewise, no branch namespace other than
`worktree-*` appears in this system at all.
