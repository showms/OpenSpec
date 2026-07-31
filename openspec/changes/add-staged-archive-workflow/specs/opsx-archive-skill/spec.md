## MODIFIED Requirements

### Requirement: OPSX Archive Skill

The system SHALL provide an `/opsx:archive` skill that orchestrates a completed change through the staged archive CLI lifecycle.

#### Scenario: Archive a change with all artifacts complete

- **WHEN** agent executes `/opsx:archive` with a change name
- **AND** all artifacts and tasks are complete
- **THEN** the agent SHALL inspect staged status for the change
- **AND** prepare the archive through `openspec archive <name> --stage prepare --json` only when no staged state exists
- **AND** consume the returned candidate work without rediscovering archive inputs
- **AND** reconcile only plan-owned candidates
- **AND** present the immutable CLI validation review for user approval
- **AND** delegate formal spec writes and the change move to finalize
- **AND** display the CLI completion receipt

#### Scenario: Change selection prompt

- **WHEN** agent executes `/opsx:archive` without specifying a change
- **THEN** the agent SHALL infer the change from conversation context or auto-select it when only one active change exists
- **AND** when ambiguous, prompt from active changes only
- **AND** announce the selected change and how to override it

#### Scenario: Existing staged state is resumed

- **WHEN** status reports prepared, validated, committing, or completed state
- **THEN** the skill SHALL report that state and follow only its structured legal next actions
- **AND** SHALL NOT create another plan or inspect internal state files

### Requirement: Artifact Completion Check

The skill SHALL present artifact-completion findings returned by prepare before candidate work.

#### Scenario: Incomplete artifacts warning

- **WHEN** prepare reports artifacts other than `done` or `skipped`
- **THEN** the skill SHALL display the affected artifacts
- **AND** ask the user whether to continue
- **AND** abort the uncommitted plan if the user cancels

#### Scenario: All artifacts complete

- **WHEN** prepare reports every artifact as `done` or `skipped`
- **THEN** the skill SHALL proceed without an artifact warning

### Requirement: Task Completion Check

The skill SHALL present task-completion findings returned by prepare before candidate work.

#### Scenario: Incomplete tasks found

- **WHEN** prepare reports incomplete tasks
- **THEN** the skill SHALL display the incomplete count
- **AND** ask the user whether to continue
- **AND** abort the uncommitted plan if the user cancels

#### Scenario: All tasks complete

- **WHEN** prepare reports all tasks complete
- **THEN** the skill SHALL proceed without a task warning

#### Scenario: No tasks output

- **WHEN** prepare reports no tasks artifact output
- **THEN** the skill SHALL proceed without a task warning

### Requirement: Spec Sync Prompt

The skill SHALL reconcile deltas only into plan-owned candidates and SHALL validate an immutable review snapshot before finalization.

#### Scenario: Prepare returns candidate work

- **WHEN** prepare returns non-null `agentWork`
- **THEN** the skill SHALL use its context, guidance, rules, work items, and scopes as the complete sync input
- **AND** SHALL NOT rerun discovery/instruction commands or inspect the plan manifest
- **AND** SHALL execute the returned structured validate action after candidate work

#### Scenario: Prepare returns move-only work

- **WHEN** prepare returns `agentWork` as `null`
- **THEN** the skill SHALL execute the returned validate action without invoking spec sync

#### Scenario: User excludes selected capabilities

- **WHEN** the user requests single-change archive without reconciling one or more discovered capabilities
- **THEN** the skill SHALL prepare with repeated `--exclude-spec` values for those capabilities
- **AND** SHALL present the complete included/excluded partition before candidate work
- **AND** SHALL report excluded deltas as not synced in the final result

#### Scenario: User proceeds with reconciliation

- **WHEN** the user accepts the proposed reconciliation scope
- **THEN** the skill SHALL run candidate-mode sync inline
- **AND** write only paths in `agentWork.writeScope`
- **AND** invoke validate for the selected change
- **AND** present the complete returned current-validation review before final approval
- **AND** retain the validation identifier only through the structured finalize action

#### Scenario: User archives without reconciliation

- **WHEN** the user explicitly chooses archive without spec reconciliation
- **THEN** the skill SHALL abort the uncommitted candidate plan
- **AND** prepare a new `--skip-specs` plan for the same change
- **AND** validate and confirm the move-only snapshot before finalize

### Requirement: Archive Process

The skill SHALL use CLI-owned forward-only finalization and SHALL distinguish applied specs from a completed archive.

#### Scenario: Successful archive

- **WHEN** the user confirms the immutable validation review
- **THEN** the skill SHALL execute the returned finalize command and arguments
- **AND** use the completion receipt as authority for archive name, location, and applied specs
- **AND** SHALL NOT independently write main specs or move the change

#### Scenario: Finalize is interrupted after spec writes

- **WHEN** finalize reports that reviewed specs are applied but the change remains active
- **THEN** the skill SHALL report that partial forward state accurately
- **AND** preserve and offer the returned structured resume action
- **AND** SHALL NOT invoke abort or claim rollback

#### Scenario: Finalize reports a target conflict

- **WHEN** finalize finds a target equal to neither its prepared base nor reviewed candidate
- **THEN** the skill SHALL report the conflicting target
- **AND** SHALL NOT overwrite it, reconstruct another finalize command, or claim success

#### Scenario: Finalize retry returns a completion receipt

- **WHEN** a resumed or repeated finalize returns an existing completion receipt
- **THEN** the skill SHALL report the archive as complete from that receipt

#### Scenario: A validation is superseded

- **WHEN** candidate work is validated again
- **THEN** the skill SHALL discard the prior finalize action
- **AND** SHALL present and retain only the action for the current validation

#### Scenario: Archive destination already exists unexpectedly

- **WHEN** prepare or finalize reports an unowned existing archive destination
- **THEN** the skill SHALL report the CLI diagnostic
- **AND** SHALL NOT choose another path or perform a direct move

## ADDED Requirements

### Requirement: Staged CLI Compatibility

The archive skill SHALL remain usable when installed ahead of the CLI version that introduces staged archive.

#### Scenario: CLI supports staged archive

- **WHEN** the staged status probe returns a versioned response or recognized staged diagnostic
- **THEN** the skill SHALL use the staged lifecycle

#### Scenario: Older CLI rejects staged options

- **WHEN** the CLI explicitly reports that `--stage` is unsupported
- **THEN** the skill SHALL announce legacy mode
- **AND** follow the retained legacy workflow
- **AND** explain that staged review and resumability are unavailable

#### Scenario: Supported CLI reports a real failure

- **WHEN** the CLI recognizes staged archive but status or prepare fails
- **THEN** the skill SHALL report the failure
- **AND** SHALL NOT treat it as version skew or fall back to agent-owned formal mutation
