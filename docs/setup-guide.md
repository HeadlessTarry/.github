# 📖 Setup Guide: Adding Auto-Merge to a Repository

This guide walks through adding Dependabot auto-merge to a repository in the `HeadlessTarry` organization.

## 📋 Prerequisites

- Repository must be in the `HeadlessTarry` organization
- You must have admin access to the repository
- The repository must have CI checks defined in a GitHub Actions workflow
- The `headlesstarry-auto-prs` GitHub App must be installed on the repository (see [Step 2](#step-2-install-the-github-app))

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
        permissions: {}
        secrets:
          AUTO_PRS_PRIVATE_KEY: ${{ secrets.AUTO_PRS_PRIVATE_KEY }}
    ```

To find the latest commit SHA, visit [HeadlessTarry/.github](https://github.com/HeadlessTarry/.github) and copy the full commit hash from the latest commit on the `main` branch.

## Step 2: Install the GitHub App

The `headlesstarry-auto-prs` GitHub App is required for approving Dependabot PRs and enabling auto-merge. GitHub's built-in `GITHUB_TOKEN` cannot approve PRs on the same repository.

1. Go to **HeadlessTarry org Settings** → **GitHub Apps** → **headlesstarry-auto-prs**
2. Click **Install App**
3. Select the repository (or all repositories)
4. Grant the requested permissions

## Step 3: Add repository secrets

Add the following secrets to your repository (**Settings** → **Secrets and variables** → **Actions** → **New repository secret**):

| Secret name | Value |
|---|---|
| `AUTO_PRS_PRIVATE_KEY` | The PEM private key for the `headlesstarry-auto-prs` GitHub App |

Also add the following **variable** (**Settings** → **Secrets and variables** → **Actions** → **Variables** tab → **New repository variable**):

| Variable name | Value |
|---|---|
| `AUTO_PRS_CLIENT_ID` | The Client ID of the `headlesstarry-auto-prs` GitHub App (starts with `Iv`) |

> **Note:** If you're retrofitting an org that has multiple consumer repos, these can also be configured as **organization secrets/variables** (accessible to all repos) rather than per-repo. The reusable workflow resolves them from the calling repository's context.

## Step 4: Enable auto-merge in repository settings

1. Go to your repository on GitHub
2. Navigate to **Settings** → **General**
3. Scroll to **Pull Requests**
4. Check **Allow auto-merge**
5. Set the default merge method to **Squash merge**

## Step 5: Configure repository rulesets

The `HeadlessTarry` organization has an org-level ruleset called "Pull Requests are Mandatory" that requires pull requests for changes to `main`. This ruleset must be configured correctly for auto-merge to work.

### Required configuration

1. Go to **Settings** → **Rules** → **Rulesets** (or **Organization Settings** → **Rules** → **Rulesets** for org-level rules)
2. Find the "Pull Requests are Mandatory" ruleset and edit it
3. Ensure the following rules are enabled:
   - **Require a pull request before merging** — requires all changes to go through a PR
   - **Require review from code owners** — optional, if you use CODEOWNERS
4. Ensure the **Restrict updates** rule is **NOT** enabled — see below for details

### Bypass configuration

Add the `headlesstarry-auto-prs` GitHub App to the bypass list with mode **Always allow**. This is required so the app can:

- Approve PRs (bypasses "require review" rules)
- Enable auto-merge (bypasses rules that would otherwise block the merge)

Other recommended bypass actors:

- **OrganizationAdmin** → **Always allow** (so org admins can always push in emergencies)

### ⚠️ Do NOT enable "Restrict updates"

If you enable **Restrict updates**, auto-merge will be blocked even if the GitHub App has bypass permissions. The error message will be: *"Merging is blocked: Cannot update this protected ref."*

This is because GitHub's auto-merge system uses an internal service to perform the merge, not the GitHub App's identity. The internal service does not inherit bypass permissions from the app that enabled auto-merge.

The **Require a pull request before merging** rule is sufficient to prevent direct pushes to `main`.

### Required status checks

Also ensure you have a ruleset (or branch protection rule) requiring status checks to pass:

1. Go to **Settings** → **Rules** → **Rulesets**
2. Create or edit a ruleset targeting `main` with **Require status checks to pass before merging** enabled
3. Add the required check names (e.g., `📦 Build Package`, or whatever your CI workflow defines)

### Identifying required check names

Required check names must match your CI job names exactly. To find them:

1. Push a commit to `main` or open a PR
2. Wait for CI to run
3. In the ruleset editor's **Require status checks** section, the search box will suggest job names as you type

## Step 6: Verify the setup

1. Wait for Dependabot to create a PR (or trigger one manually by updating a dependency)
2. Check that the **Auto-Merge** workflow runs on the PR
3. Verify the PR shows an approval from `headlesstarry-auto-prs[bot]`
4. Verify the PR shows "Auto-merge enabled" with the squash merge strategy
5. Confirm the PR is merged automatically after all required checks pass

## 🔧 Troubleshooting

### "GitHub Actions is not permitted to approve pull requests"

This error means the `GITHUB_TOKEN` is being used for the approval step instead of the GitHub App token. Ensure:

- The reusable workflow references `${{ secrets.AUTO_PRS_PRIVATE_KEY }}` (not `GITHUB_TOKEN`) for the approval step
- The stub workflow passes the secret through: `secrets: AUTO_PRS_PRIVATE_KEY: ${{ secrets.AUTO_PRS_PRIVATE_KEY }}`

### "The 'private-key' input must be set to a non-empty string"

The `AUTO_PRS_PRIVATE_KEY` secret is not accessible to the workflow. For reusable workflows, secrets must be explicitly declared and passed through — they are not inherited automatically. Ensure:

- The reusable workflow declares `secrets: AUTO_PRS_PRIVATE_KEY: required: true` in `workflow_call`
- The stub workflow passes the secret: `secrets: AUTO_PRS_PRIVATE_KEY: ${{ secrets.AUTO_PRS_PRIVATE_KEY }}`
- The secret exists as an **organization secret** or **repository secret** on the consumer repo

### "At least 1 approving review is required by reviewers with write access"

The GitHub App needs both `pull-requests: write` **and** `contents: write` permissions in its GitHub App settings. The `contents: write` permission is required for GitHub to consider the approval as coming from a "reviewer with write access."

After updating the app's permissions, you must re-approve the app installation on the repository for the new permissions to take effect.

### "Cannot update this protected ref"

The **Restrict updates** rule is enabled on the ruleset targeting `main`. This rule is incompatible with GitHub's native auto-merge. Disable it — the **Require a pull request before merging** rule is sufficient to prevent direct pushes.

### Auto-merge workflow skipped

Ensure the PR author is `dependabot[bot]`. The workflow only runs for Dependabot-authored PRs (enforced by `if: github.actor == 'dependabot[bot]'`).

### PR not merging even with auto-merge and all checks passing

Verify that:

- The GitHub App has bypass permissions set to **Always allow** (not "Allow for pull request only") on all applicable rulesets
- **Restrict updates** is not enabled on any ruleset targeting `main`
- **Allow auto-merge** is enabled in repository settings

### New version of the reusable workflow

Dependabot will automatically create a PR to bump the SHA in your stub workflow. That PR will go through the auto-merge system itself, using the old pinned version. Once merged, future PRs will use the new version.

Because Dependabot PRs come from a fork, the workflow uses `pull_request_target` (not `pull_request`) so it runs in the context of the base repository and has access to secrets.
