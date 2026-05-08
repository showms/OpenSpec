## 1. Extend config validation and shared config resolution

- [x] 1.1 Update project config validation to allow reserved workflow rule targets for `apply` and `archive` while keeping unknown targets actionable.
- [x] 1.2 Add or adapt shared config resolution logic so project `context` and workflow-scoped rules can be consumed consistently by artifact, apply, and archive instruction generation.
- [x] 1.3 Add unit tests covering valid workflow targets, unknown target diagnostics, malformed rules, and backward compatibility with existing artifact keys.

## 2. Inject config guidance into apply instructions

- [x] 2.1 Extend apply instruction generation types and payloads to include injected project `context` and `rules.apply` when configured.
- [x] 2.2 Update apply instruction rendering and JSON output so workflow guidance is surfaced without breaking existing schema-driven behavior.
- [x] 2.3 Add tests covering apply instructions with and without workflow-scoped rules, including cross-platform path-safe expectations.

## 3. Inject config guidance into archive workflow guidance

- [x] 3.1 Identify the archive guidance generation path and wire in injected project `context` and `rules.archive` without changing archive enforcement semantics.
- [x] 3.2 Update archive workflow templates or supporting generation code so archive guidance remains additive to existing readiness checks, prompts, and summaries.
- [x] 3.3 Add tests covering archive guidance with and without workflow-scoped rules.

## 4. Update prompts, docs, and examples

- [x] 4.1 Update config help text and prompts to document `rules.apply` and `rules.archive` alongside existing artifact rule keys.
- [x] 4.2 Update user-facing docs and examples to show how shared `context` and workflow-scoped rules apply across artifact, apply, and archive flows.
- [x] 4.3 Add migration guidance describing when teams should keep artifact-specific rules versus move workflow guidance into `apply` or `archive` targets.

## 5. Verify implementation quality

- [x] 5.1 Run the relevant test suites for project config parsing, artifact/apply instructions, and archive guidance.
- [x] 5.2 Add or update Windows-oriented assertions where file path handling or output paths are involved.
- [x] 5.3 Manually review the resulting instruction surfaces to confirm artifact behavior is unchanged and apply/archive guidance now includes the expected config injection.