---
name: "taskflow-tasks"
description: "Decompose a reviewed Taskflow folder into immutable implementation-ready task specifications, conflict-safe execution groups, dependencies, routing tiers, and ROADMAP waves."
argument-hint: "[<taskflow slug> — default: the single taskflow under .claude/taskflow/]"
---

# taskflow-tasks — immutable specifications and execution waves

## Select the taskflow

- Default root: `.claude/taskflow/`; work in one `<slug>/` folder.
- Honor a supplied slug; use the only sub-folder when unambiguous,
  otherwise ask the owner. Do not inspect or migrate legacy workflow artifacts.

## Precondition

The frame is locked, reviewed, and has no unresolved product question or
unapplied finding. Stop and surface either condition rather than guess.

## Outputs

Create `<slug>/tasks/README.md` with the coefficient legend, model rubric,
group table, and a pointer to `../ROADMAP.md`. The ROADMAP owns live waves and
state; these files own static specifications.

Create one immutable file per task: `<slug>/tasks/<id>-<slug>.md`.

```markdown
---
id: "b3-pairing-plane"
title: "One-line task title"
group: "B"
sequence: 3
repo: "repo-or-submodule-path"
depends_on: ["a2-library-gate"]
importance: 1
complexity: 1
security_critical: false
production_touching: false
model_hint: "fast"
taskflow_refs: ["02-target-architecture.md", "04-protocol.md"]
---

## Goal

## Scope & seams

## Definition of Done
```

`sequence` is strictly increasing within its group. **Never add `status`.**
Task files are immutable; all live state exists only in `ROADMAP.md`.

## Update ROADMAP.md

Populate execution waves and one row per task:

```text
| Task (spec) | needs | imp/cx | model | Status | Run / PR | Updated |
| [b3-pairing-plane](tasks/b3-pairing-plane.md) | a2 | 9/7 | top | ⬜ pending | | |
```

Record these rules beside the board:

1. The board is the only mutable task-state record; the ready set is computed
   from `needs` plus completed rows, never stored separately.
2. Only `/taskflow-execute` changes board rows or the progress log after
   ground-truth verification; implementers report but do not edit specs or the
   board.
3. A workspace-wide planning system, if one exists, gets one thin pointer to
   this ROADMAP, never a duplicate record per task.

Add human approval gates for production, money, secrets, and irreversible
effects. Update the taskflow README to say tasks were specified and make a
surgical commit limited to the taskflow folder.

## Coefficients and routing

- **Importance (1–10):** consequence of an incorrect or missing task; use it
  to order otherwise-ready work and communicate risk.
- **Complexity (1–10):** architectural depth, cross-cutting surface, risk, and
  correctness cliffs.
- Complexity ≥8 → `top`; 5–7 → `mid`; ≤4 → `fast`. Either
  `security_critical` or `production_touching` raises one tier, with `top`
  remaining `top`. Tiers map to consumer-project-approved models.

## Ownership and parallelism

1. A group is one merge-conflict domain. Execute its tasks strictly in ascending
   `sequence`. If file overlap is uncertain, put both tasks in the same group.
2. Different groups may run in parallel only when `depends_on` allows it;
   separate repositories are separate groups by default.
3. Express cross-group ordering with `depends_on`. End artifact-producing groups
   with an explicit integration/publish gate for downstream consumers.
4. Draw the resulting waves in ROADMAP. One task should be one reviewable,
   PR-able unit; split oversized work and merge trivial same-file work.

Any user or agent may invoke `/taskflow-execute`; dispatch still begins only
after the explicit owner GO required by its approval gates.
