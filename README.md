# 🔄 HeadlessTarry Organization Configuration

This repository contains organization-wide GitHub Actions reusable workflows for the `HeadlessTarry` organization.

## Reusable Workflows

### Auto-Merge

`.github/workflows/auto-merge.yml` — Automatically approves and enables auto-merge for Dependabot pull requests. Consumer repos call this workflow via a per-repo stub.

See [docs/setup-guide.md](docs/setup-guide.md) for instructions on adding this workflow to a new repository.

## Security

- Stubs reference this workflow by **full commit SHA**, not a branch name. A compromised `main` branch does not propagate to consumer repos.
- Dependabot automatically PRs SHA bumps when the workflow is updated, and those PRs flow through the auto-merge system itself (using the old pinned version).
- The roster repo has strict branch protection (required reviews + CI on `main`).
