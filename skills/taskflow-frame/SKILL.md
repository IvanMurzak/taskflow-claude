---
name: "taskflow-frame"
description: "Frame a system or feature change from verified repository evidence and owner decisions, then write a self-contained Taskflow architecture set and ROADMAP. Use before a structural feature or architectural change, not for a one-off bug fix."
argument-hint: "<what to frame> [taskflow slug — writes .taskflow/YYYY-MM-DD-<slug>]"
---

# taskflow-frame — establish the architecture frame

## Artifact location

- Create and use exactly one `.taskflow/YYYY-MM-DD-<kebab-slug>/` folder for
  this taskflow. Prefix the caller-provided kebab-case slug with the local
  calendar date on which the folder is created; do not rename existing folders.
  A caller may provide the slug but cannot redirect artifacts outside
  `.taskflow/`. If the folder exists, read it before extending it; never
  clobber it.
- Do not read, migrate, or fall back to legacy workflow artifacts. They are
  archival and outside this workflow.

## Non-negotiables

1. **Evidence first.** Every factual statement about existing code must have a
   `file:line` reference found in this session.
2. **Owner decides product policy; you decide mechanisms.** Ask about only
   choices that change the outcome: deployment target, compatibility, identity,
   UX golden path, monetization, scope, or irreversible behavior. Offer a safe
   recommended option first and record every answer.
3. **Artifacts are the deliverable.** Write the taskflow folder and make a
   surgical commit limited to it; do not modify unrelated files in a shared
   checkout.

## Process

1. Resolve the kebab-case `<slug>`, prefix it with today's local date as
   `YYYY-MM-DD-<slug>`, and state the path. Extract the problem; ask 2–4 focused
   owner questions while beginning exploration.
2. Explore each affected repository or subsystem independently. Require exact
   files, line numbers, existing behavior, change seams, risks, and a concise
   evidence summary. Add exploration when an owner answer exposes another seam.
3. Write this minimum set in `<slug>/`:

   | File | Required content |
   |---|---|
   | `README.md` | Status, problem, locked decisions table, summary, document map, glossary. |
   | `ROADMAP.md` | This taskflow's implementation ledger: status, timeline/waves, gates, status board skeleton, and progress log. |
   | `01-current-architecture.md` | Evidence-only current behavior, edge cases, and seam index with `file:line` references. |
   | `02-target-architecture.md` | Principles, models/roles, decisions ledger D1..Dn, trade-offs, and owner-facing open questions. |
   | `03-*.md` | End-to-end actor flows, including failure, expiry, and offline paths where relevant. |
   | `04-*.md` | Subsystem rules, data structures, precedence, and test approach. |
   | `05-infrastructure.md` | Deployment/secrets delta and rollback, when applicable. |
   | `06-migration-rollout.md` | Phases, dependencies, gates, legacy disposition, risks, and rollback. |
   | `07-security.md` | Credential inventory, threat-to-control table, handling rules, and security gates. |
   | `08-user-workflows.md` | Persona journeys with counted step/click budgets as release gates. |

4. Put this banner in `ROADMAP.md`: it is this taskflow's implementation ledger,
   not a workspace-wide product roadmap. Include design status, implementation
   status, last-updated date, execution timeline, human-approval gates, progress
   log, and the board schema:

   ```text
   | Task (spec) | needs | imp/cx | model | Status | Run / PR | Updated |
   ```

   At this stage the board may be a skeleton. It becomes populated only in
   `/taskflow-tasks`. The board is the only task-state record; the executing
   orchestrator alone later updates it after verification.
5. Record every owner answer as D1, D2, and so on in both the README table and
   target-architecture ledger. Amendments are `REVISED` with date and reason,
   never silent rewrites.
6. On later refinements, update the relevant artifacts, sweep for stale terms,
   append the decision history, and commit only the taskflow folder.

## Quality bar

- Every requirement maps to a mechanism, and every mechanism to a requirement.
- Migration has dependencies, gates, rollback, and user impact.
- New mechanisms appear in the threat table; journeys have honest counted UX
  budgets.
- No unresolved placeholder lacks a specific owner question.

When stable, direct the owner to `/taskflow-review`, then `/taskflow-tasks`.
