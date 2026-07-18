# Reusable Terraform Workflow

Path: `.github/workflows/terraform.yml`

Runs the standard Terraform lifecycle — `fmt → init → validate → plan → apply/destroy` — as a single reusable workflow that consumer repos call with `uses:`.

## Features

- **Version pinning** — pin both the Terraform version (`terraform-version`) and the workflow ref (`@v1`).
- **Keyless cloud auth** — OIDC for AWS, Azure, and GCP; no long-lived cloud secrets in GitHub.
- **PR plan comments** — plan output posted as a single sticky comment that updates in place.
- **Safe applies** — `apply`/`destroy` gated behind the `action` input *and* an optional GitHub Environment for required-reviewer approval.
- **Concurrency guard** — one run per ref + working-directory + workspace; applies never race.

## Inputs

| Input | Default | Description |
|-------|---------|-------------|
| `working-directory` | `.` | Root Terraform module directory. |
| `terraform-version` | `1.9.8` | Exact version, constraint, or `latest`. |
| `action` | `plan` | `plan`, `apply`, or `destroy`. |
| `backend-config-file` | `""` | File passed via `-backend-config`. |
| `var-file` | `""` | File passed via `-var-file`. |
| `workspace` | `""` | Workspace to select/create (empty = default). |
| `cloud-provider` | `none` | `none`, `aws`, `azure`, or `gcp`. |
| `aws-role-to-assume` | `""` | IAM role ARN for AWS OIDC. |
| `aws-region` | `us-east-1` | AWS region. |
| `environment` | `""` | GitHub Environment (enables required reviewers / env secrets). |
| `runs-on` | `ubuntu-latest` | Runner label. |
| `fail-on-fmt` | `true` | Fail when `terraform fmt -check` finds unformatted files. |
| `comment-on-pr` | `true` | Post plan output to the triggering PR. |

## Secrets (all optional)

| Secret | Purpose |
|--------|---------|
| `azure-client-id` / `azure-tenant-id` / `azure-subscription-id` | Azure OIDC login. |
| `gcp-workload-identity-provider` / `gcp-service-account` | GCP OIDC login. |
| `extra-env` | Newline-separated `KEY=VALUE` pairs exported before Terraform runs (e.g. `TF_VAR_*`). |

## Outputs

| Output | Description |
|--------|-------------|
| `plan-exitcode` | `terraform plan -detailed-exitcode`: `0` = no changes, `2` = changes. |

## Usage

See [`examples/terraform-caller.yml`](../examples/terraform-caller.yml). Minimal PR plan:

```yaml
jobs:
  plan:
    uses: OWNER/ci-workflows/.github/workflows/terraform.yml@v1
    with:
      working-directory: infra
      action: plan
      cloud-provider: aws
      aws-role-to-assume: arn:aws:iam::111122223333:role/gha-terraform-plan
```

### Passing extra environment variables (`extra-env`)

Use the `extra-env` secret to export `KEY=VALUE` pairs (one per line) before Terraform runs — handy for `TF_VAR_*` inputs and provider tokens. Values are masked in logs.

```yaml
jobs:
  plan:
    uses: OWNER/ci-workflows/.github/workflows/terraform.yml@v1
    with:
      working-directory: infra
      action: plan
    secrets:
      extra-env: |
        TF_VAR_db_password=${{ secrets.DB_PASSWORD }}
        TF_VAR_environment=dev
        CLOUDFLARE_API_TOKEN=${{ secrets.CLOUDFLARE_API_TOKEN }}
```

Notes:
- The `|` block scalar keeps each pair on its own line; mix literals with `${{ secrets.* }}` references.
- Only static values belong here — for cloud auth prefer OIDC (`cloud-provider`) over static keys.

## Prerequisites

1. **OIDC trust** — configure a cloud IAM role/identity that trusts GitHub's OIDC provider (`token.actions.githubusercontent.com`) and scope its trust policy to your repo/branch.
2. **Remote state backend** — S3+DynamoDB, azurerm, or GCS, referenced via `backend-config-file`.
3. **Environments** — for `apply`, create a GitHub Environment (e.g. `production`) with required reviewers.
