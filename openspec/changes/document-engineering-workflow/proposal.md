## Why

The engineering workflow for this repo now lives in `openspec/config.yaml` instead of a
prompt pasted by hand on every task. That file states *what* the workflow is, but not *why*
it is shaped that way — and the shape came out of a long investigation into which review
tools can actually be invoked, and what each of them can see.

Without that reasoning written down, the next person to touch the config (including a future
session of the same agent) will re-litigate settled questions, or worse, "fix" a deliberate
choice back into a broken one. Two of the decisions in the config look arbitrary until you
know the evidence behind them: why the commit happens before the reviews, and why one review
is invoked through a headless subprocess.

Task: none

## What Changes

Replace the placeholder `README.md` with documentation of the workflow: how it works first,
then the decision record — options considered, what was rejected, and why.

## Capabilities

### New Capabilities
- `workflow-documentation`: the repo documents its config-driven engineering workflow and the
  decision record behind it

### Modified Capabilities

## Impact

- Rewritten: `README.md`
- No code, no dependencies, no APIs affected
