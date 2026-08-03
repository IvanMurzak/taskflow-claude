# Taskflow for Claude Code

Taskflow is a four-stage lifecycle that a user or agent can invoke to turn a fuzzy change
into repository-evidenced architecture, adversarial verification, immutable task
specifications, and ROADMAP-driven execution.

```text
/taskflow-frame → /taskflow-review → /taskflow-tasks → /taskflow-execute
```

## Package

This repository is the Claude Code package. Its plugin manifest is
`.claude-plugin/plugin.json`, and its four public skills live under `skills/`.
Any user or agent may invoke each stage; no skill pins a model.

## Workflow contract

- New artifacts live only in `.claude/taskflow/<slug>/`.
- `ROADMAP.md` is the sole mutable task-state record. Task specs are immutable
  and never contain `status`.
- Task groups run sequentially by `sequence`; independent groups may run in
  parallel when their `needs` dependencies allow it.
- `security_critical` and `production_touching` raise the model-routing tier.
- Production, money, secrets, irreversible effects, and product decisions
  require an explicit owner gate.

Legacy `.claude/design/` folders are archives and are not read or migrated.

## License

MIT
