## ADDED Requirements

### Requirement: Prepared candidate reconciliation
The specs sync skill SHALL support an archive-supplied mapping that directs semantic reconciliation into prepared candidate specs.

#### Scenario: Archive supplies candidate targets
- **WHEN** archive invokes the sync workflow with an explicit list of delta, base, and candidate paths from a prepared plan
- **THEN** the agent SHALL read only those listed inputs
- **AND** reconcile each delta into its listed candidate
- **AND** SHALL NOT write the corresponding main specs

#### Scenario: Archive prepared a subset
- **WHEN** a prepared plan includes some discovered capabilities and excludes others
- **THEN** the sync workflow SHALL receive only included work items
- **AND** SHALL NOT read, reconcile, or create candidates for excluded deltas

#### Scenario: Archive supplies the prepare snapshot
- **WHEN** archive invokes the sync workflow with `agentWork`
- **THEN** the agent SHALL use its snapshotted project context, ordered archive-guidance and specs-rule string arrays, operation summaries, and scopes directly
- **AND** SHALL NOT refetch instructions, rediscover artifacts, inspect the plan manifest, or expand the write scope

#### Scenario: Candidate mapping contains an unknown delta
- **WHEN** an archive-supplied delta path is not present in the prepared plan's explicit input list
- **THEN** the sync workflow SHALL stop and report the mismatch
- **AND** SHALL NOT discover or substitute another delta through pattern matching

#### Scenario: Candidate mapping targets a new capability
- **WHEN** the prepared plan maps a delta to a candidate for a capability without a main spec
- **THEN** the agent SHALL produce canonical main-spec content in that candidate
- **AND** apply the existing new-capability Purpose behavior

#### Scenario: Candidate reconciliation completes
- **WHEN** all listed candidates have been reconciled
- **THEN** the skill SHALL summarize changes by candidate and capability
- **AND** direct the caller to archive validation rather than claiming main specs are synced

#### Scenario: Standalone sync has no candidate mapping
- **WHEN** a user invokes `/opsx:sync` directly without an archive-supplied candidate mapping
- **THEN** the skill SHALL obtain CLI-prepared candidate mappings and main-spec base hashes for the selected deltas
- **AND** perform semantic reconciliation only in those candidates
- **AND** preserve the existing user-facing reconciliation summary

### Requirement: CLI-owned standalone sync commit
The specs sync skill SHALL delegate standalone formal main-spec writes to a short CLI-owned commit coordinated with archive mutation.

#### Scenario: Standalone candidate work is ready
- **WHEN** the agent completes every selected standalone sync candidate
- **THEN** the skill SHALL execute the returned structured commit action
- **AND** the CLI SHALL acquire the planning-root mutation lock
- **AND** recheck every selected main-spec base hash or prepared absence before writing

#### Scenario: A selected main spec changed during sync
- **WHEN** the formal commit finds a selected main spec equal to neither its prepared base state nor its candidate
- **THEN** it SHALL fail without writing any selected main spec
- **AND** direct the user to restart sync from the current main-spec state

#### Scenario: Standalone sync commit succeeds
- **WHEN** every selected main spec still matches its prepared base state or candidate
- **THEN** the CLI SHALL atomically apply and verify the candidate bytes
- **AND** release the mutation lock
- **AND** the skill SHALL report the resulting main specs as synced

#### Scenario: Archive finalize is running
- **WHEN** standalone sync reaches formal commit while archive finalize holds the mutation lock
- **THEN** sync SHALL wait within the documented bound or return an actionable busy result
- **AND** SHALL recheck base hashes after acquiring the lock

#### Scenario: Semantic work does not hold the mutation lock
- **WHEN** the agent reads deltas or edits standalone candidates
- **THEN** it SHALL NOT hold the planning-root mutation lock
