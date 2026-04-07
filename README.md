# Reusable Terraform Docs

A reusable GitHub Actions workflow that automatically generates and commits Terraform module documentation using [terraform-docs](https://terraform-docs.io/).

## Purpose

This workflow runs `terraform-docs` against a specified directory and commits any resulting documentation changes back to the branch. It is designed to be called on pull request events so that documentation is always kept in sync with the Terraform source.

The workflow handles:

- Downloading and verifying the `terraform-docs` binary via SHA256 checksum
- Generating documentation according to a configuration file
- Committing only documentation changes back to the branch
- Retrying the push with a rebase if concurrent workflow runs cause a non-fast-forward conflict
- CI re-triggering on the resulting commit without requiring a PAT (see [CI Re-triggering](#ci-re-triggering))

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `terraform_docs_config_file` | No | `.terraform-docs.yml` | Path to the terraform-docs configuration file |
| `working_directory` | No | `.` | Directory to run terraform-docs in |
| `terraform_docs_version` | No | `v0.20.0` | Version of terraform-docs to install |

## Secrets

| Secret | Required | Description |
|---|---|---|
| `token` | No | GitHub token for checkout and push. Defaults to `GITHUB_TOKEN`. See [CI Re-triggering](#ci-re-triggering). |

## Permissions

The calling workflow must grant these permissions:

| Permission | Reason |
|---|---|
| `contents: write` | Commit and push documentation changes |
| `pull-requests: write` | Required if the token needs to interact with pull requests |

## Calling this workflow

```yaml
name: Terraform Docs

on:
  pull_request:
    types:
      - opened
      - edited
      - synchronize
      - reopened

permissions:
  contents: write
  pull-requests: write

jobs:
  docs:
    uses: ac-on-ac/reusable-terraform-docs/.github/workflows/docs.yml@v1.0.0
```

With optional inputs:

```yaml
jobs:
  docs:
    uses: ac-on-ac/reusable-terraform-docs/.github/workflows/docs.yml@v1.0.0
    with:
      terraform_docs_config_file: .config/terraform-docs.yml
      working_directory: modules/networking
      terraform_docs_version: v0.20.0
    secrets:
      token: ${{ secrets.MY_PAT }}
```

## CI Re-triggering

By default, commits pushed by `github-actions[bot]` using `GITHUB_TOKEN` do not re-trigger GitHub Actions workflows. This would leave required status checks in a **"Waiting for status to be reported"** state after the docs commit.

This workflow works around that by using `http.extraheader` for git credentials (rather than embedding the token in the remote URL). This approach is not detected by GitHub's loop-prevention logic, so downstream workflows re-fire on the resulting commit **without** needing a PAT.

A PAT only needs to be passed via the `token` secret if `GITHUB_TOKEN` lacks sufficient repository access in your specific configuration (e.g. certain private repository setups).

## Input validation

The workflow validates both `working_directory` and `terraform_docs_config_file` before running:

- Both paths must resolve within the repository root — path traversal values (e.g. `../../other-repo`) are rejected
- The config file must exist at the specified path

## Binary verification

The `terraform-docs` tarball is verified against the official `SHA256SUMS` file published alongside each release before extraction. If the checksum does not match, the workflow fails immediately.

## Releases

This repository uses the [reusable-manual-release](https://github.com/ac-on-ac/reusable-manual-release) workflow to create releases. Releases are triggered manually from the **Actions** tab.
