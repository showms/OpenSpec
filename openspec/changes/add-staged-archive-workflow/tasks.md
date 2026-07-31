## 1. Staged CLI Contract

- [ ] 1.1 Add/register prepare/status/validate/finalize/abort and root-level cleanup stages plus repeatable include/exclude options in CLI help and completion metadata without changing no-stage option meanings
- [ ] 1.2 Implement centralized validation for change-bound versus root-level stages, validation UUIDs, prepare-only include/exclude/skip selection, duplicate/unknown/overlap conflicts, `--no-validate`, current-validation identity, and JSON confirmation
- [ ] 1.3 Define versioned prepare/status/validate/finalize/abort/cleanup/retry/completion JSON shapes, structured actions, and archive-specific diagnostics
- [ ] 1.4 Add compatibility tests proving direct archive flags, prompts, output, and JSON meanings remain unchanged
- [ ] 1.5 Add command tests for every valid stage and invalid option combination

## 2. Immutable Plan and Validation Storage

- [ ] 2.1 Add explicit constants/types for deterministic change keys, immutable minimal manifests, prompt-source hashes, candidates, current-validation pointers/snapshots, commit markers, receipts, and the root mutation lock
- [ ] 2.2 Derive `.archive-plan` from the selected planning home, create its ignore file, and publish one complete active plan per change plus immutable validation snapshots through generated temporary siblings and atomic rename
- [ ] 2.3 Validate every stored shape and resolve relative authority with canonical containment, nearest-existing-parent, symlink checks, and root/change identity binding
- [ ] 2.4 Implement current-validation advancement, superseded-snapshot cleanup, completed-receipt retention, and root cleanup that never removes prepared/validated/committing plans
- [ ] 2.5 Add storage tests for malformed state, incomplete publication, traversal/symlink escapes, repeated same-change prepare, current-validation replacement, receipt retry, force-cleaning diagnostics, and cleanup boundaries
- [ ] 2.6 Add Windows storage/path tests for drive/case aliases, device names, alternate data streams, slash variants, and trailing-dot/space aliases

## 3. Prepare Stage

- [ ] 3.1 Extract shared readiness/task helpers, resolve deltas only from concrete specs outputs, and apply validated all/include/exclude/mixed/none selection
- [ ] 3.2 Resolve the existing local-date/no-double-prefix archive destination and build a deterministic `lstat`-based change inventory without following symlinks
- [ ] 3.3 Snapshot selected delta/base/target state, readiness warnings, prompt-source hashes, operation summaries, and exact scopes without persisting context/guidance/rule text
- [ ] 3.4 Create included candidates from exact base bytes or the shared Purpose-aware skeleton and support move-only/mixed-schema plans
- [ ] 3.5 Publish the change-bound immutable plan without locks/formal writes and return complete selection, `agentWork`, warnings, and structured status/validate/abort actions
- [ ] 3.6 Add prepare contract tests for no formal mutation, exact scopes, no follow-up discovery, all/include/exclude/mixed/none selection, invalid overlap, and complete inventory
- [ ] 3.7 Add prepare integration tests for selected stores, nested capabilities, Windows paths, repeated same-change prepare, prompt-source drift, and schemas without specs outputs

## 4. Validate Stage

- [ ] 4.1 Recheck plan integrity, prompt-source hashes when reconstructing agent work, complete change inventory, included delta/base/target/archive freshness, root identity, and path safety
- [ ] 4.2 Validate stable candidate bytes for canonical main-spec structure, operation outcomes, and preservation of unaffected requirements/scenarios
- [ ] 4.3 Produce complete deterministic unified diffs/statistics and base/candidate/review hashes
- [ ] 4.4 Publish exact reviewed bytes plus an immutable validation record, atomically advance the current-validation pointer, invalidate prior finalize actions, and support move-only snapshots
- [ ] 4.5 Return versioned review and structured status/finalize/abort actions without writing main specs or moving the change
- [ ] 4.6 Add validate tests for parse/operation/preservation failures, stale inputs, changed main-spec base hashes, complete review output, candidate edits, superseded snapshots, and no formal mutation

## 5. Mutation Lock, Status, Finalize, Abort, Cleanup, and Resume

- [ ] 5.1 Implement one planning-root mutation lock with owner metadata/token, bounded acquisition, owner-checked release, and explicit crash-repair behavior without age-only stealing
- [ ] 5.2 Make staged finalize, standalone sync formal commit, direct archive formal mutation, abort, and cleanup use the root mutation lock and recheck relevant base hashes after acquisition
- [ ] 5.3 Implement non-mutating staged status with none/prepared/validated/committing/completed states, freshness details, applied-target progress, locations, and structured legal actions
- [ ] 5.4 Require the current validation, recheck source/destination/included-base freshness, preflight every target, and create the validation-bound commit marker only after a conflict-free preflight
- [ ] 5.5 Classify targets as prepared-base/absence, reviewed-candidate, or conflict and atomically write only prepared-base targets from the current immutable validation snapshot
- [ ] 5.6 Stop conflicts without overwrite/rollback and return preserved evidence plus actionable status/repair/resume guidance
- [ ] 5.7 Verify reviewed target hashes and implement staged same-filesystem rename with retryable handle diagnostics and explicit `EXDEV` rejection
- [ ] 5.8 Recover crash-after-rename only with this plan's matching commit marker, write/retain the completion receipt, and make repeated finalize return the same result
- [ ] 5.9 Implement abort for uncommitted plans, reject abort after commit start, report receipts for completed plans, and implement bounded root cleanup
- [ ] 5.10 Add failure/concurrency tests for changed base hashes, partial-write resume, already-applied targets, external conflicts, busy commits, sync/archive serialization, rename retry, missed responses, status, cleanup, and abort boundaries
- [ ] 5.11 Add Windows/macOS/Linux verification for owner-safe locking, atomic individual replacement, path aliases, sharing violations, same-filesystem rename, and cross-device rejection

## 6. Single Archive and Sync Skill Integration

- [ ] 6.1 Extend sync templates to consume archive `agentWork`, edit only listed candidates, and obtain CLI-prepared candidates/base hashes for standalone sync
- [ ] 6.2 Rewrite the single archive template to prepare, reconcile candidates, validate an immutable snapshot, present the complete review, confirm, and execute returned structured actions
- [ ] 6.3 Handle readiness warnings and archive-without-sync by aborting only uncommitted plans and re-preparing with `--skip-specs`
- [ ] 6.4 Handle retryable finalize/conflict results by accurately reporting applied specs versus active/archived change state and preserving resume guidance
- [ ] 6.5 Add exact old-CLI capability detection and prove real staged failures never fall back to agent-owned formal writes or moves
- [ ] 6.6 Implement standalone sync's short CLI-owned mutation-lock commit, all-target base preflight, atomic candidate writes, all-or-no-write conflict behavior, and generated temporary cleanup
- [ ] 6.7 Regenerate single archive/sync skills, update parity hashes, and test that templates do not rediscover inputs, write formal specs directly, or reconstruct staged commands

## 7. Bulk Archive Integration

- [ ] 7.1 Record an explicit per-change included/excluded capability partition and deterministic dependency/order graph before preparing plans
- [ ] 7.2 Pass included and excluded capabilities explicitly, use `--skip-specs` for none, compare prepare's complete returned partition, and abort uncommitted plans on discovery drift
- [ ] 7.3 Run the complete lifecycle sequentially for overlapping included targets so later prepare observes earlier committed specs
- [ ] 7.4 Allow non-overlapping candidate work/validation independently while formal finalization remains serialized by the root mutation lock
- [ ] 7.5 Preserve completed/failed/retryable/dependency-skipped/user-skipped/sync-skipped reporting and mixed-schema/move-only behavior
- [ ] 7.6 Add bulk tests for selection drift, ordering, partial/retryable results, cancellation, mixed schemas, review identity, and fallback; regenerate the bulk skill/parity hash

## 8. Documentation and Verification

- [ ] 8.1 Document staged commands, include/exclude selection, single-plan/current-validation identity, forward-only commit semantics, status, abort boundary, resume actions, and diagnostics
- [ ] 8.2 Document the shared root mutation lock, standalone sync base-hash behavior, completed-receipt retention, explicit cleanup, ignored-state loss, and manual conflict repair
- [ ] 8.3 Document that reviewed specs may remain applied while rename fails, including Windows handle-release guidance, unsupported cross-filesystem movement, and commit-bound recovery
- [ ] 8.4 Update generated help, completion expectations, supported-tool references, and add a changeset
- [ ] 8.5 Run focused storage/validation/lock/resume/receipt/template/CLI/path suites plus `pnpm build`, `pnpm lint`, the full test suite, and Windows/macOS/Linux CI
- [ ] 8.6 Run `openspec validate add-staged-archive-workflow --strict` and verify regenerated skills/parity hashes are clean
