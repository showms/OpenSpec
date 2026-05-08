## MODIFIED Requirements

### Requirement: Inject context into all artifact instructions

The system SHALL inject the `context` field from project config into instructions for all artifacts and into the `apply` and `archive` workflow instruction surfaces, wrapped in XML-style `<context>` tags.

#### Scenario: Context applied to all artifacts
- **WHEN** instructions are loaded for any artifact (proposal, specs, design, tasks)
- **THEN** context section appears in all artifact instruction outputs

#### Scenario: Context applied to apply instructions
- **WHEN** apply instructions are generated for a change and config contains a `context` field
- **THEN** the apply instruction output includes the configured context

#### Scenario: Context applied to archive workflow guidance
- **WHEN** archive workflow guidance is generated and config contains a `context` field
- **THEN** the archive instruction output includes the configured context

#### Scenario: Config has no context field
- **WHEN** config omits the `context` field or `context` is undefined
- **THEN** artifact, apply, and archive instruction outputs do not include `<context>` tags

### Requirement: Format context with XML-style tags

The system SHALL wrap context content in `<context>` opening and `</context>` closing tags with content on separate lines for artifact, apply, and archive instruction surfaces.

#### Scenario: Context appears before apply rules and template
- **WHEN** apply instructions are generated with both context and workflow rules
- **THEN** the `<context>` section appears before the `<rules>` section and before the instruction template content

#### Scenario: Context appears before archive rules and guidance
- **WHEN** archive guidance is generated with both context and workflow rules
- **THEN** the `<context>` section appears before the `<rules>` section and before the archive guidance content
