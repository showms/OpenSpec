## 1. Project Config Update

- [x] 1.1 Resolve the active project config with the shared `config.yaml`/`config.yml` lookup and create `config.yaml` only when neither file exists.
- [x] 1.2 Prepare an existing config update with the YAML document API, set the canonical `schema` field, remove the obsolete `defaultSchema` field, and preserve unrelated entries and comments.
- [x] 1.3 Integrate the prepared config update into flagged and interactive `schema init` default selection without changing `--no-default` or normal schema generation.

## 2. Regression Coverage

- [x] 2.1 Add a command-level test proving `--default` creates a missing config with `schema` and that subsequent change creation selects the initialized schema.
- [x] 2.2 Add command-level coverage for updating `config.yaml` while preserving comments and unrelated fields and removing a stale `defaultSchema`.
- [x] 2.3 Add cross-platform path coverage proving an existing `config.yml` is updated in place without creating a competing `config.yaml`.
- [x] 2.4 Add or retain coverage proving `--no-default` leaves the active project config unchanged.

## 3. Release Metadata

- [x] 3.1 Add a patch changeset describing that `schema init --default` now sets the effective project default without discarding existing configuration.

## 4. Verification

- [x] 4.1 Run the focused schema command tests and any project-config tests affected by the implementation.
- [x] 4.2 Run lint, build, and strict OpenSpec validation for `fix-schema-init-default-config`.
- [x] 4.3 Confirm the new filename and path cases run in the repository's existing Windows CI coverage, updating test inclusion only if required.
