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

Installation SHALL detect whether the workflow is already present via a version marker,
so re-running the install prompt on an already-installed project does not duplicate the
workflow's rules.

#### Scenario: Same version already installed
- **WHEN** the target's `rules.tasks` already contains the current version marker
- **THEN** the install prompt makes no changes to `config.yaml`
- **AND** reports that the workflow is already installed

#### Scenario: Older version already installed
- **WHEN** the target's `rules.tasks` contains a version marker older than the current one
- **THEN** the install prompt shows the old block and the new block
- **AND** asks the user whether to replace the old block before writing anything

#### Scenario: Not yet installed
- **WHEN** the target's `rules.tasks` contains no version marker for this workflow
- **THEN** the install prompt appends the current workflow payload to `rules.tasks`
- **AND** appends the `Task: <ID>` capture bullet to `rules.proposal` if an equivalent rule
  is not already present

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
