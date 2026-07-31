## Purpose

Define a reviewable archive workflow that keeps semantic spec reconciliation agent-driven while OpenSpec owns exact reviewed snapshots, freshness checks, serialized formal writes, resumable progress, and the final change move.

## ADDED Requirements

### Requirement: Prepared archive lifecycle

The system SHALL support a durable archive plan that separates candidate editing and review from formal main-spec writes and change movement.

#### Scenario: Prepare creates a reviewable plan

- **WHEN** a user prepares an archive for an active change
- **THEN** the system SHALL create the change's single active plan under `openspec/.archive-plan/`
- **AND** return its state, readiness findings, selected delta inputs, candidate paths, and archive destination
- **AND** leave main specs and the active change location unchanged

#### Scenario: Prepare selects all discovered specs

- **WHEN** prepare is invoked without `--include-spec` or `--skip-specs`
- **THEN** the system SHALL include every concrete specs output discovered for that change

#### Scenario: Prepare selects a capability subset

- **WHEN** prepare is invoked with repeated `--include-spec <capability>` values
- **THEN** the system SHALL create candidates for exactly those discovered capabilities
- **AND** record every other discovered delta as excluded
- **AND** excluded deltas SHALL NOT gain main-spec write authority

#### Scenario: Prepare excludes selected capabilities

- **WHEN** prepare is invoked with repeated `--exclude-spec <capability>` values
- **THEN** the system SHALL include every discovered capability except those named
- **AND** SHALL report the resulting complete included and excluded sets

#### Scenario: Include and exclude selections conflict

- **WHEN** a capability is duplicated, unknown, or appears in both include and exclude selections
- **THEN** prepare SHALL fail before publishing a plan
- **AND** SHALL report the invalid capability selection

#### Scenario: Prepare skips all specs

- **WHEN** prepare is invoked with `--skip-specs`
- **THEN** the system SHALL record a no-spec selection and create no candidates
- **AND** the change SHALL remain eligible for validation and finalization

#### Scenario: Plan survives process boundaries

- **WHEN** prepare completes and the invoking process exits
- **THEN** a later process SHALL be able to inspect, validate, finalize, or abort the plan using its change name
- **AND** the plan SHALL remain bound to its prepared planning root and canonical change identity

#### Scenario: Schema has no specs artifact

- **WHEN** the selected schema exposes no concrete `specs` artifact outputs
- **THEN** prepare SHALL create a move-only plan
- **AND** SHALL NOT infer delta specs from unrelated artifacts

### Requirement: Single discoverable plan per change

The system SHALL keep at most one staged archive plan for a change and expose its recoverable state by change name.

#### Scenario: Two sessions prepare the same change

- **WHEN** a second session prepares a change that already has prepared, validated, committing, or completed state
- **THEN** the system SHALL NOT create another candidate workspace
- **AND** SHALL return the existing state and a structured status action
- **AND** neither session SHALL hold a mutation lock while performing candidate work or review

#### Scenario: Inspect current plan state

- **WHEN** a user requests staged status for a change
- **THEN** the system SHALL report `none`, `prepared`, `validated`, `committing`, or `completed`
- **AND** SHALL report the current validation identity, already-applied targets, active/archive location, freshness diagnostics, and legal next actions when applicable
- **AND** SHALL NOT require a retained plan identifier or internal manifest inspection

#### Scenario: Inspect a committing conflict

- **WHEN** status finds a committing plan with a target conflict
- **THEN** it SHALL report the target, prepared-base, reviewed-candidate, and current hashes
- **AND** SHALL provide explicit preserve-newer-work and resume guidance
- **AND** SHALL NOT silently rebind the commit, discard external work, or claim that abort is available

#### Scenario: Plan publication is interrupted

- **WHEN** a process stops before the complete plan workspace is published at its deterministic change-bound path
- **THEN** the incomplete workspace SHALL NOT be loadable as a prepared plan
- **AND** a previously published state for another change SHALL remain unaffected

### Requirement: Self-contained agent work package

The system SHALL make the prepare response sufficient for candidate reconciliation without rediscovering archive inputs or machine-owned state.

#### Scenario: Prepare returns candidate work

- **WHEN** a plan contains spec candidates
- **THEN** prepare SHALL return project context, ordered archive guidance, ordered specs rules, explicit work items, and parsed operation summaries
- **AND** each work item SHALL identify its capability, delta, optional base, candidate, and eventual target
- **AND** the response SHALL provide exact read scope and candidate-only write scope
- **AND** the response SHALL provide structured validate and abort actions

#### Scenario: Agent receives an exact write boundary

- **WHEN** prepare returns candidate work
- **THEN** only plan-owned candidate paths SHALL appear in the write scope
- **AND** eventual main-spec targets SHALL be identified as non-writable until finalize

#### Scenario: Prepare returns move-only work

- **WHEN** spec reconciliation is skipped or the schema has no concrete specs outputs
- **THEN** prepare SHALL return `agentWork` as `null`
- **AND** SHALL still return structured validate and abort actions

#### Scenario: Machine-owned state stays internal

- **WHEN** prepare serializes agent work
- **THEN** prepared hashes, root fingerprints, commit markers, validation snapshots, and completion receipts SHALL remain outside the agent work contract

#### Scenario: Prompt inputs are not persisted as plan authority

- **WHEN** prepare returns project context, archive guidance, or specs rules
- **THEN** the durable plan SHALL retain only hashes for their source inputs rather than their text
- **AND** status SHALL reconstruct candidate work only while those source hashes still match
- **AND** SHALL require abort and re-prepare when the prompt inputs changed

### Requirement: Candidate spec boundary

The system SHALL isolate agent-authored reconciliation in plan-owned candidates until a reviewed snapshot is finalized.

#### Scenario: Existing main spec receives a candidate

- **WHEN** a delta targets an existing main spec
- **THEN** prepare SHALL initialize the candidate from the exact main-spec bytes
- **AND** the agent SHALL reconcile the delta into the candidate rather than the main spec

#### Scenario: New capability receives a candidate

- **WHEN** a delta targets a capability without a main spec
- **THEN** prepare SHALL provide a plan-owned canonical candidate skeleton
- **AND** the candidate SHALL use the existing new-capability Purpose behavior

#### Scenario: Candidate paths are explicit

- **WHEN** prepare returns candidate work
- **THEN** every included delta SHALL have an explicit candidate and eventual target mapping
- **AND** consumers SHALL NOT discover candidate authority through glob or pattern matching

### Requirement: Immutable validation snapshot

The system SHALL preserve the exact candidate bytes and review associated with each validation identifier.

#### Scenario: Candidate validation succeeds

- **WHEN** a user validates a prepared candidate
- **THEN** the system SHALL verify parsing, canonical main-spec structure, safe paths, and source/base freshness
- **AND** SHALL verify required structural outcomes for every selected delta operation
- **AND** SHALL preserve unaffected requirements and scenarios
- **AND** SHALL copy the exact reviewed candidate bytes into an immutable validation snapshot
- **AND** SHALL return a complete deterministic diff, statistics, and an opaque validation identifier
- **AND** SHALL leave main specs and the active change unchanged

#### Scenario: Move-only validation succeeds

- **WHEN** a plan contains no candidates
- **THEN** validation SHALL create an immutable validation record with an empty review-file list
- **AND** SHALL bind it to the reviewed warnings, selection, archive destination, and change inventory

#### Scenario: Required delta outcome is missing

- **WHEN** an added requirement is absent, a removed requirement remains, a rename is incomplete, or a modified requirement/scenario is absent
- **THEN** validation SHALL fail with an actionable capability-specific diagnostic
- **AND** SHALL NOT issue a validation identifier for that candidate snapshot

#### Scenario: Existing unaffected content is lost

- **WHEN** a candidate drops an existing requirement or scenario not named for removal or replacement
- **THEN** validation SHALL fail before any main-spec write

#### Scenario: Editable candidate changes after validation

- **WHEN** an editable candidate changes after a successful validation
- **THEN** the previous validation identifier SHALL continue to identify only its immutable reviewed bytes
- **AND** validating the new candidate SHALL issue a separate identifier and snapshot
- **AND** SHALL make the previous identifier ineligible for finalize

#### Scenario: Validation is superseded

- **WHEN** the same plan is validated again
- **THEN** the new immutable snapshot SHALL become the plan's current validation
- **AND** finalize SHALL accept only that current validation identifier
- **AND** superseded snapshots SHALL be eligible for explicit generated-state cleanup

### Requirement: Serialized forward-only finalization

The system SHALL serialize formal archive mutation per planning root and make an interrupted commit safe to resume without rolling back reviewed spec writes.

#### Scenario: Finalize a reviewed plan

- **WHEN** the user confirms finalization with a validation identifier
- **AND** that identifier is the plan's current validation
- **AND** source, base, target, and archive freshness checks pass after acquiring the root mutation lock
- **THEN** the system SHALL write the exact immutable validation snapshot to included main-spec targets
- **AND** SHALL rename the active change to its prepared archive destination
- **AND** SHALL write and return an authoritative completion receipt

#### Scenario: Target is still at its prepared base

- **WHEN** finalize examines an included target whose current state equals the prepared base hash or prepared absence
- **THEN** finalize SHALL atomically write the reviewed candidate snapshot

#### Scenario: Target already equals the reviewed candidate

- **WHEN** finalize or a retry examines an included target already equal to the reviewed candidate hash
- **THEN** finalize SHALL treat that target as completed
- **AND** SHALL NOT rewrite or roll it back

#### Scenario: Target contains unrelated newer work

- **WHEN** an included target equals neither its prepared base state nor the reviewed candidate
- **THEN** finalize SHALL stop with a commit-conflict diagnostic
- **AND** SHALL NOT overwrite or roll back that target

#### Scenario: Standalone sync changes an included main spec

- **WHEN** standalone sync commits a different version of an included main spec after archive prepare
- **THEN** archive validation or finalization SHALL report the prepared base as stale or conflicted
- **AND** SHALL NOT archive the change from that stale plan
- **AND** SHALL direct the user to abort and prepare again from the new main-spec state

#### Scenario: Initial target preflight finds a conflict

- **WHEN** finalize finds a target conflict before any formal write has begun
- **THEN** the system SHALL NOT create the commit marker
- **AND** the user SHALL remain able to abort the uncommitted plan

#### Scenario: Process stops after some spec writes

- **WHEN** finalization stops after writing some reviewed main specs but before moving the change
- **THEN** those reviewed writes SHALL remain applied
- **AND** the plan SHALL retain its commit marker and immutable validation snapshot
- **AND** a later finalize with the same validation identifier SHALL resume remaining work

#### Scenario: Archive rename fails

- **WHEN** every included target equals the reviewed snapshot but the change-directory rename fails
- **THEN** the system SHALL leave the reviewed main specs applied
- **AND** leave the change active
- **AND** return an actionable retryable diagnostic and structured resume action
- **AND** SHALL NOT report the archive as complete

#### Scenario: Process stops after archive rename

- **WHEN** the active source is absent and the prepared archive destination matches the prepared change inventory
- **AND** every included target matches the reviewed snapshot
- **THEN** a retry SHALL write the completion receipt and report the archive as completed

#### Scenario: Caller retries a completed plan

- **WHEN** finalize receives a plan with a matching retained completion receipt
- **THEN** it SHALL return the same authoritative archive result without repeating formal writes

### Requirement: Commit-time abort boundary

The system SHALL allow a plan to be discarded before formal writes begin and SHALL require forward recovery after they begin.

#### Scenario: Abort an uncommitted plan

- **WHEN** a user aborts a plan without a commit marker
- **THEN** the system SHALL remove its candidates and validation snapshots
- **AND** SHALL leave main specs and the active change unchanged

#### Scenario: Abort after commit starts

- **WHEN** a user attempts to abort a plan with a commit marker
- **THEN** the system SHALL reject abort
- **AND** direct the user to resume finalize or repair an explicitly reported conflict
- **AND** SHALL NOT roll back reviewed main-spec writes

#### Scenario: Abort a completed plan

- **WHEN** abort receives a completed plan
- **THEN** it SHALL report the retained completion result
- **AND** SHALL NOT describe the committed archive as aborted

### Requirement: Explicit durable-state cleanup

The system SHALL retain recovery-critical state and remove generated state only through bounded explicit rules.

#### Scenario: Cleanup encounters completed and superseded state

- **WHEN** root-level cleanup finds an expired completed receipt, superseded validation snapshot, or abandoned generated publication temporary
- **THEN** it SHALL remove only entries identified through the generated-state model
- **AND** SHALL report what it removed

#### Scenario: Cleanup encounters active recovery state

- **WHEN** cleanup finds a prepared, validated, or committing plan
- **THEN** it SHALL preserve that plan
- **AND** SHALL report its change and current state

### Requirement: Root-level formal mutation coordination

The system SHALL use one owner-aware mutation lock per planning root for OpenSpec-owned formal main-spec writes and archive movement.

#### Scenario: Another archive commit is running

- **WHEN** staged finalize, direct archive, or standalone sync formal commit attempts mutation while the root mutation lock is held
- **THEN** the later operation SHALL wait within its documented bound or return an actionable busy diagnostic
- **AND** SHALL recheck freshness after acquiring the lock

#### Scenario: Unrelated changes finalize

- **WHEN** two changes have plans that target unrelated specs under the same planning root
- **THEN** their prepare, candidate work, and validation MAY proceed independently
- **AND** their formal finalization SHALL run serially

#### Scenario: Standalone sync commits candidate work

- **WHEN** standalone sync finishes agent-driven reconciliation
- **THEN** a CLI-owned formal commit SHALL acquire the root mutation lock
- **AND** SHALL recheck every selected main-spec base hash before writing
- **AND** SHALL atomically apply and verify all conflict-free candidate bytes
- **AND** a base mismatch SHALL leave all selected main specs unchanged

#### Scenario: Lock owner releases the lock

- **WHEN** a process releases the root mutation lock
- **THEN** it SHALL remove the lock only when the stored owner token matches that process
- **AND** SHALL NOT steal or remove another live owner's lock solely because of elapsed time

### Requirement: Rename-only staged movement

The staged workflow SHALL move changes only through a same-filesystem rename in this version.

#### Scenario: Same-filesystem rename succeeds

- **WHEN** the active change and prepared archive destination support rename
- **THEN** finalize SHALL rename the complete change directory without a copy/remove fallback

#### Scenario: Windows rename is temporarily blocked

- **WHEN** Windows reports a sharing or permission condition that prevents rename
- **THEN** finalize SHALL return a retryable diagnostic explaining that open handles may need to be released
- **AND** SHALL keep the active change and resumable plan

#### Scenario: Cross-filesystem rename is requested

- **WHEN** rename fails with a cross-device condition on Windows, macOS, or Linux
- **THEN** staged finalize SHALL report that automatic cross-filesystem movement is unsupported
- **AND** SHALL NOT create or publish a partial archive copy

### Requirement: Defensive path and inventory checks

The system SHALL reject path aliases and filesystem drift that could redirect plan authority outside the selected root.

#### Scenario: Caller supplies an invalid identifier

- **WHEN** a validation identifier is outside the exact generated UUID form
- **THEN** the system SHALL reject it before resolving or reading a validation path

#### Scenario: Windows path alias is unsafe

- **WHEN** a Windows path uses a drive change, case alias, device name, alternate data stream, trailing dot/space, or slash variant that escapes or aliases a protected path
- **THEN** the system SHALL reject the plan or operation before formal mutation

#### Scenario: POSIX path escapes through a symlink

- **WHEN** a candidate, target, or plan path resolves outside its allowed root through a symlink
- **THEN** the system SHALL reject the operation before formal mutation

#### Scenario: Change tree changes after prepare

- **WHEN** a regular file, directory entry, or recorded symlink target in the active change differs from the prepared inventory
- **THEN** validation/finalize SHALL report the plan as stale
- **AND** SHALL require a newly prepared plan

### Requirement: Bulk staged archive behavior

The bulk archive workflow SHALL compose reviewed single-change commits without silently expanding prepared scope.

#### Scenario: Batch scope is reviewed before mutation

- **WHEN** multiple changes are selected
- **THEN** bulk SHALL inspect every selected change and resolve capability order before preparing the first formal commit
- **AND** no main spec or change location SHALL be modified during inspection

#### Scenario: Bulk uses explicit capability selection

- **WHEN** bulk prepares a change after inspecting its concrete specs outputs
- **THEN** it SHALL pass every included capability explicitly with repeated `--include-spec`
- **AND** when at least one capability is included, SHALL pass every explicitly excluded capability with repeated `--exclude-spec`
- **AND** when no capability is included, SHALL use `--skip-specs` and expect every discovered capability in the returned excluded set
- **AND** SHALL compare prepare's returned complete partition with the inspected partition before candidate work
- **AND** a mismatch SHALL abort the uncommitted plan and report selection drift
- **AND** a newly added delta SHALL NOT silently enter the previously reviewed batch scope

#### Scenario: Overlapping changes are processed sequentially

- **WHEN** selected changes update the same capability
- **THEN** bulk SHALL complete prepare, candidate work, validate, confirm, and finalize for the earlier change
- **AND** SHALL prepare the later change from the earlier committed main spec

#### Scenario: One bulk commit fails

- **WHEN** one change fails or becomes retryable during a batch
- **THEN** already completed commits SHALL remain completed
- **AND** dependent later changes SHALL be skipped
- **AND** unrelated changes MAY continue

#### Scenario: Mixed-schema batch

- **WHEN** a batch contains changes with and without concrete specs outputs
- **THEN** every change SHALL follow prepare, validate, confirm, and finalize
- **AND** only changes with included concrete specs outputs SHALL receive candidate work
