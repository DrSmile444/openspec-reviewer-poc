## Context

Empty repo used as a PoC for driving the engineering workflow from `openspec/config.yaml`.
No build system, no source code yet.

## Goals / Non-Goals

**Goals:**
- A root `README.md` that states what the repo is and how the openspec workflow is wired.
- Exercise the full propose → apply → review → commit loop as declared in `config.yaml`.

**Non-Goals:**
- Any application code.
- Documenting the openspec CLI itself.

## Decisions

- Plain markdown, no generator or template engine — the repo has no toolchain.
- Keep the README short; it is a signpost, not documentation.

## Risks / Trade-offs

- The change is intentionally trivial, so it exercises the workflow but not its
  behaviour under a large diff.
