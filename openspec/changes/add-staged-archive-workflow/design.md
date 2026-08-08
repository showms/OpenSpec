## Context

Archive has two user-facing entry points:

- generated archive skills perform semantic spec reconciliation and move the change through agent instructions;
- `openspec archive` deterministically applies deltas and moves the change in one CLI invocation.

The staged workflow gives both paths the same safety boundary. Agents edit only plan-owned candidates. The CLI freezes the exact reviewed bytes and archive payload manifest, writes formal specs, moves the change, and records enough state to continue after an interrupted process. When normal resume is impossible, evidence-bound repair actions keep recovery inside the same CLI-owned boundary.

Archive uses a short-lived planning-root archive commit lock. Direct archive, staged finalize, and each bulk item acquire the same lock before their authoritative final preflight and hold it through formal-spec writes, movement, and durable receipt publication. Candidate reconciliation and user review happen outside the lock.

The lock serializes CLI-owned archive commits, not every possible formal-spec writer. Standalone `/opsx:sync` remains agent-driven and manual edits remain outside the lock, so both are unsupported during archive finalization. Hash checks still detect observable drift from those external writers. A future all-writer gate would require standalone sync to use a candidate plus a short CLI-owned commit phase.

```bash
openspec archive add-login --stage prepare --json
openspec archive add-login --stage prepare --include-spec auth --exclude-spec billing --json
openspec archive add-login --stage prepare --archive-name 2026-08-02-add-login-2 --json
openspec archive add-login --stage status --json
openspec archive add-login --stage validate --json
openspec archive add-login --stage finalize --validation <validation-id> --approval <approval-token> --yes --json
openspec archive add-login --stage repair --recovery <recovery-id> --resolution <decision> --approval <approval-token> --yes --json
openspec archive add-login --stage abort --json
openspec archive --stage cleanup --json
```

## Goals / Non-Goals

**Goals:**

- Isolate agent-authored Markdown reconciliation in plan-owned candidates.
- Make the exact reviewed candidate bytes the only bytes a staged finalize can write.
- Bind approval to the exact complete review and archive payload manifest rather than to a summary or reconstructed command.
- Keep the approval binding independent of the final archive destination, which is selected and recorded when commit begins.
- Detect source or base drift before formal writes begin.
- Serialize direct, staged, and bulk archive commits for one planning root during their short formal commit windows.
- Make interrupted per-file writes and archive movement safe to retry without rollback.
- Give formal-spec conflicts and orphaned movement evidence an explicit, reviewed, CLI-owned recovery path.
- Recover a missing or damaged primary plan when one consistent source-local recovery capsule remains.
- Keep one active plan and one current validation per change, discoverable by change name.
- Keep status completely read-only and derive recovery identifiers from a versioned canonical hash of the observed evidence.
- Preserve a complete review without forcing an unbounded JSON or terminal response.
- Keep staged behavior consistent across repo-local roots, selected stores, Windows, macOS, and Linux.
- Preserve the direct archive interface while sharing the same safety primitives.
- Keep bulk behavior predictable by processing every selected change sequentially.

**Non-Goals:**

- Provide an all-or-nothing transaction across several specs and a directory move.
- Coordinate standalone sync or manual formal-spec writers with the archive commit lock.
- Hold a lock during prepare, candidate reconciliation, review, conflict investigation, or user approval.
- Convert standalone `/opsx:sync` into a CLI-owned candidate, validation, and finalize lifecycle.
- Automatically merge content that changes after a formal commit starts.
- Roll back reviewed spec writes that completed before an interruption.
- Guess which source, destination, marker, or receipt owns an orphaned commit.
- Recover after the user or another tool deletes every primary and source-local copy of commit-critical evidence.
- Support competing staged plans for one change.
- Treat prompt read/write scopes as an operating-system security boundary.
- Add individual scenario removal through `MODIFIED`; omission continues to preserve existing scenarios.

## Decisions

### D1: Use the existing archive command as the staged surface

`ArchiveOptions` gains the following staged fields:

```ts
type ArchiveStage =
  | 'prepare'
  | 'status'
  | 'validate'
  | 'finalize'
  | 'repair'
  | 'abort'
  | 'cleanup';

type ArchiveRepairResolution =
  | 'prepare-spec-conflict-resolution'
  | 'reconstruct-plan'
  | 'reclaim-archive-lock'
  | 'resume-source'
  | 'adopt-destination'
  | 'quarantine-source-and-adopt-destination'
  | 'rebind-destination';

interface ArchiveOptions {
  stage?: ArchiveStage;
  includeSpec?: string[];
  excludeSpec?: string[];
  validation?: string;
  approval?: string;
  recovery?: string;
  resolution?: ArchiveRepairResolution;
  archiveName?: string;
  // existing options remain
}
```

`prepare`, `status`, `validate`, `finalize`, `repair`, and `abort` require a change name. Root-level `cleanup` rejects a change name. Selection options are valid only for `prepare`; `--skip-specs` creates a move-only plan. `--archive-name` is valid for prepare and for a `rebind-destination` repair only. `--recovery` is required for repair and is also accepted by recovery-mode validate/finalize. `--resolution` is valid only for repair. Staged invocations reject `--no-validate`. Finalize requires `--validation` and `--approval`. Every repair first recomputes and matches the supplied evidence-derived recovery ID. `prepare-spec-conflict-resolution` is a generated-state-only preparatory repair: without an approval token it immediately persists the repair record and recovery candidate, then returns the recovery validate action while leaving formal specs and archive locations unchanged. Every other repair without `--approval` persists only its complete repair review and returns an approval token; executing that returned repair requires the token and `--yes` in JSON mode.

An archive name is one validated basename, not a path. It rejects separators, traversal, platform aliases, reserved marker names, and collision with the active change or generated state. When omitted, the existing local-date naming rule remains authoritative. Prepare reports a currently observable default-name collision before candidate work, while finalize still binds and rechecks the actual destination for its commit date.

The command returns versioned JSON with structured next actions. Approval tokens are opaque identifiers stored with a validation or repair-review record and bind its plan ID, validation or recovery ID, review path, review hash, byte length, delivery mode, payload-manifest hash, selected repair decision, and every explicit repair input. A normal validation approval deliberately does not bind the final archive destination. Generated skills execute returned command and argument arrays instead of reconstructing staged or recovery commands.

Alternative considered: a separate `archive-plan` command. Keeping one archive command preserves discoverability and reuses the existing root/store selection surface.

### D2: Derive state from a small single-plan record

The selected planning home owns generated state at:

```ts
path.join(path.dirname(changesDir), '.archive-plan')
```

```text
openspec/.archive-plan/
├── .gitignore
├── archive-commit.lock
├── active/<change-key>/
│   ├── manifest.json
│   ├── bases/specs/<capability>/spec.md
│   ├── candidates/specs/<capability>/spec.md
│   ├── validations/<validation-id>/
│   │   ├── record.json
│   │   ├── review.diff
│   │   ├── payload-manifest.json
│   │   └── specs/<capability>/spec.md
│   ├── conflicts/<conflict-id>/
│   │   ├── record.json
│   │   └── current/specs/<capability>/spec.md
│   ├── quarantine/<recovery-id>/...
│   ├── current-validation.json
│   └── commit.json
├── repairs/<recovery-id>/
│   ├── record.json
│   ├── review.diff
│   ├── candidate/specs/<capability>/spec.md
│   └── validations/<validation-id>/...
├── completed/<commit-token>/
│   └── <retained plan and immutable review evidence>
├── receipts/<change-key>/<commit-token>.json
└── tmp/
```

After `commit.json` is durable and before the first formal-spec replacement, finalize also publishes a reserved source-local recovery capsule under the active change. The capsule contains an ignore-all `.gitignore`, the validated commit record, exact immutable reviewed specs, persisted bases or absence records, payload manifest, and identity hashes needed to reconstruct the active plan. The capsule path and every file in it use explicit generated constants, are hidden from normal source-control discovery by that local ignore file, and are excluded from the reviewed user-payload manifest. A complete capsule is published from a temporary sibling and is carried by the directory rename into the destination.

`change-key` is a deterministic safe key derived from the selected root identity and validated change name. Every manifest also has a random immutable `planId`, so a retained receipt and a later active change with the same name are different archive instances. Caller input never becomes an internal path without validation and containment checks.

Existing included bases are stored byte-for-byte under `bases/`; prepared absence remains an explicit typed manifest entry. Candidates never double as base storage because agent reconciliation is allowed to replace their contents. Validation, review generation, conflict diagnosis, and recovery always refer to the immutable base file or absence record.

The plan has no competing head or successor-generation chain. State is derived from recognized files rather than a second mutable state machine; approved post-commit conflict resolutions are append-only amendments to one commit lineage:

- no plan directory: `none`;
- manifest and candidates only: `prepared`;
- current validation without commit: `validated`;
- commit without completion: `committing`;
- commit with an unexpected target: `conflicted`;
- matching completion receipt: `completed`;
- recognized but malformed, incomplete, or internally inconsistent plan state: `broken`;
- commit-bound source, destination, or marker evidence that cannot be matched safely: `orphaned`.

Prepare builds a complete temporary plan and publishes it with an atomic rename. Repeating prepare returns the existing plan status. Normal validation builds a complete temporary snapshot and atomically replaces `current-validation.json`. Once `commit.json` exists, the original validation is fixed; a conflict resolution can add only a recovery-scoped immutable validation and append-only commit amendment bound to that original lineage.

After a completion receipt is durable, the completed plan is rotated from the reusable active slot to `completed/<commit-token>/` and indexed by an explicit receipt path. A crash before or during rotation is recoverable from the matching plan ID, commit token, and receipt. A later active change with the same name can publish a new plan without being mistaken for the retained receipt. Completed plans and receipts are retained for 30 days from the receipt timestamp; cleanup removes both only through their validated explicit ownership records.

`broken` and `orphaned` are fail-closed states. Status reports the damaged records and safe read-only evidence without creating or updating any file. Its recovery ID is the SHA-256 hash of a versioned canonical evidence document built in memory. Normal abort, finalize, and cleanup preserve ambiguous evidence; persistent recovery state begins only when an explicit evidence-bound repair action is invoked.

A valid source-local capsule can repair a missing or damaged primary active-plan record only when its plan, validation, commit, source, destination, payload, and target evidence form one consistent lineage. Reconstruction builds a complete temporary active plan and publishes it atomically; it never combines records from competing lineages. If both primary and capsule evidence exist but disagree, status reports `orphaned` instead of selecting one. Loss of every evidence copy remains unrecoverable by design.

These atomic publications protect against a process stopping during a file update. The archive commit lock serializes cooperating archive processes during finalization, but it does not serialize standalone agent-driven sync or manual formal-spec edits.

Alternative considered: successor records plus a mutable head revision. Serialized archive commits do not need competing heads or automatic successor records, so the state can be derived from the durable files that already define recovery.

### D3: Prepare owns discovery, selection, and candidate creation

Prepare performs the deterministic front half:

1. Resolve the selected root and canonical active change path.
2. Gather artifact and task readiness through shared helpers.
3. Obtain concrete delta specs from `artifactPaths.specs.existingOutputPaths`.
4. Validate and apply the all/include/exclude/none selection.
5. Record the complete discovered capability set and every delta hash, persist every included main-spec base byte-for-byte or record typed absence, and record path authority, readiness warnings, prompt-source hashes, and any explicit archive name.
6. Create one candidate for every included capability by copying the persisted base bytes or using the shared Purpose-aware new-spec skeleton.
7. Publish the complete plan and return candidate-only `agentWork` plus structured validate, status, and abort actions.

Prepare does not write formal specs, move the change, resolve the archive date, or freeze unrelated files in the change directory. It does perform a best-effort same-filesystem check against the nearest existing archive-root parent and reports an observed default or explicit destination collision before candidate work; finalize remains authoritative because the date, mount topology, and destination may change later. Validation and finalization re-discover the complete delta capability set, so a newly added, removed, or renamed delta invalidates the prepared partition even when it would have been excluded. Selected delta files and other recorded authority inputs must remain unchanged.

Validation builds a complete `lstat` payload manifest for every file, empty directory, mode, and symlink target under the active change, excluding only explicit archive-owned recovery paths. The review presents the complete formal-spec diff separately from this exact payload path/hash manifest so the user can tell which bytes are being written to formal specs and which existing bytes are merely being moved. Artifact and task readiness is recalculated at validation and immediately before commit. Any readiness or payload-manifest difference stops finalize before formal writes and returns the validate action so the changed archive scope can be reviewed. After commit starts, payload drift becomes `archive_source_conflict`; the receipt carries the readiness and payload-manifest hash accepted at commit start.

Includes select exactly the named capabilities. Excludes select every discovered capability except the named capabilities. When both are supplied, the caller must provide a disjoint expected partition. Unknown, duplicate, overlapping, or drifting selections fail without formal mutation. `--skip-specs` selects none.

### D4: Agent work is candidate-only

Prepare returns the complete semantic work package:

- selected delta paths;
- persisted main-spec base paths or typed absence;
- candidate paths;
- informational future target paths;
- project context, archive guidance, specs rules, and operation summaries;
- explicit read and write scopes;
- the structured validate action.

The archive skill passes this package directly into candidate-mode sync. A `prepare-spec-conflict-resolution` repair action returns the same bounded shape extended with the persisted original base, original reviewed snapshot, captured current target, and a recovery candidate initialized from current content. The agent does not rediscover deltas, inspect plan internals, write formal specs, move the change, repair markers, or broaden the returned scope.

Normal prepared-candidate reconciliation preserves every existing requirement and scenario not explicitly removed by a supported delta operation. Whole-requirement `REMOVED` remains its only removal authority. Conflict recovery starts from captured current content and preserves unrelated current semantics by default; any intentional current-to-recovery removal is authority only after it is exposed in the immutable amendment review and explicitly approved.

Standalone `/opsx:sync` without archive-supplied candidate work keeps its existing behavior. Users do not run it while staged finalization is active.

### D5: Validate freezes one complete reviewed snapshot

Validate rechecks the selected root, change identity, complete discovered capability set, recorded delta hashes, persisted included base hashes or absence, prompt-source hashes, current readiness, archive payload manifest, and candidate/target path containment. It reads each candidate and payload file once into the byte buffers used for parsing or manifest hashing, immutable storage, and review generation, so those outputs cannot describe different reads of a changing file. It verifies:

- canonical main-spec structure and parsing;
- absence of delta-operation headings in candidates;
- required ADDED, MODIFIED, REMOVED, and RENAMED outcomes;
- preservation of unaffected requirements and every base scenario not covered by whole-requirement removal.

Semantic outcome validation uses one shared parsed requirement/scenario model:

- requirement and scenario identities use the same normalization and duplicate/multiplicity rules as direct archive parsing;
- an existing spec keeps its Purpose, unaffected requirements, and unmentioned scenarios; unrelated semantic additions, removals, and renames fail validation even when the resulting Markdown parses;
- ADDED content must be present without creating a conflicting duplicate;
- MODIFIED descriptions and named scenarios must reflect the delta while every base scenario omitted by the delta remains present;
- RENAMED removes the old identity and establishes the new identity before any MODIFIED outcome is checked against the new name;
- whole-requirement REMOVED is the only removal authority; scenario omission is not deletion authority;
- formatting-only changes are visible in the complete review but do not authorize semantic drift.

Validation writes the complete deterministic spec diff and payload manifest review to `review.diff`, calculates its byte length, hash, payload-manifest hash, and per-file statistics, and stores the exact candidate bytes under a new lowercase UUID validation directory. It creates an opaque approval token bound to that validation, the exact review delivery metadata, selection, readiness, and payload manifest. The token deliberately does not bind the final archive date, name, or path; a date rollover alone does not invalidate the reviewed content. Finalize reads only from this immutable validation snapshot and requires the bound token; the token is identity binding, while `--yes` or the interactive prompt remains the user's confirmation of execution under the current destination policy.

Each successful validation atomically advances `current-validation.json`. A new validation supersedes the previous finalize action. Validation is rejected after `commit.json` exists.

Editing a candidate after successful validation leaves the immutable snapshot and review intact but makes that validation ineligible for finalize. Status returns a candidate-changed diagnostic and the structured validate action; a new successful validation receives a new identifier and supersedes the old one.

`INLINE_ARCHIVE_REVIEW_MAX_BYTES` is a documented constant, initially 256 KiB:

- reviews at or below the limit are returned completely inline and by file path;
- larger reviews return `delivery: "file"`, the complete path, length, hash, statistics, and a structured read action;
- failure to generate, persist, or hash the complete review publishes no validation identifier.

For file delivery, the archive skill uses the structured read action to inspect and present the complete review when tool limits allow. Otherwise it gives the user the durable path, hash, byte length, payload-manifest hash, and statistics and requires explicit acknowledgement that the complete file was reviewed before executing the approval-token-bound finalize action. A summary or model-generated description is never treated as approval of unseen bytes.

Move-only plans produce an immutable validation record with an empty spec diff plus the complete payload manifest, current readiness warnings, selection, and approval token. The confirmation explicitly lists every discovered delta capability that will remain unsynced.

### D6: Finalize uses a short archive commit lock and remains forward-only and idempotent

Direct archive, staged finalize, and each sequential bulk item share one planning-root archive commit lock at `.archive-plan/archive-commit.lock`. Validation identity and obviously stale inputs may be checked before acquisition, but the authoritative source, target, readiness, payload, destination, and containment preflight runs only after the lock is held. The lock remains held through commit publication, formal-spec replacement, movement, durable receipt publication, and the current invocation's matching generated cleanup.

The lock is published exclusively and records a schema version, root identity, plan and validation identity, a random invocation nonce, process ID, host identity, and acquisition time. A live invocation releases only the lock whose nonce it owns. The lock is not reentrant, and no caller may steal it merely because a fixed amount of time elapsed. Contention returns `archive_commit_busy` with the observed owner information and a status action; it does not begin or resume formal mutation.

A process crash can leave the lock record behind. Status reports that record as read-only evidence. When the recorded host is the current host and the recorded process can be proven absent, status may return a `reclaim-archive-lock` repair decision. Unknown host identity, an apparently live process, permission-denied liveness checks, malformed ownership, or changing lock evidence fails closed with `preserve-and-stop`. Explicit repair creates the first persistent recovery review, rechecks the evidence-derived recovery ID, and removes only the exact abandoned lock after approval. Cleanup never age-steals an archive commit lock.

The lock covers one bounded finalize invocation. A retryable movement failure or another non-completed return releases the live owned lock; a later finalize reacquires it and reruns the authoritative checks before resuming. This prevents two archive commits from passing final preflight together without turning candidate editing or user review into a long-lived lease.

Standalone `/opsx:sync` and manual formal-spec edits do not participate in this archive-only lock. They remain unsupported while finalization is active, and per-target rechecks plus final hash verification detect observable drift without claiming to serialize every external writer. A future all-writer gate requires standalone sync to adopt a candidate and short CLI-owned commit phase; manual edits remain outside cooperative locking.

After acquiring the archive commit lock and before creating `commit.json`, finalize:

1. Requires the requested identifier to equal the current validation and the opaque approval token to match that validation's complete review metadata.
2. Rechecks root/change identity, the complete discovered capability set, recorded delta hashes, persisted base hashes or absence, current candidate hashes, prompt sources that remain authority, payload manifest, and path containment.
3. Recalculates artifact/task readiness and rejects a difference from the current validation before formal writes.
4. Classifies every target as prepared base/absence, reviewed candidate, or unexpected content.
5. Resolves the explicit archive name or the local-date default while preserving an existing `YYYY-MM-DD-` prefix.
6. Verifies the final archive destination does not exist and the active source plus nearest existing archive-root parent are observably on the same device.

Any unexpected target before commit leaves the plan abortable and requires abort/re-prepare. A destination collision before commit can also be handled by aborting and preparing with an explicit archive name. A conflict-free preflight atomically creates `commit.json`, binding the validation ID, approval token identity, reviewed hashes, payload-manifest hash, local archive date, archive name, archive path, lock nonce, and a random commit token. The destination binding is recorded execution state rather than part of the validation approval. Finalize then publishes the matching source-local recovery capsule before the first formal-spec write; if capsule publication fails, no formal target is changed.

For each included target in deterministic normalized order:

- base hash or prepared absence means pending: atomically replace it with the immutable reviewed bytes;
- reviewed candidate hash means already applied: skip it;
- any other value means conflicted: preserve the formal target, atomically snapshot its exact current bytes and hashes under a generated `conflicts/<conflict-id>/` record, stop, and return a read-scoped `agentRecovery` evidence package for the persisted base, reviewed snapshot, current evidence, delta source, and relevant plan metadata.

The archive skill, rather than the user, reads the conflict package, compares base/reviewed/current content, identifies the likely source and semantic effect of newer work, and presents a concrete recovery direction. Status returns a read-only evidence-derived recovery ID and the `prepare-spec-conflict-resolution` repair decision. Explicit repair rechecks that evidence, persists the first recovery review and plan-owned recovery candidate initialized from the captured current target, and returns only the original base, original review, captured current target, selected delta, recovery candidate, rules, scopes, and structured recovery validate action. Candidate reconciliation may restore the original review or preserve and merge newer work, but it writes only the recovery candidate.

Recovery-mode validate requires the captured current target to remain unchanged, verifies canonical structure and the original delta outcomes, and produces a complete amendment review from captured current bytes to the recovery candidate. Because the user is explicitly resolving an exceptional conflict, unrelated differences are permitted only when they are visible in that amendment review; they are never inferred as authorized by the original delta. A new validation and approval token bind the original validation, conflict evidence, current target hash, amendment bytes, and payload manifest. Recovery-mode finalize requires both the recovery ID and new approval token, appends an immutable commit amendment, atomically applies the resolved target, and resumes the original commit. The receipt records the original review and every approved amendment.

Neither the skill nor the CLI silently overwrites the current target, invokes standalone sync during the commit, claims an automatic rebase, or applies a recovery candidate without a new immutable review and user approval.

Every retry before movement also rechecks the complete discovered delta set, recorded source-authority hashes, and reviewed payload manifest. Drift after `commit.json` exists reports `archive_source_conflict`, snapshots the safe current evidence, preserves already applied specs and the active change, and returns an evidence-bound repair investigation package. It cannot return to abort/re-prepare because forward-only commit has already begun, and it never moves a change whose current payload no longer matches the approved archive scope.

Atomic replacement uses an exclusively created generated temporary sibling in the target directory, writes the exact reviewed bytes, syncs the file, uses the platform-appropriate replace primitive, syncs the parent directory where supported, and verifies the resulting target hash. The guarantee covers process interruption and retry; it is not described as a multi-file or hardware power-loss transaction where a platform cannot provide that durability. New target directories are created and containment-checked before the sibling is opened. Windows transient sharing failures use bounded retries and leave the reviewed snapshot unchanged; exhausted retries preserve the owned temporary for recognized cleanup and return a retryable diagnostic. Existing file mode is preserved where meaningful, while new files use the documented default mode.

After all targets match, finalize re-verifies every target, source authority, payload entry, and the bound destination, then moves the active change. If the destination appeared after `commit.json` was created, finalize reports `archive_destination_conflict`, preserves the source and applied specs, keeps the original commit-bound path, and returns agent-investigation guidance. Normal finalize never chooses a new date or overwrites the destination; only an explicitly approved repair may append a new destination binding.

Immediately before movement finalize exclusively creates and syncs a reserved commit-token marker inside the active change. The marker binds the plan, commit token, source, destination, validation lineage, payload manifest, and recovery-capsule hash. A pre-existing or mismatched marker is `orphaned` recovery evidence, not overwrite authority. A successful directory move carries the marker and capsule into the archive destination. Finalize then writes the authoritative receipt, rotates the completed plan out of the reusable active slot, and removes the matching marker and capsule from the archived change. Failure to remove matching recovery residue after receipt durability is cleanup work, not archive failure.

This ordering handles a stopped process:

- before a target write: the base hash remains and retry writes it;
- after a target write: the candidate hash is recognized and retry skips it;
- after the marker is written but before movement: retry recognizes the matching marker in the active source and continues the same commit;
- after the directory move but before the receipt: source is absent and the destination carries the matching commit token, so retry writes the receipt;
- after the receipt but before marker removal, plan rotation, or response: retry returns the same receipt and safely completes only matching generated cleanup.

There is no rollback. Once `commit.json` exists, abort is unavailable. A conflicting target is never overwritten by normal finalize. The archive commit lock makes archive-versus-archive conflicts exceptional, while unsupported standalone sync or manual edits can still produce them. The archive agent investigates the preserved evidence, explains whether restoring the reviewed bytes or preserving newer work is safer, starts the selected recovery only through explicit repair, and asks the user to approve its exact amendment review. The user is not expected to compare hashes or edit formal files manually. Silent overwrite and automatic rebase remain outside this change.

### D7: Status, abort, and cleanup expose recovery without hidden mutation

Status derives and reports `none`, `prepared`, `validated`, `committing`, `conflicted`, `broken`, `orphaned`, or `completed`. It includes plan identity, current validation identity, approval binding, review delivery metadata, payload-manifest hash, included/excluded capabilities, applied and pending targets, active/archive location, archive-lock evidence, conflict or damaged-state evidence, diagnostics, and structured legal next actions. Status is completely read-only: it does not create a directory, snapshot evidence, advance or repair a pointer, update a timestamp, rotate a plan, or clean generated state.

For recovery-capable states, status constructs a versioned canonical evidence document in memory. It contains the selected root and change identities; normalized root-relative paths; typed absence, file, directory, and symlink entries; stable schema/parse results and content hashes for the relevant plan, validation, target, source, destination, marker, capsule, receipt, and archive-lock evidence; and deterministic target classifications. Entries use forward-slash logical paths and stable lexical ordering. Volatile scan times, mtimes, absolute display paths, and diagnostic prose are excluded. The recovery ID is `recovery-v1-<sha256>`, where the lowercase hexadecimal digest is calculated from the UTF-8 bytes of canonical JSON. If evidence changes during the read or cannot be read consistently, status returns `archive_evidence_unstable` without a recovery ID or repair action.

Abort removes a prepared or validated plan that has no `commit.json`. It leaves formal specs and the active change untouched. Once commit exists, abort reports resume or agent-investigation guidance instead of deleting evidence. A completed plan returns its receipt.

Root-level cleanup removes only explicitly recognized generated temporaries, superseded validation snapshots, residual matching markers or capsules from completed archives, quarantined identical source copies whose recovery receipt has passed the 30-day retention boundary, and completed plan/receipt pairs older than 30 days from their validated receipt timestamp. It never removes prepared, validated, committing, conflicted, broken, or orphaned plans, never age-steals `archive-commit.lock`, and refuses cleanup that could overlap an active or ambiguous archive commit. Cleanup reports the plan ID, ownership record, age, and reason for every deletion.

The plan directory and source-local capsule are generated recovery state. Documentation warns that force-cleaning all generated copies during a commit destroys automatic recovery. The duplicate capsule tolerates loss of either the primary plan or capsule, not deliberate or accidental deletion of both. The product describes the guarantee as resumable while at least one complete consistent recovery lineage is retained, not as a database transaction.

### D8: Persistent conflict and damaged-state recovery begins only through evidence-bound repair

Status performs read-only discovery of primary records, exact reserved capsule names in the active source, bounded archive destinations that contain the reserved capsule or marker, formal-target conflicts, and archive-lock ownership. It returns the evidence-derived recovery ID plus only the repair decisions whose preconditions are currently provable. It does not persist a recovery package. The caller starts one decision, including any explicit new archive name, through `repair`; repair first recomputes the canonical evidence and requires the same recovery ID. `prepare-spec-conflict-resolution` then persists its repair record and candidate and returns recovery validation directly because it changes only generated state. Every other decision persists a complete repair review and returns an approval-token-bound execution action before it may change formal, source, destination, lock, or primary-plan state. If evidence changes, every preview and previously returned repair action becomes stale.

`repair` accepts only a structured action returned by status and never accepts an arbitrary source, destination, marker path, or delete target:

- `prepare-spec-conflict-resolution` is legal when one captured formal target still matches the conflict evidence. It persists the base/reviewed/current-bound repair review, creates one recovery candidate initialized from the captured current content, and returns the recovery validate action without changing the formal target.
- `reconstruct-plan` is legal when one complete capsule lineage matches the current source or destination and formal targets. It atomically republishes the primary plan and returns status without changing formal specs or movement state.
- `reclaim-archive-lock` is legal only when the exact lock record is stable, belongs to the selected root, and its same-host owner process is provably absent. It removes only that nonce-bound abandoned lock after approval; unknown or apparently live ownership remains preserved.
- `resume-source` is legal when the active source and its payload match, the bound destination is absent, and the only defect is a missing or foreign marker. Repair preserves any foreign marker as evidence, publishes the commit-bound marker, and returns the normal finalize resume action.
- `adopt-destination` is legal when the active source is absent and the bound destination exactly matches the reviewed payload, capsule, commit, and target evidence. Repair writes the authoritative receipt and performs only matching generated cleanup.
- `quarantine-source-and-adopt-destination` is legal when both locations contain the same reviewed payload lineage. Repair atomically moves the duplicate active source into the plan-owned quarantine, writes the receipt for the destination, and retains the quarantine until its receipt-bound cleanup boundary; it does not delete the duplicate during repair.
- `rebind-destination` is legal when the active source exactly matches the reviewed lineage but the bound destination is occupied by foreign content. It requires an explicit new validated archive basename whose destination is absent, appends the same immutable destination amendment containing the old and new bindings to the primary plan and source-local capsule, and returns the normal finalize resume action. It never moves or overwrites the foreign destination.

If no action's preconditions are provable, status offers only `preserve-and-stop` plus a complete read-only recovery report in its response. Repair never combines disagreeing records, adopts a merely similar tree, recursively deletes evidence, age-steals a lock, or treats user approval as a substitute for missing bytes. Source, destination, marker, capsule, plan, validation, target, payload, or lock changes after investigation change the canonical recovery ID and require a new status investigation.

### D9: Staged movement is rename-only

Staged finalize renames the active change within the selected planning home:

- same-filesystem rename completes the archive;
- Windows sharing violations and similar temporary handle failures return a retryable diagnostic while specs remain applied and the change remains active;
- `EXDEV` returns an explicit unsupported diagnostic and does not copy or remove the source.

Prepare and finalize compare the source device with the nearest existing destination parent when the platform exposes stable device identity. A known mismatch fails before commit; the rename still handles `EXDEV` because mount topology can change or evade preflight. This keeps staged publication simple and prevents a partially copied archive from appearing at the final path.

### D10: Direct archive uses the same commit primitives behind its existing interface

Without `--stage`, the command preserves its positional argument, prompts, flags, JSON result fields, deterministic delta builder, and supported movement behavior. Internally it:

1. creates a direct plan and deterministic candidates;
2. snapshots the accepted candidate bytes and archive payload manifest;
3. acquires the same planning-root archive commit lock and invokes the shared target classification, recovery capsule, atomic spec replacement, commit marker, movement, repair, and receipt logic;
4. maps the result back to the existing direct output contract.

Direct `--no-validate` may skip its existing optional validators, but it still performs path containment, exact candidate snapshotting, target classification, scenario-preservation, and commit checks.

A direct invocation encountering an active staged plan reports that staged status instead of creating a competing plan. A stale uncommitted direct plan is rebuilt on a later direct invocation.

The direct movement policy retains cross-device and Windows compatibility through a generated temporary directory under the destination archive root. Immediately before copy it builds a complete `lstat` tree manifest containing relative paths, regular-file bytes and hashes, empty directories, file modes where meaningful, and symbolic-link target text without following links outside the source. Copy reproduces only those recorded entries, verifies the temporary tree, and rechecks that the source still matches the manifest before atomically publishing the final name on the destination filesystem.

Source removal occurs only after publication and another source-manifest match. A stopped copy leaves no partial final archive. If publication succeeded but source removal did not, retry recognizes the matching commit marker and manifest: it removes an unchanged source, but preserves both locations and returns agent-led investigation when the source changed. Platforms that cannot safely recreate a recorded symbolic link fail before publication and leave the source untouched rather than copying the link target as a regular file.

### D11: Bulk archive is sequential

Bulk determines a stable change order, explicit included/excluded capability partition, and optional explicit archive name for every selected change. It then runs the complete lifecycle one item at a time:

```text
prepare A → reconcile A → validate A → confirm exact approval A → finalize A
prepare B from A's completed state → reconcile B → validate B → confirm exact approval B → finalize B
```

Prepare returns the complete discovered partition, and bulk compares it with the inspected partition before candidate work. Drift aborts that uncommitted item. Completed earlier changes remain complete when a later item fails or the user cancels. Later items are reported as pending or skipped; bulk never runs two formal commits concurrently, and each item's finalize acquires and releases the shared archive commit lock independently.

Mixed-schema and move-only changes use the same review and confirmation contract with no candidates.

### D12: Paths and identifiers are defensive and cross-platform

All paths use `path.join()` and `path.resolve()`. Existing paths use canonical `realpath`; new targets use the nearest existing canonical parent. Containment uses `path.relative()` and rejects traversal, root escape, malformed identifiers, absolute stored entries, and symlink redirection.

Capability IDs retain their discovered forward-slash logical form for ordering and records, but selection and target authority reject duplicate normalized IDs and any two capabilities that resolve to the same canonical or platform-equivalent target. Rechecks occur immediately before reads, temporary creation, replacement, and movement. The archive commit lock coordinates cooperating archive processes; these checks remain defensive path validation and do not claim to defeat standalone sync, manual edits, or a hostile process that swaps filesystem objects between every check and use.

Windows checks also reject alternate drives, device names, alternate data streams, trailing-dot/space aliases, and case-insensitive aliases of protected paths. Validation identifiers use one lowercase UUID form. Recovery identifiers use the fixed `recovery-v1-<64 lowercase hex characters>` canonical-evidence form. Both are checked before lookup or path construction. Explicit archive names are validated as basenames before any destination path is resolved.

Generated temporary files, validation directories, direct-copy directories, and commit-token markers are tracked by explicit constants and ownership records. Cleanup never discovers deletion authority from an unrestricted glob.

### D13: Generated skills detect staged support narrowly

Archive skills probe staged status before work begins:

- a versioned staged response or recognized staged diagnostic selects the staged lifecycle;
- an exact unsupported-option result selects the retained legacy workflow and announces its reduced review/resume guarantees;
- any real failure from a CLI that recognizes staged archive remains authoritative and never falls back to agent-owned formal writes or movement.

### D14: Separate the state engine from command presentation

The implementation is split into narrow modules for plan/capsule storage and schema validation, archive commit locking, canonical evidence construction, pure state derivation, candidate and payload validation, target classification, atomic replacement, movement and tree manifests, repair execution, receipts/cleanup, and human/JSON presentation. The pure state engine accepts validated evidence and returns one derived state plus legal structured actions; it does not read or mutate the filesystem itself.

Direct archive, staged archive, and bulk orchestration call the same storage, classification, replacement, movement, and recovery primitives. Existing direct output mapping stays at the adapter boundary. Failpoint tests target every durable publication boundary independently before the direct adapter is switched over, reducing the chance that staged implementation changes established direct behavior accidentally.

## Risks / Trade-offs

- **A process stops after some reviewed spec writes** → Target hashes identify pending and already-applied files, so finalize resumes forward.
- **A formal spec changes unexpectedly during commit** → Preserve and snapshot it, report `conflicted`, prepare a recovery candidate from exact base/reviewed/current evidence, and require a new amendment review and approval before resuming.
- **Two archive commits overlap** → The planning-root archive commit lock admits only one direct, staged, or bulk archive finalizer before authoritative preflight.
- **Standalone sync or a manual edit overlaps archive finalize** → Those writers remain outside the archive-only lock. Hash checks report observable stale or conflicted content, but an all-writer guarantee requires standalone sync to gain a candidate-based short CLI commit phase.
- **A finalizer crashes while holding the archive lock** → Preserve the lock as evidence, never steal it by age, derive its recovery ID through read-only status, and allow explicit `reclaim-archive-lock` repair only when the same-host owner process is provably absent.
- **Windows holds the change directory open** → Keep reviewed specs applied and return a retryable rename diagnostic.
- **Staged movement crosses filesystems** → Detect an observable device mismatch before commit and still reject `EXDEV` explicitly; direct archive retains verified temporary-copy compatibility.
- **The bound destination appears after commit starts** → Preserve the active change and applied specs; normal finalize keeps the original binding, while an evidence-bound repair may rebind only to a user-supplied empty basename and records both paths.
- **The local date changes after validation** → Finalize binds the then-current destination without invalidating content approval; the token intentionally does not bind location, while the receipt records the actual archive path.
- **A plan record or commit marker is damaged** → Report `broken` or `orphaned`, reconstruct only from one consistent capsule lineage, or expose evidence-bound repair actions whose preconditions and approval token are rechecked immediately before mutation.
- **A completed receipt collides with a later same-name change** → Separate immutable plan identity and retained receipt history from the reusable active name slot, warn about the default destination during prepare, and allow an explicit validated archive basename.
- **Direct copy sees source drift or unsupported links** → Refuse publication or source removal, preserve both sides when necessary, and return the recorded tree evidence to the agent.
- **The success response is missed** → Return the retained receipt or recover it from the commit-token marker carried into the archive.
- **A complete review is too large for inline output** → Persist it and return a hash-bound file action.
- **A user approves only a summary of a file-delivered review** → Require acknowledgement of the complete durable review and use an opaque approval token bound to its validation ID, path, hash, length, delivery mode, and payload manifest.
- **A non-spec payload file changes after validation** → Invalidate the review before commit or report source conflict after commit; never present a spec-only diff as approval of an unbounded directory move.
- **The primary plan is removed during commit** → Reconstruct it only from the complete matching source-local capsule. If every generated copy is force-cleaned, preserve observable formal state and report that exact automatic recovery is no longer possible.
- **Evidence changes while status is reading it** → Return `archive_evidence_unstable` without a recovery ID or persistent recovery state; a later status invocation starts from a fresh read-only observation.
- **The state and compatibility surface becomes too complex** → Keep state derivation pure, isolate durable primitives from presentation, switch the direct adapter last, and exercise every publication boundary with failpoints and legacy-output fixtures.
- **Serial bulk is slower than parallel candidate preparation** → Prefer predictable bases, reviews, and failure reporting over throughput for a low-frequency command.
- **Structural validation misses a semantic mistake** → Preserve the complete diff and require explicit user confirmation.
