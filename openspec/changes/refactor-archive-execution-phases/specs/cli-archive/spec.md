## ADDED Requirements

### Requirement: Preserve the one-shot archive contract during phased migration

The existing `openspec archive <change>` command SHALL preserve its established syntax, flags, prompts, JSON envelope, diagnostics, spec outcomes, and recovery behavior while sharing the same planning and finalization safety primitives as phased archive attempts.

#### Scenario: Existing direct archive succeeds
- **WHEN** a user runs the existing archive command with the same change and options as before this change
- **THEN** it produces the same observable archive name, main-spec outcome, warnings, summary, and exit status
- **AND** does not require the user to manage an archive-attempt plan

#### Scenario: Existing JSON archive is blocked
- **WHEN** the one-shot JSON command encounters an existing validation, confirmation, path, task, spec-update, or destination blocker
- **THEN** it preserves the existing JSON envelope and diagnostic code for that condition
- **AND** keeps stdout free of human prompt prose

#### Scenario: Existing archive skips specs
- **WHEN** a user runs `openspec archive <change> --skip-specs`
- **THEN** the command preserves the existing no-spec-update behavior
- **AND** still uses the shared destination claim, change move, and recovery guarantees

#### Scenario: Existing archive disables validation
- **WHEN** a user explicitly runs the one-shot command with validation disabled and confirms as required
- **THEN** the command preserves its existing warning and mutation behavior
- **AND** still refuses to retire a capability without a validation verdict
