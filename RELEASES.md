## 0.1.1

### Changed

- `ci.yaml`: the `test` job no longer passes `--tests` to `cargo hack nextest run`,
  which was building every test target per feature combination causing excessive
  run time for little benefit.

## 0.1.0

### Added

- Reusable Rust workflows: `ci.yaml`, `audit.yaml`, `release.yaml`,
  `patch.yaml`, `publish.yaml`, `crates.yaml`, `wasm.yaml`.
- Composite actions: `git-identity`, `cargo-version`, `release-notes`.
- Shared mise tasks for Rust workspaces (`mise/rust.toml`).
- `lint.yaml` running actionlint over this repository's workflows.
- Tagged releases of this repository via the `Release` workflow.
- `release.yaml`, `publish.yaml` and `patch.yaml` resolve their composite
  actions at the same commit as the workflow, so pinning a tag pins the
  actions too.
