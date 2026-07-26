---
name: design-review
description: Adversarially review and refactor an existing design-doc folder with three parallel independent reviewers - code ground-truth verification, external spec/standards conformance, and internal consistency/completeness - then apply ALL confirmed findings in one coherent batch and commit. Operates on a per-design sub-folder under .claude/design/ (or a caller-specified path). Use when the user asks to "re-check / re-verify / review / refactor the design", or before decomposing a design into tasks.
argument-hint: [<design slug or path> — default: the single design under .claude/design/]
---

# design-review — adversarial design verification & refactor

**Model requirement:** MUST run on the Fable-tier model (critical architectural review). If the
session model is lower, STOP and ask the user to switch (`/model fable`).

## Which design folder

- **Default root:** `.claude/design/`. Review the per-design **sub-folder**
  `<root>/<slug>/` (designs are modular — one sub-folder each).
- **Caller override:** honor an explicitly named path/slug. If the root holds exactly one design
  sub-folder, use it; if several, ask which slug (or take the one the user names). Confirm the
  resolved path before spawning reviewers.
- Read the whole sub-folder (README, ROADMAP, all numbered docs, and `tasks/` if present) before
  reviewing.

## Principle

A design review is only worth doing adversarially: reviewers try to BREAK the design, not
summarize it. Findings must be verified against ground truth (code, spec text) before being
applied — plausible-but-wrong findings are worse than none.

## Process

### 1. Fan out THREE reviewers in parallel (one Agent call each, same message)

**Reviewer A — ground truth vs code.** Verify every FACTUAL claim about existing code against
the actual sources (file:line). Includes feasibility claims ("X already does Y"). Must also
verify cross-language/byte-level claims (hashing, encodings, endianness) where the design
depends on parity. Report format: numbered findings with severity, doc+section, what the doc
says, what the code says (file:line), one-line fix — AND a "verified correct" list.

**Reviewer B — external spec/standards conformance.** Fetch the CURRENT authoritative specs
(WebFetch/WebSearch: protocol specs, RFCs, vendor client behavior — including real SDK source
where it defines de-facto behavior). Classify violations by MUST/SHOULD with citations. Pay
special attention to: discovery/metadata locations, identifier canonicalization, security BCPs,
and what real clients actually do vs what the spec says.

**Reviewer C — internal consistency + completeness.** Cross-doc contradictions (a decision
stated differently in two places; leftovers from earlier revisions), requirement traceability
(every owner requirement → mechanism, fully guaranteed, edge cases stress-tested), phase/plan
completeness (every decision has a home in some phase; dependency graph edges real;
**ROADMAP timeline/gates match the migration doc and task specs**; **ROADMAP's status board is
the ONLY place task state exists** — a `status` field in task frontmatter or a status copy in
any other doc is a finding), threat-model coverage of NEWLY added mechanisms, UX budget honesty
(recount the steps from the mechanics), and a stale-text sweep. Must verify each candidate
against the actual doc text before reporting.

Severity scale for all three: **P0** = breaks a guarantee/golden path or states a falsehood that
materially affects the design; **P1** = gap/risk/unhandled path; **P2** = wording/staleness.

### 2. Consolidate — after ALL THREE report
Merge findings; dedupe overlaps (spec + consistency often hit the same text). Decide fixes for
every finding — including design changes (not just wording). A finding that revises an owner
decision gets the decision marked **REVISED + date + why** in the ledger, and is flagged to the
user prominently in the final report.

### 3. Apply in ONE batch
Edit all docs in a single pass (batched Edits per file). Then run a **stale-term grep sweep**
over the sub-folder for every construct you replaced (old parameter shapes, old numbers, old
storage locations) — fix stragglers. Update the README status line AND the ROADMAP progress log
to record the review.

### 4. Commit + report
Surgical commit (design sub-folder only) with a message summarizing spec/ground-truth/
consistency findings separately. Report to the user: (a) design changes they must be aware of
(decision revisions first), (b) factual corrections, (c) what was verified sound — so the review
builds justified confidence, not just a diff.

## Anti-patterns

- Applying fixes before all reviewers return (they overlap on the same text).
- Reporting reviewer output verbatim instead of verified, consolidated findings.
- Treating MUST-violations as "hardening later".
- Skipping the "verified correct" inventory — knowing what checked out is half the value.
