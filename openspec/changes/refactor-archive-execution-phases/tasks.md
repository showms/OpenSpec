## 1. Characterize and Extract the Existing Archive Transaction

- [ ] 1.1 Add focused characterization coverage for current human output, JSON envelopes, diagnostic codes, flags, prompts, exit codes, store selection, and successful results before moving implementation code.
- [ ] 1.2 Extract archive result, diagnostic, option, check, and internal mutation types from `ArchiveCommand` without changing exported behavior.
- [ ] 1.3 Extract canonical path validation, fingerprints, and stale-input comparison helpers with their existing path-escape, symlink, casing, and concurrent-edit tests.
- [ ] 1.4 Extract archive destination naming, availability checks, exclusive claims, and claim recovery with the existing local-date and collision behavior intact.
- [ ] 1.5 Extract target snapshots, ordinary spec writes, capability retirement, rollback, and backup finalization behind a single-change transaction interface.
- [ ] 1.6 Extract change movement and EPERM/EXDEV fallback recovery while preserving verified-copy and retained-destination semantics on Windows, macOS, and Linux.
- [ ] 1.7 Reduce `ArchiveCommand` to orchestration and presentation over the extracted services and prove the complete pre-existing archive test suite is unchanged.

## 2. Define and Store Archive Attempts

- [ ] 2.1 Add explicit constants for the archive-attempt contract version, supported versions, generated temporary prefix, sibling lock suffix, plan file, result file, input directories, and output directories.
- [ ] 2.2 Define runtime-validated single and batch plan schemas, root and change snapshots, typed checks, target entries, contributing deltas, overlapping-attempt summaries and available actions, `sync`/`retire`/`already-synced`/`skip` decisions, derived effects, semantic-conflict decisions, user-selected artifact-rule resolutions, validation results, and conflict-group results.
- [ ] 2.3 Implement private archive-attempt workspace creation with `fs.mkdtemp()`, restrictive permissions where supported, generated plan and entry IDs, publication only after overlap checks pass or explicit new-attempt authorization is accepted, and no project-root mutation.
- [ ] 2.4 Snapshot concrete deltas and one baseline per canonical target into ID-addressed attempt-workspace files and record explicit mappings from every change contribution to its target and candidate.
- [ ] 2.5 Generate `result.json` with pending target results and explicit contribution inclusion or skip decisions; require a final semantic decision for every target before validation.
- [ ] 2.6 Implement attempt loading that treats plan paths and serialized fields as untrusted, re-establishes the canonical managed attempt-workspace root, and reports the plan path needed to resume that exact attempt.
- [ ] 2.7 Implement sibling attempt locks with exclusive create, plan/process/operation/nonce ownership, identity-checked release in `finally`, one lock per single or batch attempt, no long-lived lock during agent work or user review, and a fixed attempt-then-archive-root acquisition order for finalization.
- [ ] 2.8 Add `archive-attempt inspect`, explicit `unlock --yes` for repeatedly verified attempt locks and associated phased archive-root claims, and token-bound `cleanup --attempt ... --yes`; never steal a claim automatically from its age or process metadata alone or accept an arbitrary recursive-delete target.
- [ ] 2.9 Run lifecycle cleanup and overlap discovery at the start of every prepare, automatically delete only unlocked attempts proven consumed by matching complete archives, return stale, partial, orphaned, invalid, or locked attempts with their actions, and block default workspace creation when a resumable single or batch attempt includes any selected change.
- [ ] 2.10 Add attempt-workspace storage, cleanup, and lock tests for unusual capability IDs, Windows separators and casing, symlinks and junctions, permission failures, replaced locks, crashed owners, read-only inspection, consumed detection, cleanup-token invalidation, unrelated temporary directories, abandoned attempts, and proof that cleanup never reverses project mutations.

## 3. Implement Prepare and Typed Checks

- [ ] 3.1 Add non-interactive `archive-attempt prepare --change <name> [--new-attempt --yes] --json` CLI routing with normal repo/store root selection, prepare-time lifecycle and overlap results, rejection of `--new-attempt` without `--yes`, and stdout-pure JSON.
- [ ] 3.2 Build a planner that resolves the change, schema, artifact status, task progress, archive root, `finalize-local-date` naming policy, preview path, and existing built-in validation facts once.
- [ ] 3.3 Discover spec work only from `artifactPaths.specs.existingOutputPaths` and map each explicit delta to a canonical main-spec target without directory inference.
- [ ] 3.4 Load one current config snapshot and attach project context, archive operation guidance, and per-entry `specs` artifact rules as separate fields.
- [ ] 3.5 Record fingerprints for the selected root, complete change tree and metadata, config, task/artifact facts, deltas, baselines, archive root, and naming policy without binding plan validity to a preview date.
- [ ] 3.6 Return built-in checks through the explicit check-ID list with stable `pass`, `warning`, `blocked`, `confirmationRequired`, message, and fix fields.
- [ ] 3.7 Add prepare tests for valid changes, default blocking on resumable same-change attempts, explicit `--new-attempt --yes` coexistence, single/batch overlaps without candidate migration, locked-overlap rejection, consumed-attempt deletion, retained-attempt choices, incomplete artifacts/tasks, invalid changes, destination previews and collisions, selected stores, no-spec schemas, empty specs outputs, and malformed instruction inputs.
- [ ] 3.8 Add cross-platform prepare tests proving path output and identity checks work on Windows, macOS, and Linux without hardcoded separators.

## 4. Implement Candidate Validation and Tokens

- [ ] 4.1 Add non-interactive `archive-attempt validate --plan <plan-path> --json` routing and reject unsupported contract versions before reading candidate work.
- [ ] 4.2 Acquire the attempt lock; re-resolve the root, change, schema, artifact paths, config, checks, archive root, naming policy, delta targets, contributions, and allowed attempt-workspace entries instead of trusting serialized plan paths; and reject candidate or manifest reads whose fingerprints change during parsing.
- [ ] 4.3 Reject stale plans when change metadata or files, task/artifact facts, config, delta inputs, baseline targets, root identity, archive root, or naming policy changed after prepare, while allowing the local preview date to roll forward.
- [ ] 4.4 Validate that the result manifest contains exactly one explicit `sync`, `retire`, `already-synced`, or `skip` decision for every planned target plus explicit inclusion or skip state for every batch contribution, with no unknown, duplicate, missing, or pending entry.
- [ ] 4.5 Reject plan, manifest, input, and candidate paths that escape their canonical roots, traverse symlinks, change file type, alias mutation targets, or rely on case-insensitive duplicates.
- [ ] 4.6 Validate every `sync` candidate as a canonical main spec, reject delta operation sections, derive `create` or `update`, and report entry-scoped structured diagnostics without changing the project.
- [ ] 4.7 Validate every `retire` result against current metadata authorization, requiring every included batch contributor to authorize aggregate retirement, plus main-spec containment, content-safety rules, and validation-enabled requirements.
- [ ] 4.8 Validate `already-synced` and user-selected `skip` explicitly and prevent an absent candidate or missing baseline from silently becoming a deletion, retirement, synchronized result, or skip.
- [ ] 4.9 Produce baseline-to-final-state diffs, spec-level effects, requirement-operation counts, and a canonical token covering the version, normalized plan, results, candidates, live inputs, archive policy, typed checks, contributions, confirmed semantic conflicts, and selected and suppressed artifact-rule sources.
- [ ] 4.10 Add validation tests for every decision and derived effect, exact diff output, structural failures, a main spec edited after agent baseline read, concurrent candidate/manifest writes, stale inputs, manifest tampering, token determinism, path attacks, unauthorized retirement, unresolved rule conflicts, warning preservation, local-date rollover, and zero project writes.

## 5. Implement Finalization and Recovery

- [ ] 5.1 Add non-interactive `archive-attempt finalize --plan <plan-path> --validation-token <token> --yes --json` routing with explicit confirmation enforcement.
- [ ] 5.2 Acquire the attempt lock, re-run complete validation immediately before mutation, compare every live target with its baseline again immediately before that target is changed, and reject tokens invalidated by candidate, result, check, project, root, archive-policy, contribution, semantic-conflict, or artifact-rule-resolution changes.
- [ ] 5.3 Capture the current local date once, derive and recheck the dated destination, preserve already date-prefixed names, and acquire the archive-root transaction lock before the first spec mutation.
- [ ] 5.4 Capture all mutation targets, apply validated `sync` candidates with target-local atomic replacement, and defer retirements until ordinary writes succeed.
- [ ] 5.5 Apply authorized retirements with the existing real-root containment, displaced-file verification, empty-directory pruning, warning, and recovery behavior.
- [ ] 5.6 Verify active delta and authorization fingerprints, move the change with the extracted cross-device fallback, and verify archived inputs before commit.
- [ ] 5.7 Restore attempted spec mutations and return the change to its active path after pre-commit failures reported while the process remains running, while preserving completed fallback archives when cleanup fails and making no durable-journal or abrupt-termination rollback claim.
- [ ] 5.8 Emit versioned finalization JSON containing the existing archive result fields, semantic decisions, derived effects, exact dated destination, warnings, root envelope, and stable diagnostics, then delete the consumed attempt workspace and release owned locks in `finally`.
- [ ] 5.9 Add race and recovery tests for edits before writes, edits before retirement, attempt and archive-root claims, replaced locks, local-midnight rollover, token invalidation, move failures, in-process rollback conflicts, EPERM/EXDEV fallback, retained complete archives, abrupt termination after partial writes, explicit claim cleanup, and a new idempotent prepare from the resulting current state.

## 6. Preserve the One-Shot Archive Command

- [ ] 6.1 Implement a compatibility candidate builder that maps the existing deterministic `findSpecUpdates`, `buildUpdatedSpec`, skip, retirement, already-synced, warning, and totals behavior onto the shared semantic decisions and derived effects.
- [ ] 6.2 Route one-shot archive through the shared planner and single-change transaction services without spawning archive-attempt CLI subprocesses.
- [ ] 6.3 Preserve `--skip-specs`, declined spec updates, `--no-validate`, `--yes`, interactive selection, non-interactive failure guidance, store flags, and validation-disabled retirement behavior.
- [ ] 6.4 Preserve human output ordering, JSON envelopes, diagnostics, exit codes, and recovery messages through snapshot or exact assertion coverage.
- [ ] 6.5 Run the full existing archive, specs-apply, store-root, and CLI end-to-end suites and fix only regressions introduced by the refactor.

## 7. Migrate Single-Change Agent Workflows

- [ ] 7.1 Add archive-candidate mode to the sync workflow with explicit plan version, selected entry IDs, immutable input paths, candidate output paths, and supplied artifact-rule snapshots.
- [ ] 7.2 Make candidate sync write only managed outputs; idempotently recognize outcomes already represented by a new baseline after an interrupted attempt; record `sync`, `retire`, or `already-synced` for processed targets; honor user-selected `skip`; and return a semantic summary and baseline-to-final-state diff without claiming validation or archive completion.
- [ ] 7.3 Keep standalone `/opsx:sync` discovery, current rule loading, main-spec writes, validation, retirement, and output unchanged when no archive plan is supplied.
- [ ] 7.4 Update the single-change archive skill to handle overlapping-attempt results by asking the user to resume an exact plan, inspect or clean and retry, explicitly create a new attempt, or cancel; then render typed checks, perform candidate sync inline, validate, display `CREATE`/`UPDATE`/`RETIRE`/`ALREADY SYNCED`/`SKIPPED` effects plus exact diffs and requirement counts, request final confirmation, and finalize through the CLI.
- [ ] 7.5 Make cancellation discard the attempt and prove that main specs and the active change remain unchanged before finalization.
- [ ] 7.6 Add explicit contract-version negotiation and older-CLI guidance that offers upgrade or the direct one-shot archive command without manually moving the change.
- [ ] 7.7 Regenerate skill and command assets, update parity hashes, and add tests proving no generated archive workflow writes main specs or invokes a filesystem move directly.
- [ ] 7.8 Add single-change integration tests for rules/context separation, no-spec schemas, explicit skips, already-synced entries, idempotent restart after partially applied outcomes, create/update effects, retirements, invalid candidates, exact confirmation diffs, existing-target guidance without alternate dates, finalize-time dates, cancellation, final confirmation, and selected stores.

## 8. Add Batch and Mixed-Schema Orchestration

- [ ] 8.1 Add `archive-attempt prepare-batch --change <name>... [--new-attempt --yes]` to report every retained single or batch attempt overlapping the selection, create no workspace by default when resumable overlaps exist, and otherwise resolve the selected changes into one batch plan and map the potential canonical change-to-target graph.
- [ ] 8.2 Snapshot each canonical target baseline once, attach every contributing delta and its artifact-rule snapshot in stable `created`-date then change-name order, and create one candidate output per shared target.
- [ ] 8.3 Extend candidate sync to combine compatible intent across all included contributions, report true semantic conflicts without silently applying last-created-wins, pause on incompatible artifact rules, and regenerate the candidate after the user selects the controlling rule for every rule conflict.
- [ ] 8.4 Use a later `created` date as the visible proposed resolution only for incompatible change intent, require explicit confirmation for every semantic conflict, require a user decision when dates are missing or equal, and never use creation date to resolve artifact-rule conflicts.
- [ ] 8.5 Add `archive-attempt validate-batch` to exclude explicitly skipped contribution edges, derive effective connected conflict groups, validate every target and artifact-rule selection, produce aggregate diffs, effects, requirement counts and contribution summaries, preflight destinations, and bind the deterministic group order plus semantic and rule-conflict decisions into one token.
- [ ] 8.6 Add `archive-attempt finalize-batch` that captures one current local date, holds the batch-attempt then archive-root locks, rechecks every group and target immediately before mutation, and commits or rolls back all candidate mutations and change moves within each connected group for failures returned while the process remains running.
- [ ] 8.7 Update bulk archive templates to prepare all changes, finish aggregate candidate work, validate the entire batch, show aggregate diffs and conflicts, and ask once for confirmation before the first group.
- [ ] 8.8 Persist committed-group receipts in the retained batch attempt, report exact completed, failed, and unfinalized groups, resume only after verifying completed archives and spec fingerprints and revalidating remaining groups, and allow a new prepare from current project state after an abruptly interrupted group is explicitly abandoned.
- [ ] 8.9 Preserve mixed-schema behavior so no-spec changes contribute no target edges and proceed as independent readiness-and-move groups.
- [ ] 8.10 Add batch tests for overlapping single and batch attempts, explicit new-attempt authorization without candidate migration, shared-target aggregate candidates, transitive conflict groups, compatible and incompatible intent, user-selected artifact-rule precedence, newer-date semantic proposals, missing/equal dates, skipped contributions, mixed schemas, invalid later groups, in-process group rollback, abrupt termination with partial group state, wider-batch partial completion, retry under one batch-attempt lock, and zero writes before full prevalidation.

## 9. Documentation, Generated Assets, and Compatibility

- [ ] 9.1 Document the archive-attempt commands, attempt identity and resume path, overlap choices, explicit `--new-attempt --yes`, single/batch coexistence rules, versioned JSON fields, semantic decisions and derived effects, typed checks, exact diff contract, validation token, confirmation, prepare-time cleanup, cleanup tokens, inspect, unlock, discard, and recovery or re-prepare behavior.
- [ ] 9.2 Update archive, customization, agent-contract, and migration documentation to keep artifact rules, operation guidance, skill steps, and CLI checks distinct.
- [ ] 9.3 Document one-shot compatibility, independent skill/CLI upgrades, OS-temporary attempt-workspace privacy, lifecycle cleanup without TTL, operation and archive-root locks, finalize-time local dates, in-process single-change and conflict-group rollback, the absence of a persistent journal or abrupt-termination atomicity, and non-global batch atomicity.
- [ ] 9.4 Update generated templates and help text through the canonical generation pipeline and verify every tracked output by explicit asset name.

## 10. Verification and Rollout

- [ ] 10.1 Add focused unit coverage for all new single and batch schemas, semantic decisions and effects, semantic and artifact-rule conflict resolutions, check IDs, fingerprints, aggregate diff serialization, token generation, lifecycle cleanup and cleanup tokens, locks, and path boundaries.
- [ ] 10.2 Add CLI integration coverage for every prepare, explicit-new prepare, prepare-batch, validate, validate-batch, finalize, finalize-batch, inspect, unlock, cleanup, and discard success and failure envelope.
- [ ] 10.3 Run Windows CI coverage for temporary paths, exclusive attempt locks, case-insensitive aliases, symlink/junction handling, target-local writes, finalize-time dates, EPERM fallback, lifecycle cleanup, and store roots.
- [ ] 10.4 Run macOS and Linux coverage for realpath aliases, permissions, exclusive attempt locks, cross-device fallback, archive-root claims, signals, explicit stale-lock recovery, and lifecycle cleanup.
- [ ] 10.5 Run build, lint, focused archive/attempt/template tests, the full test suite, and strict OpenSpec validation.
- [ ] 10.6 Perform a release-audit pass for CLI/skill version skew, generated asset coverage, documentation, and changeset requirements before enabling phased archive in generated skills.
