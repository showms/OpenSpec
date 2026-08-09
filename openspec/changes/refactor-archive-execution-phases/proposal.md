## Why

OpenSpec currently has two archive implementations with different ownership boundaries: `openspec archive` performs deterministic spec merging and a rollback-capable filesystem transaction, while archive skills ask an agent to merge specs and then move the change themselves. This duplication makes the skill path depend on prompt compliance for checks that should be enforced by the CLI, and it prevents both paths from sharing one safe, reviewable archive contract.

## What Changes

- Introduce a versioned phased archive contract: prepare an archive attempt, perform agent work, validate, review an exact baseline-to-final-state diff, explicitly confirm, and finalize.
- Make prepare resolve the selected root, change, schema, completion state, delta inputs, target specs, archive root and finalize-time local-date naming policy, project context, archive guidance, and applicable `specs` artifact rules as structured data.
- Give agent-driven archive work a managed candidate workspace so semantic spec reconciliation does not modify main specs before validation and final confirmation.
- Make CLI validation reject stale plans, unsafe or unexpected paths, invalid candidate specs, unauthorized capability retirement, destination conflicts, and other conditions that would make finalization unsafe.
- Make CLI finalization revalidate the approved plan, resolve the archive name from the local date captured when finalization starts, claim the archive root, promote validated candidates, move the change, and preserve the existing in-process rollback and recovery guarantees.
- Change the single-change archive skill so the agent owns only semantic spec work and user-facing explanation; the CLI owns enforceable checks and all final project mutations.
- Give candidate results the explicit semantic decisions `sync`, `retire`, `already-synced`, and `skip`, while deriving the concrete `create`, `update`, `delete`, or no-file-change effect for review and finalization.
- Define batch behavior around conflict groups: changes connected through a shared main-spec target produce one aggregate result and baseline-to-final-state diff per target, resolve true semantic conflicts explicitly, and roll back together when a running finalize operation reports a failure, while the wider batch may still complete partially by group.
- Preserve mixed-schema behavior: changes without concrete `specs` artifact outputs skip semantic spec work and still use the same validate/finalize path.
- Preserve the existing `openspec archive <change>` command, flags, JSON envelope, diagnostics, and automation behavior through a compatibility path that reuses the new archive transaction boundaries.
- Keep standalone spec sync behavior intact while adding a candidate-workspace mode for archive-driven semantic reconciliation.
- Protect attempt-changing CLI operations with short-lived, attempt-specific locks; run lifecycle cleanup on every prepare; automatically remove only attempts proven consumed; and require the user to choose whether stale, partial, orphaned, invalid, or locked attempts are cleaned, recovered, or retained.
- Detect resumable attempts that already include a selected change before creating another candidate workspace. Require the user to resume an existing attempt, clean up retained state, explicitly authorize a concurrent new attempt, or cancel; never implicitly reuse, overwrite, or duplicate an attempt.
- Resolve incompatible artifact rules by asking the user which conflicting rule controls the candidate, keeping every non-conflicting rule in force, regenerating the candidate from that decision, and binding the resolution into validation.
- Do not add a persistent transaction journal or promise cross-file atomicity after abrupt process termination. Completed main-spec writes may remain, and a later attempt may use the current project state as its new baseline and reconcile only the remaining outcome.
- Do not add configurable enforceable operation checks in this change; future typed checks can be added individually when concrete enforcement requirements emerge.

## Capabilities

### New Capabilities

- `archive-execution-phases`: Define the versioned archive-attempt prepare, candidate-work, diff review, validate, confirmation, finalize, in-process recovery, interrupted-attempt handling, attempt lifecycle, conflict-group batch, and mixed-schema archive contract.

### Modified Capabilities

- `cli-archive`: Preserve the existing one-shot command while moving enforceable checks and project mutations onto shared phased archive primitives.
- `opsx-archive-skill`: Replace direct main-spec writes and direct filesystem moves with prepared candidate work followed by CLI validation and finalization.
- `specs-sync-skill`: Support an archive-supplied candidate workspace and rule snapshot without changing standalone sync behavior.
- `openspec-conventions`: Recognize agent-produced, CLI-validated candidate specs as an archive merge path alongside the compatible programmatic merge path.

## Impact

- Affected CLI and core areas include archive command routing, archive diagnostics, root and schema resolution, artifact status resolution, spec input discovery, path safety, validation, archive claims, snapshots, in-process rollback, interruption handling, and cross-device move recovery.
- Archive workflow templates and generated skills stop moving changes directly and consume structured phase responses instead.
- Archive-driven spec sync writes only to a managed candidate workspace until finalization, including one aggregate candidate for every shared batch target; standalone `/opsx:sync` continues to write main specs directly.
- A temporary archive attempt may be complete or may stop partway through, and it may outlive one conversation. When a resumable single or batch attempt already includes a selected change, prepare returns that overlap without creating another plan; the client must resume by plan path, clean up the retained attempt, cancel, or explicitly request a distinct attempt with `--new-attempt --yes`.
- Candidate validation and finalization use baseline fingerprints rather than a long-lived main-spec lock, so an external edit after the agent reads a baseline makes that attempt stale instead of being overwritten.
- Abrupt termination can leave already completed spec writes in place. After the user resolves stale claims and retained attempt state, a new prepare treats those writes as current baseline state; candidate sync must recognize already-applied outcomes and remain idempotent.
- The phased JSON contracts require explicit versioning and compatibility tests so generated skills and installed CLIs fail safely when their supported contract versions differ.
- Existing archive behavior remains available during migration, allowing the internal refactor and skill rollout to land incrementally without a breaking CLI change.
