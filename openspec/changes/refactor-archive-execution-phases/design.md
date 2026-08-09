## Context

Archive execution is currently split across two independently evolving paths:

- `openspec archive` discovers delta specs, builds deterministic replacements, validates them, snapshots target specs, claims the archive destination, applies spec mutations, moves the change, and rolls back when a pre-commit failure occurs.
- The single and bulk archive skills inspect status and tasks themselves, invoke an agent-driven semantic sync that writes main specs directly, and move change directories themselves.

The runtime-input work added current project context, `operations.archive.guidance`, and artifact rules to the skill path without changing this ownership split. The follow-up must preserve the CLI's accumulated path, concurrency, retirement, rollback, store-root, and cross-device guarantees while allowing an agent to own only the semantic transformation of a delta and baseline into a candidate main spec.

The current agent-driven sync path reads a main spec and later edits it directly, without binding the write to the version the agent read. An editor, another agent, Git operation, or other process can therefore change the main spec in between. The phased path closes that window with prepared baseline fingerprints, validation against live targets, token binding, and another comparison immediately before each CLI-owned mutation; it does not try to hold a filesystem lock while an agent or user thinks.

The design must also tolerate generated skills and CLIs being upgraded independently, work on Windows, macOS, and Linux, and avoid adding durable project state for an archive attempt that has not been finalized.

## Goals / Non-Goals

**Goals:**

- Establish one versioned archive execution contract shared by CLI and skill paths.
- Keep semantic spec reconciliation agent-driven for the phased skill path.
- Keep enforceable readiness, path, validation, concurrency, retirement, mutation, move, and recovery behavior in the CLI.
- Leave the selected project or store unchanged until the user approves finalization.
- Preserve single-change rollback across candidate promotion and the archive move for failures returned while the finalizing process remains running.
- Preserve existing `openspec archive` syntax, flags, JSON output, diagnostic codes, and automation behavior.
- Define honest batch and mixed-schema semantics before migrating bulk archive.
- Make contract-version mismatches and stale work fail with actionable recovery guidance.

**Non-Goals:**

- Replace semantic judgment with exact deterministic Markdown output comparison.
- Add configurable enforceable operation checks.
- Provide all-or-nothing atomicity across an entire bulk archive batch.
- Change the standalone `/opsx:sync` user experience.
- Persist archive attempts as version-controlled OpenSpec artifacts.
- Add a persistent transaction journal or guarantee cross-file atomicity when a process is terminated abruptly.
- Deprecate the existing one-shot archive command in this change.

## Decisions

### D1: Add a separate machine-oriented archive-attempt namespace

The phased contract is exposed through additive commands rather than overloading positional values accepted by `openspec archive [change-name]`:

```text
openspec archive-attempt prepare --change <name> [--new-attempt --yes] --json
openspec archive-attempt validate --plan <plan-path> --json
openspec archive-attempt finalize --plan <plan-path> --validation-token <token> --yes --json
openspec archive-attempt discard --plan <plan-path> --json
openspec archive-attempt inspect --plan <plan-path> --json
openspec archive-attempt unlock --plan <plan-path> --yes --json
openspec archive-attempt cleanup --attempt <attempt-dir> --cleanup-token <token> --yes --json
openspec archive-attempt prepare-batch --change <name>... [--new-attempt --yes] --json
openspec archive-attempt validate-batch --plan <batch-plan-path> --json
openspec archive-attempt finalize-batch --plan <batch-plan-path> --validation-token <token> --yes --json
```

`prepare` and `prepare-batch` accept the normal selected-root options. The resulting plan records the resolved root identity; later commands re-resolve and verify that identity rather than accepting a different root. Phase commands are non-interactive and emit JSON only. Generated skills own explanation and prompting, while `--yes` on finalize, batch finalize, stale-lock removal, explicit attempt cleanup, and `--new-attempt` is the authorization required for their machine-driven mutation. `--new-attempt` without `--yes` is rejected.

This avoids reserving change names such as `prepare`, `validate`, or `finalize` under the existing archive command. A nested `openspec archive prepare` surface was rejected because it would reinterpret valid positional change names and complicate the established `--` handling for unusual names. Separate flat commands were rejected because three or four top-level archive verbs would be harder to version and discover as one protocol.

### D2: Use a versioned, CLI-managed temporary attempt workspace

After overlap checks pass, prepare creates a private attempt workspace with `fs.mkdtemp(path.join(os.tmpdir(), ARCHIVE_ATTEMPT_PREFIX))`. The generated prefix and file names are tracked in constants. The workspace contains:

```text
plan.json
result.json
inputs/
  deltas/<entry-id>.md
  baselines/<entry-id>.md
outputs/
  <entry-id>/spec.md
```

Entry IDs are generated identifiers looked up explicitly from the plan; capability names are never concatenated into attempt-workspace paths. Input files are immutable snapshots for the agent to read. Output files are candidate main specs. `result.json` starts with every entry in a `pending` state and requires an explicit semantic decision before validation.

Each completed result instead records one semantic decision:

```ts
type SpecDecision = 'sync' | 'retire' | 'already-synced' | 'skip';
type SpecEffect = 'create' | 'update' | 'delete' | 'none';
```

The sync agent selects `sync`, `retire`, or `already-synced`; only an explicit user choice selects `skip`. Validation derives the effect from that decision and the current target state: `sync` creates or updates, `retire` deletes, and `already-synced` or `skip` changes no main-spec file. `already-synced` means the prepared baseline already represents the delta outcome; `skip` means the user explicitly chose not to reconcile specs. A missing candidate never implies retirement, already-synced work, or a skip decision.

The project remains unchanged during prepare, agent work, validate, and discard. A temporary attempt workspace may live on another volume, including on Windows. Finalization therefore reads candidate content and uses target-local atomic writes and the existing move fallback rather than assuming a cross-volume rename from the attempt directory will work.

OS temporary storage was chosen over a hidden directory inside the OpenSpec root so unfinished attempts do not appear in version control, change discovery, store health, or user-authored project content. Finalize and discard remove the attempt workspace. Every single or batch prepare first runs lifecycle cleanup. It may automatically remove only an unlocked attempt when every prepared member is absent from active changes and has a complete archived change matching its prepared tree fingerprint. Stale, partially completed, orphaned, invalid, and locked attempts are returned with their state and available actions so the skill can ask whether to clean, recover, or retain them; an unrelated temporary directory is never offered for deletion.

An invalid attempt may not contain a usable plan, so explicit cleanup accepts an attempt directory plus a short-lived cleanup token returned by the prepare-time scan. Cleanup re-establishes that the path is a direct, non-link child of `os.tmpdir()`, has the exact generated prefix, still has the same filesystem identity and scanned metadata fingerprint, can be exclusively locked, and still matches the token before deleting it. The command cannot be used as a general recursive-delete surface. Removing an attempt never rolls back main-spec writes or archive moves that an earlier process already completed.

Before publishing a workspace, prepare compares the selected change set with retained single and batch attempts in the same canonical root. If an active matching attempt or a partially completed batch includes any selected change, the default response reports every overlap, its state, plan path when trustworthy, and available actions, and creates no new plan or workspace. The client must ask the user to resume one exact attempt, inspect or clean retained state and retry, cancel, or explicitly authorize a distinct attempt with `--new-attempt --yes`. An overlap with a live or unresolved lock must be inspected before new work starts; the new-attempt flag does not steal or bypass a claim.

Explicit new-attempt authorization permits more than one attempt to name the same change, but it never reuses, overwrites, or migrates candidates from an older attempt. The same rule applies across single and batch preparation: a batch cannot silently absorb a single-change candidate, and a single prepare cannot silently detach a change from a retained batch. `--new-attempt --yes` bypasses only the resumable-overlap decision after the overlaps have been reported; every readiness, path, stale-claim, and safety check still applies.

### D3: Make the plan self-describing but never trusted

`ArchivePlanV1` contains:

```ts
interface ArchivePlanV1 {
  contractVersion: 1;
  planId: string;
  root: RootIdentity;
  change: ChangeSnapshot;
  archive: {
    root: string;
    changeName: string;
    namingPolicy: 'finalize-local-date';
  };
  checks: ArchiveCheck[];
  operationInputs?: { context?: string; operationGuidance?: string[] };
  specWork: SpecWorkEntry[];
  resultPath: string;
}
```

Each spec-work entry records the concrete delta path from `artifactPaths.specs.existingOutputPaths`, the canonical main-spec target, attempt input and output paths, whether the target existed, its baseline fingerprint, the delta fingerprint, and the current `specs` artifact-rule snapshot. Context, archive guidance, and artifact rules remain structurally distinct.

Prepare reads one project-config snapshot. The plan identity covers the selected-root identity, change metadata and relevant files, task/artifact facts, config snapshot, delta inputs, baseline targets, archive root and naming policy, and contract version. A plan is an optimistic-concurrency record for one archive attempt, not a per-change singleton or an authorization token. When no resumable overlap exists, prepare creates a new plan ID and snapshots. When an overlap exists, prepare discovers it but does not adopt it: another client resumes only by selecting the returned plan path, and a distinct concurrent attempt is created only after `--new-attempt --yes` explicitly authorizes that choice.

The plan stores the `finalize-local-date` naming policy rather than a fixed dated destination. Prepare and validate may return a clearly marked preview path. Finalize captures the current local date once when the operation starts, derives every destination used by that operation from that date, and then checks and claims those concrete destinations. A change name that already begins with a valid date prefix keeps its existing name. If the derived destination already exists, finalization blocks without overwriting it, choosing an alternate date, or inventing a numeric suffix; recovery guidance asks the user to inspect and resolve the existing destination.

Validate and finalize treat every stored path and field as untrusted, rederive allowed roots and targets through the same internal resolvers, and compare explicit entries rather than relying on filename patterns.

### D4: Return typed checks instead of prompt prose

Prepare and validate report checks in a common shape:

```ts
interface ArchiveCheck {
  id: ArchiveCheckId;
  status: 'pass' | 'warning' | 'blocked';
  message: string;
  confirmationRequired: boolean;
  fix?: string;
}
```

Built-in check IDs are maintained in an explicit constant list. Validation failures, unsafe paths, invalid metadata, stale inputs, and destination collisions are blocked. Incomplete artifacts or tasks retain their established warning-and-confirm behavior. Operation guidance cannot create, suppress, or change a check. This change structures existing checks but does not introduce configurable typed checks.

The skill renders warnings and obtains the user's decision. Before final confirmation it also renders the validated baseline-to-final-state diff, including a full deletion diff for retirement, the spec-level effect (`CREATE`, `UPDATE`, `RETIRE`, `ALREADY SYNCED`, or `SKIPPED`), and requirement-level ADDED, MODIFIED, REMOVED, and RENAMED counts. Finalize requires `--yes` whenever any confirmation is needed and rejects a plan with a blocked check.

### D5: Limit agent ownership to candidate semantics

The archive skill passes the immutable delta and baseline snapshots, candidate output paths, project context, archive guidance, and relevant `specs` rules to the inline sync workflow. In archive-candidate mode the sync workflow:

1. reads only the plan's selected spec entries;
2. performs the semantic merge into the corresponding candidate output, using every contributing delta when a batch entry groups changes by one canonical target;
3. applies artifact rules only to candidate spec content;
4. records `sync`, `retire`, or `already-synced` for every entry it processes;
5. leaves `skip` to an explicit user decision; and
6. produces a semantic summary plus a baseline-to-final-state diff for user review.

It does not write a main spec, delete a capability, move a change, reinterpret a CLI check, or widen the selected entry list. A skipped entry creates no candidate and the workflow does not expose or apply its artifact rules to agent work. Standalone sync continues to resolve and write main specs as it does today.

The agent remains responsible for meaning and clear rewriting. CLI validation enforces reliable structural and safety properties, but it does not compare a candidate byte-for-byte with the deterministic merger or claim that natural-language intent has been proven.

Candidate reconciliation is idempotent against the prepared baseline. If a prior finalizing process was terminated after some main-spec outcomes were already applied, a later prepare snapshots that current state. The agent records `already-synced` for outcomes the new baseline already represents and produces candidates only for the remaining semantic result; it does not require a persistent transaction journal or reconstruct the earlier process's mutation order.

### D6: Validate candidates and issue a concurrency token

Validate reloads the attempt and performs these checks without mutating the project:

- the contract version is supported and the selected root resolves to the same identity;
- the change, config, delta snapshots, baseline targets, artifact/task state, archive root, and naming policy still match prepare;
- `result.json` contains exactly one explicit `sync`, `retire`, `already-synced`, or `skip` decision for every planned entry and no unknown entry;
- attempt inputs, outputs, and manifest are regular files within the canonical attempt-workspace root, with no symlink traversal;
- candidate and manifest fingerprints remain stable across each validation read, so concurrent agent writes fail validation instead of producing a mixed snapshot;
- every `sync` candidate passes main-spec structure and content validation and contains no delta operation sections;
- every `retire` result is authorized by valid change metadata and targets a path inside the real specs root;
- `already-synced` is explicit and matches the prepared baseline, while `skip` records an explicit user choice; neither can conceal a candidate deletion;
- all destination paths are still safe and mutually distinct.

On success, validate returns a `validationToken` computed from a canonical representation of the contract version, plan, result manifest, candidate fingerprints, derived effects, rendered diffs, live-input fingerprints, check results, confirmed semantic-conflict decisions, and user-selected artifact-rule conflict resolutions. The token is an optimistic-concurrency value rather than a secret. Any relevant edit after validation causes finalize's recomputation to differ and requires a new prepare/validate cycle.

### D7: Make finalize the only project-mutating phase

Finalize requires the plan, matching validation token, and explicit `--yes`. It acquires the attempt lock, reruns validation, captures the current local date once, and then:

1. derives the dated archive destination from the captured date, acquires the archive-root transaction lock, and settles every concrete destination;
2. captures snapshots for every candidate target;
3. compares each live target with its prepared baseline again immediately before mutation and promotes validated candidate writes using target-local atomic replacement;
4. performs authorized retirements only after ordinary writes succeed;
5. verifies the active change and delta fingerprints again;
6. moves the change with the existing cross-device and Windows fallback;
7. verifies the archived source where required; and
8. commits or rolls back snapshots using the existing in-process recovery rules; and
9. deletes the consumed attempt on success and releases locks in `finally` without removing a lock it no longer owns.

If the running operation reports a failure before a complete archive is secured, target specs are restored and the change remains or returns to its active path. If the verified fallback archive is complete but cleanup of its staged source fails, the complete archive and committed spec state are retained with recovery diagnostics, matching current behavior.

The phased path does not add a persistent transaction journal. If the process is terminated abruptly, completed candidate writes, retirements, or moves may remain. After the user confirms the owning process is gone and resolves any retained claims, the agent may inspect and retain or clean the old attempt, then prepare again. A new attempt treats current main specs as its baseline, classifies already-applied outcomes explicitly, and reconciles the rest. A complete matching archive consumes the old attempt; simultaneous active and archive copies, content that matches neither an expected current outcome nor a complete archive, or other ambiguous filesystem state requires explicit user handling. Attempt cleanup itself never reverses completed project mutations.

Finalize emits the existing archive result fields plus attempt and candidate summaries. It never executes agent-authored commands; candidates are data validated and applied by OpenSpec.

### D8: Keep one-shot archive as a compatibility adapter

`openspec archive <change>` remains the user-facing one-shot command. Internally it calls the extracted planner and transaction services without spawning CLI subprocesses. Its compatibility merge adapter uses the existing deterministic `findSpecUpdates` and `buildUpdatedSpec` behavior to populate `sync`, `retire`, `already-synced`, or `skip` decisions and their derived effects, then reuses shared validation and finalization.

`--skip-specs`, declining spec updates, `--no-validate`, incomplete-task confirmation, human output, JSON envelopes, diagnostic codes, retirement behavior, store selection, and exit-code behavior remain unchanged. In particular, validation-disabled archive still cannot retire a capability.

The read-only `openspec instructions archive` surface remains available for older generated skills. A new phased skill obtains the same runtime inputs from prepare and does not need a second config read.

### D9: Fail safely across independently upgraded skills and CLIs

Every response includes `contractVersion`. A skill supports an explicit version list and rejects an unsupported version before candidate work. If `archive-attempt` is absent, the skill reports that the installed CLI does not support phased archive and offers an upgrade command or the existing direct `openspec archive <change>` compatibility path. It does not fall back to manually moving a change.

This is stricter than the fail-open archive-input lookup because the phase command owns enforceable checks and final mutations rather than optional prompt context.

### D10: Build aggregate candidates and finalize conflict groups with in-process rollback

`prepare-batch` resolves every selected change into one batch attempt before agent work. Before doing so, it reports every retained single or batch attempt that includes any selected change and creates no new batch workspace until the user resolves the overlaps or explicitly authorizes a distinct attempt. It builds a potential graph whose nodes are changes and canonical main-spec targets, with an edge whenever a change has a planned delta contribution to a target. After explicit contribution skips are recorded, validation rederives the effective graph from included contributions only. Each effective connected component is a conflict group with in-process rollback: changes that share an included target directly or transitively finalize together, while skipped edges and disconnected groups do not create an unnecessary dependency.

Each canonical target has one immutable baseline snapshot, one candidate output, and an ordered list of contributing delta snapshots with their own artifact-rule snapshots. Every non-conflicting rule from every included contribution constrains the aggregate candidate. When rules conflict, the agent pauses that target and reports the conflicting rule text and source changes; the user selects which rule controls that conflict, the result records the selected and suppressed rule sources, and the candidate is regenerated under that choice before validation. Change creation date never resolves an artifact-rule conflict.

The stable delta input order is the change metadata `created` date from oldest to newest, then change name for deterministic ordering. That order guides semantic reconciliation but does not silently discard earlier intent. Compatible contributions are combined. For truly incompatible change intent, a later `created` date supplies the proposed resolution; missing dates, equal dates, and every destructive or ambiguous resolution remain visible in the conflict summary and require explicit user confirmation. Future explicit dependency or superattempt metadata may outrank creation date without changing this contract version.

A batch entry records which change contributions participate and which the user explicitly skipped. The agent produces one final target decision and candidate from the included contributions. An aggregate `retire` decision is authorized only when every included change contribution to that target comes from metadata that validly declares `retire_capabilities: true`; a later creation date or final confirmation does not manufacture missing deletion authorization. Validation requires a concrete user selection for every reported artifact-rule conflict and returns, for each target, the aggregate baseline-to-final-state diff, the derived spec effect, requirement-operation counts, per-change contribution summary, semantic-conflict resolution, and artifact-rule resolution. The batch token binds all of those facts.

`validate-batch` requires every conflict group and destination preflight to pass before the first mutation. After one user confirmation, each `finalize-batch` invocation captures the current local date once and finalizes conflict groups in deterministic order. Immediately before each group and each target mutation, it rechecks the live change, baseline, candidate, contribution, authorization, and destination fingerprints. Within a group, the CLI applies every aggregate candidate and moves every member change as one rollback-capable transaction while the process remains running. If a running group reports a failure before commit, all of that group's spec mutations and change moves are restored. Earlier completed groups remain complete and later groups remain active, so the wider batch is not globally atomic and reports exact group-level partial completion and retry guidance. The retained batch attempt records committed-group receipts; retry verifies those receipts against archived changes and final spec fingerprints, skips verified completed groups, and revalidates only the remaining groups against current state.

Abrupt process termination may leave part of the current group applied because no durable transaction journal exists. After stale claims are explicitly resolved, the user may keep or clean the retained attempt and start a new prepare against current project state. Already-applied aggregate outcomes then form part of the new baseline and semantic reconciliation converges the remaining active changes. The contract does not describe an abruptly interrupted conflict group as atomic.

Mixed-schema changes participate independently. A change with no concrete `artifactPaths.specs.existingOutputPaths` contributes no target edge and proceeds as its own readiness-and-move group without inventing spec work or fetching `specs` rules.

Global all-or-nothing rollback across disconnected groups was rejected because reversing already completed archive moves and fallback-copy outcomes would expand the failure surface substantially. Aggregate candidates plus connected groups with in-process rollback keep shared-target semantics coherent without requiring sequential re-prepare or a virtual baseline per change.

### D11: Use operation-scoped attempt locks and lifecycle cleanup

Every attempt has a sibling lock path created with exclusive `wx` semantics. The lock records the plan ID, process ID, random nonce, operation, and acquisition time. Prepare holds it only while publishing a complete attempt workspace; validate, finalize, discard, explicit cleanup, and lifecycle cleanup hold it for their whole operation. Agent candidate work and user review do not hold a long-lived lock. `inspect` is read-only and reports a best-effort snapshot without claiming the attempt. Validation detects concurrent candidate or manifest writes by fingerprinting reads before and after parsing, and finalization detects later edits through the validation token and live baseline comparisons.

Locks are released in `finally`. The owner closes its handle and removes the lock only after the current path identity and nonce still match the file it created. Finalization always acquires the attempt lock before the archive-root transaction lock and releases them in reverse order. A successful single finalize deletes its consumed attempt before releasing the sibling lock. A batch uses one batch attempt and one attempt lock for its whole validation or finalization operation; a partially completed batch retains that attempt with exact group state for retry.

An exclusive-create failure is treated as an active claim. There is no time-based automatic lock stealing: process IDs and timestamps are diagnostic, not sufficient proof that a lock is stale across every platform. An archive-root claim created by phased finalization also records the plan ID, operation, and its own nonce. `inspect` reports the attempt, project, archive, attempt-lock, and associated archive-root-claim state. `unlock --yes` is an explicit recovery action after the user has established that no owning archive process remains; it removes only selected claims whose repeatedly verified identities, plan ownership, and nonces still match and does not mutate attempt data or project content. Automatic cleanup skips every locked attempt.

Attempt cleanup uses project state rather than TTL and runs at the start of every prepare. Finalize and discard remove their own attempt workspaces. The scan examines only exact generated-prefix directories and automatically removes one only when it can acquire the attempt lock, every prepared member is absent from active changes, and every member has a complete archived change matching the plan's root, name, and prepared tree fingerprint. An active matching attempt is resumable by its plan path; a changed active attempt is stale; a partially completed batch is retained for retry; a missing member without a matching archive is orphaned; malformed or unverifiable state is invalid. Stale, partial, orphaned, invalid, and locked attempts are retained and returned with explicit clean, recover or retry where supported, and keep choices. A resumable attempt that overlaps the requested change set blocks default attempt creation until the user chooses an action. Unrecognized directories are left untouched and are not presented as OpenSpec cleanup candidates.

The attempt lock protects temporary attempt data. The existing archive-root transaction lock separately serializes project mutations and dated destination claims; finalize holds both until commit or rollback completes.

### D12: Extract internal services before exposing the protocol

The current archive module is split behind behavior-preserving interfaces:

- selection and root resolution;
- archive planning and typed checks;
- attempt-workspace storage;
- validation and fingerprinting;
- destination claims;
- target snapshots and spec mutation;
- change movement and recovery;
- human/JSON presentation; and
- the one-shot compatibility merger.

The phase commands and legacy command call these services directly. No implementation path shells out to another OpenSpec command. Existing exported helpers and tests remain stable until callers have migrated.

## Risks / Trade-offs

- **Temporary attempts can be abandoned** -> Every prepare scans generated attempts, automatically removes only unlocked attempts proven consumed by a matching archive, and reports stale, partial, orphaned, invalid, or locked attempts for an explicit clean, recover, retry, or keep decision without age-based deletion.
- **Candidate files may contain sensitive project content** -> Use OS-private temporary directories, minimize copied inputs, avoid logging contents, and remove attempt workspaces promptly.
- **Inputs can change between every phase** -> Fingerprint change, config, delta, baseline, destination, manifest, and candidate state; validate twice and bind finalization to the resulting token.
- **The main spec can change after the agent reads its baseline** -> Agent work stays in the candidate workspace; validation, finalization, and each immediate pre-mutation check compare the live target with the prepared baseline and require a new attempt when they differ.
- **Agent output can be structurally valid but semantically wrong** -> Show the exact validated baseline-to-final-state diff, per-change contribution summary, and every proposed semantic-conflict resolution before confirmation, and never describe CLI validation as proof of natural-language equivalence.
- **Cross-volume staging prevents direct atomic rename** -> Use target-local atomic writes and the existing snapshot/rollback transaction rather than moving candidate files into place.
- **Creation time does not prove semantic causality** -> Use `created` only for deterministic input ordering and a visible proposed resolution; preserve compatible intent and require confirmation for true conflicts instead of silently applying last-created-wins.
- **Batch finalization can partially complete** -> Validate the entire batch first, provide in-process rollback for each connected conflict group, finalize groups deterministically, and return resumable group-level partial-state diagnostics.
- **Abrupt termination can leave partially applied main specs or a partially moved conflict group** -> Do not add a persistent transaction journal or claim crash atomicity; retain inspectable attempt and lock state, require explicit stale-claim handling, and allow a new idempotent prepare to treat current project state as its baseline and converge the remaining result.
- **Artifact rules can conflict across batch contributions** -> Keep all non-conflicting rules, require the user to select the controlling rule for each conflict, regenerate the candidate under that choice, and bind selected and suppressed rule sources into validation.
- **A process can crash while holding an attempt lock** -> Never steal by age, report lock ownership diagnostics, and require explicit inspect/unlock recovery when liveness cannot be proven safely.
- **Multiple attempts for one change can duplicate work or confuse recovery** -> Default prepare reports resumable single and batch overlaps without creating a workspace; only a user-confirmed `--new-attempt --yes` creates another independent attempt, and candidates never migrate implicitly between attempts.
- **New skills can be installed with older CLIs** -> Version the contract, fail with upgrade guidance, and offer the existing one-shot CLI path instead of reproducing manual archive behavior.
- **Refactoring a heavily tested archive path can regress edge cases** -> Land internal extraction first with the existing suite unchanged, then add phase-contract characterization tests before migrating skills.

## Migration Plan

1. Extract planner, checks, claims, snapshot, mutation, move, rollback, and presentation services without changing `openspec archive` behavior.
2. Add archive-attempt types, semantic decisions and effects, operation locks, prepare-time lifecycle and overlap inspection, explicit cleanup, and prepare/discard commands behind additive CLI routing, with contract and path-safety tests.
3. Add candidate validation, exact diff output, tokens, immediate live-baseline checks, finalize-time local-date resolution, and finalize by reusing the extracted in-process transaction; retain the existing command as the only default path.
4. Move the one-shot deterministic merge through the shared candidate/transaction model and prove output and diagnostic parity.
5. Add archive-candidate mode to the sync workflow and migrate the single-change archive skill.
6. Add batch preparation, aggregate candidates, created-date-guided semantic-conflict review, explicit artifact-rule selection, in-process conflict-group rollback, interruption/re-prepare behavior, group-level partial-result contracts, and migrate bulk archive.
7. Update generated assets, documentation, compatibility guidance, and end-to-end coverage.

Rollback is staged: the skill-template migration can be reverted while the additive phase commands remain unused, and the one-shot archive compatibility path remains available throughout. Attempt data is temporary and versioned, so rollback requires no durable project-data migration.

## Open Questions

None. The command spelling and contract version are intentionally explicit in this proposal so review can change them before implementation without creating a compatibility commitment.
