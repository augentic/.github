# augentic/.github

The organisation's GitHub profile and its default community health files.

- [profile/README.md](profile/README.md) is the profile shown at
  [github.com/augentic](https://github.com/augentic).
- [SECURITY.md](SECURITY.md) and [SUPPORT.md](SUPPORT.md) are the defaults
  GitHub shows for every Augentic repository that does not carry its own.

## Where the shared tooling went

The reusable workflows, composite actions, mise tasks, and release notes that
used to live here moved to [`augentic/toolkit`](https://github.com/augentic/toolkit),
where the `conventions/` tree and the `conventions` program keep every
repository's shared files in step. Pin that repository's tags:

```yaml
jobs:
  ci:
    uses: augentic/toolkit/.github/workflows/ci.yaml@v0.3.0
```

```toml
# mise.toml
[task_config]
includes = ["git::https://github.com/augentic/toolkit.git//mise/rust.toml?ref=v0.3.0"]
```

The tags `v0.1.0` through `v0.2.0` stay on this repository, so a reference to
`augentic/.github/...@v0.2.0` keeps resolving; nothing new is tagged here.
