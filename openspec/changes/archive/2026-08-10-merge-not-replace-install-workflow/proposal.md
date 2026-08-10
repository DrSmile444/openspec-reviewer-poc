## Why

The "Installation via agent" prompt in `README.md` downloads this repo's `openspec/config.yaml`
and writes it to the same path in the target project, gated only by "show diff and ask before
overwriting." When the target already has its own `schema`, its own `context`, and its own
per-artifact rules (design/specs/adrs/plan/self_review, etc. — shape varies by schema), that
"diff" is a full-file replacement: it wipes all of it. Tested against a real project with a
custom schema, own project context, and existing design/specs/adrs rules — the generated diff
proposed dropping every one of them.

The only thing this workflow actually needs to install is the `rules.tasks` block (branch →
implementation → verify → commit → 3 parallel reviews → fix → commit) and one `rules.proposal`
bullet (the `Task: <ID>` capture). Everything else in `config.yaml` is project identity and must
never be touched by an install.

Task: none

## What Changes

- Extract the portable workflow payload (the `rules.tasks` block and the `rules.proposal`
  bullet) out of `openspec/config.yaml` into a standalone `openspec/workflow-rules.md`, so it is
  the single source of truth both the README prompt and this repo's own config derive from.
- Add a stable version marker as the first line of the payload's `rules.tasks` block (e.g.
  `openspec-reviewer-poc workflow v1 — branch, implement, verify, commit, review, fix,
  commit.`), so an install can detect whether the workflow is already present.
- Rewrite the "Installation via agent" prompt in `README.md`: when `openspec/config.yaml`
  already exists, the agent reads it, checks `rules.tasks` for the version marker, and:
  - same version marker present → no-op, report "already installed"
  - older version marker present → show old-vs-new block, ask before replacing
  - no marker present → append the payload to `rules.tasks` (and the Task-ID bullet to
    `rules.proposal` if missing), never touching `schema`, `context`, or any other rules key,
    and show a diff scoped to only the added lines before writing
- Add one sentence to the payload text clarifying that its branch/verify/commit/review/fix/
  commit steps are process scaffolding around plan- or spec-derived implementation tasks, not
  implementation tasks themselves — so schemas whose own rules say "derived from plan.md, do
  not add tasks not in the plan" don't cause a task-generating model to drop the scaffolding.
- Narrow the "Same thing without an agent" bash-only section to the fresh-install case only
  (no existing `config.yaml`), with an explicit note that merging into an existing file
  requires the agent path.
- Update this repo's own `openspec/config.yaml` `rules.tasks` to carry the version marker and
  reference `workflow-rules.md` as the source, so the reference install and the merge-detection
  logic stay in sync.

## Capabilities

### New Capabilities
- `workflow-installation`: how the branch/review workflow is installed into a target project —
  merging additively into an existing `openspec/config.yaml` (with idempotent re-install)
  instead of overwriting it, versus a plain copy on a fresh install.

### Modified Capabilities

## Impact

- Rewritten: `README.md` ("Installation via agent" section)
- New: `openspec/workflow-rules.md`
- Modified: `openspec/config.yaml` (`rules.tasks` gains the version marker; content otherwise
  unchanged)
- No application code, no dependencies, no APIs affected
