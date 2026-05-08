## Context

Project config currently injects `context` and artifact-targeted `rules` only when generating artifact instructions. That leaves `/opsx:apply` and `/opsx:archive` without the same project-level guidance, even though they are part of the same end-to-end workflow.

This change is cross-cutting because it touches config validation, apply instruction generation, archive workflow guidance, config help text, docs, and tests. It also has to stay backward compatible with existing artifact keys in `openspec/config.yaml`.

Key constraints:

- Reuse the existing `context` and `rules` model rather than introducing a new config format.
- Preserve current artifact instruction behavior.
- Preserve current archive command enforcement and safety prompts.
- Keep behavior cross-platform for macOS, Linux, and Windows.

## Goals / Non-Goals

**Goals:**

- Make the existing top-level `context` available to `apply` and `archive` instruction surfaces.
- Allow reserved workflow rule targets for `apply` and `archive` while keeping artifact rule keys unchanged.
- Surface validation errors clearly when workflow rule targets are malformed or unsupported.
- Keep implementation aligned with existing instruction-loading and config-validation patterns.

**Non-Goals:**

- Redesign the structure of `openspec/config.yaml`.
- Change archive runtime enforcement semantics, readiness checks, or prompts.
- Expand workflow rule injection to `verify` or `sync`.
- Change how artifact-specific rules are matched and injected today.

## Decisions

### 1. Reserve workflow rule targets inside the existing `rules` object

Use reserved keys such as `rules.apply` and `rules.archive` inside the existing `rules` object rather than creating a parallel workflow config section.

This keeps the user mental model simple: `context` remains shared project guidance, and `rules` remains the place for instruction-time constraints. The only extension is that some rule keys now refer to workflow instruction surfaces instead of artifacts.

Alternatives considered:

- Add a new top-level `workflowRules` section. Rejected because it duplicates an existing concept and increases migration cost.
- Add nested workflow structure under a new config block. Rejected because it complicates validation and documentation for limited benefit.

### 2. Treat workflow targets as explicit reserved names during validation

Config validation should continue to validate artifact keys against the active schema, while also allowing the reserved workflow targets `apply` and `archive`.

This preserves today’s strict validation behavior for typos and unknown targets, but extends the allowlist in a narrow, intentional way.

Alternatives considered:

- Accept any unknown key and ignore it later. Rejected because it weakens existing validation and makes config mistakes harder to diagnose.
- Infer workflow targets from naming conventions. Rejected because explicit reserved targets are simpler to explain and test.

### 3. Inject workflow guidance at existing instruction-generation surfaces

`apply` should receive injected `context` plus `rules.apply` when apply instructions are generated. `archive` should receive injected `context` plus `rules.archive` when archive workflow guidance is generated.

Artifact rules should continue to apply only to matching artifact instructions. Workflow rules should apply only to their corresponding workflow instruction surfaces.

This keeps precedence simple and avoids ambiguous merges between artifact-specific and workflow-specific guidance.

Alternatives considered:

- Merge all artifact and workflow rules together for apply/archive. Rejected because it blurs scope and creates surprising interactions.
- Recompute workflow guidance ad hoc in command handlers. Rejected because it spreads config logic across multiple paths instead of reusing existing instruction assembly patterns.

### 4. Document migration as additive, not breaking

Existing configs that only use artifact keys should keep working unchanged. Teams that currently encode apply/archive guidance elsewhere can gradually move that guidance into `rules.apply` and `rules.archive` without changing artifact rules.

This keeps adoption low-risk and aligns with the proposal’s backward-compatibility goal.

Alternatives considered:

- Deprecate some artifact rule keys immediately. Rejected because this proposal is about extending instruction surfaces, not forcing config reorganization.

## Risks / Trade-offs

- [Validation surface grows] → Keep the reserved target list small and explicit (`apply`, `archive`) and continue failing fast on unknown keys.
- [Instruction behavior diverges across artifacts and workflows] → Keep scope boundaries explicit: artifact rules only affect artifacts, workflow rules only affect workflow instruction surfaces.
- [Archive guidance becomes confused with archive enforcement] → Limit this change to instruction generation and preserve existing archive safety checks and prompts.
- [Docs drift from implementation] → Update config help text, docs, examples, and tests in the same implementation change.
- [Cross-platform path issues in tests or templates] → Continue using Node.js path utilities and avoid hardcoded path separators in implementation and tests.

## Migration Plan

1. Extend config validation to recognize reserved workflow targets alongside artifact keys.
2. Update apply instruction generation to expose injected `context` and `rules.apply`.
3. Update archive workflow guidance generation to expose injected `context` and `rules.archive`.
4. Update config prompts, docs, and examples to show artifact and workflow rule targets together.
5. Add unit and integration coverage for validation, apply injection, archive injection, and backward compatibility.

Rollback is straightforward because the change is additive: removing workflow-target support reverts behavior to the current artifact-only model.

## Open Questions

- Should archive guidance reuse a shared instruction injection helper with apply/artifact generation, or keep archive-specific assembly with a shared config resolution step?
- Should the implementation expose workflow-injected guidance through new explicit fields on apply/archive instruction payloads, or only fold it into rendered instruction text?
- Do we need docs examples that demonstrate both `context` plus `rules.apply`/`rules.archive` together in the same config snippet?