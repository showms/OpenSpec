## MODIFIED Requirements

### Requirement: OPSX Archive Skill

The system SHALL provide an `/opsx:archive` skill that archives completed changes in the experimental workflow and includes injected project context and archive-scoped workflow rules in archive guidance.

#### Scenario: Archive guidance includes project context
- **WHEN** agent executes `/opsx:archive` for a change and config contains a `context` field
- **THEN** the archive workflow guidance includes that configured project context

#### Scenario: Archive guidance includes workflow rules
- **WHEN** agent executes `/opsx:archive` for a change and config contains `rules.archive`
- **THEN** the archive workflow guidance includes those archive-specific rules

#### Scenario: Archive behavior remains unchanged without workflow rules
- **WHEN** agent executes `/opsx:archive` and config omits `rules.archive`
- **THEN** the skill preserves existing archive readiness checks, prompts, and archive behavior

### Requirement: Skill Output

The skill SHALL provide clear feedback about the archive operation without changing existing archive enforcement semantics.

#### Scenario: Archive guidance remains additive
- **WHEN** archive guidance is generated with injected context or archive rules
- **THEN** the added guidance supplements the existing archive workflow output
- **AND** does not replace artifact completion checks, task warnings, sync prompts, or archive execution summaries
