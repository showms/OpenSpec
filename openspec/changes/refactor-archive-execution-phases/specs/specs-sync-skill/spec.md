## ADDED Requirements

### Requirement: Archive candidate workspace mode

The specs sync skill SHALL support an archive-supplied candidate mode that performs semantic reconciliation only for explicitly planned entries and writes no main spec directly.

#### Scenario: Archive supplies planned candidate entries
- **WHEN** archive invokes sync inline with a supported plan, selected entry list, immutable baseline and delta snapshots, candidate paths, and `specs` rule snapshot
- **THEN** sync reconciles only those entries into their corresponding candidate paths
- **AND** reuses the supplied artifact-rule snapshot without fetching it again
- **AND** records an explicit `sync`, `retire`, or `already-synced` result for every processed target

#### Scenario: Batch entry has multiple contributing changes
- **WHEN** one candidate entry supplies multiple change deltas for the same canonical main-spec target
- **THEN** sync reconciles every included contribution against the one prepared baseline into one aggregate candidate
- **AND** uses creation date as the stable input order and proposed priority without silently discarding compatible earlier intent
- **AND** reports every incompatible semantic choice with its contributing changes for explicit user confirmation

#### Scenario: Included artifact rules conflict
- **WHEN** included contributions supply artifact rules that cannot both control the same candidate content
- **THEN** sync pauses that target and reports the conflicting rule text and source changes
- **AND** waits for the user to select the controlling rule for each conflict
- **AND** preserves all non-conflicting rules
- **AND** regenerates the candidate under the selected rules and records the selected and suppressed rule sources

#### Scenario: User skips a prepared contribution
- **WHEN** the archive workflow records `skip` for a change contribution or target
- **THEN** sync excludes that skipped work from semantic reconciliation
- **AND** does not read its artifact rules or create a candidate solely for that skipped work

#### Scenario: Candidate mode receives an unknown entry
- **WHEN** the caller asks sync to process an entry that is not present in the prepared plan
- **THEN** sync reports the invalid entry and stops
- **AND** does not widen the selection or write any main spec

#### Scenario: Candidate mode finishes semantic work
- **WHEN** all selected candidates have been reconciled
- **THEN** sync reports a semantic summary, per-change contributions, and baseline-to-final-state diff for user review
- **AND** does not claim CLI validation or archive completion
- **AND** leaves candidate promotion, retirement, and the archive move to finalization

#### Scenario: A new baseline already contains part of the intended outcome
- **WHEN** a new archive attempt uses current main specs that already represent one or more delta outcomes from an earlier interrupted attempt
- **THEN** sync treats those outcomes idempotently and records `already-synced` where appropriate
- **AND** reconciles only the remaining intended outcome into candidates
- **AND** does not duplicate requirements, scenarios, renames, or retirements already represented by the baseline

#### Scenario: Standalone sync is invoked
- **WHEN** the user invokes `/opsx:sync` without an archive candidate plan
- **THEN** standalone root resolution, artifact discovery, rule loading, main-spec writes, validation, and output remain unchanged
