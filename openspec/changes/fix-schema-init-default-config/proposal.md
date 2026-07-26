## Why

`openspec schema init --default` reports that the new schema was set as the project default, but it writes an unused `defaultSchema` key while schema resolution reads `schema`. As a result, subsequent changes continue to use the previous or built-in default, and updating an existing config can also discard unrelated configuration presentation or bypass an existing `config.yml`.

## What Changes

- Persist the selected schema under the project config's active `schema` key so subsequent commands actually use it.
- Update the existing project config file regardless of whether it is named `config.yaml` or `config.yml`, without creating a competing config file.
- Preserve unrelated configuration values and comments while updating the schema selection.
- Remove the obsolete `defaultSchema` key when repairing a config through `schema init --default`.
- Add command-level regression coverage for new configs, existing configs, alternate config filenames, and effective schema selection.
- Correct the schema-init behavioral specification to use the canonical project config key.

## Capabilities

### New Capabilities

None.

### Modified Capabilities

- `schema-init-command`: Setting a newly initialized schema as the project default must update the canonical project configuration and take effect for subsequent changes without disturbing unrelated config content.

## Impact

- `src/commands/schema.ts` project-config update path for `schema init`.
- Shared project-config path resolution and YAML document handling.
- `test/commands/schema.test.ts` command-level regression coverage.
- `openspec/specs/schema-init-command/spec.md` default-schema scenarios.
- Patch release metadata for the CLI behavior fix.
