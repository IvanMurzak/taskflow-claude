# Claude-Strategy

Taskflow is a dual-platform, four-stage lifecycle that turns a fuzzy change into
repository-evidenced architecture, a reviewed plan, immutable task specs, and
ROADMAP-driven execution.

```text
/taskflow-frame → /taskflow-review → /taskflow-tasks → /taskflow-execute
     frame              verify              specify            orchestrate
```

Owners invoke every stage manually. The session model remains their choice;
the `top` / `mid` / `fast` task tiers map to the consumer project's approved
models and never pin a vendor model.

## Install

### Claude Code

```text
/plugin marketplace add --source local --path "C:\Projects\AI\claude-plugins"
/plugin install taskflow@ivan-private-plugins
```

### Codex

Add a marketplace that packages [`codex/taskflow`](codex/taskflow/) and install
its `taskflow` entry with the Codex plugin CLI:

```powershell
codex plugin add taskflow@<your-marketplace>
```

The Codex package is self-contained; its manifest is
[`codex/taskflow/.codex-plugin/plugin.json`](codex/taskflow/.codex-plugin/plugin.json).

## Lifecycle

| Command | Role |
|---|---|
| `/taskflow-frame` | Researches repository ground truth, captures owner product decisions, and writes the architecture frame plus `ROADMAP.md`. |
| `/taskflow-review` | Runs adversarial code, external-spec, and internal-consistency verification, applying only confirmed non-product corrections. |
| `/taskflow-tasks` | Creates immutable task specifications, conflict-safe groups, dependencies, routing tiers, and execution waves. |
| `/taskflow-execute` | Reconciles and orchestrates from the ROADMAP status board, which is its sole writable task-state record. |

## Contract

- Active artifacts live only at `.claude/taskflow/<slug>/`; a caller may select
  a slug but may not redirect Taskflow artifacts outside that root.
- Existing `.claude/design/` folders are archives. Taskflow neither reads,
  migrates, nor falls back to them.
- Every taskflow contains `README.md`, `ROADMAP.md`, numbered architecture
  documents as appropriate, and later a `tasks/` directory.
- Task specs are immutable and never contain a status. `ROADMAP.md` is the
  single source of task state; only `/taskflow-execute` changes it after
  evidence such as a merged change and passing CI.
- Specs include `sequence` within a merge-conflict group,
  `security_critical`, and `production_touching`. Work is sequential within a
  group and parallel across independent groups.
- Production, money, secrets, irreversible effects, and owner product choices
  need an explicit owner gate. Taskflow presents a safe recommended option and
  records the confirmed decision.
- Use the project's normal forge, CI, and execution system when present. When
  one is unavailable, use isolated worktrees and locally verifiable evidence.

## License

MIT
