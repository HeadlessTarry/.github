# 📖 Setup Guide: Adding Auto-Merge to a Repository

This guide walks through adding Dependabot auto-merge to a repository in the `HeadlessTarry` organization.

## Prerequisites

- Repository must be in the `HeadlessTarry` organization
- You must have admin access to the repository
- The repository must have CI checks defined in a GitHub Actions workflow

## Step 1: Add the stub workflow

Create `.github/workflows/auto-merge.yml` in your repository with the following content, replacing `<full-commit-sha>` with the latest commit SHA from the roster repo:

```yaml
---
name: Auto-Merge

on:
  pull_request_target:
    types: [opened, synchronize]

jobs:
  auto-merge:
    uses: HeadlessTarry/.github/.github/workflows/auto-merge.yml@<full-commit-sha>
    permissions:
      pull-requests: write
      contents: write
```

To find the latest commit SHA, visit [HeadlessTarry/.github](https://github.com/HeadlessTarry/.github) and copy the full commit hash from the latest commit on the `main` branch.

## Step 2: Enable auto-merge in repository settings

1. Go to your repository on GitHub
2. Navigate to **Settings** → **General**
3. Scroll to **Pull Requests**
4. Check **Allow auto-merge**
5. Set the default merge method to **Squash merge**

## Step 3: Configure branch protection

1. Go to **Settings** → **Branches**
2. Click **Add branch protection rule** for `main`
3. Enable the following:
   - **Require status checks to pass before merging** — tick all CI jobs that must pass
   - **Require merge queue** — enable with merge method **Squash merge**
   - **Require signed commits** — if applicable to your repository

### Identifying required check names

Required check names must match your CI job names exactly. To find them:

1. Push a commit to `main` or open a PR
2. Wait for CI to run
3. In **Settings** → **Branches** → **Branch protection rules** → the search box under **Require status checks**, the job names will appear as suggestions

For example, if your CI workflow defines jobs named `build` and `home-assistant` (as in the HomeAssistant repo), both must be checked.

## Step 4: Verify the setup

1. Wait for Dependabot to create a PR (or trigger one manually by updating a dependency)
2. Check that the **Auto-Merge** workflow runs on the PR
3. Verify the PR shows "Auto-merge enabled" with the squash merge strategy
4. Confirm the PR enters the merge queue after all required checks pass

## Troubleshooting

### Auto-merge workflow skipped

Ensure the PR author is `dependabot[bot]`. The workflow only runs for Dependabot-authored PRs.

### Auto-merge not enabled despite workflow success

Check that **Allow auto-merge** is enabled in repository settings. Also verify the repository hasn't exceeded merge queue limits.

### PR not merging even with auto-merge

Verify that all required status checks are passing. GitHub will only merge when every required check passes.

### New version of the reusable workflow

Dependabot will automatically create a PR to bump the SHA in your stub workflow. That PR will go through the auto-merge system itself, using the old pinned version. Once merged, future PRs will use the new version.
