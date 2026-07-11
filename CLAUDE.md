# CLAUDE.md — Claude-Design plugin

Guidance for Claude Code when editing THIS plugin (not for consumers using it).

## What this is

A four-skill architecture-design lifecycle plugin: `/design-system` → `/design-review` →
`/design-tasks` → `/design-implement`. Skills live at `skills/<name>/SKILL.md`; the folder name
IS the skill name. Names deliberately carry the `design-` family prefix: UIs often hide the
plugin namespace, so each name must be self-descriptive standalone (owner decision, 2026-07-11).

## Invariants — keep these when editing

1. **The four skills share one contract; change it in ALL of them or none.** The shared
   contract is: the location convention (default root `.claude/design/`, caller override, one
   sub-folder per design), the ROADMAP shape (§4 of `system`), the **status-board row schema**
   (`Task (spec) | needs | imp/cx | model | Status | Run / PR | Updated`), the three status
   rules (board = only mutable state; single writer = orchestrator; global plan store gets one
   pointer task), and the model rubric (≥8 top / 5–7 mid / ≤4 fast / security bumps a tier).
   Grep all four SKILL.md files when touching any of these.
2. **`model: fable` frontmatter is mandatory on the three design-time skills**
   (`design-system`, `design-review`, `design-tasks`) — plus the in-body STOP instruction if
   the session model is lower. Explicit owner decision: architecture quality outranks token
   cost. **`design-implement` deliberately has NO model frontmatter** (owner decision,
   2026-07-11): orchestration inherits the session model; the implementers' tiers come from
   the ROADMAP board.
3. **Task specs are immutable; ROADMAP is the single source of task state.** Never add a
   `status` field to the task frontmatter schema in `tasks`, and never weaken `implement`'s
   single-writer rule.
4. **Cross-references use the bare family names** (`/design-review`) — they match what users
   see. Every skill declares an `argument-hint` exposing the optional design-folder argument;
   keep it when editing frontmatter.
5. **Stay consumer-agnostic.** The skills may *prefer* a pipeline system when the consumer
   project has one, but must define the fallback (isolated worker agents in worktrees). Never
   hardcode a specific project's paths beyond the `.claude/design/` default.

## Releasing a change

Bump `version` in `.claude-plugin/plugin.json`, commit + push this repo, then bump the submodule
pointer + marketplace version in the parent marketplace repo (`Personal-Plugin-Marketplace`).
