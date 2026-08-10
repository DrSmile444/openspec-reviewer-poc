## Purpose

Defines how the branch/review workflow is installed into a target project: merging
additively into an existing `openspec/config.yaml` instead of overwriting it, with
idempotent re-installation.

## Requirements

### Requirement: Fresh install copies the reference files verbatim

When the target project has no `openspec/config.yaml`, installation SHALL copy
`openspec/config.yaml`, `openspec/workflow-rules.md`, and the two vendored skills verbatim,
exactly as the current process does.

#### Scenario: No existing config.yaml
- **WHEN** a target project has no `openspec/config.yaml`
- **THEN** the install prompt downloads and writes `config.yaml` and the two skills
  directly, with no merge step

### Requirement: Existing config.yaml is merged, never overwritten wholesale

When `openspec/config.yaml` already exists in the target project, installation SHALL
preserve its `schema`, `context`, and every rules key other than `tasks` and `proposal`
unchanged, and SHALL modify only `rules.tasks` and `rules.proposal`.

#### Scenario: Target has its own schema, context, and other artifact rules
- **WHEN** the target's `openspec/config.yaml` declares a `schema` other than this repo's,
  a `context` block, and rules for artifacts such as `design`, `specs`, `adrs`, `plan`, or
  `self_review`
- **THEN** the installed result leaves `schema`, `context`, and all of those rules keys
  byte-for-byte unchanged
- **AND** the diff shown to the user before writing contains only additions to
  `rules.tasks` and `rules.proposal`, not a full-file replacement

### Requirement: Installation is idempotent

Installation SHALL detect whether the workflow is already present by comparing the target's
`rules.tasks` against the reference payload for an equivalent (not necessarily identical)
description of the same sequence, so re-running the install prompt on an already-installed
project does not duplicate the workflow's rules. No literal version-marker string SHALL be
written into a target's `config.yaml` for this purpose.

#### Scenario: Equivalent workflow already installed
- **WHEN** the target's `rules.tasks` already describes the branch/verify/commit/review/fix/
  commit sequence, whether or not its wording matches the reference payload verbatim
- **THEN** the install prompt makes no changes to `config.yaml`
- **AND** reports that the workflow is already installed

#### Scenario: Meaningfully different installed version found
- **WHEN** the target's `rules.tasks` describes a recognizably older or incomplete version of
  this workflow (e.g. missing one of the three reviews, or a different step order)
- **THEN** the install prompt shows the existing rules and the reference payload
- **AND** asks the user whether to replace the existing ones before writing anything

#### Scenario: Not yet installed
- **WHEN** the target's `rules.tasks` contains nothing resembling this workflow
- **THEN** the install prompt appends the current workflow payload to `rules.tasks`
- **AND** appends the `Task: <ID>` capture bullet to `rules.proposal` if an equivalent rule
  is not already present

#### Scenario: Confirmation covers a wrong semantic match
- **WHEN** the install prompt judges the target's `rules.tasks` to already contain, lack, or
  differ from the reference payload
- **THEN** it shows the diff of what it intends to write, or explicitly states it intends no
  changes, before writing anything - the human confirmation is the safety net for a
  comparison that has no literal string to check against

### Requirement: Installed rules do not collide with plan-derived task rules

The installed `rules.tasks` payload SHALL state that its branch/verify/commit/review/fix/
commit steps are process scaffolding around the target's own implementation tasks, not
implementation tasks themselves.

#### Scenario: Target schema requires tasks to derive from a plan
- **WHEN** the target's existing `rules.tasks` includes a rule restricting tasks to those
  derived from a plan or spec document
- **THEN** the installed payload's own text makes clear its scaffolding steps are not
  subject to that derivation rule

### Requirement: Non-agent installation is limited to fresh installs

The bash-only installation path (no agent) SHALL be documented as valid only when no
`openspec/config.yaml` exists yet in the target project.

#### Scenario: User tries the bash-only path on a project with an existing config.yaml
- **WHEN** a user reads the "Same thing without an agent" section
- **THEN** it states that merging into an existing `config.yaml` requires the agent-based
  path, and that the bash snippet is for fresh installs only
