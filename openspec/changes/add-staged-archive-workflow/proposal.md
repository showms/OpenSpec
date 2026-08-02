## Why

Archive needs one reviewable path that keeps agent-authored Markdown reconciliation away from formal specs until the user approves the exact result. The workflow must also recover safely when a single archive operation is interrupted after only part of its work completes.

## What Changes

- Extend `openspec archive <change>` with the staged lifecycle `prepare`, `status`, `validate`, `finalize`, `resolve`, `repair`, and `abort`, plus root-level `cleanup`.
- Make `prepare` discover the selected delta specs, persist exact main-spec base files or explicit absence records, and create plan-owned candidate specs without changing formal specs or moving the change.
- Allow repeatable `--include-spec` and `--exclude-spec` selection for a single archive; `--skip-specs` creates a move-only plan.
- Keep semantic Markdown reconciliation agent-driven, but limit archive-driven sync to the candidate paths returned by the CLI.
- Make `validate` freeze the exact candidate bytes and archive payload manifest, persist the complete review, and return an opaque validation identifier plus an approval token bound to the exact review delivery metadata.
- Make `finalize` require the current validation identifier, its approval token, and explicit confirmation; recheck source, payload, and base hashes; apply each reviewed spec with atomic per-file replacement; and move the change only after every included target and payload entry match the reviewed snapshot.
- Make interrupted finalization resumable by classifying each target as pending, already applied, or conflicted. The CLI preserves conflicting bytes and returns base, reviewed, and current evidence so the archive skill can investigate and explain the conflict; an evidence-bound `resolve` flow prepares a recovery candidate, freezes a new reviewed amendment, and resumes only after user approval. It does not silently overwrite, roll back, or automatically rebase content.
- Give `broken` and `orphaned` states an evidence-bound `repair` flow. Safe repairs can reconstruct primary state from a matching recovery capsule, resume from a verified source, adopt a verified destination, quarantine an identical duplicate source, or explicitly rebind an occupied destination; ambiguous evidence remains preserved and non-mutating.
- Bind the archive date and destination when formal commit begins, allow an explicit validated archive name to avoid same-day name reuse, report a distinct destination conflict if that bound path later becomes unavailable, and retain a completion receipt so a missed success response can be recovered safely.
- Duplicate commit-critical reviewed bytes and records into a source-local recovery capsule before formal writes, bind it to the plan and commit token, carry it through movement, and use it to reconstruct a missing primary plan when exactly one consistent lineage remains. Deleting every generated evidence copy remains outside the recovery guarantee.
- Expose staged state by change name, including damaged or orphaned recovery state. Give every plan an independent identity, retain completed receipts for 30 days outside the reusable active-plan slot, and provide explicit abort and cleanup behavior for generated state.
- Recheck the complete discovered delta set, readiness, and archive payload manifest at validation and finalization. The review distinguishes the complete formal-spec diff from the exact path/hash manifest of the other files that will be moved, and any post-validation payload drift requires a new review before commit.
- Define archive around a documented single-writer assumption without adding a finalize or planning-root writer lock in this change: generated skills serialize normal archive use, and users do not run another archive, standalone sync, or manual formal-spec edit during finalization.
- Treat concurrent formal-spec writers as unsupported. Hash checks report observable stale or conflicted content but do not coordinate simultaneous writers or eliminate the race in which two writers pass preflight before either changes a target. A future writer lock would need to cover direct and staged archive, standalone sync, and every other cooperative formal-spec writer together.
- Process bulk archives sequentially, including preparation, review, confirmation, and finalization for each selected change.
- Preserve the direct `openspec archive <change>` interface as a compatibility adapter over the same candidate, validation, per-target classification, atomic-write, resume, and receipt primitives.
- Keep a narrow legacy fallback for generated skills running with a CLI that does not support staged archive.

## Capabilities

### New Capabilities

- `archive-staged-workflow`: Defines candidate preparation, immutable spec/payload review, approval binding, single-writer forward-only finalization, interruption recovery, reviewed conflict resolution, evidence-bound orphan repair, status, abort, cleanup, completion receipts, and sequential bulk composition.

### Modified Capabilities

- `cli-archive`: Adds the staged, resolve, and repair command surface; capability and destination selection; review approval identity; resumable finalization; recovery diagnostics; and direct-command compatibility.
- `opsx-archive-skill`: Moves formal spec writes, conflict resolution, orphan repair, and archive movement from agent-authored shell steps to the CLI-owned staged lifecycle.
- `specs-sync-skill`: Adds archive-supplied normal and conflict-recovery candidate reconciliation while leaving standalone sync behavior unchanged.

## Impact

- Affects archive CLI parsing and execution, root/store-aware paths, archive validation, formal spec writes, archive movement and repair, JSON diagnostics, generated archive/sync skills, and bulk archive orchestration.
- Adds generated state under `openspec/.archive-plan/` and a temporary source-local recovery capsule, including immutable base files, candidates, validation and resolution snapshots, conflict and orphan evidence, payload manifests, commit state, complete review files, plan identities, approval tokens, and 30-day completion receipts that do not block a later same-name change.
- Changes failure behavior: reviewed spec writes that completed before an interruption remain applied, a later finalize resumes from the recorded state instead of rolling them back, and the archive skill—not the user—performs conflict or orphan investigation from preserved evidence before offering evidence-bound resolve or repair actions.
- Adds cross-platform verification for path safety, atomic file replacement and durability boundaries, Windows rename failures, cross-device staged movement, direct-copy tree verification, interrupted writes, damaged recovery state, missed responses, sequential bulk behavior, and direct-command compatibility.
