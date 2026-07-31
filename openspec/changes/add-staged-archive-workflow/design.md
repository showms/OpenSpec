## Context

See `proposal.md` for motivation. Today the generated archive skill and direct CLI implement different archive paths:

- The skill performs readiness inspection, semantic spec sync, verification, and the final directory move through agent instructions.
- `ArchiveCommand` validates and deterministically rebuilds specs in-process before writing them and moving the change.
- Archive-driven `/opsx:sync` writes main specs before the archive outcome is known.

The staged path must work across separate CLI processes and agent sessions, selected stores as well as repo-local roots, and Windows as well as POSIX filesystems.

The important domain property is that main specs may be synced before a change is archived. Standalone `/opsx:sync` already creates that state intentionally. Therefore finalization does not need to simulate a database transaction across several files and a directory move. It needs to ensure that every formal write came from an exact reviewed snapshot, never overwrites unrelated newer work, and can be retried safely.

```bash
openspec archive add-login --stage prepare --json
openspec archive add-login --stage prepare --include-spec auth --exclude-spec billing --json
openspec archive add-login --stage status --json
openspec archive add-login --stage validate --json
openspec archive add-login --stage finalize --validation <validation-id> --yes --json
openspec archive add-login --stage abort --json
openspec archive --stage cleanup --json
```

## Goals / Non-Goals

**Goals:**

- Keep semantic Markdown reconciliation agent-driven while isolating it in plan-owned candidates.
- Make the exact reviewed candidate bytes the only bytes a staged finalize can write.
- Detect source, target, and path drift before overwriting newer work.
- Make interrupted formal writes resumable without backups, rollback, or a multi-file journal.
- Coordinate OpenSpec-owned formal main-spec mutation with one planning-root mutation lock held only during staged finalize, standalone sync commit, direct archive mutation, abort, and completed-plan cleanup.
- Keep at most one active archive plan and one current validation snapshot per change, discoverable by change name.
- Persist only state required for freshness, exact review identity, commit recovery, and completion retry.
- Preserve direct archive behavior and generated-skill compatibility.
- Define bulk and mixed-schema behavior without a batch/base-plan CLI surface.

**Non-Goals:**

- Provide all-or-nothing rollback across several main specs and the change-directory move.
- Guarantee that a move failure leaves main specs at their prepared base; reviewed spec writes may remain applied.
- Hold locks while an agent edits candidates or a user reviews output.
- Add per-target locks, multiple competing plans for one change, backups, or a write-ahead mutation journal.
- Automatically copy/remove a staged archive across filesystems or around a rename failure.
- Replace the agent with a semantic Markdown merge algorithm.
- Make prompt context, guidance, read scope, or write scope an operating-system security boundary.
- Change the existing positional change-name contract or add an `archive-plan` command.

## Decisions

### D1: Dispatch staged execution inside the existing archive command

Extend `ArchiveOptions` with:

```ts
type ArchiveStage = 'prepare' | 'status' | 'validate' | 'finalize' | 'abort' | 'cleanup';

interface ArchiveOptions {
  stage?: ArchiveStage;
  includeSpec?: string[];
  excludeSpec?: string[];
  validation?: string;
  // existing options remain
}
```

`ArchiveCommand.execute()` validates option combinations and dispatches to a staged archive service when `stage` is present. With no stage, it continues through the existing direct path and keeps the current flags, prompts, and output contract.

Stage-specific rules:

- `prepare`, `status`, `validate`, `finalize`, and `abort` require a change name. Root-level `cleanup` rejects a change name.
- `--include-spec <capability>` and `--exclude-spec <capability>` are repeatable and valid only for `prepare`.
- Omitting selection includes every discovered concrete specs output. Includes select exactly those capabilities. Excludes include every discovered capability except those named. When both are present they must be disjoint and the response reports the resulting complete partition.
- Duplicate, unknown, or overlapping include/exclude values and unsupported skip/selection combinations fail before plan publication. `--skip-specs` selects none.
- Staged invocations reject `--no-validate`.
- `finalize` requires `--validation`; other stages reject it.
- Human finalize prompts unless `--yes` is supplied; JSON finalize requires `--yes`.
- Root-selection options apply to every stage. Plan loading derives the active-plan key from the selected root and change identity.

Direct archive acquires the same root mutation lock around its existing formal writes and move. Standalone sync performs semantic work in candidates and delegates its short formal write phase to a CLI-owned, base-hash-checked commit under that lock. This serializes every cooperative OpenSpec main-spec writer without holding a lock during agent work.

### D2: Use one durable active plan per change

Derive the plan root from the selected planning home with:

```ts
path.join(path.dirname(changesDir), '.archive-plan')
```

Do not derive it from the current working directory.

```text
openspec/.archive-plan/
├── .gitignore
├── locks/
│   └── mutation.lock
└── active/<change-key>/
    ├── manifest.json
    ├── candidates/
    │   └── specs/<capability>/spec.md
    ├── validations/<validation-id>/
    │   ├── record.json
    │   └── specs/<capability>/spec.md
    ├── current-validation.json
    ├── commit.json       # appears when formal writes begin
    └── completed.json    # retained completion receipt
```

`change-key` is a deterministic safe key derived from the root fingerprint and validated canonical change identity. It is an internal path component, not caller input. Validation IDs use one lowercase UUID lexical form; caller-supplied validation IDs are checked before path construction and resolved under the selected plan root with containment checks.

`manifest.json` is immutable after prepare. A complete plan is built in a generated temporary sibling and published by renaming it to the deterministic active path. Publication collision means the change already has prepared, validated, committing, or completed state; prepare returns that state and a structured status action rather than creating a competing plan.

Each successful validation publishes a new immutable UUID snapshot and atomically advances `current-validation.json`. The prior validation identifier immediately becomes ineligible for finalize and its snapshot becomes explicit generated cleanup state. Once `commit.json` exists, the current validation pointer is fixed to the commit-bound identifier.

The plan root contains an ignore-all `.gitignore`. Aborted plans are removed. Completed plans retain `completed.json` for a documented receipt period so retries and status can return the authoritative prior result. Root-level cleanup removes only explicit generated superseded validations, abandoned publication temporaries, and completed plan directories whose receipts exceed that period. It never removes prepared, validated, or committing plans automatically.

The workspace is durable workflow state, not a source repository artifact. Documentation warns that force-cleaning ignored files can destroy recovery state. Status reports committing state prominently before users run workspace cleanup tools.

### D3: Keep trusted state small and immutable

The immutable manifest contains only machine-owned authority:

```ts
interface ArchivePlanManifestV1 {
  version: 1;
  rootFingerprint: string;
  changeName: string;
  changeKey: string;
  changePath: string;       // planning-root relative
  archivePath: string;      // planning-root relative
  skipSpecs: boolean;
  specSelection: {
    mode: 'all' | 'include' | 'exclude' | 'mixed' | 'none';
    requestedIncluded: string[];
    requestedExcluded: string[];
    included: string[];
    excluded: string[];
  };
  createdAt: string;
  warnings: ArchivePlanWarning[];
  changeInventory: ArchiveTreeEntry[];
  files: ArchivePlanFile[];
  promptSourceHashes: ArchivePromptSourceHashes;
}
```

Each included file records capability, delta/base/candidate/target relative paths, prepared delta hash, and either the prepared base hash or prepared target absence. Excluded files carry no candidate or target-write authority.

The plan does not persist project context, guidance, or rule text. It records hashes for their source files and returns the current serializable `agentWork` in prepare. Status may reconstruct `agentWork` from current sources only when every recorded source hash still matches; otherwise it marks candidate work stale and directs the caller to abort and prepare again. Operation summaries, included/excluded mappings, and explicit path authority remain derivable from the manifest.

Normal plan loading uses explicit constant-defined paths. It does not glob for required plan files.

### D4: Prepare owns discovery and candidate creation

Prepare performs the deterministic front half:

1. Resolve the selected root and validate the change path with containment and canonical-path checks.
2. Gather artifact readiness and task progress through shared core helpers.
3. Obtain concrete delta paths exclusively from `artifactPaths.specs.existingOutputPaths`.
4. Apply the all/include/exclude/mixed/none selection rule and reject duplicate, unknown, overlapping, or unsupported capability selections.
5. Resolve the archive name using existing local-date/no-double-prefix behavior and reject an existing destination.
6. Snapshot the complete active change tree without following symlinks outside it. Record regular-file hashes, directory entries, and symlink target text so a moved tree can be identified after a crash.
7. Snapshot selected deltas, included main-spec base hashes/target absence, prompt-source hashes, warnings, and the complete resulting selection.
8. Create candidates only for included deltas:
   - copy an existing main spec byte-for-byte;
   - create the shared Purpose-aware canonical skeleton for a new capability.
9. Publish the complete immutable plan directory at the change's deterministic active-plan path, or return the already-existing state without replacing it.

Prepare does not perform semantic merging, modify main specs, move the change, or acquire the root mutation lock.

Prepare returns:

- `archivePlan`: change-bound state, selection, warnings, work directory, and confirmation need;
- `agentWork`: semantic inputs and exact scopes, or `null` for move-only work;
- `nextActions`: structured status, validate, and abort command/argument arrays.

### D5: Agent work is candidate-only

The staged archive skill passes `agentWork` directly into candidate-mode sync:

```text
read context, guidance, and rules
→ read each listed delta, optional base, and candidate
→ write only listed candidate paths
→ execute the returned validate action
```

The agent does not rediscover deltas, derive paths, open the manifest, broaden selection, write main specs, move the change, or construct staged commands.

`targetPath` is informational until formal commit. Archive-supplied candidate work proceeds to archive validate/finalize. Standalone `/opsx:sync` uses the same candidate shape but returns a structured CLI commit action that binds target paths, prepared base hashes/absence, and candidate hashes without creating an archive plan or moving a change.

The standalone sync commit is intentionally short-lived: the agent finishes all semantic candidate edits first, then the CLI acquires the root mutation lock, rechecks every prepared base, validates candidate structure, atomically writes conflict-free candidates, verifies their hashes, and releases the lock. A base mismatch writes nothing and directs the user to rerun sync from the new main-spec state.

These scopes are workflow contracts, not OS access controls. Formal authority comes from the immutable manifest, validation snapshot, hashes, path checks, and finalize behavior.

### D6: Validate creates an immutable reviewed snapshot

Validate loads the immutable plan and rechecks:

- selected root/change identity;
- complete active change inventory;
- included delta hashes;
- existing base hashes or continued target absence;
- archive destination absence;
- recorded prompt-source hashes when reconstructing agent work;
- plan and candidate path containment.

For every candidate, validate reads one stable byte snapshot and:

1. validates main-spec parsing and canonical structure;
2. rejects delta operation headings;
3. checks structural outcomes for ADDED, MODIFIED, REMOVED, and RENAMED operations;
4. checks preservation of existing requirements and scenarios not named for removal/replacement;
5. creates a complete deterministic unified diff and statistics;
6. publishes the exact candidate bytes and review metadata under a new immutable validation UUID directory.

The validation record binds:

- root/change identity;
- validation ID;
- selected capabilities/delta hashes;
- prepared base hashes/absence;
- exact candidate hashes;
- exact review hashes;
- validation time.

Finalize reads candidate bytes from the current immutable validation directory, not from the editable candidate workspace.

Revalidation creates another immutable validation snapshot and atomically advances the current-validation pointer. The previous ID still identifies its old bytes for diagnostics but is no longer eligible for finalize. If the editable candidate changes, validating it produces a new ID and invalidates previously returned finalize actions.

Move-only validation creates an empty file list and an immutable validation record bound to readiness warnings, selection, archive destination, and change inventory.

The validator checks structural intent, not prose quality. The user-facing review remains the semantic approval boundary.

### D7: Use one planning-root mutation lock for formal spec writes

All OpenSpec-owned formal main-spec mutation and archive movement under one planning root is serialized by `locks/mutation.lock`.

The lock is used by:

- staged finalize;
- the standalone sync CLI commit that applies agent-produced candidates after rechecking their recorded main-spec base hashes;
- abort, while it checks/removes a plan;
- completed-receipt garbage collection;
- direct archive during its existing spec-write/move section.

It is not used by archive/sync discovery, candidate editing, validate, diff review, or user confirmation. Standalone sync therefore does all semantic reconciliation before invoking one short CLI-owned commit.

The lock contains an owner token and process metadata. Release removes the lock only when the stored token matches the holder. A lock is never stolen solely because its mtime exceeds a fixed threshold. Crash recovery uses an explicit owner-liveness/repair policy and actionable diagnostics.

A single coarse lock intentionally serializes unrelated formal spec commits. These writes are low frequency, and this avoids per-target locks, lock ordering, target aliases, and deadlock analysis. Every archive finalize, direct archive, and standalone sync commit rechecks relevant base hashes after acquiring the lock, so a waiting writer observes earlier committed work before writing.

The direct no-stage path snapshots the main-spec bases used for its in-memory rebuild. If any base changed before it acquires the mutation lock, it rebuilds and revalidates from the locked current state before writing; it never writes an earlier rebuild over a newer base.

Hash checks are authoritative only for writers participating in this protocol. External editors can still change files outside OpenSpec. Finalize reads each target again immediately before replacement and verifies all candidate hashes afterward; a non-participating writer that races the final replacement remains an external filesystem race and is reported accurately if detected, not described as an operating-system security guarantee.

### D8: Finalize makes idempotent forward progress

Finalize acquires the root mutation lock and follows this order:

1. If `completed.json` exists and matches the plan, return its authoritative success result.
2. Load `current-validation.json`, require the requested validation identifier to match it, and verify validation/root/change identity.
3. Recheck the complete change inventory, included delta hashes, every included main-spec base/candidate state, path safety, and source/destination state. An existing destination is accepted only by the commit-bound crash-after-rename recovery case described below.
4. If no commit marker exists, preflight every target as prepared-base/absence, reviewed-candidate, or conflict. A conflict stops while the plan is still abortable.
5. After conflict-free preflight and immediately before the first formal write, create `commit.json` atomically. It binds the plan to the chosen validation ID and reviewed hashes.
6. If `commit.json` already exists, require the same current validation identity and resume that commit.
7. For each target in deterministic normalized order, read its current state again:
   - current hash/absence equals the prepared base state: write the reviewed validation snapshot atomically;
   - current hash equals the reviewed candidate hash: treat it as already completed;
   - otherwise: stop with `archive_commit_conflict` and do not overwrite the newer content.
8. Verify every included target now equals its reviewed candidate hash.
9. Rename the active change to the prepared archive path.
10. Write `completed.json` atomically with the archive result and applied-spec summary.
11. Return the receipt and release the lock.

There are no backups and no rollback. Every formal spec byte written by this path was already validated and reviewed. If writing or rename fails, `commit.json` and the validation snapshot remain, and the next finalize repeats the classification above.

Once `commit.json` exists, abort is rejected. The plan must be resumed or manually repaired if an external conflict prevents progress.

If a process stops after rename but before `completed.json`, retry first requires this plan's matching `commit.json`, then observes:

- active source absent;
- prepared archive destination present;
- archive tree matching the prepared change inventory;
- every included target matching the reviewed candidate.

It then writes the completion receipt and returns success. If source/destination identity is ambiguous, it preserves both evidence and fails closed.

This design provides exactly-once observable completion through a retained receipt, not rollback-based atomicity.

### D9: Staged movement is rename-only

Staged finalize uses a same-filesystem directory rename. It does not fall back to recursive copy/remove.

- A transient `EPERM`, sharing violation, or similar rename failure leaves reviewed spec writes in place, keeps the active change, retains `commit.json`, and returns a retryable diagnostic.
- `EXDEV` returns an explicit unsupported cross-filesystem diagnostic. The user must place the planning root on one filesystem or move it through an external/manual workflow.
- Finalize never publishes a partially copied archive directory.

This is a deliberate staged-path limitation. The direct no-stage path retains its existing compatibility behavior in this change.

### D10: Status, abort, and cleanup manage durable state explicitly

Status resolves the deterministic active-plan path from the selected root and change name. It reports one of `none`, `prepared`, `validated`, `committing`, or `completed`, the current validation identity when eligible, already-applied targets, active/archive location, freshness diagnostics, and structured legal next actions. It never mutates formal state.

Abort acquires the root mutation lock and:

- removes a prepared plan when neither `commit.json` nor `completed.json` exists;
- reports the existing completion receipt when the plan already completed;
- rejects a plan with `commit.json` and directs the caller to resume finalize;
- leaves main specs and the active change untouched for every successful abort.

A validation snapshot alone does not prevent abort. All validation directories are plan-owned and are removed with an uncommitted plan.

Root-level cleanup acquires the mutation lock and removes only explicitly recognized generated publication temporaries, superseded validations, and completed plans older than the documented receipt period. It reports prepared, validated, and committing plans but never removes them. Because ignored workspace state can be destroyed by external cleanup tools, documentation treats committing state as recovery-critical and exposes it through status.

### D11: Resolve paths defensively

All paths use `path.join()`/`path.resolve()`. Containment uses `path.relative()`, canonical `realpath` for existing paths, and the nearest existing canonical parent for targets not yet created.

Reject:

- malformed validation IDs or internal change keys;
- absolute manifest entries;
- `.`/`..`, NUL, alternate-drive, or containment escapes;
- candidate/target symlink escapes;
- Windows drive changes, case-insensitive aliases, device names, alternate data-stream syntax, and trailing-dot/space aliases;
- change names or capability identifiers that resolve to a different path than their validated canonical identity.

The change inventory uses `lstat` and records symlink target text without following it. Finalize rechecks paths after acquiring the mutation lock.

Atomic individual-file replacement must have explicit Windows behavior and tests; it may retry documented transient sharing failures but must never replace an unrelated path.

### D12: Bulk archive composes single-change commits

Bulk first gathers status for every selected change, investigates capability conflicts, and records:

- a deterministic change order;
- an explicit included capability list for every change;
- an explicit excluded capability list and rationale.

Bulk passes every included capability with repeated `--include-spec` and, when the included set is non-empty, every explicitly excluded capability with repeated `--exclude-spec`. For an empty included set it uses `--skip-specs`; prepare then returns every discovered capability as excluded for comparison with the inspected expected partition.

Prepare rediscovers capabilities and returns its complete resulting partition. Before candidate work, bulk compares that partition with the inspected included/excluded sets. Any added, removed, or reclassified capability causes the newly published uncommitted plan to be aborted and the batch item to fail as selection drift. Only included capabilities receive candidates, validation snapshots, and formal writes. Excluded deltas are reported as `sync-skipped`.

Changes with overlapping included targets run the complete lifecycle sequentially:

```text
prepare A → agent A → validate A → confirm A → finalize A
prepare B against A's committed result → agent B → validate B → confirm B → finalize B
```

Non-overlapping changes may prepare and validate independently, but finalization is serialized by the root mutation lock and each result is confirmed separately.

Bulk completion is intentionally partial:

- completed changes remain completed;
- a failed change is reported failed or retryable;
- dependent later changes are skipped;
- unrelated changes may continue;
- mixed-schema/move-only changes use the same validation/finalize contract.

### D13: Detect old CLIs narrowly

Generated archive skills probe staged status before starting work:

- a versioned staged response or staged diagnostic means staged support exists and every real failure is authoritative;
- an exact unsupported-option/help capability result means the CLI is older, so the skill announces legacy mode and follows the retained legacy workflow;
- an invalid response from a CLI advertising staged support is an error, not permission to write main specs or move directories directly.

### D14: Use explicit diagnostics and versioned JSON

Staged diagnostics include:

- `archive_stage_invalid`
- `archive_plan_invalid`
- `archive_plan_not_found`
- `archive_plan_exists`
- `archive_plan_completed`
- `archive_plan_commit_started`
- `archive_plan_stale`
- `archive_plan_validation_failed`
- `archive_validation_required`
- `archive_validation_not_found`
- `archive_validation_not_current`
- `archive_spec_selection_invalid`
- `archive_spec_selection_conflict`
- `archive_spec_selection_drift`
- `archive_commit_busy`
- `archive_commit_conflict`
- `archive_move_retryable`
- `archive_cross_device_unsupported`
- `archive_completion_recovery_failed`
- `archive_cleanup_invalid`
- existing `archive_confirmation_required`

Every staged JSON response contains `contractVersion`, `archivePlan` (or `null`), the root envelope, and structured next actions/diagnostics where applicable. Status is the authoritative recovery surface. On retryable finalize failure, the response clearly distinguishes already-applied specs from the still-active change and returns the exact resume action.

Manifest, validation-record, commit-marker, completed-receipt, and response versions evolve independently.

## Risks / Trade-offs

- **Finalize stops after some reviewed spec writes** → Keep those writes, retain the immutable validation snapshot and commit marker, and resume by classifying targets as base/candidate/conflict.
- **External work changes a target during an interrupted commit** → Never roll it back or overwrite it; fail with `archive_commit_conflict` and preserve the plan for manual repair.
- **Two changes target the same main spec** → Serialize finalize with the root mutation lock and recheck base state after acquiring it; the later change prepares from the committed result or becomes stale/conflicted.
- **Prepare is repeated for the same change** → Return the existing change-bound plan status; do not create a competing workspace.
- **Standalone sync changes an included main spec** → Its formal commit uses the shared mutation lock; archive validate/finalize sees the changed base hash and requires a new prepare.
- **A caller misses the success response** → Retain `completed.json` and return the same authoritative result on retry.
- **Root-level serialization delays unrelated archives** → Accept the small throughput cost to remove multi-lock ordering and recovery complexity.
- **Windows keeps the change directory open** → Return a retryable rename diagnostic; ask the user to release the handle and rerun finalize.
- **Planning root spans filesystems** → Reject staged rename with an actionable `EXDEV` diagnostic; do not copy/remove automatically.
- **Structural validation misses a semantic mistake** → Return the complete deterministic review and require explicit confirmation.
- **Validation snapshots consume disk** → Keep only the current snapshot eligible, clean superseded snapshots explicitly, remove them on abort, and garbage-collect completed plans after the receipt-retention period.
- **Prompt context or rules contain secrets** → Do not persist their text; retain only source hashes and reconstruct agent work only while those sources remain unchanged.
- **Ignored workflow state is force-cleaned** → Document the recovery impact, expose committing plans through status, and never claim durability against external deletion.
- **A stale lock survives a crashed process** → Use owner metadata and explicit repair/liveness logic; never steal solely by age.

## Migration Plan

1. Add staged options and versioned JSON diagnostics while keeping the no-stage interface stable.
2. Add deterministic per-change plan publication, minimal candidate/current-validation storage, path validation, status, cleanup, and completed-receipt retention.
3. Implement prepare/status/validate/abort with one active plan per change.
4. Implement the owner-aware root mutation lock and make staged finalize, direct archive, and standalone sync formal commits participate in it.
5. Implement idempotent target classification, base-hash conflict detection, atomic individual writes, rename-only movement, retryable failures, and commit-bound completion recovery.
6. Add candidate mode and CLI-owned formal commit to sync, then migrate the single archive skill with old-CLI fallback.
7. Migrate bulk archive to explicit selection and ordered per-change lifecycles.
8. Regenerate skills/parity fixtures, update docs/completions, and run cross-platform tests.
9. Roll back by removing the staged option and skill branches; direct archive remains available throughout.
