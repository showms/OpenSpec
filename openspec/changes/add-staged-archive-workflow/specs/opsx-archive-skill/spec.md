## MODIFIED Requirements

### Requirement: OPSX Archive Skill

The system SHALL provide an `/opsx:archive` skill that sequentially orchestrates a completed change through the short-locked staged archive CLI lifecycle.

#### Scenario: Archive a change with all artifacts complete

- **WHEN** agent executes `/opsx:archive` with a change name
- **AND** all artifacts in the schema are complete
- **AND** all tasks are complete
- **THEN** the agent SHALL inspect staged status for the change
- **AND** prepare only when no staged state exists
- **AND** reconcile only plan-owned candidates returned by prepare
- **AND** validate and present the complete immutable formal-spec diff, archive payload manifest, and review delivery identity
- **AND** execute the approval-token-bound finalize action only after user approval
- **AND** display the CLI completion receipt with the archived location

#### Scenario: Change selection prompt

- **WHEN** agent executes `/opsx:archive` without specifying a change
- **THEN** the agent SHALL infer the change from conversation context or auto-select it when only one active change exists
- **AND** when ambiguous, prompt from active changes only
- **AND** announce the selected change and how to override it

#### Scenario: Existing staged state is resumed

- **WHEN** staged status reports prepared, validated, committing, conflicted, broken, orphaned, or completed state
- **THEN** the skill SHALL report that state and follow only its structured legal next actions
- **AND** SHALL NOT create another plan, inspect internal plan files outside returned evidence scopes, or construct replacement staged or repair commands

### Requirement: Artifact Completion Check

The skill SHALL present artifact-completion findings returned by staged prepare before candidate work.

#### Scenario: Incomplete artifacts warning

- **WHEN** prepare reports one or more artifacts with status other than `done` or `skipped`
- **THEN** display a warning listing incomplete artifacts
- **AND** prompt user for confirmation to continue
- **AND** proceed if user confirms
- **AND** abort the uncommitted plan if the user cancels

#### Scenario: All artifacts complete

- **WHEN** prepare reports every artifact as `done` or `skipped`
- **THEN** proceed without an artifact warning

### Requirement: Task Completion Check

The skill SHALL present task-completion findings returned by staged prepare before candidate work.

#### Scenario: Incomplete tasks found

- **WHEN** prepare reports incomplete tasks
- **THEN** display a warning showing the incomplete count
- **AND** prompt user for confirmation to continue
- **AND** proceed if user confirms
- **AND** abort the uncommitted plan if the user cancels

#### Scenario: All tasks complete

- **WHEN** prepare reports all tasks complete
- **THEN** proceed without a task-related warning

#### Scenario: No tasks file

- **WHEN** prepare reports no tasks artifact output
- **THEN** proceed without a task-related warning

### Requirement: Spec Sync Prompt

The skill SHALL reconcile selected delta specs only into staged candidate files and SHALL use CLI validation as the review boundary.

#### Scenario: Delta specs exist

- **WHEN** prepare returns one or more included candidate work items
- **THEN** the skill SHALL present the complete included/excluded capability partition
- **AND** prompt the user to proceed with the selected reconciliation scope, archive without syncing, or cancel
- **AND** if reconciliation is selected, execute archive-supplied candidate sync inline and wait for it to finish
- **AND** write only paths in the returned write scope
- **AND** execute the returned validate action after candidate work
- **AND** make the complete immutable formal-spec diff, payload manifest, review path, hash, byte length, and approval identity available before final approval
- **AND** stop without finalizing when sync or validation fails

#### Scenario: No delta specs

- **WHEN** prepare returns `agentWork: null`
- **THEN** the skill SHALL proceed to the returned move-only validate action without invoking spec sync

#### Scenario: User archives without spec reconciliation

- **WHEN** the user explicitly chooses archive without syncing selected delta specs
- **THEN** the skill SHALL abort the uncommitted candidate plan
- **AND** prepare a new move-only plan with `--skip-specs`
- **AND** validate and present that move-only review before final approval

#### Scenario: Move-only review contains excluded deltas

- **WHEN** validation returns an empty formal-spec diff for a change that has discovered excluded deltas
- **THEN** the skill SHALL list every capability that will remain unsynced
- **AND** present the complete archive payload manifest before asking for approval

#### Scenario: Review is delivered by file

- **WHEN** validation reports `delivery: file`
- **THEN** the skill SHALL present the review statistics, byte length, hash, and complete review path
- **AND** use the returned structured read action to inspect the complete file before requesting final approval
- **AND** present the payload-manifest hash and bind the approval request to the exact path, hash, byte length, delivery mode, and payload manifest represented by the returned approval token
- **AND** explain that this approval covers reviewed spec and payload content but does not bind the final archive date, name, or path
- **AND** SHALL NOT call a summary or truncated excerpt the complete review

#### Scenario: User declines final staged review

- **WHEN** the user does not approve the current immutable review
- **THEN** the skill SHALL stop without executing finalize
- **AND** report that the validated plan remains available through status or can be removed through the returned abort action
- **AND** SHALL NOT reinterpret the decline as permission to archive without syncing

### Requirement: Archive Process

The skill SHALL delegate formal writes and archive movement to the short-locked staged finalizer.

#### Scenario: Successful archive

- **WHEN** the user confirms the current immutable validation review
- **THEN** the skill SHALL execute the returned finalize command and arguments containing the matching approval token
- **AND** use the completion receipt as authority for archive name, location, applied specs, skipped specs, warnings, and schema
- **AND** SHALL NOT independently write main specs or move the change
- **AND** preserve `.openspec.yaml` through CLI-owned movement

#### Scenario: Commit-bound archive destination conflicts

- **WHEN** finalize reports `archive_destination_conflict`
- **THEN** the skill SHALL inspect the returned source, destination, marker, receipt, and commit evidence
- **AND** explain the likely ownership problem and present safe recovery options
- **AND** when the source lineage is complete, offer only the returned evidence-bound `rebind-destination` repair with a user-supplied safe archive basename
- **AND** request explicit user approval before executing that repair
- **AND** SHALL NOT choose another path itself, overwrite the destination, abort the commit, or perform a direct move

#### Scenario: Finalize is interrupted

- **WHEN** finalize reports that reviewed specs are partially or fully applied while the change remains active
- **THEN** the skill SHALL report applied, pending, and movement state accurately
- **AND** preserve and offer the returned resume action
- **AND** SHALL NOT claim rollback or completion

#### Scenario: Finalize reports a conflict

- **WHEN** finalize reports a target that equals neither its prepared base nor reviewed snapshot
- **THEN** the skill SHALL consume the returned read-scoped `agentRecovery` evidence package
- **AND** read and compare the persisted base, immutable reviewed snapshot, current evidence, selected delta, hashes, and relevant plan metadata within the returned scopes
- **AND** diagnose the semantic difference and likely source of the conflict so the user does not have to compare files or hashes manually
- **AND** present whether restoring the original review or preserving newer work is safer
- **AND** execute the returned `prepare-spec-conflict-resolution` repair action to create a plan-owned recovery candidate only after the user selects a direction
- **AND** reconcile only that recovery candidate, run the returned recovery validate action, present the complete amendment review, and request explicit user approval before recovery finalize
- **AND** SHALL NOT silently overwrite, write formal specs directly, invoke standalone sync during commit, auto-rebase, invent authority outside the package, or claim success

#### Scenario: Status reports broken or orphaned state

- **WHEN** staged status reports `broken` or `orphaned`
- **THEN** the skill SHALL inspect only the safe evidence and agent-investigation actions returned by the CLI
- **AND** correlate plan, source, destination, marker, validation, and receipt identities
- **AND** explain the most likely failure point and the preconditions of every returned repair option in plain language
- **AND** after the user selects an option, execute its repair preview, explain that the preview writes only generated repair-review state, and present the complete effects, retained evidence, cleanup consequences, and explicit inputs
- **AND** request approval before executing the returned approval-token-bound reconstruct, resume, adopt, quarantine, or rebind action
- **AND** SHALL NOT construct a repair command, delete evidence, or treat approval as authority for an action the CLI did not return

#### Scenario: Archive commit lock is busy

- **WHEN** finalize reports `archive_commit_busy`
- **THEN** the skill SHALL report the current archive owner information returned by the CLI
- **AND** preserve and offer the returned read-only status or retry action
- **AND** SHALL NOT bypass, delete, age-steal, or reconstruct the lock

#### Scenario: Archive commit lock is abandoned

- **WHEN** status returns an evidence-bound `reclaim-archive-lock` repair decision because the same-host owner process is provably absent
- **THEN** the skill SHALL explain which invocation left the lock and that reclamation removes only the nonce-bound lock
- **AND** run the returned repair preview and request explicit user approval before executing lock reclamation
- **AND** continue only through the returned retry action

#### Scenario: Orphaned source can resume

- **WHEN** status proves that the source payload is current, the destination is absent, and only the marker is missing or foreign
- **THEN** the skill SHALL explain that `resume-source` preserves any foreign marker and restores only the commit-bound marker
- **AND** execute the returned repair only after approval
- **AND** continue through the returned finalize resume action

#### Scenario: Orphaned destination can be adopted

- **WHEN** status proves that the source is absent and the destination exactly matches the reviewed payload and recovery capsule
- **THEN** the skill SHALL explain that `adopt-destination` records already completed movement rather than moving content again
- **AND** execute the returned repair only after approval
- **AND** report completion from its receipt

#### Scenario: Both orphaned locations match

- **WHEN** status proves that source and destination contain the same reviewed lineage
- **THEN** the skill SHALL explain that the returned repair moves the duplicate source to plan-owned quarantine without deleting it
- **AND** disclose the retention and later cleanup behavior
- **AND** execute `quarantine-source-and-adopt-destination` only after approval

#### Scenario: Orphan evidence is insufficient

- **WHEN** status returns only `preserve-and-stop` or reports that all complete recovery copies are missing
- **THEN** the skill SHALL provide the complete read-only recovery report returned by status and explain which proof is missing
- **AND** SHALL NOT repair, move, delete, overwrite, reconstruct reviewed bytes, or claim completion

#### Scenario: Finalize retry returns a receipt

- **WHEN** a repeated finalize returns an existing or recovered completion receipt
- **THEN** the skill SHALL report the archive as complete from that receipt

#### Scenario: Payload changes after review

- **WHEN** finalize reports `archive_payload_changed` before commit
- **THEN** the skill SHALL show which archived payload entries changed
- **AND** execute the returned validate action and request a new approval
- **AND** SHALL NOT reuse the prior approval token

### Requirement: Skill Output

The skill SHALL provide clear staged archive state and completion feedback.

#### Scenario: Archive complete with sync

- **WHEN** archive completes after candidate reconciliation
- **THEN** display the specs applied from the receipt
- **AND** display the archive location and schema
- **AND** display the approved payload-manifest hash and any conflict-resolution amendments

#### Scenario: Archive complete without sync

- **WHEN** archive completes from a move-only validation
- **THEN** note that discovered delta specs were not synced when applicable
- **AND** display the archive location and schema

#### Scenario: Archive complete with warnings

- **WHEN** archive completes with readiness warnings
- **THEN** include the warnings from the completion receipt
- **AND** suggest review when the archive was completed with incomplete artifacts or tasks

#### Scenario: Archive concurrency notice

- **WHEN** the skill prepares or resumes a staged archive that has not completed
- **THEN** it SHALL explain that other direct, staged, and bulk archive commits are serialized by the short-lived planning-root archive lock
- **AND** tell the user not to run standalone sync or manually edit formal specs for that planning root while finalization is active

## ADDED Requirements

### Requirement: Staged CLI Compatibility

The archive skill SHALL distinguish an older CLI from a real staged archive failure.

#### Scenario: CLI supports staged archive

- **WHEN** the staged status probe returns a versioned response or recognized staged diagnostic
- **THEN** the skill SHALL use the staged lifecycle

#### Scenario: Older CLI rejects staged options

- **WHEN** the CLI explicitly reports that `--stage` is unsupported
- **THEN** the skill SHALL announce legacy mode
- **AND** follow the retained legacy workflow
- **AND** explain that immutable staged review and interruption recovery are unavailable

#### Scenario: Supported CLI reports a real failure

- **WHEN** the CLI recognizes staged archive but status, prepare, validate, or finalize fails
- **THEN** the skill SHALL report the failure
- **AND** SHALL NOT fall back to agent-owned formal writes or movement
