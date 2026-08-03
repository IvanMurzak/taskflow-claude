---
name: "taskflow-review"
description: "Adversarially verify an existing Taskflow folder against repository code, authoritative external specifications, and internal consistency; then apply confirmed non-product corrections in one coherent batch."
argument-hint: "[<taskflow slug> — default: the single taskflow under .claude/taskflow/]"
disable-model-invocation: true
---

# taskflow-review — adversarial verification

## Select the taskflow

- Default root: `.claude/taskflow/`; review one `<slug>/` sub-folder.
- Honor a supplied slug. If the root has exactly one sub-folder, use it;
  otherwise ask the owner to select one. Confirm the resolved folder before work.
- Read the whole folder—README, ROADMAP, numbered documents, and `tasks/` if it
  exists. Never use legacy workflow artifacts as input or fallback.

## Principle

Reviewers try to disprove the taskflow. Apply a finding only after verifying it
against evidence; plausible but false corrections are harmful.

## Process

1. Run three independent reviews in parallel.

   - **Repository truth:** check every factual claim against source with
     `file:line` evidence, including feasibility and cross-language/byte-level
     parity claims. Report findings and a separate verified-correct inventory.
   - **External conformance:** use available current research to check
     authoritative specifications, standards, vendor behavior, and relevant
     SDK source. Classify MUST/SHOULD deviations with citations.
   - **Internal consistency:** check cross-document decisions, owner-requirement
     traceability, migration and ROADMAP alignment, real dependency edges,
     threat coverage, UX budgets, stale text, and the rule that ROADMAP is the
     sole task-state record.

   Use P0 for broken guarantees/material falsehoods, P1 for gaps or risks, and
   P2 for wording or stale content.
2. Wait for every review, consolidate duplicate findings, and verify the
   proposed correction. Apply factual and mechanical corrections together.
3. If evidence affects a product decision—deployment, compatibility, identity,
   UX, monetization, scope, production behavior, money, secrets, or an
   irreversible action—do not choose for the owner. Present evidence and a safe
   recommended option, then record the confirmed choice as `REVISED` with date
   and rationale in the decision ledger.
4. Edit all affected documents in one coherent batch. Sweep the taskflow folder
   for every replaced term, parameter, number, and storage location. Update the
   README status and ROADMAP progress log.
5. Make a surgical taskflow-folder commit. Report decision revisions first,
   then confirmed corrections, then the verified-correct inventory.

## Review checks

- `ROADMAP.md` has the only task status board and only the executing
  orchestrator edits it after repository/CI evidence.
- Task specs have no `status`; when present they include `sequence`,
  `security_critical`, and `production_touching`.
- Timeline, gates, dependencies, migration, and task specs agree.
- The workflow uses available forge/CI tooling where present and a local,
  isolated-worktree evidence path where it is absent.

Do not start decomposition automatically; the next manual stage is
`/taskflow-tasks`.
