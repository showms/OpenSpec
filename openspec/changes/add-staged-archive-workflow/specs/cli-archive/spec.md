## ADDED Requirements

### Requirement: Staged archive command surface

The archive command SHALL expose prepare, status, validate, finalize, resolve, repair, abort, and cleanup operations through the existing command surface.

#### Scenario: Prepare stage

- **WHEN** a user runs `openspec archive <change> --stage prepare`
- **THEN** the command SHALL prepare or return the change's active staged plan
- **AND** SHALL NOT modify formal specs or move the change

#### Scenario: Status stage

- **WHEN** a user runs `openspec archive <change> --stage status`
- **THEN** the command SHALL return the derived current state and structured legal next actions
- **AND** SHALL NOT mutate staged or formal state

#### Scenario: Status finds damaged recovery state

- **WHEN** staged records are malformed or source, destination, marker, and receipt evidence cannot be assigned consistently
- **THEN** the command SHALL return `broken` or `orphaned` as appropriate
- **AND** expose only safe evidence paths and structured agent-investigation actions
- **AND** SHALL NOT infer progress, delete evidence, or mutate formal state

#### Scenario: Validate stage

- **WHEN** a user runs `openspec archive <change> --stage validate`
- **THEN** the command SHALL validate the current candidates and publish one immutable current review
- **AND** return its validation identifier and complete-review delivery metadata

#### Scenario: Finalize stage

- **WHEN** a user runs `openspec archive <change> --stage finalize --validation <id> --approval <token>` with required confirmation
- **THEN** the command SHALL finalize only the current immutable validation
- **AND** return a completion receipt, resumable status, or actionable conflict diagnostic

#### Scenario: Resolve stage

- **WHEN** a user runs `openspec archive <change> --stage resolve --recovery <id>` for the current conflict
- **THEN** the command SHALL prepare or return plan-owned recovery candidate work and its structured recovery validate action
- **AND** SHALL NOT modify the formal target, other reviewed targets, or active change

#### Scenario: Repair stage

- **WHEN** a user runs `openspec archive <change> --stage repair --recovery <id> --resolution <decision> --approval <token>` with required confirmation
- **THEN** the command SHALL execute only the evidence-bound repair decision returned by current status
- **AND** return reconstructed status, a normal resume action, a completion receipt, or a stale-evidence diagnostic

#### Scenario: Abort stage

- **WHEN** a user runs `openspec archive <change> --stage abort`
- **THEN** the command SHALL remove an uncommitted plan or explain why commit evidence must be retained

#### Scenario: Cleanup stage

- **WHEN** a user runs `openspec archive --stage cleanup`
- **THEN** the command SHALL remove only recognized disposable staged state
- **AND** preserve every active recovery plan

### Requirement: Prepare selection contract

The archive command SHALL make the selected specs scope explicit and stable before candidate work.

#### Scenario: Prepare includes every discovered spec

- **WHEN** prepare receives no include, exclude, or skip option
- **THEN** it SHALL include every concrete specs output discovered for the change
- **AND** return the complete included/excluded partition

#### Scenario: Prepare includes named capabilities

- **WHEN** prepare receives repeated `--include-spec <capability>` values
- **THEN** it SHALL include exactly those discovered capabilities
- **AND** report every other discovered capability as excluded

#### Scenario: Prepare excludes named capabilities

- **WHEN** prepare receives repeated `--exclude-spec <capability>` values
- **THEN** it SHALL include every discovered capability except those values
- **AND** report the excluded capabilities explicitly

#### Scenario: Prepare receives an explicit partition

- **WHEN** prepare receives both include and exclude values
- **THEN** the two lists SHALL be disjoint and cover the discovered capabilities exactly
- **AND** any mismatch SHALL fail with `archive_spec_selection_invalid`

#### Scenario: Prepare skips specs

- **WHEN** prepare receives `--skip-specs`
- **THEN** it SHALL select no capabilities
- **AND** return every discovered capability as excluded

#### Scenario: Prepare receives an archive name

- **WHEN** prepare receives `--archive-name <basename>`
- **THEN** it SHALL record that exact safe basename as the intended archive name
- **AND** reject separators, traversal, reserved generated names, platform aliases, and destinations outside the selected archive root

#### Scenario: Prepare observes destination or device trouble

- **WHEN** prepare can already observe that the intended destination exists or the active source and archive root are on different devices
- **THEN** it SHALL return an actionable preflight diagnostic before candidate work
- **AND** SHALL NOT silently rename, copy, remove, or publish a plan with an unusable explicit destination

### Requirement: Prepare JSON agent contract

The archive command SHALL return a complete versioned work package without exposing mutable machine authority.

#### Scenario: Candidate work is returned

- **WHEN** prepare includes one or more capabilities
- **THEN** JSON SHALL return `contractVersion`, root identity, immutable plan identifier, intended archive-name policy, archive plan status, current readiness warnings, complete discovered capability set, selection, and `agentWork`
- **AND** `agentWork` SHALL contain explicit delta/persisted-base/candidate mappings, guidance, rules, summaries, read scope, write scope, and validate action

#### Scenario: Move-only work is returned

- **WHEN** prepare includes no capabilities
- **THEN** JSON SHALL return `agentWork: null`
- **AND** include structured validate, status, and abort actions

#### Scenario: Internal authority is omitted

- **WHEN** prepare returns JSON
- **THEN** the response SHALL omit internal change keys, commit tokens, mutable state paths, and cleanup ownership records
- **AND** consumers SHALL use returned structured actions instead of constructing internal paths

### Requirement: Immutable validate review contract

The archive command SHALL make the complete current review available while bounding normal response size.

#### Scenario: Review is delivered inline

- **WHEN** the complete review is within the documented inline byte limit
- **THEN** JSON SHALL include the complete formal-spec diff and payload-manifest review body, path, hash, byte length, payload-manifest hash, statistics, validation identifier, approval token, and finalize action

#### Scenario: Review is delivered by file

- **WHEN** the complete review exceeds the documented inline byte limit
- **THEN** JSON SHALL omit the diff body
- **AND** return `delivery: file`, the complete review path, hash, byte length, payload-manifest hash, statistics, validation identifier, approval token, and structured read/finalize actions
- **AND** the finalize action SHALL carry the opaque approval token bound to that complete path, hash, byte length, delivery mode, and payload-manifest hash

#### Scenario: Review generation fails

- **WHEN** the complete review cannot be safely generated, persisted, or hashed
- **THEN** the command SHALL fail with `archive_review_generation_failed`
- **AND** SHALL NOT publish a validation identifier, approval token, or finalize action

#### Scenario: Candidate is validated again

- **WHEN** validation succeeds again before commit starts
- **THEN** the new identifier SHALL become current
- **AND** a new approval token SHALL bind only the new review
- **AND** finalize SHALL reject every superseded identifier or approval token with `archive_validation_not_current`

#### Scenario: Candidate changes after validation

- **WHEN** finalize finds candidate bytes different from the current validation snapshot
- **THEN** it SHALL fail with `archive_validation_stale`
- **AND** return the current validate action without beginning commit

#### Scenario: Archive payload changes after validation

- **WHEN** finalize finds the active-change payload manifest different from the current validation
- **THEN** it SHALL fail before commit with `archive_payload_changed` and return the validate action
- **AND** after commit it SHALL fail with `archive_source_conflict` and return safe recovery evidence

#### Scenario: Discovery or readiness changes after prepare

- **WHEN** validate or finalize rediscovers delta capability or artifact/task readiness state
- **THEN** it SHALL reject an added, removed, or renamed delta capability as `archive_spec_selection_stale`
- **AND** finalize SHALL reject readiness that differs from the current validation as `archive_readiness_changed` before commit and return the validate action
- **AND** return the freshly calculated readiness warnings for user acceptance
- **AND** SHALL NOT reuse the prepare-time readiness result as final authority

#### Scenario: Validation follows commit start

- **WHEN** validation is requested after commit state exists
- **THEN** the command SHALL reject it with the current resume or conflict action
- **AND** preserve the commit-bound validation

### Requirement: Staged final approval

The archive command SHALL keep staged review approval separate from the established direct spec-update confirmation behavior.

#### Scenario: Human staged finalize requests approval

- **WHEN** staged finalize runs interactively without `--yes`
- **THEN** it SHALL display the current validation identifier, formal specs created/updated, unsynced capabilities, complete review delivery metadata, payload-manifest hash, readiness warnings, and destination
- **AND** prompt for approval defaulting to No

#### Scenario: User declines staged final approval

- **WHEN** the user declines staged final approval
- **THEN** the command SHALL preserve the validated plan, candidates, immutable review, and approval token
- **AND** leave formal specs and the active change unchanged
- **AND** return status and abort actions

#### Scenario: Staged finalize is preconfirmed

- **WHEN** staged finalize receives `--yes`, the current validation identifier, and its bound approval token
- **THEN** it SHALL skip the interactive prompt and continue through current pre-commit checks

### Requirement: Resumable finalize contract

The archive command SHALL make a single-writer staged finalize safe to retry after interruption.

#### Scenario: Finalize receives stale inputs before commit

- **WHEN** a recorded delta, base target, or archive destination changes before commit state exists
- **THEN** finalize SHALL fail without formal writes
- **AND** return an abort and re-prepare action

#### Scenario: Retry follows partial spec writes

- **WHEN** commit state exists and some targets already equal the reviewed snapshot while others remain at their prepared bases
- **THEN** finalize SHALL skip the applied targets
- **AND** atomically write the remaining reviewed targets
- **AND** continue toward the commit-bound archive destination

#### Scenario: Retry finds conflicting content

- **WHEN** commit state exists and a target equals neither its prepared base nor reviewed snapshot
- **THEN** finalize SHALL fail with `archive_commit_conflict`
- **AND** preserve and snapshot the current target
- **AND** return status plus a read-scoped `agentRecovery` evidence package containing the persisted base, reviewed snapshot, current evidence, selected delta, hashes, relevant plan metadata, and explicit safe scopes
- **AND** return an evidence-bound resolve action for plan-owned recovery candidate work
- **AND** SHALL NOT offer a silent overwrite, automatic rebase, rollback, direct formal write, or standalone-sync action

#### Scenario: Bound destination conflicts after commit starts

- **WHEN** the commit-bound archive destination becomes occupied without the matching commit marker
- **THEN** finalize SHALL fail with `archive_destination_conflict`
- **AND** preserve the bound destination, source, applied specs, and recovery evidence
- **AND** return structured agent-investigation plus an evidence-bound rebind repair action without choosing another destination itself

#### Scenario: Source authority conflicts after commit starts

- **WHEN** a retry finds the complete delta set, selected delta bytes, or another recorded source-authority input changed after commit state exists
- **THEN** finalize SHALL fail with `archive_source_conflict`
- **AND** preserve already applied specs and return safe current evidence plus agent-investigation actions
- **AND** SHALL NOT return abort-and-prepare, continue spec writes, or move the active change

#### Scenario: Rename failure is retryable

- **WHEN** every reviewed target is applied but the staged directory rename is temporarily unavailable
- **THEN** finalize SHALL return `archive_move_retryable`
- **AND** distinguish applied specs from the still-active change
- **AND** return the exact resume action

#### Scenario: Retry follows completed movement

- **WHEN** source is absent and the commit-bound destination contains the matching commit marker
- **THEN** finalize SHALL recover and return the completion receipt
- **AND** SHALL NOT repeat formal spec writes

#### Scenario: Primary plan is recovered from source-local capsule

- **WHEN** the primary active plan is missing or damaged and one complete capsule lineage matches the current source or destination and formal targets
- **THEN** status SHALL return `broken` with an evidence-bound `reconstruct-plan` repair action
- **AND** successful repair SHALL atomically republish the primary plan before returning its derived resumable status

#### Scenario: Completed finalize is repeated

- **WHEN** a matching retained completion receipt exists
- **THEN** finalize SHALL return that same result

#### Scenario: A later change reuses a completed name

- **WHEN** prepare finds no active plan but retained receipts exist for the same change name
- **THEN** it SHALL create a new random plan identifier and reusable active slot
- **AND** SHALL leave the older receipts in 30-day history
- **AND** status SHALL NOT report the later active change as completed from an older plan identity

#### Scenario: Same-day reused name has a destination collision

- **WHEN** a later same-name change would use an archive basename that already exists
- **THEN** prepare or pre-commit finalize SHALL report the exact collision before formal writes
- **AND** return the abort-and-prepare action for an explicit `--archive-name`
- **AND** SHALL NOT infer which historical receipt or destination the caller intended

### Requirement: Evidence-bound resolve and repair contract

The archive command SHALL expose conflict resolution and orphan repair as structured, stale-safe actions rather than user-constructed filesystem commands.

#### Scenario: Resolve returns candidate work

- **WHEN** the current conflict receives `--stage resolve --recovery <id>`
- **THEN** JSON SHALL return the original base, original review, captured current target, selected delta, plan-owned recovery candidate, explicit scopes, and structured recovery validate action
- **AND** omit internal mutable authority and unrelated plan files

#### Scenario: Recovery validation succeeds

- **WHEN** `--stage validate --recovery <id>` validates the current recovery candidate against unchanged conflict evidence
- **THEN** JSON SHALL return a complete current-to-recovery amendment review, new validation identifier, approval token, and recovery finalize action
- **AND** bind them to the original validation and recovery evidence lineage

#### Scenario: Repair action is current

- **WHEN** repair receives the current recovery ID, listed decision, and all required explicit inputs without an approval token
- **THEN** it SHALL persist and return a non-mutating complete repair review plus an approval token and exact execution action
- **AND** bind the token to the evidence manifest, decision, effects, cleanup consequences, and explicit inputs

#### Scenario: Approved repair action is current

- **WHEN** repair receives the exact returned execution action with its evidence-bound approval token and required confirmation
- **THEN** it SHALL recheck the complete evidence manifest immediately before mutation
- **AND** execute only `reconstruct-plan`, `resume-source`, `adopt-destination`, `quarantine-source-and-adopt-destination`, or `rebind-destination` when that action's preconditions remain true

#### Scenario: Repair action is stale or unsupported

- **WHEN** evidence changed, the decision was not returned by current status, required bytes are missing, or supplied paths differ from the structured action
- **THEN** repair SHALL fail with `archive_recovery_stale` or `archive_repair_not_allowed`
- **AND** preserve source, destination, formal targets, markers, capsules, receipts, plan records, and quarantine
- **AND** return a fresh status action

#### Scenario: Rebind receives a new archive name

- **WHEN** an allowed `rebind-destination` repair receives a user-supplied safe basename whose destination is absent
- **THEN** it SHALL append an immutable destination amendment containing both old and new bindings
- **AND** return the normal finalize resume action
- **AND** SHALL NOT modify the occupant at the old destination

### Requirement: Staged archive option validation

The archive command SHALL reject ambiguous or unsupported staged option combinations before formal mutation.

#### Scenario: Change-bound stage omits a change

- **WHEN** prepare, status, validate, finalize, resolve, repair, or abort is requested without a change name
- **THEN** the command SHALL fail with an actionable missing-change diagnostic

#### Scenario: Cleanup receives a change

- **WHEN** root-level cleanup receives a change name
- **THEN** the command SHALL reject the unsupported combination

#### Scenario: Identifier is malformed

- **WHEN** a caller supplies a validation or recovery identifier outside the generated lowercase UUID form
- **THEN** the command SHALL reject it before resolving a validation, resolution, repair, or evidence path

#### Scenario: Staged archive requests no-validate

- **WHEN** a staged invocation includes `--no-validate`
- **THEN** the command SHALL reject the combination

#### Scenario: Prepare combines skip and selection

- **WHEN** prepare receives `--skip-specs` with include or exclude values
- **THEN** the command SHALL fail without publishing a plan

#### Scenario: Non-prepare stage receives selection

- **WHEN** status, validate, finalize, resolve, repair, abort, or cleanup receives include, exclude, or skip selection
- **THEN** the command SHALL reject the combination

#### Scenario: Finalize omits validation identity

- **WHEN** finalize is requested without `--validation`
- **THEN** the command SHALL fail with `archive_validation_required`

#### Scenario: Finalize omits approval identity

- **WHEN** finalize is requested without `--approval` or with a token not bound to the current validation and recovery lineage
- **THEN** the command SHALL fail with `archive_approval_required` or `archive_approval_mismatch`

#### Scenario: Resolve or repair omits recovery identity

- **WHEN** resolve or repair is requested without `--recovery`
- **THEN** the command SHALL fail with `archive_recovery_required`

#### Scenario: Repair receives invalid resolution options

- **WHEN** repair omits `--resolution`, receives an unlisted decision, receives `--archive-name` for a decision other than rebind, or rebind omits `--archive-name`
- **THEN** the command SHALL reject the combination before mutation

#### Scenario: Stage receives approval outside execution

- **WHEN** prepare, status, validate, resolve, abort, cleanup, or a repair invocation that does not exactly match a returned execution action receives `--approval`
- **THEN** the command SHALL reject the combination

#### Scenario: Stage receives recovery outside recovery flow

- **WHEN** prepare, status, abort, cleanup, or a normal validate/finalize invocation receives `--recovery`
- **THEN** the command SHALL reject the combination

#### Scenario: Non-repair stage receives resolution

- **WHEN** prepare, status, validate, finalize, resolve, abort, or cleanup receives `--resolution`
- **THEN** the command SHALL reject the combination

#### Scenario: Non-prepare stage receives archive name

- **WHEN** status, validate, finalize, resolve, abort, cleanup, or a non-rebind repair receives `--archive-name`
- **THEN** the command SHALL reject the combination

#### Scenario: Direct archive receives staged-only options

- **WHEN** an invocation without `--stage` receives include, exclude, archive-name, validation, approval, recovery, or resolution options
- **THEN** the command SHALL reject them without changing the established meanings of existing direct flags

#### Scenario: JSON finalize lacks confirmation

- **WHEN** JSON finalize is requested without `--yes`
- **THEN** the command SHALL fail with `archive_confirmation_required`
- **AND** preserve resumable state

#### Scenario: JSON mutating repair lacks confirmation

- **WHEN** JSON repair would reconstruct, publish, quarantine, adopt, or rebind state without `--yes`
- **THEN** the command SHALL fail with `archive_confirmation_required`
- **AND** preserve all recovery evidence

### Requirement: Single-writer archive usage

The archive command SHALL communicate that staged finalization requires single-writer use rather than providing a concurrent transaction or writer lock.

#### Scenario: Status reports an active commit

- **WHEN** status reports committing or conflicted state
- **THEN** the response SHALL instruct the caller not to start another archive, standalone sync, or manual formal-spec edit for the selected root
- **AND** return only resume, evidence-bound resolve/repair, or non-mutating agent-investigation actions

#### Scenario: Command describes the no-lock boundary

- **WHEN** help, status, or a staged diagnostic describes finalization safety
- **THEN** the command SHALL state that it does not provide a finalize, process, lease, session-owned, time-based, or planning-root writer lock
- **AND** SHALL identify concurrent formal-spec writers as unsupported
- **AND** SHALL NOT present source or target hash checks as mutual exclusion

#### Scenario: Unexpected drift is observed

- **WHEN** the command detects a source or target change outside the staged lifecycle
- **THEN** it SHALL report stale or conflicted state
- **AND** SHALL NOT claim that unrelated concurrent writers were serialized

### Requirement: Direct archive compatibility

The archive command SHALL preserve its existing direct interface while sharing staged safety primitives internally.

#### Scenario: Archive runs without stage

- **WHEN** a user runs `openspec archive <change>` without `--stage`
- **THEN** existing positional arguments, flags, prompts, JSON result fields, and delta-application meanings SHALL remain available
- **AND** deterministic candidates and archive payload SHALL pass through exact snapshotting, target classification, recovery capsules, atomic writes, movement recovery/repair, and completion receipts

#### Scenario: Direct archive skips optional validation

- **WHEN** a confirmed direct archive uses `--no-validate`
- **THEN** it MAY skip existing optional validators
- **AND** SHALL still enforce path containment, exact candidate snapshotting, scenario preservation, target classification, and commit checks

#### Scenario: Direct archive encounters a staged plan

- **WHEN** direct archive finds an active staged plan for the change
- **THEN** it SHALL report that plan's status
- **AND** SHALL NOT create a competing direct plan or move the change independently

#### Scenario: Direct archive uses movement compatibility

- **WHEN** direct archive requires copy-based movement for Windows or cross-device compatibility
- **THEN** it SHALL build a complete `lstat`-based source manifest containing regular-file bytes and hashes, empty directories, file modes, and symlink target text without following links outside the source
- **AND** recreate that manifest in an owned temporary destination, fail safely if the platform cannot recreate a required symlink, and verify the complete temporary tree
- **AND** recheck the source manifest immediately before atomically publishing the final name within the destination filesystem
- **AND** remove the source only after publication and only while it still matches the verified manifest
- **AND** a failed copy SHALL NOT leave a partial final archive path

#### Scenario: Direct archive stops after copy publication

- **WHEN** the final destination is published but the source cannot be proven unchanged or source removal is interrupted
- **THEN** the command SHALL preserve both trees and completion evidence
- **AND** report a recoverable source-drift or cleanup state with structured agent-investigation actions
- **AND** SHALL NOT recursively delete a changed source

### Requirement: Cross-platform staged paths and movement

The archive command SHALL resolve staged state and movement safely on Windows, macOS, and Linux.

#### Scenario: Windows staged workspace

- **WHEN** staged archive runs on Windows
- **THEN** returned paths SHALL use resolved Windows paths
- **AND** path checks SHALL reject traversal, drive changes, case aliases, device names, alternate data streams, trailing-dot/space aliases, unsafe archive names, and unbound repair paths

#### Scenario: POSIX staged workspace

- **WHEN** staged archive runs on macOS or Linux
- **THEN** returned paths SHALL use resolved POSIX paths
- **AND** path checks SHALL reject traversal, symlink escape, and targets outside the selected root

#### Scenario: Staged movement crosses devices

- **WHEN** staged rename reports a cross-device condition
- **THEN** the command SHALL return `archive_cross_device_unsupported`
- **AND** SHALL NOT copy or remove the active change

## MODIFIED Requirements

### Requirement: Confirmation Behavior

The spec update confirmation SHALL provide clear visibility into changes before they are applied.

#### Scenario: Displaying confirmation

- **WHEN** prompting for confirmation
- **THEN** display a clear summary showing:
  - Which specs will be created (new capabilities)
  - Which specs will be updated (existing capabilities)
  - The source path for each spec
- **AND** format the confirmation prompt as:
  ```
  The following specs will be updated:

  NEW specs to be created:
    - cli-archive (from changes/add-archive-command/specs/cli-archive/spec.md)

  EXISTING specs to be updated:
    - cli-init (from changes/update-init-command/specs/cli-init/spec.md)

  Update 2 specs and archive 'add-archive-command'? [y/N]:
  ```

#### Scenario: Handling confirmation response

- **WHEN** waiting for user confirmation
- **THEN** default to "No" for safety (require explicit "y" or "yes")
- **AND** skip confirmation when `--yes` or `-y` flag is provided

#### Scenario: User declines confirmation

- **WHEN** user declines the confirmation
- **THEN** skip spec updates
- **AND** continue with the archive operation
- **AND** display a success message indicating specs were not updated

## REMOVED Requirements

### Requirement: Non-blocking confirmation

**Reason**: Its behavior is now part of `Confirmation Behavior`; keeping both requirements created contradictory decline outcomes.

**Migration**: Use the updated `User declines confirmation` scenario under `Confirmation Behavior`. The established CLI behavior of skipping spec updates and continuing the archive is retained.
