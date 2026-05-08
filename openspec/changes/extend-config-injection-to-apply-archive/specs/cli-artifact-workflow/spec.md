## MODIFIED Requirements

### Requirement: Apply Instructions Command

The system SHALL generate schema-aware apply instructions via `openspec instructions apply`, including injected project context and apply-scoped workflow rules when configured.

#### Scenario: Generate apply instructions
- **WHEN** user runs `openspec instructions apply --change <id>`
- **AND** all required artifacts (per schema's `apply.requires`) exist
- **THEN** the system outputs:
  - `contextFiles` mapping artifact IDs to arrays of concrete paths for all existing artifacts
  - schema-specific instruction text
  - progress tracking file path (if `apply.tracks` is set)
  - injected project context when config defines `context`
  - injected apply workflow rules when config defines `rules.apply`

#### Scenario: Apply instructions JSON output
- **WHEN** user runs `openspec instructions apply --change <id> --json`
- **THEN** the system outputs JSON with:
  - `contextFiles`: object mapping artifact IDs to arrays of concrete paths for existing artifacts
  - `instruction`: the apply instruction text
  - `tracks`: path to progress file or null
  - `applyRequires`: list of required artifact IDs
  - injected context and rules fields when available

#### Scenario: Apply instructions without workflow rule target
- **WHEN** user runs `openspec instructions apply --change <id>` and config omits `rules.apply`
- **THEN** the system still returns apply instructions successfully
- **AND** preserves existing schema-driven behavior
