## ADDED Requirements

### Requirement: Staged archive command surface

The archive command SHALL expose staged execution through the existing `openspec archive <change-name>` command.

#### Scenario: Prepare stage

- **WHEN** a user runs `openspec archive add-login --stage prepare --json`
- **THEN** the command SHALL prepare the single durable active plan for `add-login`
- **AND** return versioned `archivePlan`, `agentWork`, and `nextActions` fields
- **AND** include all discovered concrete specs outputs when no selection override is provided
- **AND** SHALL NOT update main specs or move the change

#### Scenario: Prepare a selected specs subset

- **WHEN** a user repeats `--include-spec <capability>` during prepare
- **THEN** the command SHALL include exactly the named discovered capabilities
- **AND** report all other discovered capabilities as excluded

#### Scenario: Prepare excluding selected specs

- **WHEN** a user repeats `--exclude-spec <capability>` during prepare
- **THEN** the command SHALL exclude exactly the named discovered capabilities
- **AND** include and report every other discovered capability

#### Scenario: Prepare finds existing staged state

- **WHEN** prepare is requested for a change that already has prepared, validated, committing, or completed state
- **THEN** the command SHALL NOT publish a second plan
- **AND** SHALL return the existing state and a structured status action

#### Scenario: Status stage

- **WHEN** a user runs `openspec archive add-login --stage status --json`
- **THEN** the command SHALL report the change's current staged state, freshness, current validation, already-applied specs, active/archive location, and legal next actions
- **AND** SHALL NOT require a plan identifier or mutate formal state

#### Scenario: Status reports a committing conflict

- **WHEN** staged status finds a commit-started target conflict
- **THEN** the command SHALL return the conflicting path and prepared-base, reviewed-candidate, and current hashes
- **AND** SHALL return explicit manual repair and resume guidance

#### Scenario: Validate stage

- **WHEN** a user runs `openspec archive add-login --stage validate --json`
- **THEN** the command SHALL validate the plan's current editable candidates and archive preconditions
- **AND** preserve the exact reviewed bytes in a new immutable validation snapshot
- **AND** return the complete deterministic review and an opaque validation identifier
- **AND** return a structured finalize action containing that identifier

#### Scenario: Finalize stage

- **WHEN** a user runs `openspec archive add-login --stage finalize --validation <validation-id> --yes --json`
- **THEN** the command SHALL apply only that immutable reviewed snapshot
- **AND** require it to be the change's current validation
- **AND** serialize formal mutation with the planning-root mutation lock
- **AND** return the authoritative archive receipt on completion

#### Scenario: Abort stage

- **WHEN** a user runs `openspec archive add-login --stage abort --json`
- **AND** formal writes have not begun
- **THEN** the command SHALL discard the plan
- **AND** report that no candidate changes were applied

#### Scenario: Cleanup stage

- **WHEN** a user runs `openspec archive --stage cleanup --json`
- **THEN** the command SHALL remove only expired completed state, superseded validations, and abandoned generated publication temporaries
- **AND** preserve and report every prepared, validated, or committing plan

### Requirement: Prepare JSON agent contract

The archive command SHALL return a prepare payload that an agent can execute without additional discovery.

#### Scenario: Candidate work is returned

- **WHEN** prepare resolves one or more spec candidates
- **THEN** `agentWork` SHALL include snapshotted context, ordered guidance/rules, explicit work items, operation summaries, read scope, and candidate-only write scope
- **AND** no status rerun, instruction lookup, artifact discovery, path derivation, or manifest read SHALL be required

#### Scenario: Included and excluded deltas are returned

- **WHEN** prepare selects a subset
- **THEN** `archivePlan.specSelection` SHALL distinguish included and excluded capabilities
- **AND** `agentWork` SHALL contain only included inputs and candidates

#### Scenario: Follow-up commands are returned

- **WHEN** prepare succeeds
- **THEN** `nextActions` SHALL provide status, validate, and abort as separate `command` and `args` values
- **AND** consumers SHALL NOT need to shell-quote a preformatted command string

#### Scenario: Move-only plan is returned

- **WHEN** prepare creates no spec candidates
- **THEN** `agentWork` SHALL be `null`
- **AND** `nextActions` SHALL still provide status, validate, and abort

#### Scenario: Internal authority is omitted

- **WHEN** prepare serializes `agentWork`
- **THEN** prepared hashes, root fingerprints, validation snapshots, commit markers, lock paths, and receipts SHALL NOT be included

### Requirement: Immutable validate review contract

The archive command SHALL bind each validation identifier to an immutable candidate and review snapshot.

#### Scenario: Candidate review is returned

- **WHEN** candidate validation succeeds
- **THEN** the response SHALL contain the complete unified diff, statistics, base hash, candidate hash, and review hash for every candidate
- **AND** SHALL identify the immutable validation snapshot reviewed by the user

#### Scenario: Move-only review is returned

- **WHEN** validation succeeds for a plan without candidates
- **THEN** the response SHALL contain an empty review-file list
- **AND** SHALL still return a validation identifier and structured finalize action

#### Scenario: Candidate is validated again

- **WHEN** a candidate is validated more than once
- **THEN** each successful validation SHALL create a separate immutable snapshot
- **AND** the newest snapshot SHALL become the current validation
- **AND** prior validation identifiers SHALL be rejected by finalize

### Requirement: Idempotent finalize contract

The archive command SHALL resume a previously started finalize without rolling back reviewed main-spec writes.

#### Scenario: Included main spec changed after prepare

- **WHEN** validate or finalize finds an included main spec equal to neither its prepared base state nor its reviewed candidate
- **THEN** the command SHALL fail with a stale or conflict diagnostic
- **AND** SHALL NOT move the change
- **AND** SHALL NOT overwrite the changed main spec

#### Scenario: Finalize receives a superseded validation

- **WHEN** finalize receives a validation identifier that is not the plan's current validation
- **THEN** the command SHALL reject it without formal mutation
- **AND** SHALL return the current status and legal next actions

#### Scenario: Retry after partial spec writes

- **WHEN** finalize is retried with the validation identifier recorded by the commit marker
- **THEN** targets already equal to the reviewed snapshot SHALL be treated as complete
- **AND** targets still equal to their prepared base SHALL receive the reviewed snapshot
- **AND** any other target SHALL cause a conflict diagnostic without overwrite

#### Scenario: Retry after completed archive

- **WHEN** a caller retries finalize after completion
- **THEN** the command SHALL return the retained completion receipt
- **AND** SHALL NOT repeat formal writes or the change move

#### Scenario: Rename failure is retryable

- **WHEN** formal spec writes complete but directory rename fails without moving the source
- **THEN** the response SHALL report which specs are already applied
- **AND** identify the change as still active
- **AND** provide a structured resume action

#### Scenario: Commit already started

- **WHEN** abort is requested after the plan's commit marker exists
- **THEN** the command SHALL reject abort with an actionable commit-started diagnostic
- **AND** direct the caller to resume finalize

### Requirement: Staged archive option validation

The archive command SHALL reject ambiguous or unsafe staged option combinations.

#### Scenario: Change-bound stage omits a change

- **WHEN** `prepare`, `status`, `validate`, `finalize`, or `abort` is requested without a change name
- **THEN** the command SHALL fail with an actionable missing-change diagnostic

#### Scenario: Cleanup receives a change

- **WHEN** root-level `cleanup` is requested with a change name
- **THEN** the command SHALL reject the unsupported combination

#### Scenario: Identifier is malformed

- **WHEN** a caller supplies a validation identifier outside the exact generated UUID form
- **THEN** the command SHALL reject it before resolving a validation path

#### Scenario: Staged archive requests no-validate

- **WHEN** any staged invocation includes `--no-validate`
- **THEN** the command SHALL reject the combination

#### Scenario: Prepare combines selection and skip

- **WHEN** prepare receives `--skip-specs` with `--include-spec` or `--exclude-spec`
- **THEN** the command SHALL fail without publishing a plan

#### Scenario: Prepare receives invalid capability selection

- **WHEN** prepare receives an unknown or duplicate include/exclude value, or the same capability appears in both sets
- **THEN** the command SHALL fail with `archive_spec_selection_invalid`
- **AND** SHALL NOT publish a plan

#### Scenario: A non-prepare stage receives selection

- **WHEN** status, validate, finalize, abort, or cleanup receives `--include-spec`, `--exclude-spec`, or `--skip-specs`
- **THEN** the command SHALL reject the stage-option combination

#### Scenario: Finalize omits validation identity

- **WHEN** finalize is requested without `--validation`
- **THEN** the command SHALL fail with `archive_validation_required`

#### Scenario: JSON finalization lacks confirmation

- **WHEN** `finalize --json` is requested without `--yes`
- **THEN** the command SHALL fail with `archive_confirmation_required`
- **AND** leave the plan resumable

### Requirement: Direct archive compatibility

The archive command SHALL preserve its existing direct interface while coordinating formal mutation with staged finalize.

#### Scenario: Archive without stage

- **WHEN** a user runs `openspec archive <change-name>` without `--stage`
- **THEN** existing flags, prompts, merge behavior, output, and JSON contracts SHALL retain their meaning
- **AND** formal direct archive mutation SHALL participate in the planning-root mutation lock

#### Scenario: Direct archive waits behind staged finalize

- **WHEN** direct archive reaches formal mutation while staged finalize or standalone sync holds the root mutation lock
- **THEN** direct archive SHALL wait within the documented bound or return an actionable busy result
- **AND** SHALL recheck its archive preconditions and the main-spec bases used for rebuilding after acquiring the lock
- **AND** SHALL rebuild and revalidate from the locked current state before writing when a base changed

### Requirement: Cross-platform staged path and rename behavior

The archive command SHALL resolve staged paths consistently and SHALL not publish copy/remove fallback archives.

#### Scenario: Windows plan workspace

- **WHEN** staged archive runs on Windows
- **THEN** returned plan/candidate paths SHALL use resolved Windows paths
- **AND** safety checks SHALL reject traversal, drive changes, case aliases, device names, alternate data streams, and target escapes

#### Scenario: POSIX plan workspace

- **WHEN** staged archive runs on macOS or Linux
- **THEN** returned plan/candidate paths SHALL use resolved POSIX paths
- **AND** safety checks SHALL reject traversal, symlink escape, and targets outside the selected root

#### Scenario: Rename is temporarily unavailable

- **WHEN** same-filesystem staged rename fails because the directory is in use
- **THEN** the command SHALL preserve the active change and resumable commit
- **AND** return an actionable retryable diagnostic

#### Scenario: Cross-filesystem move is requested

- **WHEN** staged rename reports a cross-device condition
- **THEN** the command SHALL return `archive_cross_device_unsupported`
- **AND** SHALL NOT copy the change to the final or temporary archive destination
