## MODIFIED Requirements

### Requirement: Schema init supports setting project default
The CLI SHALL offer to set the newly created schema as the project default and SHALL persist the selection in the active project config's canonical `schema` field so subsequent commands use that schema without losing unrelated configuration content.

#### Scenario: Set as default interactively
- **WHEN** user runs `openspec schema init my-workflow` in interactive mode
- **AND** user confirms setting it as the project default
- **THEN** the system records `schema: my-workflow` in the active project config
- **AND** subsequent changes created without a schema override use `my-workflow`

#### Scenario: Set as default via flag
- **WHEN** user runs `openspec schema init my-workflow --default`
- **THEN** the system creates the schema
- **AND** records `schema: my-workflow` in the active project config
- **AND** reports that the schema was set as the project default

#### Scenario: Create a missing project config
- **WHEN** user runs `openspec schema init my-workflow --default`
- **AND** neither `openspec/config.yaml` nor `openspec/config.yml` exists
- **THEN** the system creates `openspec/config.yaml` with `schema: my-workflow`

#### Scenario: Update the existing alternate config filename cross-platform
- **WHEN** a project on macOS, Linux, or Windows has `openspec/config.yml` and no `openspec/config.yaml`
- **AND** user runs `openspec schema init my-workflow --default`
- **THEN** the system updates `openspec/config.yml` with `schema: my-workflow`
- **AND** does not create `openspec/config.yaml`

#### Scenario: Preserve unrelated project configuration
- **WHEN** the active project config contains comments and unrelated entries such as `context`, `rules`, `references`, or `store`
- **AND** user runs `openspec schema init my-workflow --default`
- **THEN** the system updates only the default schema selection
- **AND** preserves the unrelated entries and comments

#### Scenario: Repair an obsolete generated default field
- **WHEN** the active project config contains a `defaultSchema` field from an earlier `schema init --default` invocation
- **AND** user runs `openspec schema init my-workflow --default`
- **THEN** the system records `schema: my-workflow`
- **AND** removes the obsolete `defaultSchema` field

#### Scenario: Skip setting default
- **WHEN** user runs `openspec schema init my-workflow --no-default`
- **THEN** the system creates the schema without modifying the active project config
