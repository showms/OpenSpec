## ADDED Requirements

### Requirement: Prepared archive lifecycle

The system SHALL prepare one reviewable archive plan for a change without modifying formal specs or moving the active change.

#### Scenario: Prepare creates candidate work

- **WHEN** a user prepares a change with selected delta specs
- **THEN** the system SHALL record the complete discovered delta capability set and selected delta hashes
- **AND** persist the exact bytes of every existing included main-spec base, or record typed absence for a new capability
- **AND** create one plan-owned candidate for every included capability
- **AND** return readiness warnings, the complete included/excluded partition, candidate-only agent work, and structured next actions
- **AND** leave formal specs and the active change unchanged

#### Scenario: Prepare creates move-only work

- **WHEN** the selected schema has no concrete specs output or the user prepares with `--skip-specs`
- **THEN** the system SHALL create a move-only plan with no candidates
- **AND** return the discovered excluded capabilities and structured validate, status, and abort actions

#### Scenario: Prepare selects a subset

- **WHEN** a user supplies valid included and excluded capability selections
- **THEN** the system SHALL create candidates only for included capabilities
- **AND** report the complete resulting partition
- **AND** preserve excluded deltas without reading or writing their target specs

#### Scenario: Prepare receives an invalid selection

- **WHEN** a selection contains an unknown capability, duplicate value, overlapping include/exclude value, or an incomplete explicit partition
- **THEN** the system SHALL fail with an actionable selection diagnostic
- **AND** SHALL NOT publish a plan or modify formal state

#### Scenario: Prepare receives an explicit archive name

- **WHEN** a user supplies one safe archive basename during prepare
- **THEN** the system SHALL record it as the intended destination name
- **AND** report any currently observable destination collision or cross-device condition before candidate work
- **AND** SHALL reject path separators, traversal, platform aliases, reserved generated names, and names that resolve outside the selected archive root

#### Scenario: Prepare observes a default-name collision

- **WHEN** the default local-date destination currently exists for a newly prepared same-name change
- **THEN** prepare SHALL report the collision and the structured abort-and-prepare action for choosing an explicit archive name
- **AND** SHALL NOT silently add a suffix or overwrite the existing archive

#### Scenario: Prepare is repeated

- **WHEN** a plan already exists for the selected change
- **THEN** the system SHALL return the existing plan status and legal next actions
- **AND** SHALL NOT create a competing plan

#### Scenario: A completed change name is reused

- **WHEN** a new active change uses the same name as a previously completed staged archive
- **THEN** prepare SHALL assign a new immutable random plan identifier
- **AND** SHALL NOT treat the retained completion receipt as an active plan
- **AND** SHALL preserve the older receipt as history until its retention expires

### Requirement: Candidate reconciliation boundary

The system SHALL limit archive-driven semantic reconciliation to explicit plan-owned candidates.

#### Scenario: Existing main spec receives a candidate

- **WHEN** an included delta targets an existing main spec
- **THEN** prepare SHALL persist the exact main-spec bytes as the plan-owned base
- **AND** initialize the candidate from those exact bytes
- **AND** archive-driven sync SHALL reconcile the delta into that candidate

#### Scenario: New capability receives a candidate

- **WHEN** an included delta targets a capability without a main spec
- **THEN** prepare SHALL initialize the candidate with the canonical Purpose-aware new-spec skeleton
- **AND** archive-driven sync SHALL write the resulting main-spec content only to that candidate

#### Scenario: Agent work is self-contained

- **WHEN** prepare returns candidate work
- **THEN** the work package SHALL contain the selected deltas, persisted bases or typed absence, candidates, operation summaries, guidance, rules, and explicit read/write scopes
- **AND** consumers SHALL use those inputs without rediscovering delta or candidate authority
- **AND** write only the returned candidate paths

#### Scenario: Modified delta omits a base scenario

- **WHEN** a normal prepared candidate's `MODIFIED` delta does not mention a scenario present in its base requirement
- **THEN** candidate reconciliation SHALL preserve that scenario
- **AND** SHALL treat only whole-requirement `REMOVED` as removal authority in this workflow

### Requirement: Immutable validation review

The system SHALL bind every finalize action to one complete immutable review of exact candidate bytes.

#### Scenario: Candidate validation succeeds

- **WHEN** all candidate files parse as canonical main specs and satisfy their selected delta outcomes
- **THEN** validation SHALL rediscover the complete delta capability set and current artifact/task readiness
- **AND** verify recorded source, base, prompt-source, and selection freshness
- **AND** preserve unaffected requirements and every base scenario not covered by whole-requirement removal
- **AND** enforce shared requirement/scenario identity and multiplicity rules
- **AND** verify `ADDED`, partial `MODIFIED`, `RENAMED` before `MODIFIED`, and whole-requirement `REMOVED` outcomes without unrelated semantic additions, removals, or renames
- **AND** preserve the existing Purpose content for existing capabilities
- **AND** store the exact candidate bytes in an immutable validation snapshot
- **AND** store a complete deterministic formal-spec diff plus an exact archive payload path/hash manifest with their hashes, byte length, and per-file statistics
- **AND** issue an opaque current validation identifier
- **AND** issue an opaque approval token bound to the validation identifier, complete review delivery metadata, payload-manifest hash, selection, and readiness
- **AND** leave formal specs and the active change unchanged

#### Scenario: Candidate validation fails

- **WHEN** a candidate is malformed, contains delta-operation headings, misses a required delta outcome, or loses protected base content
- **THEN** validation SHALL report capability-specific errors
- **AND** SHALL NOT issue a validation identifier or modify formal state

#### Scenario: Prepared delta discovery drifts

- **WHEN** validation or finalization rediscovers a delta capability that was added, removed, or renamed after prepare
- **THEN** the system SHALL report the prepared capability partition as stale
- **AND** SHALL NOT validate or begin commit from that plan
- **AND** return an abort-and-prepare action while the plan is still uncommitted

#### Scenario: Source authority drifts after commit starts

- **WHEN** a finalize retry finds the complete delta set, selected delta bytes, or another recorded source-authority input changed after commit state exists
- **THEN** finalize SHALL report `archive_source_conflict`
- **AND** snapshot safe current evidence and preserve the active change, applied specs, and recovery records
- **AND** return agent-investigation guidance instead of abort-and-prepare
- **AND** SHALL NOT continue formal writes or move a change whose current deltas no longer explain the reviewed result

#### Scenario: Readiness changes after prepare

- **WHEN** validation recalculates artifact or task readiness after prepare without changing selected spec bytes
- **THEN** the immutable validation SHALL record the freshly calculated readiness and warnings
- **AND** finalization SHALL recalculate readiness before commit and reject any difference with a validate action
- **AND** the completion receipt SHALL record the readiness accepted at commit start

#### Scenario: Archive payload changes after validation

- **WHEN** a non-generated file, empty directory, mode, or symlink target under the active change differs from the current validation payload manifest
- **THEN** finalization before commit SHALL report `archive_payload_changed` and return the validate action
- **AND** finalization after commit SHALL report `archive_source_conflict` with safe current evidence
- **AND** SHALL NOT move an unreviewed payload tree

#### Scenario: Validation is repeated

- **WHEN** a prepared plan is validated again before commit starts
- **THEN** the new immutable snapshot SHALL become the current validation
- **AND** the prior identifier SHALL remain diagnostic history but SHALL NOT be eligible for finalize

#### Scenario: Candidate changes after validation

- **WHEN** candidate bytes no longer match the current immutable validation snapshot
- **THEN** finalize SHALL reject that validation as stale
- **AND** return the legal validate action for creating a new immutable review

#### Scenario: Complete review fits inline delivery

- **WHEN** the complete review is within the documented inline byte limit
- **THEN** validation SHALL return the complete review inline
- **AND** also return its durable path, hash, byte length, payload-manifest hash, and approval token

#### Scenario: Complete review requires file delivery

- **WHEN** the complete review exceeds the documented inline byte limit
- **THEN** validation SHALL return `delivery: file`, its complete path, hash, byte length, payload-manifest hash, statistics, approval token, and a structured read action
- **AND** SHALL NOT label a summary or truncated excerpt as the complete review

#### Scenario: File-delivered review is approved

- **WHEN** a consumer requests finalization for a review delivered by file
- **THEN** the consumer SHALL explicitly acknowledge the complete file path, hash, and byte length returned by validation
- **AND** finalize SHALL require the opaque approval token bound to those exact values and the payload-manifest hash
- **AND** a summary, excerpt, or statistics-only acknowledgement SHALL NOT count as approval of the immutable review

#### Scenario: Complete review cannot be created

- **WHEN** the complete review cannot be fully generated, persisted, or hashed
- **THEN** validation SHALL fail closed with `archive_review_generation_failed`
- **AND** SHALL NOT issue a validation identifier

#### Scenario: Move-only validation succeeds

- **WHEN** a prepared plan contains no candidates
- **THEN** validation SHALL create an immutable empty-spec diff plus a complete archive payload manifest bound to the readiness warnings and selected archive scope
- **AND** explicitly list discovered capabilities that will remain unsynced
- **AND** issue a current validation identifier and approval token

### Requirement: Single-writer forward-only finalization

The system SHALL finalize a reviewed archive through resumable forward progress rather than rollback, under the documented single-writer usage contract.

#### Scenario: Finalize starts a reviewed commit

- **WHEN** a user confirms the current validation identifier
- **AND** supplies the opaque approval token bound to its complete review delivery metadata and payload-manifest hash
- **AND** the complete discovered delta set and selected partition remain current
- **AND** every recorded source and target base remains current
- **AND** the complete active-change payload matches the reviewed manifest
- **AND** current artifact and task readiness has been recalculated and accepted
- **AND** the archive destination is available
- **THEN** finalize SHALL bind the local archive date, final destination, validation identifier, approval identity, reviewed and payload hashes, and commit token before writing formal specs
- **AND** publish a complete matching source-local recovery capsule before the first formal-spec replacement
- **AND** preserve an existing valid date prefix on the change name

#### Scenario: Pending target is applied

- **WHEN** an included target still equals its prepared base hash or prepared absence
- **THEN** finalize SHALL atomically replace that target with the exact immutable reviewed bytes
- **AND** verify the resulting target hash

#### Scenario: Individual target replacement is interrupted

- **WHEN** finalize writes a reviewed target
- **THEN** it SHALL create an exclusive owned temporary sibling in the target directory
- **AND** write and sync the exact reviewed bytes before using the platform replacement primitive
- **AND** sync the parent directory where the platform supports it, verify the published hash, preserve the intended file mode, and clean up only its owned temporary path
- **AND** use a bounded retry for documented transient Windows sharing failures
- **AND** SHALL promise process-interruption recovery but SHALL NOT claim hardware power-loss durability or a multi-file atomic transaction

#### Scenario: Reviewed target is already applied

- **WHEN** an included target already equals the reviewed candidate hash
- **THEN** finalize SHALL treat it as completed
- **AND** SHALL NOT rewrite or roll it back

#### Scenario: Target is unexpected before commit

- **WHEN** preflight finds an included target equal to neither its prepared base nor reviewed candidate before commit state exists
- **THEN** finalize SHALL report the plan as stale
- **AND** leave the plan abortable
- **AND** direct the user to abort and prepare again

#### Scenario: Target is unexpected after commit starts

- **WHEN** a retry finds an included target equal to neither its prepared base nor reviewed candidate after commit state exists
- **THEN** finalize SHALL preserve the unexpected content and snapshot its exact bytes as plan-owned conflict evidence
- **AND** report the plan as conflicted with the current, base, and reviewed paths and hashes
- **AND** return a read-scoped `agentRecovery` evidence package containing the selected delta, relevant plan metadata, explicit safe scopes, and those three evidence versions
- **AND** the archive agent SHALL diagnose the semantic difference and present recovery options to the user
- **AND** user approval SHALL be required before any content-changing recovery action
- **AND** the CLI SHALL return an evidence-bound resolve action for creating a plan-owned recovery candidate
- **AND** the CLI and agent SHALL NOT silently rebase, overwrite, roll back, invoke standalone sync during commit, or claim success

#### Scenario: Process stops after some target writes

- **WHEN** finalization stops after one or more reviewed targets were written
- **THEN** those writes SHALL remain applied
- **AND** a later finalize with the commit-bound validation identifier SHALL classify and resume the remaining targets

#### Scenario: Finalization resumes on a later date

- **WHEN** an interrupted commit is retried on a later local calendar date
- **THEN** it SHALL retain the archive date and destination recorded when commit began

#### Scenario: Bound archive destination becomes occupied

- **WHEN** commit has started and the bound destination is later occupied by content without the matching commit-token marker
- **THEN** finalize SHALL report `archive_destination_conflict`
- **AND** preserve the bound date and destination, active source, applied specs, and recovery evidence
- **AND** normal finalize SHALL NOT abort the commit, choose an alternate destination, overwrite the occupant, or move the source
- **AND** return evidence and a structured repair action that can rebind only to an explicit user-supplied empty destination

### Requirement: Reviewed conflict resolution

The system SHALL resolve a post-commit formal-spec conflict through a new immutable amendment review without giving the agent direct formal-spec write authority.

#### Scenario: Resolve prepares recovery candidate work

- **WHEN** a conflicted plan receives its current evidence-bound resolve action
- **THEN** the system SHALL create one plan-owned recovery candidate initialized from the captured current target
- **AND** return the persisted original base, original reviewed snapshot, captured current target, selected delta, rules, and explicit recovery read/write scopes
- **AND** SHALL NOT modify the formal target, other reviewed targets, or active change

#### Scenario: Recovery candidate is validated

- **WHEN** the recovery candidate remains based on the captured current hash and satisfies canonical main-spec structure plus the original delta outcomes
- **THEN** validation SHALL create a complete immutable amendment review from captured current bytes to recovery candidate bytes
- **AND** bind a new validation identifier and approval token to the original validation, conflict evidence, current hash, amendment bytes, and payload manifest
- **AND** show every difference not authorized by the original delta for explicit user review

#### Scenario: Approved recovery amendment is applied

- **WHEN** recovery-mode finalize receives the current recovery ID, validation identifier, approval token, and explicit confirmation
- **THEN** it SHALL append an immutable commit amendment
- **AND** atomically apply the exact recovery snapshot to the conflicted target
- **AND** resume the original forward-only commit
- **AND** record the original review and amendment lineage in the completion receipt

#### Scenario: Recovery evidence changes

- **WHEN** the captured current target, payload manifest, source authority, or recovery candidate changes after recovery validation
- **THEN** the system SHALL reject the recovery approval as stale
- **AND** preserve the latest current bytes and require a new resolve or validate action as appropriate

### Requirement: Archive movement and completion recovery

The system SHALL move a fully applied staged change without publishing a partial archive and SHALL recover a missed completion response.

#### Scenario: Same-filesystem staged rename succeeds

- **WHEN** every included target equals the reviewed snapshot and the active change can be renamed within the selected planning home
- **THEN** finalize SHALL exclusively create and sync a commit-token marker in the active source before movement
- **AND** bind the marker to the plan, commit, validation lineage, payload manifest, source-local capsule, source, and destination
- **AND** carry that marker and recovery capsule into the final archive destination
- **AND** write an authoritative completion receipt
- **AND** remove the generated marker and capsule after the receipt is durable
- **AND** report the final archive location and applied specs

#### Scenario: Process stops after marker creation

- **WHEN** a matching commit-token marker exists in the active source and movement has not happened
- **THEN** a retry SHALL verify its plan identifier, token, source, and destination binding
- **AND** resume the same movement without repeating completed spec writes

#### Scenario: Movement marker is missing or inconsistent

- **WHEN** movement evidence contains a pre-existing marker, malformed marker, mismatched token, or source/destination combination inconsistent with commit state
- **THEN** status SHALL report the plan as `orphaned`
- **AND** preserve all source, destination, plan, and marker evidence
- **AND** return an opaque recovery identifier and only the repair decisions whose preconditions are provable
- **AND** SHALL NOT guess ownership

#### Scenario: Staged rename is temporarily blocked

- **WHEN** Windows or another platform temporarily prevents the directory rename
- **THEN** finalize SHALL leave reviewed spec writes applied and the active change in place
- **AND** return a retryable diagnostic and structured resume action
- **AND** SHALL NOT report the archive as complete

#### Scenario: Staged rename crosses filesystems

- **WHEN** staged movement reports a cross-device condition
- **THEN** finalize SHALL return `archive_cross_device_unsupported`
- **AND** SHALL NOT copy, remove, or publish a partial final archive

#### Scenario: Process stops after directory movement

- **WHEN** the active source is absent and the commit-bound destination contains the matching commit token
- **THEN** a retry SHALL recognize the completed movement
- **AND** write and return the completion receipt without repeating spec writes or movement

#### Scenario: Marker cleanup fails after receipt durability

- **WHEN** the authoritative completion receipt is durable but removal of the matching generated marker or source-local capsule fails
- **THEN** finalize SHALL still report completion from the receipt
- **AND** report the matching generated path as cleanup residue
- **AND** cleanup MAY later remove only that receipt-bound residue

#### Scenario: Completed finalize is retried

- **WHEN** a matching completion receipt already exists
- **THEN** finalize SHALL return the same authoritative result
- **AND** SHALL NOT repeat formal mutation
- **AND** the completed plan SHALL remain outside the reusable active change slot

### Requirement: Evidence-bound orphan and broken repair

The system SHALL preserve ambiguous recovery evidence and perform only explicitly approved repair actions whose complete preconditions remain provable.

#### Scenario: Repair decision is previewed

- **WHEN** a caller selects one currently allowed repair decision and supplies every required explicit input without an approval token
- **THEN** repair SHALL remain non-mutating
- **AND** persist a complete repair review containing the evidence manifest, decision, effects, retained evidence, cleanup consequences, and explicit inputs
- **AND** return an approval token and exact execution action bound to that repair review

#### Scenario: Primary plan is reconstructed

- **WHEN** the primary active-plan record is missing or damaged
- **AND** one complete source-local capsule matches the source or destination, validation lineage, commit, payload, and formal targets
- **THEN** repair SHALL atomically reconstruct the primary plan from that capsule
- **AND** return the derived status without changing formal specs or movement state

#### Scenario: Source is verified for resume

- **WHEN** the active source and payload match the commit, the destination is absent, and the marker alone is missing or foreign
- **THEN** an approved `resume-source` repair SHALL preserve any foreign marker as evidence
- **AND** publish the commit-bound marker and return the normal finalize resume action

#### Scenario: Destination is verified for adoption

- **WHEN** the active source is absent and the bound destination exactly matches the reviewed payload, capsule, commit, and target evidence
- **THEN** an approved `adopt-destination` repair SHALL write the authoritative completion receipt
- **AND** perform only receipt-bound generated cleanup

#### Scenario: Matching source and destination both exist

- **WHEN** the active source and bound destination both contain the same reviewed payload lineage
- **THEN** an approved `quarantine-source-and-adopt-destination` repair SHALL atomically move the duplicate source to a plan-owned quarantine
- **AND** write the destination receipt
- **AND** retain the quarantined copy until its receipt-bound cleanup retention expires
- **AND** SHALL NOT delete the duplicate during repair

#### Scenario: Bound destination contains foreign content

- **WHEN** the active source exactly matches the reviewed lineage and the bound destination is occupied by different content
- **THEN** an approved `rebind-destination` repair SHALL require a user-supplied safe basename whose destination is absent
- **AND** append the old and new destination bindings to immutable repair history
- **AND** return the normal finalize resume action
- **AND** SHALL NOT modify the foreign destination

#### Scenario: Repair evidence is insufficient or stale

- **WHEN** no repair preconditions can be proven or any bound evidence changes after status
- **THEN** repair SHALL preserve every source, destination, plan, capsule, marker, receipt, and formal target
- **AND** return `preserve-and-stop` with a durable recovery report or a fresh status action
- **AND** SHALL NOT combine disagreeing lineages, adopt similar content, or use confirmation as a substitute for missing evidence

#### Scenario: Every recovery copy is missing

- **WHEN** commit may have started but neither a complete primary plan nor a complete matching capsule remains
- **THEN** status SHALL report that exact automatic resume cannot be proven
- **AND** preserve all observable formal and archive state for external recovery
- **AND** SHALL NOT reconstruct reviewed bytes from summaries, hashes, or current targets

### Requirement: Staged status and lifecycle cleanup

The system SHALL expose recovery state by change name and preserve commit evidence until completion.

#### Scenario: Status reports current state

- **WHEN** a user requests staged status
- **THEN** the system SHALL report `none`, `prepared`, `validated`, `committing`, `conflicted`, `broken`, `orphaned`, or `completed`
- **AND** include the immutable plan identifier, current validation and approval binding when applicable, review delivery, payload-manifest hash, included/excluded capabilities, applied and pending targets, locations, diagnostics, evidence paths, and structured legal next actions

#### Scenario: Active change name has only historical receipts

- **WHEN** an active change exists with no active plan and retained receipts use the same change name but different plan identities
- **THEN** status SHALL report `none` for the active change
- **AND** expose older receipts only as history
- **AND** prepare SHALL remain a legal next action

#### Scenario: Stored plan state is malformed or incomplete

- **WHEN** a plan, pointer, commit record, or receipt exists but fails schema, hash, containment, identity, or cross-record consistency checks
- **THEN** status SHALL report `broken`
- **AND** preserve the suspect files and return their safe evidence paths for agent-led diagnosis
- **AND** return `reconstruct-plan` only when one complete matching source-local capsule proves the lineage
- **AND** otherwise SHALL NOT infer progress or mutate formal state

#### Scenario: Movement evidence has no consistent owner

- **WHEN** source, destination, marker, and receipt evidence cannot be reconciled to one plan and commit token
- **THEN** status SHALL report `orphaned`
- **AND** preserve all evidence and return a recovery identifier plus evidence-bound agent-investigation and repair actions
- **AND** SHALL NOT move, delete, or overwrite either location

#### Scenario: Abort removes an uncommitted plan

- **WHEN** a user aborts a prepared or validated plan without commit state
- **THEN** the system SHALL remove its generated candidates and validation snapshots
- **AND** leave formal specs and the active change unchanged

#### Scenario: Abort follows commit start

- **WHEN** a user requests abort after commit state exists
- **THEN** the system SHALL preserve recovery evidence
- **AND** reject abort with resume or agent-investigation guidance

#### Scenario: Cleanup removes disposable state

- **WHEN** root-level cleanup encounters recognized superseded validations, abandoned generated temporaries, receipt-bound marker or capsule residue, quarantined duplicate sources past their receipt-bound retention, or completed receipts older than 30 days
- **THEN** it SHALL remove only those explicitly owned paths
- **AND** report what was removed

#### Scenario: Cleanup encounters active recovery state

- **WHEN** cleanup encounters a prepared, validated, committing, conflicted, broken, or orphaned plan
- **THEN** it SHALL report and preserve that plan

### Requirement: Single-writer usage contract

The system SHALL describe staged archive as requiring single-writer use and SHALL NOT claim that finalize or the planning root is protected by a writer lock.

#### Scenario: Normal staged operation

- **WHEN** a staged archive lifecycle is active
- **THEN** generated skills SHALL wait for each command to finish before starting the next
- **AND** SHALL instruct the user not to run another archive, standalone sync, or manual formal-spec edit until finalization finishes or the uncommitted plan is aborted

#### Scenario: No locking guarantee

- **WHEN** staged archive explains its concurrency boundary
- **THEN** it SHALL state that this workflow does not provide a finalize, process, lease, session-owned, time-based, or planning-root writer lock
- **AND** SHALL describe concurrent archive, standalone sync, and manual formal-spec writes as unsupported
- **AND** SHALL describe hash drift checks as detection rather than mutual exclusion
- **AND** SHALL NOT claim that every simultaneous-preflight or last-writer race is prevented

#### Scenario: Unexpected external change is detected

- **WHEN** a recorded source or target changes outside the supported single-writer workflow
- **THEN** the system SHALL report stale or conflicted state according to whether commit has begun
- **AND** SHALL NOT describe the workflow as an all-or-nothing transaction

### Requirement: Sequential bulk composition

The system SHALL compose bulk archive from complete sequential single-change lifecycles.

#### Scenario: Bulk scope is established

- **WHEN** a user selects multiple changes for bulk archive
- **THEN** bulk SHALL determine a stable change order and explicit included/excluded capability partition for every change
- **AND** present the scope before formal mutation

#### Scenario: Bulk processes selected changes

- **WHEN** the user proceeds with bulk archive
- **THEN** bulk SHALL prepare, reconcile, validate, confirm the exact approval-bound review, and finalize each change before preparing the next change
- **AND** the next change SHALL observe all completed formal specs from earlier changes

#### Scenario: Bulk selection drifts

- **WHEN** prepare returns a capability partition different from the inspected partition
- **THEN** bulk SHALL abort that uncommitted plan
- **AND** report selection drift without continuing that item

#### Scenario: One bulk item fails or is cancelled

- **WHEN** a selected change fails, conflicts, or is cancelled
- **THEN** completed earlier changes SHALL remain completed
- **AND** bulk SHALL report the current item accurately
- **AND** later changes SHALL remain unstarted or be explicitly reported as skipped

#### Scenario: Mixed-schema bulk item has no specs

- **WHEN** a selected change has no concrete specs output
- **THEN** bulk SHALL use the same move-only validation and confirmation lifecycle

### Requirement: Defensive cross-platform paths

The system SHALL keep every staged path and generated identifier within its selected planning root on Windows, macOS, and Linux.

#### Scenario: Invalid identifier or stored path

- **WHEN** a caller or stored plan supplies a malformed validation identifier, absolute authority entry, traversal, or containment escape
- **THEN** the system SHALL reject it before reading or writing the resolved target

#### Scenario: Windows path alias is unsafe

- **WHEN** a Windows path uses a drive change, device name, alternate data stream, trailing-dot/space alias, or case-insensitive alias of a protected path
- **THEN** the system SHALL reject the operation before formal mutation

#### Scenario: POSIX path escapes through a symlink

- **WHEN** a candidate, target, source, or destination resolves outside its allowed root through a symlink
- **THEN** the system SHALL reject the operation before formal mutation

#### Scenario: Recovery action supplies a path

- **WHEN** a repair request supplies an archive name, recovery identifier, or path-like value not present in its structured evidence-bound action
- **THEN** the system SHALL reject it before resolving or mutating source, destination, quarantine, marker, capsule, or receipt state
