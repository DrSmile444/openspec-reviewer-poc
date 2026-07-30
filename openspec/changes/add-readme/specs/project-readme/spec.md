## ADDED Requirements

### Requirement: Root README

The repository SHALL expose a `README.md` at its root that states the repository's
purpose and describes how the openspec workflow is configured for it.

#### Scenario: Reader opens the repository root
- **WHEN** someone lists the files at the repository root
- **THEN** a `README.md` is present
- **AND** it names the repository's purpose in its first paragraph

#### Scenario: Reader wants to know the workflow
- **WHEN** someone reads `README.md`
- **THEN** it points to `openspec/config.yaml` as the place where the
  branch → implement → review → commit workflow is declared
