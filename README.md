# Shared GitHub Resources

Shared Github CI and release workflows for Augentic repositories. Consumers
should pin a release tag (`@vX.Y.Z`) rather than `@main`; see
[Versioning](#versioning).

## Versioning

This repository is released as `vX.Y.Z` tags with a matching GitHub release,
starting at `v0.1.0`. While on `0.x`, a **minor** bump signals a breaking
change (renamed inputs, changed behaviour, removed workflows) and a **patch**
bump is a fix. Release notes live in [RELEASES.md](RELEASES.md).

Pin the tag in every reference to this repository:

```yaml
jobs:
  ci:
    uses: augentic/.github/.github/workflows/ci.yaml@v0.1.0
```

```toml
# mise.toml
[task_config]
includes = ["git::https://github.com/augentic/.github.git//mise/rust.toml?ref=v0.1.0"]
```

Because `release.yaml`, `publish.yaml` and `patch.yaml` resolve their
composite actions at their own commit (see
[Composite actions in reusable workflows](#composite-actions-in-reusable-workflows)),
pinning the workflow tag pins everything it runs.

Let Dependabot open bump PRs for the pinned workflows by adding a
`github-actions` entry to the consuming repository's `.github/dependabot.yml`:

```yaml
version: 2
updates:
  - package-ecosystem: github-actions
    directory: /
    schedule:
      interval: weekly
    groups:
      actions:
        patterns:
          - "*"
```

`@main` still works for trying unreleased changes but is not the supported
reference: it can change under a consumer at any time.

## Required secrets

Some reusable workflows require secrets to be provisioned by the calling
repository.

### `release.yaml`

Creates a release branch from `main` and opens a "bump version" PR back into
`main`. The PR is opened using a GitHub App installation token so that branch
protection / required-status-checks fire on the PR.

| Secret | Required | Used for |
|---|---|---|
| `APP_ID` | yes | App ID of a GitHub App installed on the repository with `contents: write` and `pull_requests: write` permissions. Passed to `actions/create-github-app-token` as `client-id` (see below). |
| `APP_PRIVATE_KEY` | yes | PEM-encoded private key for the same GitHub App. |
| `CARGO_REGISTRY_TOKEN` | yes | Token used by `cargo update --workspace` when private-registry dependencies are present. |

These secrets are validated before any job runs: a calling repository that
inherits the workflow without `APP_ID` / `APP_PRIVATE_KEY` available fails
immediately with `Secret APP_ID is required, but not provided while calling.`

#### Why a GitHub App token

The "bump version" PR cannot be opened with the default `GITHUB_TOKEN`,
because PRs created by it do not trigger other workflows — so branch
protection and required status checks would never fire and the PR could not be
merged. The workflow instead mints a short-lived installation token from a
GitHub App (`actions/create-github-app-token`), which is treated as a real
actor and lets CI run on the PR.

#### Setting up the GitHub App

1. Create a GitHub App in the `augentic` org (Settings → Developer settings →
   GitHub Apps → New GitHub App).
2. Grant it the repository permissions **Contents: Read and write** and
   **Pull requests: Read and write**. No webhook is required.
3. Note the **App ID** and generate a **private key** (a `.pem` file).
4. **Install** the App on the org and grant it access to every repository that
   calls `release.yaml`.
5. Provision the secrets. Prefer **organization** Actions secrets shared with
   the consuming repositories — the same pattern as `CARGO_REGISTRY_TOKEN` —
   so every release-capable repo inherits them:
   - `APP_ID` — the numeric App ID.
   - `APP_PRIVATE_KEY` — the full PEM key, including the `BEGIN`/`END` lines.

`actions/create-github-app-token` deprecated its `app-id` input in favour of
`client-id`. The workflow passes `APP_ID` as `client-id`: GitHub accepts either
the App ID or the Client ID as the JWT issuer, so the existing secret keeps
working and nothing needs to be re-provisioned. If you would rather store the
App's Client ID (`Iv1...`) in `APP_ID`, that works too.

### `publish.yaml`

No secrets beyond `GITHUB_TOKEN`. The workflow is safe to re-run after a
partial failure (for example when the downstream `crates.yaml` job is rate
limited): the `Release <version>` commit is only made when `RELEASES.md` still
says `Unreleased`, the `v<version>` tag is only pushed when it does not already
exist, and the GitHub release is skipped when one already exists for the tag.
The tag job checks out the head of the release branch rather than the
triggering commit so that a re-run sees the commit pushed by the earlier
attempt.

### `crates.yaml`

| Secret | Required | Used for |
|---|---|---|
| `CARGO_REGISTRY_TOKEN` | yes | Authenticates `cargo publish --workspace --locked`. |

Re-runnable: workspace members already on crates.io at the current version are
passed to `cargo publish` as `--exclude`, and when crates.io answers `429 Too
Many Requests` (new crates are admitted slowly, see
[crates.io rate limits](https://crates.io/docs/rate-limits)) the job sleeps
until the time crates.io names and retries, up to 12 attempts. Any other
publish failure fails the job immediately.

### `patch.yaml`

| Secret | Required | Used for |
|---|---|---|
| `CARGO_REGISTRY_TOKEN` | yes | Token used by `cargo update --workspace` when private-registry dependencies are present. |

### `wasm.yaml`

| Secret | Required | Used for |
|---|---|---|
| `AZURE_CLIENT_ID` | yes | OIDC federated identity for the Azure CLI. |
| `AZURE_TENANT_ID` | yes | Azure tenant for the federated identity. |
| `AZURE_SUBSCRIPTION_ID` | yes | Azure subscription containing the storage account and container app. |

### `self-release.yaml`

Not reusable; it tags and releases this repository (see
[Releasing this repository](#releasing-this-repository)). No secrets beyond
`GITHUB_TOKEN`.

## Releasing this repository

Maintainers cut a release of the shared workflows as follows:

1. Open a PR that adds a new `## X.Y.Z` section at the top of
   [RELEASES.md](RELEASES.md), above the previous version, and writes the
   notes for it (`### Added` / `### Changed` / `### Fixed`). Line 1 of the
   file is the version the `Release` workflow will tag. Squash-merge the PR.
2. In Actions, run the **Release** workflow (`self-release.yaml`) on `main`.
3. The workflow reads the version from line 1, pushes an annotated `vX.Y.Z`
   tag at the head of `main`, and creates a GitHub release named
   `Release vX.Y.Z` whose body is that section of `RELEASES.md` followed by
   GitHub's generated "What's Changed" list (Dependabot PRs are filtered out
   by [.github/release.yaml](.github/release.yaml)). The release is marked as
   latest and created immutably.

The workflow refuses to run when:

- it is dispatched on a branch other than `main`;
- line 1 of `RELEASES.md` is not exactly `## MAJOR.MINOR.PATCH`;
- the section under line 1 is empty (write the notes first);
- `vX.Y.Z` is already tagged **and** released, which means line 1 was not
  bumped since the last release.

It is safe to re-run after a partial failure: an existing tag without a
release is reused, and an existing release is skipped.

Unlike `publish.yaml` for the Rust repositories, nothing is committed back:
`main` is governed by the organisation's **Merge** ruleset (PR with review and
signed commits), so there is no `Released <date>` line in `RELEASES.md` and
the date lives on the GitHub release. "Unreleased" is simply a version on
line 1 that has no tag yet.

## Conventions

### Composite actions in reusable workflows

A reusable workflow must never reference this repository's composite actions
as `augentic/.github/.github/actions/<name>@main`: a consumer pinned to
`@v0.1.0` would still run the actions from `main`. `uses:` cannot take an
expression, so the tag cannot be substituted at release time either.

Instead, each job that needs a composite action checks out this repository at
the commit the reusable workflow itself is running from, using the `job`
context, and references the actions by local path:

```yaml
      - uses: actions/checkout@v7            # consumer repository

      - name: Check out shared actions
        uses: actions/checkout@v7
        with:
          repository: ${{ job.workflow_repository }}
          ref: ${{ job.workflow_sha }}
          path: .augentic
          persist-credentials: false
      - run: echo '/.augentic/' >> .git/info/exclude

      - uses: ./.augentic/.github/actions/git-identity
```

Local `uses:` paths must live inside the workspace, so the checkout lands in
`.augentic/` next to the consumer's code. The `.git/info/exclude` line hides
it from git without touching tracked files, so neither `git commit -am` nor
`peter-evans/create-pull-request` (which stages untracked files by default)
can carry it into a consumer branch. The composite actions run with the
workspace root as their working directory, so `cargo` and `git` still act on
the consumer repository.

`job.workflow_repository` / `job.workflow_sha` are newer than the `job`
context type bundled with the pinned actionlint, so
[.github/actionlint.yaml](.github/actionlint.yaml) ignores those two reports
per workflow; add an entry there when a new workflow adopts the pattern.
