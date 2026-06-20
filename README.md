# CI/CD Reusable Workflows

This repository contains GitHub Actions Reusable Workflows for building and deploying to Roblox, as well as dispatching events to trigger updates in dependent repositories.

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

## 2. Trigger Template (`trigger-template.yml`)

This workflow is used by shared library packages (e.g., Economy, Pet systems) to dispatch update signals to all dependent game places.  
It parses a JSON array of repositories and triggers them to pull the latest library version.

**Example usage in your Library repository (e.g., `RBEconomy/.github/workflows/deploy.yml`):**

```yaml
name: Trigger Dependent Places

on:
  push:
    branches:
      - '**'

jobs:
  trigger:
    uses: RBEngine/actions/.github/workflows/trigger-template.yml@main
    with:
      # Specify the target repositories as a JSON array
      target_repos: '["MyGame-arena", "MyGame-tournament", "MyGame-lobby"]'
    secrets:
      # Requires a Personal Access Token (PAT) with access to trigger the target repositories
      PAT_TOKEN: ${{ secrets.PERSONAL_ACCESS_TOKEN }}
```
