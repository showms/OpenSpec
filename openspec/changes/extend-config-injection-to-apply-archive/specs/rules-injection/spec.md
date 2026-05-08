## MODIFIED Requirements

### Requirement: Inject rules only for matching artifact

The system SHALL inject rules from config into instructions only when the target matches either an artifact ID or a reserved workflow target in the `rules` object.

#### Scenario: Rules exist for the artifact
- **WHEN** loading instructions for `proposal` and config has `rules: { proposal: ["Rule 1", "Rule 2"] }`
- **THEN** instruction output includes a rules section with both rules

#### Scenario: Workflow rules exist for apply
- **WHEN** loading apply instructions and config has `rules: { apply: ["Use sub-agents only when explicitly allowed"] }`
- **THEN** apply instruction output includes a rules section with that workflow guidance

#### Scenario: Workflow rules exist for archive
- **WHEN** loading archive workflow guidance and config has `rules: { archive: ["Update the knowledge base after archive"] }`
- **THEN** archive instruction output includes a rules section with that workflow guidance

#### Scenario: Artifact and workflow rules remain scoped
- **WHEN** config has both `rules.tasks` and `rules.apply`
- **THEN** task artifact instructions include only `rules.tasks`
- **AND** apply instructions include only `rules.apply`

#### Scenario: No rules for requested target
- **WHEN** instructions are loaded for a target with no matching entry in the `rules` object
- **THEN** instruction output does not include `<rules>` tags

### Requirement: Validate artifact IDs during instruction loading

The system SHALL validate keys in the `rules` object against the active schema artifact IDs plus reserved workflow targets and emit actionable validation output for unknown targets.

#### Scenario: Reserved workflow targets are valid
- **WHEN** config has `rules: { apply: [...], archive: [...] }`
- **THEN** no validation warning is emitted for those targets

#### Scenario: Unknown target in rules
- **WHEN** instructions are loaded and config has `rules: { verify: [...] }`
- **THEN** the system emits a validation message identifying `verify` as an unknown target
- **AND** the message lists valid artifact IDs plus reserved workflow targets for that schema

#### Scenario: Validation failure surfaced to workflow callers
- **WHEN** apply or archive instruction generation encounters malformed or unknown rule targets
- **THEN** the failure is surfaced to the caller with actionable correction guidance
