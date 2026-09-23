# Shared GitHub Resources

Shared Github CI and release workflows for Augentic repositories.

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
