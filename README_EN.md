# Repository Sync Hub

[简体中文](./README.md)｜[English](./README_EN.md)

This repository hosts reusable GitHub Actions workflows for automatically mirroring a source repository to a target repository.

## Architecture

```
source-repo/
├── .github/
│   └── workflows/
│       └── sync.yml          # Trigger: thin wrapper (~14 lines)
│           ├── listens to push → main
│           └── calls the hub's sync-core.yml
│
sync-hub/
├── .github/
│   └── workflows/
│       └── sync-core.yml     # Core logic (maintain this one file only)
│           ├── Job 1: Sync code & tags
│           │   ├── remove sync.yml (avoid trigger leakage in target)
│           │   ├── fetch target main
│           │   ├── version check (strict merge-base)
│           │   ├── git push --force-with-lease
│           │   └── git push --tags
│           ├── Job 2: Sync Releases
│           │   ├── list source releases
│           │   ├── skip existing target releases
│           │   ├── create release (metadata + assets)
│           │   └── download & re-upload assets
│           ├── Job 3: Sync Wiki
│           │   ├── clone source wiki
│           │   ├── clone target wiki (init if absent)
│           │   ├── rsync --delete full mirror
│           │   └── git push
│           └── Job 4: Sync Packages (GHCR)
│               ├── query source GHCR packages
│               ├── docker pull source image
│               ├── docker tag → target repo
│               └── docker push
│
target-repo/
├── main                      # clean mirror, no sync.yml
├── Tags                      # identical to source
├── Releases                  # identical to source (with assets)
├── Wiki/                     # identical to source wiki
└── Packages (GHCR)           # mirrored from source
```

### Core Logic

1. **Trigger**: `push` to `main` branch in source repo, or manual `workflow_dispatch`
2. **Call**: Source repo references this hub's Reusable Workflow via `uses`
3. **Cleanup**: Before pushing, remove `.github/workflows/sync.yml` to prevent trigger leakage in target
4. **Version Check**:
   - Target empty → push directly
   - Target identical to source → skip
   - Target strictly behind source → safe push (`--force-with-lease`)
   - Target ahead or diverged → **explicitly abort** to prevent overwrite
5. **Tags Sync**: `git push --tags` to keep tags consistent
6. **Releases Sync**: Copy release metadata via GitHub API, download & re-upload assets; **existing releases are not overwritten**
7. **Wiki Sync**: Full mirror via `rsync --delete`; auto-initialize if target has no wiki
8. **Packages Sync**: Sync GHCR container images via `docker pull/tag/push`; supports both user and org package ownership

### Safety Mechanisms

- **PAT Isolation**: Token is injected via the caller repo's Secret; this hub stores no credentials
- **Overwrite Protection**: `--force-with-lease` + `merge-base` double-check prevents any push that could lose commits
- **Release Deduplication**: Checks if target already has a release with the same tag before creating
- **Fail-Fast**: Jobs run independently; any failure is explicitly reported without blocking others

## File Structure

```
.github/workflows/
└── sync-core.yml          # Core sync logic (maintain this one file only)
```

## Prerequisites

### PAT Permission Requirements

| Sync Content | Required PAT Scopes |
|-------------|---------------------|
| Code + Tags + Wiki + Releases | `repo` |
| Packages (GHCR) | `repo` + `write:packages` + `read:packages` |

> If the repo has no GHCR Packages, `repo` scope alone is sufficient.

## Setup for a New Repository

### Step 1: Configure Secret

In the source repo, go to **Settings → Secrets and variables → Actions** and add:

| Name | Value |
|------|-------|
| `<PAT_SECRET_NAME>` | Your Personal Access Token (scopes see "Prerequisites" above) |

### Step 2: Create the Trigger

Create file: `.github/workflows/sync.yml`

```yaml
# Auto-sync workflow
# Purpose: mirror to target repo when main branch is updated
# Note: this file only lives in the source repo; target repo will not retain it

name: Sync to Target

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  sync:
    uses: <hub-repo>/.github/workflows/sync-core.yml@main
    with:
      target_repo: <target-repo-full-name>
    secrets:
      TARGET_PAT: ${{ secrets.<PAT_SECRET_NAME> }}
```

> `<target-repo-full-name>` format is `owner/repo`, e.g. `someone/my-repo`.

### Step 3: Verify

Push to `main` or manually trigger Actions, then check the logs to confirm successful sync.

## Maintenance Notes

- **Modify sync logic**: Update only `sync-core.yml` in this hub; all caller repos automatically pick up changes
- **Add a new repo**: Copy the thin wrapper above and change `target_repo`
- **Rename target repo**: Update `target_repo` in the source repo's `sync.yml`
- **Rename source repo**: No changes needed; `${{ github.repository }}` auto-detects
- **Exclude a sync type**: If you don't need Releases/Packages/Wiki, the workflow still runs but silently skips when no resources exist, without affecting the overall flow
