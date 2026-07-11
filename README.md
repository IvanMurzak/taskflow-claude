# Claude-Strategy

A Claude Code plugin: a **four-skill lifecycle for feature/system architecture** — from a fuzzy
problem statement to designed docs, adversarial verification, a model-routable task breakdown,
and orchestrated execution.

Skill names carry the `design-` prefix so they stay self-descriptive even when the UI hides the
plugin namespace (you see `/design-review`, not a bare `/review`). Every skill takes an
**optional design-folder argument** (slug or path); default root is `.claude/design/`.

```
/design-system  →  /design-review  →  /design-tasks  →  /design-implement
   (design)       (adversarial        (decompose)        (orchestrate
                    verification)                          execution)
```

The three design-time skills run on the **Fable-tier model** (declared in frontmatter) —
architecture is the one place where reasoning quality dominates token cost. `/design-implement`
inherits the session model (orchestration is procedural); the tasks themselves are routed to
model tiers by the coefficients.

## Install

From the private marketplace:

```bash
/plugin marketplace add --source local --path "C:\Projects\AI\claude-plugins"
/plugin install strategy@ivan-private-plugins
```

## The skills

| Skill | What it does |
|---|---|
| `/design-system` | Explores ground truth in the affected repos (parallel explorer agents, every claim gets `file:line`), asks the owner the product-shaping questions, then writes a numbered design-doc set — README with a locked-decisions table, current/target architecture with a decisions ledger, flows, infra delta, migration, security threat table, UX step budgets — plus a **ROADMAP.md** (living implementation ledger). |
| `/design-review` | Three parallel adversarial reviewers — **A**: every factual claim vs actual code; **B**: current external specs/RFCs/vendor behavior via web; **C**: internal consistency, traceability, completeness. Consolidates after all three, applies every confirmed finding in one batch, runs a stale-term sweep, commits. |
| `/design-tasks` | Decomposes the reviewed design into **immutable task specs** with `importance`/`complexity` coefficients (1–10) that select the implementing AI model tier (top/mid/fast), grouped into **merge-conflict domains** (sequential within a group, parallel across groups), and populates ROADMAP's execution waves + status board. |
| `/design-implement` | The execution orchestrator. Reads the **ROADMAP status board — the single source of task state** — reconciles it with ground truth, computes the ready set from `needs` edges, dispatches tasks (via the project's pipeline system when present) on the board's model tier, verifies merged-PR/green-CI before flipping any row, and drives wave after wave. Single writer of the board. |

## Core conventions

- **Designs are modular:** each design lives in its own sub-folder under the consumer project's
  `.claude/design/<slug>/` (root overridable per invocation). Many designs coexist side by side.
- **ROADMAP.md is mandatory** in every design and is the ONLY mutable record of implementation
  state. Task spec files never carry status; a status copy anywhere else is a bug.
- **Single writer:** only the orchestrator flips board rows — after ground-truth verification.
- **Model rubric:** complexity ≥8 → top tier; 5–7 → mid; ≤4 → fast; `security_critical` or
  production-touching bumps one tier up.
- **Owner gates:** anything touching production, money, secrets, or irreversible actions gets an
  explicit human-approval gate in ROADMAP and its own GO at dispatch time.

## License

MIT
