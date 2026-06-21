# CI/CD Reusable Workflows

This repository contains GitHub Actions Reusable Workflows for building and deploying to Roblox.

---

## 1. Deploy Template (`deploy-template.yml`)

This workflow handles checking out the code, updating submodules, running `rojo build`, and using the Roblox Open Cloud API to publish the place file. It automatically handles routing to the correct environment:
- **Master Branch:** Deploys to Official (PROD) Place.
- **Other Branches:** Deploys to Test (TEST) Place.

**Example usage in your Place repository (e.g., `MyGame-arena/.github/workflows/deploy.yml`):**

```yaml
name: Deploy Place

on:
  push:
    branches:
      - '**'
  repository_dispatch:
    types: [update_library]

jobs:
  # 1) When a developer pushes directly to this place's repository
  deploy-from-push:
    if: github.event_name == 'push'
    uses: RBEngine/actions/.github/workflows/deploy-template.yml@main
    secrets: inherit

  # 2) When this place is triggered by a central library update (e.g., RBEconomy)
  deploy-from-dispatch:
    if: github.event_name == 'repository_dispatch'
    uses: RBEngine/actions/.github/workflows/deploy-template.yml@main
    with:
      is_dispatch: true
      environment_target: ${{ github.event.client_payload.environment }}
      target_branch: ${{ github.event.client_payload.branch }}
    secrets: inherit
```

*Note: Ensure you have `ROBLOX_API_KEY`, `PROD_UNIVERSE_ID`, `PROD_PLACE_ID`, `TEST_UNIVERSE_ID`, and `TEST_PLACE_ID` configured in your Organization or Repository Secrets.*

---

## 2. Package Update Notifications (Decoupled Listener Model)

Shared package repositories (e.g. `RBClauneck/economy`, `RBNaberius/membership`)
notify the repositories that vendor them as a git submodule. They do **not**
hardcode a list of dependents — consumers subscribe on their own terms. Both
flows emit the same generic `repository_dispatch` event, so a single listener
handles everything:

| Trigger (in the package) | `event_type` | `mode` | Effect in the consumer |
| --- | --- | --- | --- |
| push to `master` | `update_submodule` | `production` | submodule pointer bumped on the consumer's `master` |
| push to any non-`master` branch | `update_submodule` | `testing` | submodule pointer bumped on a `test/package-testing` branch (created if missing) |

**Client payload**

```json
{
  "mode": "production | testing",
  "package": "<package repo name>",
  "submodule_path": "packages/<package>",
  "source_branch": "<branch that was pushed>",
  "source_sha": "<commit to pin the submodule to>"
}
```

### Master updates (broadcast)

On a push to `master` the package broadcasts to every `owner/repo` listed in its
`MASTER_SUBSCRIBERS` Actions variable (*Settings → Secrets and variables →
Actions → Variables*). The subscriber list lives in repo configuration, never in
the workflow, so adding or removing a dependent never touches the package's code.

### Non-master updates (directed)

A push to any other branch dispatches only to the package's dedicated test
repository (a fixed 1:1 partner) to refresh `test/package-testing` for pre-merge
validation.

### Subscribing as a consumer

1. Add a listener workflow (e.g. `.github/workflows/submodule-update.yml`) that
   handles `repository_dispatch: types: [update_submodule]`, resolving the target
   branch from `client_payload.mode` and pinning `submodule_path` to `source_sha`.
2. Ask the package maintainer to add your `owner/repo` to that package's
   `MASTER_SUBSCRIBERS` variable to receive `master` updates.

The package side needs a `DISPATCH_TOKEN` secret (a PAT with Contents:
read/write on the subscriber/test repos); without it the notify steps no-op.
