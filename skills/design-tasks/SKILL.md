---
name: design-tasks
description: Decompose a polished, reviewed design-doc folder into an implementable task breakdown - a tasks/ subfolder with one file per task carrying importance and complexity coefficients (used to pick the implementing AI model) and merge-conflict-safe parallelism groups - then populate the design's ROADMAP with the execution waves and status board. Operates on a per-design sub-folder under .claude/design/ (or a caller-specified path). Use when the user asks to "write down the tasks", "decompose the design", or "plan the implementation" of a finished design.
argument-hint: [<design slug or path> — default: the single design under .claude/design/]
---

# design-tasks — decompose a design into a model-routable, parallel-safe task breakdown

**Model requirement:** MUST run on the Fable-tier model (the decomposition IS architecture). If
the session model is lower, STOP and ask the user to switch (`/model fable`).

## Which design folder

- **Default root:** `.claude/design/`. Decompose the per-design **sub-folder** `<root>/<slug>/`.
- **Caller override:** honor an explicitly named path/slug; if the root holds one design, use it,
  else ask which. Confirm the resolved path.

## Precondition

The design must be LOCKED and reviewed (ideally via `/design-review`) with no open questions.
If open questions or unapplied findings remain — STOP and surface them instead of decomposing.

## Output

### `<slug>/tasks/README.md` (the index)
- Coefficient legend, the model-selection rubric, the group table.
- A pointer to `../ROADMAP.md` for the live waves + status board (do NOT duplicate the timeline
  here — ROADMAP owns it; these files own the static specs).

### One file per task: `<slug>/tasks/<group><n>-<slug>.md`

```markdown
---
id: b3-pairing-plane
title: <one line>
group: B                    # merge-conflict domain (see below)
repo: <repo / submodule path>
depends_on: [<task ids>]    # cross-group edges; within-group order is the listed sequence
importance: 1-10            # product criticality (golden path / security / revenue)
complexity: 1-10            # engineering difficulty (depth, surface, risk, unfamiliar tech)
security_critical: bool     # touches auth/credentials/token issuance/production
model_hint: top|mid|fast    # computed from the rubric
design_refs: [02, 04]       # which design docs specify this task
---
## Goal        — one or two sentences
## Scope & seams — concrete files/classes/endpoints from the design's seam index
## Definition of Done — checkable boxes (tests, gates, published artifacts)
```

**NO `status` field — ever.** Task files are **immutable specs**; they are never edited during
implementation. ALL live state (status, run id, PR, dates) lives in exactly ONE place: the
design's ROADMAP status board. A status field in task frontmatter is a bug.

### Update `<slug>/ROADMAP.md` (REQUIRED — the single source of task state)
Populate the ROADMAP the design skill created: fill the **execution waves** from the groups +
dependency edges, and the **status board** — the ONLY mutable record of task state. One row per
task, carrying everything an orchestrator needs without opening the spec files:

```
| Task (spec) | needs | imp/cx | model | Status | Run / PR | Updated |
| [<id>](tasks/<id>.md) | <short dep ids> | 9/7 | top | ⬜ pending | | |
```

Also record in ROADMAP the three status rules: **(1)** the board is the only place task state
exists (no copies in task files, plan store, or other docs); **(2)** single writer — only the
orchestrating agent flips rows + appends the progress log, after ground-truth verification
(merged PR, green CI); implementers report, they don't edit; **(3)** if the consumer project
keeps a global plan store, it gets ONE thin pointer task for the whole design, never one file
per task. The ready set is *computed* from `needs` + ✅, not stored. Add/confirm the
**human-approval gates** for any production/money/secret/irreversible task. If ROADMAP is
missing (older design), create it here per the `/design-system` §4 shape — every design must
end with a populated ROADMAP.

## The two coefficients (be honest, not flattering)

- **importance (1–10):** what breaks if this task is wrong or missing. 10 = the feature fails or
  security is compromised. Docs/comms rarely exceed 6.
- **complexity (1–10):** architectural depth + cross-cutting surface + risk. 8+ means the task
  redesigns behavior across components or has subtle correctness cliffs (crypto, routing,
  concurrency, cross-language parity).

## Model-selection rubric (record `model_hint` per task)

| Rule | Tier |
|---|---|
| complexity ≥ 8 | **top** (Fable-tier) |
| complexity 5–7 | **mid** (Sonnet-tier) |
| complexity ≤ 4 | **fast** (Haiku-tier) |
| `security_critical` or production-touching | bump one tier up |

## Grouping — the merge-conflict rule (this is the core of the skill)

1. A **group = one merge-conflict domain**: one repo (or one disjoint area of a repo). Tasks in
   the same group run **strictly sequentially in listed order** — they touch overlapping files,
   and concurrent runs would produce theoretical merge conflicts. When in doubt whether two
   tasks' file sets overlap (shared constants, generated files, csproj/lockfiles), put them in
   the SAME group.
2. **Different groups run in parallel**, subject to `depends_on`. Separate repos are separate
   groups by construction (e.g., three engine repos = three parallel groups).
3. Express cross-group ordering ONLY via `depends_on` (e.g., consumers depend on the library's
   gate task). Distinguish "dev can start" from "release gate" dependencies explicitly when they
   differ.
4. End each group that ships a consumable artifact with an explicit **gate task** (integration
   test/publish) that downstream groups depend on.
5. Draw the resulting **waves** in ROADMAP: which groups run simultaneously at each stage.

## Task sizing

One task = one PR-able unit dispatched through the project's implementation pipeline (e.g. an
`implement-task` workflow): big enough to be worth a worktree + CI cycle, small enough to
review. Split a task that would exceed ~1 PR; merge tasks that would produce trivial PRs in the
same files (they'd conflict anyway).

## Finish

Update the design README (doc map + status → "tasks designed <date>"), ensure ROADMAP's waves +
status board are populated, surgical commit (design sub-folder only), and report: task count,
waves, and the importance/complexity spread. Execution then starts via `/design-implement`
(same sub-folder) — on explicit owner GO, never automatically.
