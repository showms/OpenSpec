## ADDED Requirements

### Requirement: Prepared candidate reconciliation

The specs sync skill SHALL support archive-supplied candidate work that keeps semantic reconciliation away from formal specs.

#### Scenario: Archive supplies candidate targets

- **WHEN** archive invokes sync with explicit delta, persisted base or typed absence, candidate, and informational target mappings
- **THEN** the agent SHALL read only the listed semantic inputs
- **AND** reconcile each selected delta into its listed candidate
- **AND** SHALL NOT write the corresponding formal main spec

#### Scenario: Archive prepares a subset

- **WHEN** prepared candidate work includes some discovered capabilities and excludes others
- **THEN** sync SHALL process only the included work items
- **AND** SHALL NOT read, reconcile, or create candidates for excluded deltas

#### Scenario: Archive supplies a complete work package

- **WHEN** archive invokes sync with `agentWork`
- **THEN** the agent SHALL use its project context, ordered guidance, specs rules, operation summaries, mappings, and scopes directly
- **AND** SHALL NOT refetch instructions, rediscover artifacts, inspect the plan manifest, or expand the write scope

#### Scenario: Candidate mapping contains an unknown delta

- **WHEN** a requested delta path is absent from the prepared work package
- **THEN** sync SHALL stop and report the mismatch
- **AND** SHALL NOT discover or substitute another delta through pattern matching

#### Scenario: Candidate targets a new capability

- **WHEN** prepared work maps a delta to a candidate for a capability without a main spec
- **THEN** sync SHALL produce canonical main-spec content in that candidate
- **AND** retain the existing Purpose behavior for new capabilities

#### Scenario: Modified delta omits a base scenario

- **WHEN** a normal prepared candidate base contains a scenario that its selected `MODIFIED` delta does not mention
- **THEN** reconciliation SHALL preserve that scenario in the candidate
- **AND** SHALL treat only whole-requirement `REMOVED` as removal authority in this workflow

#### Scenario: Candidate reconciliation preserves unrelated semantics

- **WHEN** archive-supplied delta operations are reconciled into a normal prepared candidate
- **THEN** sync SHALL preserve existing Purpose content, unaffected requirements, shared requirement/scenario identity, and scenario multiplicity
- **AND** apply `RENAMED` identity before a related `MODIFIED` operation
- **AND** SHALL NOT introduce unrelated semantic additions, removals, or renames

#### Scenario: Candidate reconciliation completes

- **WHEN** all listed candidates have been reconciled
- **THEN** sync SHALL summarize changes by candidate and capability
- **AND** direct the caller to the returned archive validate action
- **AND** SHALL NOT claim that formal specs are synced or that the change is archived

#### Scenario: Archive supplies conflict recovery candidate work

- **WHEN** archive invokes sync with an evidence-bound recovery package containing the original base, original reviewed snapshot, captured current target, selected delta, and one recovery candidate
- **THEN** the agent SHALL compare those listed versions and reconcile only the recovery candidate
- **AND** initialize its reasoning from the captured current target and preserve its unrelated semantics by default
- **AND** clearly identify any intentional restoration of original reviewed bytes or removal of captured current content in its summary
- **AND** rely on the complete current-to-recovery amendment review and new user approval, rather than the original delta alone, as authority for any such intentional removal
- **AND** direct the caller to the returned recovery validate action
- **AND** SHALL NOT write the formal target, inspect unrelated plan state, invoke standalone sync, or claim that the conflict is resolved

#### Scenario: Recovery evidence changes during reconciliation

- **WHEN** a listed recovery input no longer matches the archive-supplied evidence package
- **THEN** sync SHALL stop and report the stale recovery package
- **AND** SHALL NOT rediscover, substitute, or merge from a different formal target

#### Scenario: Standalone sync has no candidate mapping

- **WHEN** a user invokes `/opsx:sync` without archive-supplied candidate work
- **THEN** sync SHALL retain its existing formal reconciliation and user-facing summary
- **AND** SHALL NOT claim participation in staged archive state or concurrent-writer coordination
