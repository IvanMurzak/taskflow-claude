---
name: system
description: Design a system/feature architecture end-to-end the way a Technical Director designs features - ground-truth exploration of the affected repos, owner product-decision Q&A, then a numbered design-doc set (with a ROADMAP) in a dedicated per-design sub-folder under .claude/design/ (or a caller-specified path) with a decisions ledger. Use when the user asks to "design a solution/system/architecture", describes a structural problem needing an architectural answer, or wants a feature designed before implementation. NOT for one-off bug fixes or tasks that fit an existing pipeline.
model: fable
---

# /design:system — architecture design process

**Model requirement:** this is a critical architectural process and MUST run on the Fable-tier
model. If the current session model is lower, STOP and ask the user to switch (`/model fable`)
before doing anything else.

## Where the design goes (location contract)

- **Default root:** `.claude/design/` in the consumer project.
- **Caller override:** if the user names a location ("design it under X", "put the files in Y"),
  use that as the root instead. Confirm the resolved path in your first message.
- **One sub-folder PER design (mandatory — designs are modular).** Never write design docs
  directly into the root. Create `<root>/<kebab-slug>/` and put everything for this design there.
  A workspace accumulates many designs side by side (`.claude/design/mcp-authorize/`,
  `.claude/design/billing-v2/`, …) — each self-contained, each independently reviewable and
  decomposable.
- If the sub-folder already exists, you are extending/redesigning — read what's there first, don't
  clobber it.

## Non-negotiables

1. **Ground truth before design.** Never assert "X works like Y" from memory — every factual
   claim about existing code carries a `file:line` reference obtained this session.
2. **The owner decides products, you decide mechanisms.** Product-shaping choices (deployment
   targets, UX golden path, compatibility policy, identity model, monetization interplay) go to
   the user via AskUserQuestion with a recommended option first. Record every answer.
3. **Docs are the deliverable.** Conversation is scratch space; the design folder is the durable
   artifact. Commit surgically (stage only the design sub-folder; never `git add -A`; never touch
   other flows' files in a shared checkout).

## Process

### 1. Frame + ask (same turn)
Resolve the design sub-folder path (above). Extract the problem. Draft 2–4 sharp product
questions whose answers change the design (deployment topology, golden-path UX, breaking-change
tolerance, identity/accounts, scope). Fire AskUserQuestion AND launch exploration in the same
turn so the user answers while agents run.

### 2. Explore ground truth (parallel)
One Explore/general-purpose agent per affected repo/subsystem, launched concurrently. Each
prompt demands: file paths + line numbers + short excerpts, the exact seams (files/classes where
the change would land), current behavior of the mechanism being redesigned, and a structured
summary "to design against". Add a follow-up explorer whenever the user's answers reveal a new
affected system.

### 3. Write the design-doc set — `<root>/<kebab-slug>/`
`README.md` + `ROADMAP.md` are REQUIRED in every design. Numbered docs adapt to the feature; the
canonical shape is:

| File | Contents |
|---|---|
| `README.md` | Status line, problem, **locked-decisions table**, one-paragraph design summary, doc map (incl. ROADMAP), glossary |
| `ROADMAP.md` | **REQUIRED, living implementation ledger** — see §4 |
| `01-current-architecture.md` | Ground truth only, with file:line refs, edge-case table, and a **seam index** (every file the redesign touches) |
| `02-target-architecture.md` | Principles, roles/diagram, core models, **decisions ledger D1..Dn** with rationale + trade-offs, explicitly-listed open questions |
| `03-…` flows | End-to-end sequence flows for every actor path, including failure/expiry/offline |
| `04+` subsystem specs | Data structures, precedence rules, test plan per subsystem |
| infra delta | What changes in deployment (compose/nginx/secrets), with rollback |
| migration/rollout | Numbered phases, dependency graph, per-phase gates, legacy disposition table, risks |
| security | Token/credential inventory, threat table (threat → control), secrets-handling rules, per-phase security gates |
| user-workflows | UX contract: step-by-step tables per persona with **counted step/click budgets** declared as release gates |

### 4. ROADMAP.md (REQUIRED)
Every design gets a `ROADMAP.md` capturing **what and when to do during implementation** — the
timeline + status, distinct from the migration doc's *rationale* and the task specs' *detail*.
It contains at least:
- A one-line disambiguation banner: *"this is THIS design's implementation ledger — NOT the
  workspace-level global product roadmap (e.g. `.claude/plans/ROADMAP.md`)"*.
- **Design status** + **implementation status** (e.g. NOT STARTED 0/N) + last-updated date.
- The **execution timeline** — phases/waves, what runs in parallel, dependency edges, gates.
- A **status board** (one row per planned unit; filled in properly once `/design:tasks` runs).
- **Human-approval gates** (anything touching production, money, secrets, irreversibility).
- A **progress log** appended over time.
At design time the status board may be a skeleton; `/design:tasks` populates it from the task
coefficients.

**The status board is the SINGLE source of truth for implementation state.** Each row carries
the spec link + orchestration metadata (`needs`, imp/cx, model, status, run/PR, updated) so the
orchestrator reads the ready set from one file. Task state exists NOWHERE else — not in task
frontmatter (specs are immutable), not in other docs, not per-task in a global plan store (which
gets one thin pointer task for the whole design). **Single writer:** only the orchestrating
agent updates the board + progress log, after verifying ground truth; implementers report.

### 5. Decisions ledger discipline
Every owner answer becomes a numbered decision (D1, D2, …) with a date, recorded in BOTH the
README table and the 02 ledger. When the user amends a decision later, mark the row REVISED with
the reason — never silently rewrite history.

### 6. Iterate
Each subsequent user message that refines the design → update the affected docs in the same
turn, keep cross-doc consistency (grep for stale terms you just replaced), append decisions,
surgical commit per iteration with a message summarizing what changed and why.

## Quality bar (self-check before declaring done)

- Every mechanism traces to a requirement; every requirement traces to a mechanism.
- UX flows counted in steps and browser hops; worst path named honestly.
- Security section has a threat table covering every NEW mechanism introduced.
- Migration has phases, a dependency graph, rollback points, and a user-impact table.
- ROADMAP exists, disambiguated from the global plan store, with timeline + gates.
- No "TBD" without an owner-facing open question attached.

When the design stabilizes, recommend running `/design:review` (same sub-folder) before task
decomposition (`/design:tasks`). The full lifecycle is `/design:system` → `/design:review` →
`/design:tasks` → `/design:implement` (execution orchestrator; owner GO required).
