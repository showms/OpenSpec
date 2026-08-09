## MODIFIED Requirements

### Requirement: OPSX Archive Skill

The system SHALL provide an `/opsx:archive` skill that prepares, semantically reconciles, validates, confirms, and finalizes a completed change through the versioned archive-attempt contract.

#### Scenario: Archive a change with all artifacts complete

- **WHEN** agent executes `/opsx:archive` with a change name
- **AND** all prepared artifact and task checks pass
- **THEN** the agent performs any semantic spec work only in the prepared candidate workspace
- **AND** obtains successful CLI validation before asking for final confirmation
- **AND** shows the validated spec effect, baseline-to-final-state diff, and requirement-operation counts before that confirmation
- **AND** invokes CLI finalization after the user confirms
- **AND** displays the archived location returned by finalization

#### Scenario: Change selection prompt

- **WHEN** agent executes `/opsx:archive` without specifying a change
- **THEN** the agent infers the change from conversation context, or auto-selects it when only one active change exists
- **AND** when ambiguous, prompts user to select from available changes, showing only active changes (excludes archive/)
- **AND** announces which change was selected and how to override

#### Scenario: A resumable attempt already includes the change

- **WHEN** prepare reports one or more resumable single or batch attempts that include the selected change
- **THEN** the skill shows each attempt's identity, scope, lifecycle and lock state, trustworthy plan path, and available actions
- **AND** asks the user to resume one exact attempt, inspect or clean retained state and retry, create a distinct attempt, or cancel
- **AND** resumes only through the plan path selected by the user
- **AND** invokes `archive-attempt prepare --new-attempt --yes` only when the user explicitly chooses another attempt
- **AND** does not reuse, overwrite, or migrate candidate work implicitly

#### Scenario: Installed CLI does not support the attempt contract

- **WHEN** the skill cannot prepare a supported archive-attempt version
- **THEN** it stops before semantic work or filesystem mutation
- **AND** offers CLI upgrade guidance or the existing one-shot `openspec archive <change>` compatibility command
- **AND** does not move the change itself

### Requirement: Task Completion Check

The skill SHALL consume the prepared CLI task check before archiving and SHALL preserve the existing explicit-confirmation behavior for incomplete tasks.

#### Scenario: Incomplete tasks found

- **WHEN** prepare returns a confirmation-required warning for incomplete tasks
- **THEN** display the warning and incomplete count to the user
- **AND** prompt for confirmation to continue
- **AND** preserve that confirmation for finalization without treating guidance as an override

#### Scenario: All tasks complete

- **WHEN** the prepared task check passes
- **THEN** proceed without task-related warning

#### Scenario: No tasks file

- **WHEN** prepare reports no task blocker or warning because no tasks file exists
- **THEN** proceed without task-related warning

### Requirement: Spec Sync Prompt

The skill SHALL use prepared spec-work entries to offer semantic spec reconciliation before archiving and SHALL keep all selected work in the managed candidate workspace until finalization.

#### Scenario: Delta specs exist

- **WHEN** the prepared plan contains one or more spec-work entries
- **THEN** the skill summarizes the planned capabilities and asks whether to reconcile them before archiving
- **AND** if user cancels, discards the attempt and stops without changing main specs or moving the change
- **AND** if user confirms reconciliation, executes archive-candidate sync inline and waits for it to complete
- **AND** obtains CLI validation for every planned result
- **AND** stops without finalization when sync or validation fails

#### Scenario: User archives without spec reconciliation

- **WHEN** the user explicitly chooses to archive without syncing prepared delta specs
- **THEN** the skill records `skip` for the selected work through the archive-attempt contract
- **AND** does not expose or apply artifact rules to agent work or create candidates for a merge that will not run
- **AND** still requires successful CLI validation and final confirmation before the move

#### Scenario: No delta specs

- **WHEN** the prepared plan contains no spec-work entries
- **THEN** proceed without a sync prompt
- **AND** do not infer delta specs from directories or unrelated artifacts

### Requirement: Archive Process

The skill SHALL delegate every project mutation and archive move to CLI finalization.

#### Scenario: Successful archive

- **WHEN** the user confirms a successfully validated archive attempt
- **THEN** the skill invokes finalization with the plan and validation token
- **AND** accepts the dated archive name resolved from the local date when finalization starts
- **AND** reports the exact archive name and path returned by the CLI
- **AND** preserves `.openspec.yaml` as part of the CLI-owned change move

#### Scenario: Archive already exists

- **WHEN** prepare, validation, or finalization reports that the target archive already exists
- **THEN** the skill stops without writing main specs or moving the change
- **AND** asks the user to inspect and resolve the existing archive destination
- **AND** does not suggest choosing another date, overwriting the destination, or adding an automatic suffix

#### Scenario: User declines final confirmation

- **WHEN** the merge is validated and the user declines final archive confirmation
- **THEN** the skill discards the attempt
- **AND** main specs and the active change remain unchanged

#### Scenario: Previous finalization ended abruptly
- **WHEN** the skill finds retained attempt or lock state from a finalizing process that no longer runs
- **THEN** it shows the observed active-change, archive, main-spec, attempt, and lock state without claiming automatic rollback
- **AND** asks the user to clean, recover or retry where supported, or retain that state
- **AND** may start a new prepare only after the user explicitly resolves stale claims and the retained-attempt decision
- **AND** accepts already-applied spec outcomes as part of the new baseline

## ADDED Requirements

### Requirement: Final Archive Review

The skill SHALL present the exact validated archive effects before requesting final confirmation.

#### Scenario: Candidate work is ready for confirmation

- **WHEN** CLI validation returns one or more prepared spec results
- **THEN** the skill labels each result as `CREATE`, `UPDATE`, `RETIRE`, `ALREADY SYNCED`, or `SKIPPED`
- **AND** shows the baseline-to-final-state diff for every created, updated, or retired target
- **AND** shows ADDED, MODIFIED, REMOVED, and RENAMED requirement counts
- **AND** presents every semantic-conflict resolution and its contributing changes
- **AND** presents every artifact-rule conflict, the user-selected controlling rule, and each suppressed conflicting rule
- **AND** makes clear that the displayed archive path is resolved using the local date when finalization starts
