## ADDED Requirements

### Requirement: Workflow mechanism is documented

`README.md` SHALL describe how the config-driven workflow runs: where the workflow is
declared, how it reaches a change's `tasks.md`, and what each step does.

#### Scenario: A step misbehaves and the maintainer needs to debug it
- **WHEN** a maintainer reads `README.md`
- **THEN** it names `openspec/config.yaml` as the source of the workflow
- **AND** it explains that `rules.tasks` is injected into `openspec instructions tasks`
- **AND** it lists the workflow steps in execution order

#### Scenario: A reviewer needs to know what each review can see
- **WHEN** a maintainer reads the review section
- **THEN** each of the three reviews states how it is invoked
- **AND** states whether it reads committed history or the working tree

### Requirement: Decision record is documented

`README.md` SHALL record the decisions behind the workflow, the alternatives that were
rejected, and the reason each was rejected.

#### Scenario: A maintainer wants to change a deliberate choice
- **WHEN** they read the decision record
- **THEN** it explains why the implementation commit precedes the reviews
- **AND** it explains why `/code-review` is invoked through a headless subprocess

#### Scenario: A maintainer wants to revive a rejected option
- **WHEN** they read the rejected-options section
- **THEN** each rejected option states what it was, why it was dropped, and what was lost
- **AND** enough detail is present to rebuild it without repeating the investigation
