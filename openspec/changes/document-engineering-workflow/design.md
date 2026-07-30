## Context

`openspec/config.yaml` carries the workflow as `rules.tasks`, injected into
`openspec instructions tasks` at propose time and baked into each change's `tasks.md`. The
rules were derived empirically: every review channel was tested against a live repo with
planted defects before being written into the config.

The repo has no build system and no source code — it exists to host and exercise this
workflow.

## Goals / Non-Goals

**Goals:**
- Explain how the workflow runs, in enough detail to debug it when a step misbehaves.
- Record the decisions, the rejected alternatives, and the evidence, so the config can be
  improved later without repeating the investigation.

**Non-Goals:**
- Documenting the OpenSpec CLI itself.
- A tutorial. The reader is the person maintaining this repo.

## Decisions

- **Structure: how it works first, decision record second.** Someone hitting a broken step
  needs the mechanism immediately; the reasoning is for whoever changes the config.
- **Include reconstruction recipes for rejected options.** A rejected option recorded as
  "we didn't do X" is not recoverable. Recorded as "X, built like this, dropped for this
  reason" it is.
- **Plain markdown, no toolchain.** The repo has none, and the README should not introduce
  one.

## Risks / Trade-offs

- The document restates facts about Claude Code internals (which commands are
  model-invocable, what each review reads). Those can change with a Claude Code release, so
  claims are dated and attributed to how they were verified, rather than asserted flatly.
- Documenting rejected options at length risks the reader mistaking them for live advice;
  they are kept in a clearly separated section.
