# openspec-reviewer-poc

A proof-of-concept repository for driving the engineering workflow from OpenSpec
configuration instead of pasting a prompt into the agent on every task.

## Workflow

The branch → implement → review → commit workflow is declared in
[`openspec/config.yaml`](openspec/config.yaml) under `rules.tasks`. Those rules are
injected into `openspec instructions tasks`, so every generated `tasks.md` carries the
workflow steps as checkboxes that the apply phase walks through.

## Layout

- `openspec/config.yaml` — schema, project context, and the per-artifact rules
- `openspec/changes/` — active changes
- `openspec/specs/` — accepted specifications
