# Reusable Language CI Workflows

Four production-grade reusable pipelines share one shape:

```
build-test  ─►  image (build → Trivy scan → push)
```

The `build-test` job is language-specific; the `image` job is identical across all
four. Images are **built and loaded locally, scanned with Trivy, and only pushed
when the scan passes** — a failed scan blocks the push.

| Language | Workflow | Build tool(s) | Test | Sonar mode |
|----------|----------|---------------|------|------------|
| Java | [`java.yml`](../.github/workflows/java.yml) | Maven / Gradle | JUnit + JaCoCo | via build tool (needs bytecode) |
| Python | [`python.yml`](../.github/workflows/python.yml) | pip / Poetry | pytest + coverage.xml | standalone scanner |
| Node.js | [`nodejs.yml`](../.github/workflows/nodejs.yml) | npm / yarn / pnpm | `test` script | standalone scanner |
| PHP | [`php.yml`](../.github/workflows/php.yml) | Composer | PHPUnit + Clover | standalone scanner |

## Pinned action versions (latest majors)

| Action | Version |
|--------|---------|
| actions/checkout | v7 |
| actions/setup-java | v5 |
| actions/setup-python | v6 |
| actions/setup-node | v7 |
| shivammathur/setup-php | v2 |
| actions/cache | v6 |
| actions/upload-artifact | v7 |
| actions/download-artifact | v8 |
| SonarSource/sonarqube-scan-action | v8 |
| aws-actions/configure-aws-credentials | v6 |
| aws-actions/amazon-ecr-login | v2 |
| docker/setup-qemu-action | v4 |
| docker/setup-buildx-action | v4 |
| docker/login-action | v4 |
| docker/metadata-action | v6 |
| docker/build-push-action | v7 |
| aquasecurity/trivy-action | 0.35.0 |

## Common inputs (all four workflows)

| Input | Default | Description |
|-------|---------|-------------|
| `working-directory` | `.` | Project directory. |
| `run-sonar` | `false` | Run SonarQube analysis. |
| `sonar-host-url` | `""` | SonarQube server URL (omit for SonarCloud). |
| `build-image` | `false` | Build a Docker image. |
| `push-image` | `false` | Push to the registry (only after a passing scan). |
| `scan-image` | `true` | Trivy-scan the image. |
| `registry` | `ghcr.io` | Container registry host. |
| `image-name` | repo | Image repository (lowercased). |
| `dockerfile` | `Dockerfile` | Dockerfile path. |
| `docker-context` | `.` | Build context. |
| `platform` | `linux/amd64` | Target platform, single arch (`linux/amd64` or `linux/arm64`). |
| `trivy-severity` | `CRITICAL,HIGH` | Severities that count as findings. |
| `fail-on-scan` | `true` | Fail the pipeline on findings. |
| `environment` | `""` | GitHub Environment for the image job (approvals). |
| `runs-on` | `ubuntu-latest` | Runner label. |

Common secrets: `sonar-token`, `registry-username`, `registry-password`
(registry auth defaults to `github.actor` + `GITHUB_TOKEN` for `ghcr.io`).

### Pushing to Amazon ECR (OIDC role auth)

Set `ecr: true` to authenticate to AWS with a short-lived assumed role instead
of static registry credentials — no long-lived keys in GitHub.

| Input | Default | Description |
|-------|---------|-------------|
| `ecr` | `false` | Use OIDC role auth + ECR login instead of `registry-username/password`. |
| `aws-role-to-assume` | `""` | IAM role ARN to assume (required when `ecr: true`). |
| `aws-region` | `us-east-1` | AWS region of the ECR registry. |
| `ecr-create-repo` | `false` | Create the repo (scan-on-push) on first push. |

When `ecr: true`:

- The registry host is derived automatically from the ECR login — leave
  `registry`/`registry-username`/`registry-password` unset. Set `image-name` to
  the **ECR repository name** (e.g. `myapp`), not a full host.
- The generic `docker/login-action` step is skipped; ECR login handles Docker auth.
- The job already requests `id-token: write`, so OIDC works out of the box.

Prerequisites: an IAM role whose trust policy allows
`token.actions.githubusercontent.com` scoped to your repo/branch, with
`ecr:GetAuthorizationToken` + push permissions (and `ecr:CreateRepository`,
`ecr:DescribeRepositories` if `ecr-create-repo: true`).

```yaml
jobs:
  ci:
    uses: OWNER/ci-workflows/.github/workflows/java.yml@v1
    with:
      build-image: true
      push-image: ${{ github.ref == 'refs/heads/main' }}
      ecr: true
      aws-role-to-assume: arn:aws:iam::111122223333:role/gha-ecr-push
      aws-region: us-east-1
      image-name: myapp
```

Common output: `image-ref` — the fully qualified image reference that was built.

## Language-specific inputs

**Java** — `java-version` (21), `java-distribution` (temurin), `build-tool`
(maven/gradle), `build-command`, `sonar-project-key`.

**Python** — `python-version` (3.12), `package-manager` (pip/poetry),
`install-command`, `test-command`, `run-lint` (ruff).

**Node.js** — `node-version` (20), `package-manager` (npm/yarn/pnpm),
`lint-script` / `test-script` / `build-script` and their `run-*` toggles.

**PHP** — `php-version` (8.3), `coverage-driver` (pcov/xdebug/none),
`php-extensions`, `composer-flags`, `run-static-analysis` +
`static-analysis-command`.

## Design notes

- **Single build, always** — the image is built exactly once for the target
  `platform`, loaded locally, scanned by Trivy, then its tags are `docker push`ed.
  There is no second build.
- **Trivy** uses `ignore-unfixed: true` so you are not blocked on vulns with no
  available fix; flip `fail-on-scan: false` to report without gating.
- **Architecture (single arch)** — set `platform: linux/amd64` **or**
  `platform: linux/arm64` (Graviton). For arm64, prefer a native runner
  (`runs-on: ubuntu-24.04-arm`) — QEMU is only set up automatically when the
  target arch differs from the runner's (cross-build), which is slower.
- **Approvals** — set `environment: production` so the push happens under a
  GitHub Environment with required reviewers.
- **Least privilege** — callers must grant `packages: write`, `id-token: write`,
  and `security-events: write` (for pushing to GHCR, OIDC, and Trivy SARIF).

## Usage

See the per-language examples: [java](../examples/java-caller.yml),
[python](../examples/python-caller.yml), [nodejs](../examples/nodejs-caller.yml),
[php](../examples/php-caller.yml).
