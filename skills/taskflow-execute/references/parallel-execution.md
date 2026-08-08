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
| `worktree_path` | absolute path to the provisioned directory |
| `branch` | the branch the hook cut, `worktree-<name>` by contract |
| `env_file` | absolute path to the dotenv file — the input to §6 |
| `reused` / `reused_evidence` | `true` plus `registry` or `git-worktree` when the slot already existed |
| `base_branch`, `submodules`, `hook_dir` | echoed back as resolved |
| `detail` | the failure reason when `status` is `failed` |

Exit **0** success · **1** the provisioning hook failed (soft-fail or hard-fail;
`detail` says which) · **2** usage, an invalid `--name`, or an invalid
`--outcome`.

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

1. **`create` runs the consumer's `worktree-create.*` hook**, resolved from
   `<project>/.pipeline/.hooks` (override with `--hook-dir <path>`). `--base` and
   `--submodules` reach that hook as `PIPELINE_WT_BASE_BRANCH` and
   `PIPELINE_WT_SUBMODULES` — the hook is what cuts the branch. **A project with
   no such hook gets a failed create (exit 1), not a provisioned slot.** A
   built-in default provisioner is planned but is not in the version this
   document was verified against; until it ships, `toolkit` in a hookless project
   behaves as `native` with an announced create failure per attempt, so withhold
   rather than retry.
2. **`--ports <n>` is recorded, not allocated,** in that same version — the
   `--json` `ports` field is always `null` and `ports_requested` echoes the
   request. Ports therefore come from the consumer hook today. §6 states the
   precedence that governs both worlds, so nothing changes in this document when
   the allocator lands.

### 3.3 `pipeline` — the run provisions itself

The orchestrator launches one run per task:

```
pipeline drive --root <pipeline_root> --run-id <id> --start <step-name> \
               [--effort code-review=<depth>] --json
```

The run creates its own slot, and **F4 through F6 are skipped entirely** — no
worker brief from §7, no orchestrator-side merge from §9. Mint the run id with
`pipeline id`; never invent one. `--merge` does not apply (§1.3).

---

## 4. Slot naming

| Item | Form | Why |
|---|---|---|
| **Slot name** | the task id, **validated against `tasks/` first** | The id reaches a filesystem path, a branch name, and a user-authored hook's environment. Validating that the id names a real file under `tasks/` before it reaches any of those is gate SG6 |
| **Branch** | the substrate's own — `worktree-<name>` | Not invented by this skill. It is the namespace `pipeline gc` already scans and reaps, and the namespace the preflight's `git branch --list worktree-*` already reports |
| **Location** | `native`: `.claude/worktrees/`, host-managed. `toolkit`: the substrate's own root, outside the repository | A worker's build artifacts never land in the project folder |

**The skill invents no branch namespace of its own.** `worktree-<name>` is the
substrate's, and it is the only one that appears anywhere in this system — in the
preflight snapshot, in reconciliation (§11), and in `gc`'s reaping (§12). Do not
create branches under any other prefix and do not look for them under one.

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
request, and its land step retries a refused merge with `gh pr merge --admin` by
default**, reporting `merged_via_admin: true` when it did. That is the command's
internal behaviour, not the orchestrator's policy — but a run that invokes it
unmodified has elevated, whatever the policy says.

Therefore:

- **The orchestrator must suppress that elevation** by passing the CLI's
  non-elevating option on every `submodule bump` invocation. That option is being
  added by a companion CLI task and **is not present in the CLI version this
  document was verified against**, which is why no flag spelling is quoted here —
  read the command's own usage output, or `docs/cli.md`, for the current
  spelling before wiring it in.
- **Until it is available**, either accept that this one command may elevate and
  **report `merged_via_admin: true` prominently in the round report**, or skip
  the guarded bump and do the pointer bump by hand. Do not leave an elevation
  unreported on the theory that it was the CLI's decision.
- The bump procedure itself — when it runs, which submodules it names, how its
  `skipped[]` entries are read — belongs to `references/submodules.md` §6 and is
  not repeated here.

This is the only exception. Task PRs never elevate, under any flag, in any tier.

---

## 10. Failure paths

Every response is defined, announced, and non-forcing. Exactly one halts the run.

| Failure | Response |
|---|---|
| **Dirty main checkout at preflight** | Stop before any dispatch. Report the dirty paths. No stash, no clean |
| **Slot creation fails** | The row stays pending. Reap the partial slot with `pipeline worktree destroy --name <task-id> --outcome completed` — `completed` reaps, and `halted` would *preserve*, which is wrong for a slot with nothing in it. **Do not retry blindly** |
| **`create` reports `reused`** | Duplicate dispatch (§5). Do not proceed with the worker. Reconcile via §11 |
| **`gh` absent or unauthenticated** | There is no PR path. Workers push branches only, rows stop at `🟣`, and the run says so **once** — not per task |
| **Base-branch fast-forward fails** | Do not force. Report and continue the round; the next round retries |
| **Pointer bump or parent push rejected** | Report. Leave pointers unbumped. No force-push, no elevation flag added by the orchestrator |
| **Port allocation exhausted** | Withhold the task exactly as in the `native` tier (§2.1), and state the range that was tried |
| **Reviewer subagent fails** | Treat as a blocking finding for that round. A second failure escalates the row to `⛔` rather than merging unreviewed |
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

---

## 11. Interruption and resume

A dispatch round is not atomic, and a session can end mid-flight. On the next
invocation the preflight snapshot exposes the wreckage before anything is
computed:

```
git worktree list                    # live slots
git branch --list worktree-*         # dispatched branches
```

plus the board's own `🔵` rows. **Reconcile all of it before any new dispatch.**
Four cases:

| Evidence | Action |
|---|---|
| **1. A `🔵` row whose PR is merged** | Verify every DoD item against the repository, then record `✅` with the merged reference. The work is done; only the bookkeeping was interrupted |
| **2. A `🔵` row with an open PR** | **Adopt it.** Do not dispatch a second worker for that task — that is the duplicate dispatch §5 exists to prevent. Pick the row up at review or merge, wherever it actually is |
| **3. A `🔵` row with a branch but no PR** | Inspect the worktree. Either resume the worker against the existing branch, or reset the row to pending and reap the slot. Decide from the tree, not from the row |
| **4. A live slot with no matching row** | A leak. `pipeline gc` reports it; `pipeline gc --clean` reaps it. A `⛔` row keeps its slot deliberately — that is not a leak, and reaping it destroys the post-mortem |

Reconciliation is the reason the in-flight table (§5) is rebuilt from repository
evidence at the start of every invocation rather than trusted from the board
alone. The board records what was *dispatched*; the tree records what *happened*.

---

## 12. Cleanup

### 12.1 Per task, on success

```
pipeline worktree finalize --name <task-id> [--json]      # where a terminal hook exists
pipeline worktree destroy  --name <task-id> --outcome completed [--json]
```

- **`destroy --outcome completed` reaps**: `PIPELINE_WT_DELETE_BRANCHES=1`, and
  the slot record is dropped.
- **`finalize` is strict must-succeed** — only an explicit `{"ok":true}` from the
  consumer's terminal hook passes, and a **missing** hook fails too. Call it only
  where the project defines one; a failure is reported and the slot preserved,
  never swallowed.
- In the `native` tier there is nothing to destroy: the host locks the slot for
  the agent's lifetime and sweeps it afterwards.

### 12.2 Per task, on failure — the worktree is kept

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

### 12.3 At run completion

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
  directories and safe-deletes merged branches (`git branch -d`, never `-D`).
  Run it as a decision, not reflexively, and never while a `⛔` row's slot is
  still wanted for inspection.
- **`--force-worktree-branches`** (requires `--clean`) additionally hard-deletes
  **unmerged** `worktree-*` branches — squash-merged branches read as unmerged
  forever, which is the case it exists for. It destroys unmerged work by
  definition, so it is **never run unattended**: it requires an explicit owner
  decision.

Finish by reporting verified results, withheld tasks and why (§2.1), preserved
worktrees and why (§12.2), any `merged_via_admin` from §9.4, and every
outstanding gate.

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
| `pipeline submodule bump` | flags owned by `references/submodules.md` §6; the non-elevating option is deliberately unquoted here (§9.4) |
| `git worktree list`, `git branch --list worktree-*`, `git diff --name-only <base>...HEAD`, `git merge-base`, `git submodule status`, `git -C <checkout> pull --ff-only` | git |
| `gh api repos/…/branches/…/protection`, `gh api repos/…/rulesets`, `gh pr merge` | GitHub CLI |

**Named only to forbid them:** `--force` on slot creation (the CLI exposes none
on any `worktree` verb), `git worktree add --force`, and `gh pr merge --admin` or
any other bypass flag. The orchestrator runs none of these, and no brief asks a
worker to. Likewise, no branch namespace other than `worktree-*` appears in this
system at all.
