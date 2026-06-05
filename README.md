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
| `APP_ID` | yes | App ID of a GitHub App installed on the repository with `contents: write` and `pull_requests: write` permissions. |
| `APP_PRIVATE_KEY` | yes | PEM-encoded private key for the same GitHub App. |
| `CARGO_REGISTRY_TOKEN` | yes | Token used by `cargo generate-lockfile` when private-registry dependencies are present. |

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

### `crates.yaml`

| Secret | Required | Used for |
|---|---|---|
| `CARGO_REGISTRY_TOKEN` | yes | Authenticates `cargo publish --workspace --locked`. |

### `patch.yaml`

| Secret | Required | Used for |
|---|---|---|
| `CARGO_REGISTRY_TOKEN` | yes | Token used by `cargo generate-lockfile` when private-registry dependencies are present. |

### `wasm.yaml`

| Secret | Required | Used for |
|---|---|---|
| `AZURE_CLIENT_ID` | yes | OIDC federated identity for the Azure CLI. |
| `AZURE_TENANT_ID` | yes | Azure tenant for the federated identity. |
| `AZURE_SUBSCRIPTION_ID` | yes | Azure subscription containing the storage account and container app. |
