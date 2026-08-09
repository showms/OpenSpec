## MODIFIED Requirements

### Requirement: Archive Process Enhancement

The archive process SHALL reconcile delta changes into canonical main specifications through either the compatible one-shot programmatic merger or a semantic candidate that passed the phased archive validation contract.

#### Scenario: One-shot archive applies changes programmatically

- **WHEN** archiving a completed change through the existing one-shot command
- **THEN** the archive command SHALL:
  1. Parse RENAMED sections first and apply renames
  2. Parse REMOVED sections and remove by normalized header match
  3. Parse MODIFIED sections and replace by normalized header match (using new names if renamed)
  4. Parse ADDED sections and append new requirements
- **AND** validate that all MODIFIED headers exist in current spec
- **AND** treat a REMOVED header that is already absent as already removed (warn and continue; a REMOVED header that names the FROM side of a RENAMED in the same delta, compared case- and whitespace-insensitively, or that differs only in case or whitespace from an existing requirement, is a conflict)
- **AND** treat an ADDED header that already exists with identical content as already synced (differing content is a conflict)
- **AND** treat a RENAMED whose source is gone but target present as already synced
- **AND** generate the updated spec in the main specs directory

#### Scenario: Phased archive uses a semantic candidate

- **WHEN** an archive-attempt plan assigns one baseline and one or more contributing change deltas to an agent-driven candidate entry
- **THEN** the agent MAY reconcile meaning and rewrite the candidate clearly instead of reproducing the programmatic merge byte-for-byte
- **AND** a shared batch target SHALL produce one aggregate candidate that preserves compatible intent from every included contribution
- **AND** the candidate SHALL retain canonical main-spec structure with no delta operation headers
- **AND** OpenSpec SHALL validate the candidate, destination, retirement intent, and current inputs before promoting it
- **AND** artifact rules SHALL constrain only the candidate spec produced from that artifact

#### Scenario: Handling conflicts during one-shot archive

- **WHEN** delta changes conflict with current spec state in the programmatic compatibility path
- **THEN** the archive command SHALL report specific conflicts
- **AND** require manual resolution before proceeding
- **AND** provide clear guidance on resolving conflicts

#### Scenario: Candidate inputs become stale

- **WHEN** a contributing delta, baseline, config, change, archive root, or naming policy changes after a semantic candidate was prepared
- **THEN** phased archive SHALL reject the stale candidate before promotion
- **AND** require a new prepare and validation cycle against current inputs

#### Scenario: New prepare follows an interrupted candidate promotion

- **WHEN** a prior finalizing process terminated after promoting only part of its candidate outcomes
- **THEN** a later prepare SHALL treat the current canonical main specs as its new baselines
- **AND** semantic reconciliation SHALL preserve outcomes those baselines already represent
- **AND** produce only the remaining changes needed to reach the delta intent
