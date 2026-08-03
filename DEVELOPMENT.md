# CLAUDE.md — Taskflow plugin

Guidance for Claude Code when editing this plugin, not for consumer projects.

## What this is

A four-skill Taskflow lifecycle: `/taskflow-frame` → `/taskflow-review` →
`/taskflow-tasks` → `/taskflow-execute`. Skill folders match their public names.

## Invariants

1. **All four skills share one contract.** The default artifact root is
   `.claude/taskflow/`, with one `<slug>/` folder per taskflow. `ROADMAP.md` is
   the only mutable implementation-state record. Its rows use:
   `Task (spec) | needs | imp/cx | model | Status | Run / PR | Updated`.
2. **Lifecycle roles stay distinct.** Frame establishes repository evidence and
   architectural decisions; review verifies them adversarially; tasks writes
   immutable specs and waves; execute is the ROADMAP status-board orchestrator.
3. **No model is pinned.** The session model is the owner's choice. `top`,
   `mid`, and `fast` in the board map to consumer-project-approved models.
4. **Task specs are immutable.** Never add a `status` field to a task spec.
   Only the executing orchestrator changes board rows or the progress log, and
   only after ground-truth verification.
5. **Model routing stays consistent.** Complexity ≥8 → top; 5–7 → mid; ≤4 →
   fast; `security_critical` or `production_touching` raises one tier.
6. **Use public bare names in cross-references.** Every Claude skill has quoted
   YAML `name`, `description`, and `argument-hint`, plus
   `disable-model-invocation: true`, so owners invoke the lifecycle manually.
7. **Remain tool-agnostic.** Prefer a consumer project's normal forge, CI, and
   execution workflow when available; otherwise use isolated worktree workers
   and locally verifiable evidence. Never assume a particular vendor or tool.
8. **Product decisions are owner-owned.** Deployment target, compatibility,
   identity, UX, monetization, scope, production changes, money, secrets, and
   irreversible actions require an explicit owner decision or GO. Present a
   safe recommended option first and record the decision.
9. **Legacy folders are archival.** Do not read, migrate, or fall back to
   legacy workflow artifacts; only `.claude/taskflow/<slug>/` is active.

## Releasing

Bump `.claude-plugin/plugin.json` and `codex/taskflow/.codex-plugin/plugin.json`
together, validate both packages, then commit and release through the relevant
marketplaces.
