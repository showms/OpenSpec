## Why

Archive currently splits responsibility inconsistently between generated skills and the direct CLI: agents can edit main specs and move change directories themselves, while `openspec archive` has a separate deterministic merge and validation path. A staged workflow is needed so agents retain semantic Markdown reconciliation while OpenSpec owns the reviewed bytes, freshness checks, formal writes, and final archive move.

The first design attempted to make spec writes and the directory move one rollback-capable filesystem transaction. That is unnecessary for OpenSpec: syncing validated main specs while leaving a change active is already a supported state. A forward-only, resumable commit gives the useful safety boundary with substantially less recovery and locking machinery.

## What Changes

- Extend the existing `openspec archive <change>` command with `--stage prepare|status|validate|finalize|abort`, root-level `--stage cleanup`, and stage-specific selection/validation options, without adding a separate archive-plan command.
- Add repeatable `--include-spec` and `--exclude-spec` selection for single-change prepare. Bulk supplies and verifies an explicit included/excluded partition so discovery drift cannot silently change reviewed scope.
- Make `prepare` inspect readiness, discover delta specs, resolve safe paths, snapshot every included main-spec base hash or absence, and create one durable active plan per change under `openspec/.archive-plan/` without modifying main specs or moving the change.
- Limit agent work to plan-owned candidate specs. Agents no longer write main specs or perform the archive move in the staged path.
- Make `validate` replace the plan's current immutable validation snapshot, return a complete diff, and issue an opaque validation identifier. Superseded validation identifiers cannot finalize.
- Make `finalize` require explicit confirmation and the current validation identifier, recheck included main-spec hashes after acquiring one planning-root mutation lock, and apply the reviewed snapshot through idempotent forward progress.
- Treat each target already equal to the reviewed candidate as completed, each target still equal to its prepared base as pending, and any other target as a conflict. Interrupted finalization resumes instead of rolling back validated spec writes.
- Expose staged status by change name so a later process can recover the current state and exact next actions without retaining a plan identifier or inspecting internal files.
- Allow only one active staged archive plan per change. A second prepare returns the existing plan status instead of publishing an independent competing plan.
- Retain a small completed receipt so a caller can safely retry after the archive committed but before the success response was observed.
- Persist only recovery-authoritative hashes, paths, candidates, the current validation snapshot, commit state, and completion receipt. Reconstruct prompt inputs only while their recorded source hashes remain current, and provide explicit cleanup for expired completed state.
- Coordinate every OpenSpec-owned formal main-spec commit: staged finalize, direct archive, and standalone sync formal writes use the same short-held planning-root mutation lock. Agent semantic work remains lock-free and candidate-only.
- Keep staged archive movement rename-only in this version. Retryable rename failures leave validated spec writes and the active change available for a later finalize attempt; staged archive does not use copy/remove fallback.
- Update single and bulk archive skills to orchestrate the staged CLI contract. Bulk passes an explicit per-change capability selection and processes overlapping capabilities sequentially.
- Preserve the existing direct `openspec archive <change>` interface and output when `--stage` is omitted. Direct archive filesystem mutation participates in the same planning-root mutation lock so it cannot race a staged finalize or standalone sync commit.
- Preserve a narrow compatibility fallback for generated skills running with an older CLI that does not support `--stage`.

## Capabilities

### New Capabilities

- `archive-staged-workflow`: Defines single-plan prepared candidates, current immutable validation identity, shared mutation-lock coordination, forward-only finalization, status/retry behavior, completed receipts, cleanup, abort behavior, rename-only limitations, and bulk composition.

### Modified Capabilities

- `cli-archive`: Adds staged status/cleanup, include/exclude selection, current-validation, resumable-finalization, and diagnostic contracts while retaining the direct archive interface.
- `opsx-archive-skill`: Moves formal archive writes and movement from agent-authored shell steps to the staged CLI lifecycle, with version-skew fallback behavior.
- `specs-sync-skill`: Makes archive-driven reconciliation candidate-only and routes standalone sync formal writes through a base-hash-checked CLI commit using the shared mutation lock.

## Impact

- Affects archive CLI parsing and execution, archive validation, root/store-aware path handling, main-spec writes, archive movement, JSON diagnostics, and generated archive/sync skills.
- Adds minimal per-change state under `openspec/.archive-plan/`, including editable candidates, one current immutable validation snapshot, a commit marker, and a retained completion receipt.
- Serializes OpenSpec-owned formal main-spec commits and archive movement per planning root. Prepare, candidate editing, validation, and user review remain lock-free.
- Changes failure semantics: once formal writes begin, staged archive only resumes forward. A failed move may leave reviewed main specs applied while the change remains active.
- Staged archive no longer promises automatic cross-filesystem or Windows copy/remove fallback. Rename failures are actionable and retryable.
- Requires cross-platform tests for path safety, include/exclude drift, base-hash conflicts, archive/sync serialization, atomic individual file replacement, partial-write resume, status/cleanup, rename failure, completed-receipt retry, mixed-schema/bulk behavior, and old-CLI compatibility.
