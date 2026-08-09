## ADDED Requirements

### Requirement: Prepare a versioned archive attempt

The system SHALL provide a non-interactive archive preparation operation that resolves one selected change and returns a versioned plan containing its root, schema, readiness checks, archive root, finalize-time local-date naming policy, destination preview, and allowed spec work without modifying the selected project or store.

#### Scenario: Prepare a change with delta specs
- **WHEN** a client prepares an archive attempt for a valid change with concrete `artifactPaths.specs.existingOutputPaths`
- **THEN** the response identifies every concrete delta, its corresponding main-spec target, and its managed candidate entry
- **AND** records the current change, config, delta, target, archive-root, and naming-policy state needed to detect stale work
- **AND** leaves main specs and the active change unchanged

#### Scenario: Prepare a schema without specs outputs
- **WHEN** the resolved schema has no `specs` artifact or the change has no concrete specs outputs
- **THEN** the plan contains no spec-work entries
- **AND** still contains readiness checks, the archive root and naming policy, and a clearly marked destination preview
- **AND** does not infer delta specs from other artifacts or directories

#### Scenario: Prepare in a selected store
- **WHEN** a client prepares a change from a registered store on Windows, macOS, or Linux
- **THEN** every returned project path is resolved from that selected store
- **AND** subsequent phases are bound to the same canonical root identity
- **AND** path handling does not depend on a platform-specific separator or path casing

#### Scenario: Prepare finds a resumable attempt for the same change
- **WHEN** one or more resumable archive attempts already include the selected active change
- **AND** the caller has not explicitly authorized a new attempt
- **THEN** prepare returns every matching attempt, its lifecycle and lock state, its plan path when trustworthy, and the actions available for that attempt
- **AND** creates no new plan, snapshots, or candidate workspace
- **AND** requires the client to ask the user whether to resume one exact attempt, inspect or clean retained state and retry, create a distinct attempt, or cancel
- **AND** does not implicitly reuse, overwrite, or migrate candidate work from an older attempt

#### Scenario: User explicitly creates another attempt for the same change
- **WHEN** the user has reviewed the reported overlapping attempts
- **AND** the caller repeats prepare with `--new-attempt --yes`
- **THEN** prepare creates a distinct plan ID, snapshots, and candidate workspace
- **AND** leaves every older attempt unchanged
- **AND** binds the new attempt to its own returned plan path
- **AND** does not let the explicit-new option bypass a live or unresolved lock, readiness check, path check, or other safety blocker

#### Scenario: Single and batch attempts overlap
- **WHEN** a selected change belongs to a retained single-change or batch attempt
- **AND** a client requests either single-change or batch preparation that includes that change
- **THEN** prepare reports every overlapping attempt before creating a new workspace
- **AND** does not move a candidate between single and batch attempts or detach a change from a batch implicitly
- **AND** requires the same resume, cleanup, explicit-new, or cancel decision

### Requirement: Keep archive inputs in their owning domains

The prepared archive plan SHALL keep project context, archive operation guidance, and `specs` artifact rules structurally separate and SHALL associate artifact rules only with candidate specs produced from the corresponding specs artifact.

#### Scenario: Context, guidance, and rules are configured
- **WHEN** prepare loads project context, `operations.archive.guidance`, and `rules.specs`
- **THEN** the plan exposes them as separate fields
- **AND** associates the artifact rules only with planned spec outputs
- **AND** does not turn artifact rules into operation guidance or enforceable checks

#### Scenario: Guidance conflicts with a CLI check
- **WHEN** archive guidance asks the agent to bypass a blocked CLI check or change a resolved path
- **THEN** the blocked check and resolved path remain controlling inputs
- **AND** finalization remains unavailable until the blocking condition is resolved

#### Scenario: Aggregate target has multiple artifact-rule snapshots
- **WHEN** multiple included change contributions produce one aggregate candidate for a shared target
- **THEN** every contribution keeps its own `specs` artifact-rule snapshot
- **AND** every non-conflicting artifact rule constrains only that aggregate candidate
- **AND** incompatible rule instructions are reported with their rule text and source changes rather than treated as archive guidance or resolved by change creation date
- **AND** the user selects which rule controls each conflict
- **AND** the result records the selected and suppressed rule sources and regenerates the candidate under that selection before validation

### Requirement: Perform semantic work in a managed candidate workspace

An agent-driven archive SHALL write semantic merge results only to candidate entries named by the prepared plan and SHALL explicitly classify each target as `sync`, `retire`, `already-synced`, or `skip`.

#### Scenario: Agent produces a candidate spec
- **WHEN** the agent semantically reconciles a planned delta and baseline
- **THEN** it writes the result to that entry's candidate path
- **AND** records `sync` for that exact entry
- **AND** leaves the corresponding main spec unchanged until finalization

#### Scenario: Agent requests capability retirement
- **WHEN** semantic reconciliation removes the final requirement and the agent intends to retire the capability
- **THEN** it records `retire` explicitly for the planned entry
- **AND** omission of a candidate file alone is not treated as retirement authorization

#### Scenario: Agent reports an already-synced entry
- **WHEN** the baseline already represents the delta outcome
- **THEN** the agent records `already-synced` explicitly
- **AND** validation compares that result with the same prepared entry rather than widening the work set

#### Scenario: New attempt follows an interrupted finalization
- **WHEN** a previous finalizing process ended abruptly after applying some main-spec outcomes and the active change remains available
- **AND** the user has inspected the retained attempt, resolved any stale claims, and explicitly cleaned or retained it before authorizing a new attempt
- **THEN** a new prepare snapshots the current main specs as its baseline
- **AND** semantic reconciliation records already-represented outcomes as `already-synced`
- **AND** produces candidates only for the remaining intended outcome
- **AND** does not require reconstruction or rollback of the earlier process's mutation order

#### Scenario: User skips spec reconciliation
- **WHEN** the user explicitly chooses not to reconcile a prepared spec entry
- **THEN** the result records `skip` for that exact entry
- **AND** no candidate or retirement is inferred for the skipped work
- **AND** final review identifies that the spec was intentionally skipped rather than already synchronized

#### Scenario: Agent produces an unknown output
- **WHEN** the result manifest names an entry or target not present in the plan
- **THEN** validation rejects the attempt
- **AND** the project and active change remain unchanged

### Requirement: Validate a prepared archive without project mutation

The system SHALL validate the complete prepared attempt before finalization and SHALL return typed pass, warning, and blocked checks plus a validation token only when all enforceable conditions pass.

#### Scenario: Candidate workspace is valid
- **WHEN** every planned result is explicit, every sync candidate is a valid main spec, every retirement is authorized, and all paths and archive roots remain safe
- **THEN** validation succeeds
- **AND** confirms that every live main-spec target still matches the baseline the agent used
- **AND** returns a token bound to the plan, results, candidates, current project inputs, and check results
- **AND** returns each target's derived `create`, `update`, `delete`, or no-file-change effect
- **AND** returns the exact baseline-to-final-state diff, including complete deletion for retirement, and ADDED, MODIFIED, REMOVED, and RENAMED counts used for final review
- **AND** does not write main specs or move the change

#### Scenario: Candidate spec is invalid
- **WHEN** a `sync` candidate cannot be parsed as a main spec, fails main-spec validation, or contains delta operation sections
- **THEN** validation returns a blocked check naming the candidate entry
- **AND** returns no usable validation token
- **AND** leaves the project unchanged

#### Scenario: Candidate path escapes the attempt
- **WHEN** a plan, manifest, input, or output resolves outside the canonical managed attempt or traverses a symbolic link to another location
- **THEN** validation blocks the attempt before reading it as an approved candidate
- **AND** no project file is changed

#### Scenario: Candidate or result changes during validation
- **WHEN** a candidate or result manifest changes while validation is reading it
- **THEN** validation reports the attempt as concurrently modified
- **AND** returns no usable validation token
- **AND** leaves the project unchanged

#### Scenario: Incomplete work needs confirmation
- **WHEN** the current artifact or task state is incomplete but existing archive behavior permits an explicit override
- **THEN** validation reports a warning with `confirmationRequired`
- **AND** does not relabel the condition as either a successful check or a non-overridable failure

### Requirement: Reject stale archive work

Validation and finalization SHALL reject an attempt when a relevant input has changed since preparation or since the validation token was issued.

#### Scenario: Delta or baseline changes after prepare
- **WHEN** a planned delta, main-spec baseline, change metadata, relevant config, artifact state, task state, archive root, or naming policy differs from its prepared state
- **THEN** validation reports the attempt as stale
- **AND** tells the client to prepare and perform semantic work again
- **AND** preserves both the current project state and the candidate workspace for inspection

#### Scenario: Project changes after validation
- **WHEN** any token-bound input, candidate, result, check, or destination changes after validation
- **THEN** finalization rejects the validation token
- **AND** performs no project mutation

### Requirement: Finalize one archive through CLI-owned mutation

The system SHALL finalize a validated archive only after explicit confirmation, revalidate the attempt, claim the destination, promote validated candidates, and move the change, with rollback for failures returned while the finalizing process remains running.

#### Scenario: Successful finalization
- **WHEN** the caller supplies the matching plan, validation token, and explicit confirmation
- **AND** final revalidation succeeds
- **THEN** the system captures the current local date once and derives the archive destination from that date
- **AND** keeps an already date-prefixed change name unchanged
- **AND** claims the archive root and verifies the derived destination
- **AND** applies all validated candidate writes and retirements
- **AND** moves the complete change to the resolved dated archive path
- **AND** reports the archived location, spec outcomes, and warnings

#### Scenario: Local date changes after prepare or validation
- **WHEN** finalization starts on a different local date from prepare or validation
- **THEN** the system uses the date captured when finalization starts
- **AND** checks the newly derived destination before any spec mutation
- **AND** does not reject the plan merely because a preview date changed

#### Scenario: Derived archive destination already exists
- **WHEN** finalization derives a destination that already exists
- **THEN** finalization blocks before any spec mutation
- **AND** does not overwrite the destination, choose another date, or invent a numeric suffix
- **AND** tells the client to inspect and resolve the existing destination

#### Scenario: Confirmation is absent
- **WHEN** finalization requires confirmation and the caller does not provide it
- **THEN** finalization returns a blocked diagnostic identifying the required confirmation
- **AND** performs no project mutation

#### Scenario: Mutation or move fails before commit
- **WHEN** the running finalization operation reports that a candidate mutation, retirement, verification, or final move failed before a complete archive is secured
- **THEN** the system restores every attempted main-spec mutation
- **AND** leaves or returns the change to its active path
- **AND** reports any recovery failure separately from the original error

#### Scenario: Verified fallback archive is complete but cleanup fails
- **WHEN** a cross-device or platform fallback secures and verifies a complete archive but cannot remove its staged source
- **THEN** the system retains the complete archive and committed spec state
- **AND** returns recovery diagnostics rather than deleting the only verified complete copy

#### Scenario: Finalizing process terminates abruptly
- **WHEN** the process terminates without running normal failure handling after one or more project mutations completed
- **THEN** the system does not claim that those completed mutations were automatically rolled back
- **AND** a retained attempt or claim is reported for explicit inspection, unlock, cleanup, recovery where available, or retention
- **AND** attempt cleanup does not reverse completed main-spec writes or archive moves
- **AND** a later prepare may use current project state as its new baseline when the active change remains available and the user explicitly resolves the retained-attempt decision

### Requirement: Negotiate archive-attempt contract versions

Every archive-attempt response SHALL identify its contract version, and clients SHALL stop before semantic work or project mutation when that version is unsupported.

#### Scenario: Client and CLI support the same version
- **WHEN** a client receives a supported archive-attempt contract version
- **THEN** it may continue using fields and checks defined by that version

#### Scenario: CLI lacks archive attempts or returns an unsupported version
- **WHEN** a generated skill cannot invoke archive attempts or receives an unsupported contract version
- **THEN** it reports the compatibility problem and upgrade guidance
- **AND** may offer the existing one-shot `openspec archive <change>` command
- **AND** does not fall back to manually moving the change

### Requirement: Finalize batches through conflict groups

A bulk archive client SHALL prepare and validate every selected change before mutation, produce one aggregate candidate per canonical main-spec target, and provide rollback within each connected shared-target conflict group for failures returned while the process remains running, while reporting wider batch completion by group.

#### Scenario: Multiple changes target the same main spec
- **WHEN** two or more selected changes include deltas for the same canonical main-spec path
- **THEN** batch preparation snapshots that main spec once as the shared baseline
- **AND** assigns every contributing delta to one aggregate candidate entry
- **AND** final review shows the aggregate baseline-to-final-state diff and each change's contribution
- **AND** the connected changes finalize together rather than requiring sequential prepare cycles

#### Scenario: A shared-target contribution is skipped
- **WHEN** the user records `skip` for one change's contribution to a shared target
- **THEN** batch validation excludes that contribution from the aggregate candidate and effective conflict graph
- **AND** the skipped edge does not force otherwise disconnected changes into one rollback group
- **AND** final review still identifies the skipped change and target

#### Scenario: Compatible contributions share one candidate
- **WHEN** contributing changes express compatible intent for one target
- **THEN** semantic reconciliation preserves all compatible intent in the aggregate candidate
- **AND** does not discard an earlier change merely because another change was created later

#### Scenario: Aggregate result retires a capability
- **WHEN** included contributions together remove the final requirement from a shared target
- **THEN** retirement is available only when every included contributing change validly declares `retire_capabilities: true`
- **AND** creation-date priority and user confirmation do not replace missing retirement authorization
- **AND** validation blocks before mutation when any required declaration is absent or invalid

#### Scenario: Contributing changes contain incompatible intent
- **WHEN** contributing changes cannot be reconciled without choosing between their meanings
- **THEN** the system orders inputs by `created` date from oldest to newest and uses the later-created change as the proposed resolution when the dates establish an order
- **AND** reports the competing changes, affected requirement, proposed resolution, and reason
- **AND** requires explicit confirmation of that resolution before finalization
- **AND** requires an explicit user decision when creation dates are missing or equal

#### Scenario: Contributing changes contain incompatible artifact rules
- **WHEN** included contributions supply artifact rules that cannot both control the same candidate content
- **THEN** the client shows the conflicting rules and their source changes
- **AND** requires the user to select the controlling rule for each conflict
- **AND** keeps every non-conflicting rule from all included contributions
- **AND** regenerates the aggregate candidate under the recorded selections
- **AND** validation blocks without a token until every rule conflict has a concrete selection

#### Scenario: Batch is ready to finalize
- **WHEN** every target candidate, conflict decision, readiness check, and derived destination preflight validates
- **THEN** batch validation returns a token bound to the aggregate candidates, diffs, contributions, semantic-conflict decisions, artifact-rule selections, and deterministic conflict-group order
- **AND** the client requests user confirmation before finalizing the first group

#### Scenario: Conflict group is about to mutate
- **WHEN** a validated batch is ready to start a conflict group or one of its target mutations
- **THEN** finalization rechecks the live changes, baselines, candidates, contributions, authorizations, and destinations for that group
- **AND** stops before that mutation when any fingerprint differs from the validated state

#### Scenario: One conflict group fails
- **WHEN** the running finalization operation reports that a conflict group failed before commit
- **THEN** every attempted spec mutation and change move in that group is restored
- **AND** earlier completed groups remain complete
- **AND** the failing and later groups remain active with exact status and retry guidance
- **AND** the result does not claim all-or-nothing batch atomicity

#### Scenario: Process terminates inside a conflict group
- **WHEN** the finalizing process terminates abruptly after applying only part of a conflict group
- **THEN** the result does not describe that interrupted group as atomically rolled back
- **AND** current project state may become the baseline of a later prepare after retained claims are explicitly handled
- **AND** already-applied compatible outcomes are preserved during the new semantic reconciliation

#### Scenario: Retry a partially completed batch
- **WHEN** the caller retries a retained batch attempt after one or more conflict groups committed
- **THEN** the system verifies completed-group receipts against their archived changes and final spec fingerprints
- **AND** skips only groups whose committed state still verifies
- **AND** revalidates the remaining groups against current project state
- **AND** uses the local date captured by the retrying finalize invocation for newly archived groups

#### Scenario: Mixed-schema batch
- **WHEN** a batch contains changes with and without concrete specs outputs
- **THEN** each change is prepared from its resolved schema
- **AND** no-spec changes contribute no candidate targets and participate as independent readiness-and-move groups

### Requirement: Protect archive attempts during CLI operations

The system SHALL give each archive attempt an exclusive operation lock and SHALL prevent validation, finalization, discard, or cleanup from concurrently mutating the same attempt.

#### Scenario: Another operation holds the attempt lock
- **WHEN** a CLI operation cannot exclusively claim the selected attempt
- **THEN** it performs no attempt or project mutation
- **AND** reports the owning plan, process, and operation information available from the lock
- **AND** does not steal the lock based only on its age

#### Scenario: Operation releases its attempt lock
- **WHEN** prepare publication, validation, finalization, discard, or cleanup succeeds or fails normally
- **THEN** the operation releases the lock during completion
- **AND** removes a lock only when it still owns the same lock identity

#### Scenario: Agent work and user review occur between CLI operations
- **WHEN** an agent writes candidates or a user reviews validated output between phase commands
- **THEN** no long-lived attempt lock is held for that waiting period
- **AND** later validation and finalization detect candidate, manifest, token-bound, or live-baseline changes before project mutation

#### Scenario: Finalization acquires both lock scopes
- **WHEN** finalization needs to mutate the project
- **THEN** it acquires the attempt lock before the archive-root transaction lock
- **AND** releases the archive-root lock before the attempt lock
- **AND** holds both until the running operation commits or completes its in-process rollback

#### Scenario: A process leaves a stale lock
- **WHEN** inspection finds a lock whose owning process is no longer running or cannot be established safely
- **THEN** automatic cleanup leaves the lock and attempt unchanged
- **AND** reports the recovery state for the attempt lock and any phased archive-root claim associated with the same plan
- **AND** requires explicit unlock confirmation before removing the stale claim
- **AND** stale-claim removal verifies that each selected identity, plan ownership, and nonce did not change and does not modify attempt data or project content

### Requirement: Dispose of archive attempts by lifecycle state

The system SHALL scan generated archive attempts at the start of every prepare, remove an attempt after successful finalization or explicit discard, automatically remove an older attempt only when current project state proves that every prepared member change was completely archived, and require user choice for every other cleanup.

#### Scenario: Prepare runs lifecycle cleanup
- **WHEN** a single or batch prepare starts
- **THEN** it scans only exact generated attempt directories
- **AND** automatically removes only unlocked attempts proven consumed by matching complete archives
- **AND** returns stale, partial, orphaned, invalid, or locked attempts with the actions available for each state

#### Scenario: Resumable attempt overlaps the requested changes
- **WHEN** lifecycle inspection finds an active matching attempt or partially completed batch that includes any requested change
- **THEN** default prepare returns the overlapping attempts and creates no new attempt workspace
- **AND** requires the user to choose resume, inspect or cleanup and retry, explicit new attempt, or cancel
- **AND** an explicit new attempt requires both `--new-attempt` and `--yes`

#### Scenario: User cancels before finalization
- **WHEN** the user declines final confirmation and the client discards the attempt
- **THEN** candidate and snapshot files are removed
- **AND** main specs and the active change remain unchanged

#### Scenario: A previous attempt has already been consumed
- **WHEN** cleanup can claim the attempt lock
- **AND** every prepared member change is absent from active changes
- **AND** every member has an archived change matching the plan's root, change name, and prepared tree fingerprint
- **THEN** cleanup removes that consumed attempt without using an age threshold

#### Scenario: Attempt is stale, partial, orphaned, invalid, or locked
- **WHEN** an active change differs from its prepared snapshot, only part of a batch completed, no matching active or archived change exists for a member, metadata cannot be verified, or the attempt lock is unavailable
- **THEN** automatic cleanup retains the attempt
- **AND** asks the user through the client to choose cleanup, recovery or retry where supported, or continued retention
- **AND** performs no cleanup until that choice is explicit

#### Scenario: User explicitly cleans an invalid attempt
- **WHEN** prepare reports an invalid generated attempt with a cleanup token and the user authorizes cleanup
- **THEN** cleanup rechecks that the attempt is the same direct non-link child of the system temporary directory with the exact generated prefix, filesystem identity, scanned metadata fingerprint, and exclusive lock
- **AND** deletes only that verified attempt
- **AND** does not mutate main specs, active changes, or archives

#### Scenario: User removes a retained attempt after interrupted finalization
- **WHEN** the user explicitly cleans or discards a retained attempt after a finalizing process ended abruptly
- **THEN** only the temporary attempt data and owned claims selected for cleanup are removed
- **AND** main-spec writes and archive moves already completed by the earlier process remain unchanged

#### Scenario: Automatic cleanup encounters an unrelated temporary directory
- **WHEN** cleanup finds a directory that does not carry the exact generated archive-attempt prefix and valid attempt metadata
- **THEN** it leaves that directory untouched
