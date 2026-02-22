# Reusable Workflows Library

A comprehensive collection of GitHub Actions reusable workflows for CI/CD, security, infrastructure, and developer experience.

All workflows live in `.github/workflows/` and are invoked via `workflow_call`.

---

## Available Workflows

| Workflow | File | Description |
| --- | --- | --- |
| [CI](#ci) | `ci.yml` | Node.js CI pipeline: install, lint, test, coverage |
| [Docker Build & Push](#docker-build--push) | `docker-build-push.yml` | Multi-platform Docker image build and registry push |
| [Deploy](#deploy) | `deploy.yml` | Application deployment with GitHub Deployments API |
| [Terraform](#terraform) | `terraform.yml` | Terraform plan / apply / destroy with multi-cloud support |
| [Release](#release) | `release.yml` | Semantic versioning, changelog, GitHub Release, npm publish |
| [Security Scan](#security-scan) | `security-scan.yml` | Dependency audit, CodeQL, container scanning, SARIF upload |
| [Notify](#notify) | `notify.yml` | Notifications to Slack, Teams, and Discord |
| [PR Checks](#pr-checks) | `pr-checks.yml` | PR validation: labels, conventional commits, size, reviewers |

---

## CI

**File:** `.github/workflows/ci.yml`

Runs a Node.js CI pipeline with optional lint and test steps. Outputs test results and coverage percentage.

### Inputs

| Name | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `node-version` | string | no | `'20'` | Node.js version |
| `working-directory` | string | no | `'./'` | Working directory for all commands |
| `run-lint` | boolean | no | `true` | Run lint step |
| `run-tests` | boolean | no | `true` | Run test step |

### Outputs

| Name | Description |
| --- | --- |
| `test-result` | Result of the test run (`success`/`failure`/`skipped`) |
| `coverage-percentage` | Code coverage percentage |

### Secrets

| Name | Required | Description |
| --- | --- | --- |
| `npm-token` | no | NPM auth token for private packages |

### Example

```yaml
jobs:
  ci:
    uses: myorg/reusable-workflows/.github/workflows/ci.yml@main
    with:
      node-version: '20'
      run-lint: true
      run-tests: true
    secrets:
      npm-token: ${{ secrets.NPM_TOKEN }}
```

---

## Docker Build & Push

**File:** `.github/workflows/docker-build-push.yml`

Builds Docker images with multi-platform support (via QEMU) and optionally pushes to a container registry.

### Inputs

| Name | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `image-name` | string | **yes** | — | Full image name incl. registry |
| `dockerfile` | string | no | `'Dockerfile'` | Path to Dockerfile |
| `context` | string | no | `'.'` | Docker build context |
| `push` | boolean | no | `false` | Push to registry |
| `platforms` | string | no | `'linux/amd64'` | Target platforms (comma-separated) |
| `build-args` | string | no | `''` | Build arguments (newline-separated) |
| `tags` | string | no | `''` | Image tags (comma-separated) |

### Outputs

| Name | Description |
| --- | --- |
| `image-digest` | Image digest (`sha256:...`) |
| `image-tags` | Built/pushed image tags |

### Secrets

| Name | Required | Description |
| --- | --- | --- |
| `registry-username` | no | Registry username |
| `registry-password` | no | Registry password/token |

### Example

```yaml
jobs:
  docker:
    uses: myorg/reusable-workflows/.github/workflows/docker-build-push.yml@main
    with:
      image-name: 'ghcr.io/myorg/myapp'
      push: true
      platforms: 'linux/amd64,linux/arm64'
      tags: 'latest,v1.2.3'
    secrets:
      registry-username: ${{ secrets.GHCR_USER }}
      registry-password: ${{ secrets.GHCR_TOKEN }}
```

---

## Deploy

**File:** `.github/workflows/deploy.yml`

Deploys an application to a target environment, tracking status via the GitHub Deployments API.

### Inputs

| Name | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `environment` | string | **yes** | — | Target environment |
| `version` | string | **yes** | — | Version/tag to deploy |
| `dry-run` | boolean | no | `false` | Simulate without changes |
| `app-name` | string | **yes** | — | Application name |

### Outputs

| Name | Description |
| --- | --- |
| `deployment-url` | URL of the deployed application |
| `deployment-id` | GitHub Deployment ID |

### Secrets

| Name | Required | Description |
| --- | --- | --- |
| `deploy-token` | **yes** | Deployment auth token |
| `cloud-credentials` | no | Cloud provider credentials |

### Example

```yaml
jobs:
  deploy:
    uses: myorg/reusable-workflows/.github/workflows/deploy.yml@main
    with:
      environment: 'production'
      version: 'v1.2.3'
      app-name: 'my-api'
    secrets:
      deploy-token: ${{ secrets.DEPLOY_TOKEN }}
      cloud-credentials: ${{ secrets.CLOUD_CREDENTIALS }}
```

---

## Terraform

**File:** `.github/workflows/terraform.yml`

Runs Terraform operations (plan, apply, destroy) with auto-detected cloud credential setup for AWS, GCP, and Azure.

### Inputs

| Name | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `working-directory` | string | no | `'.'` | Terraform config directory |
| `terraform-version` | string | no | `'1.6'` | Terraform CLI version |
| `command` | string | **yes** | — | `plan`, `apply`, or `destroy` |
| `environment` | string | **yes** | — | Target environment |
| `auto-approve` | boolean | no | `false` | Skip interactive approval |

### Outputs

| Name | Description |
| --- | --- |
| `plan-output` | Terraform plan text output |
| `apply-output` | Terraform apply/destroy output |

### Secrets

| Name | Required | Description |
| --- | --- | --- |
| `cloud-credentials` | **yes** | Cloud provider credentials (JSON) |

### Example

```yaml
jobs:
  plan:
    uses: myorg/reusable-workflows/.github/workflows/terraform.yml@main
    with:
      working-directory: './infra/staging'
      command: 'plan'
      environment: 'staging'
    secrets:
      cloud-credentials: ${{ secrets.AWS_CREDENTIALS }}

  apply:
    needs: [plan]
    uses: myorg/reusable-workflows/.github/workflows/terraform.yml@main
    with:
      working-directory: './infra/staging'
      command: 'apply'
      environment: 'staging'
      auto-approve: true
    secrets:
      cloud-credentials: ${{ secrets.AWS_CREDENTIALS }}
```

---

## Release

**File:** `.github/workflows/release.yml`

Automates semantic version bumps, changelog generation, Git tagging, GitHub Release creation, and optional npm publishing.

### Inputs

| Name | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `version-bump` | string | **yes** | — | `patch`, `minor`, or `major` |
| `prerelease` | boolean | no | `false` | Mark as prerelease |
| `generate-changelog` | boolean | no | `true` | Auto-generate changelog |

### Outputs

| Name | Description |
| --- | --- |
| `new-version` | New version string |
| `release-url` | GitHub Release URL |
| `changelog` | Generated changelog content |

### Secrets

| Name | Required | Description |
| --- | --- | --- |
| `npm-token` | no | NPM token (omit to skip npm publish) |

### Example

```yaml
jobs:
  release:
    uses: myorg/reusable-workflows/.github/workflows/release.yml@main
    with:
      version-bump: 'minor'
      prerelease: false
    secrets:
      npm-token: ${{ secrets.NPM_TOKEN }}
    permissions:
      contents: write
```

---

## Security Scan

**File:** `.github/workflows/security-scan.yml`

Runs dependency audits, CodeQL analysis, container scanning (Trivy), and uploads SARIF results.

### Inputs

| Name | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `scan-type` | string | no | `'full'` | `full` or `quick` |
| `fail-on-severity` | string | no | `'high'` | `critical`, `high`, `medium`, or `low` |

### Outputs

| Name | Description |
| --- | --- |
| `vulnerabilities-found` | Total vulnerabilities found |
| `report-url` | Link to GitHub Security tab |

### Example

```yaml
jobs:
  security:
    uses: myorg/reusable-workflows/.github/workflows/security-scan.yml@main
    with:
      scan-type: 'full'
      fail-on-severity: 'high'
    permissions:
      security-events: write
      contents: read
```

---

## Notify

**File:** `.github/workflows/notify.yml`

Sends pipeline status notifications to Slack, Microsoft Teams, and/or Discord.

### Inputs

| Name | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `status` | string | **yes** | — | Pipeline status (`success`, `failure`, etc.) |
| `environment` | string | no | `''` | Environment name for context |
| `message` | string | no | `''` | Custom message body |
| `channels` | string | no | `'slack'` | Channels: `slack`, `teams`, `discord` (comma-separated) |

### Secrets

| Name | Required | Description |
| --- | --- | --- |
| `slack-webhook` | no | Slack Incoming Webhook URL |
| `teams-webhook` | no | Teams Incoming Webhook URL |
| `discord-webhook` | no | Discord Webhook URL |

### Example

```yaml
jobs:
  notify:
    if: always()
    needs: [deploy]
    uses: myorg/reusable-workflows/.github/workflows/notify.yml@main
    with:
      status: ${{ needs.deploy.result }}
      environment: 'production'
      message: 'v1.2.3 deployed successfully'
      channels: 'slack,discord'
    secrets:
      slack-webhook: ${{ secrets.SLACK_WEBHOOK }}
      discord-webhook: ${{ secrets.DISCORD_WEBHOOK }}
```

---

## PR Checks

**File:** `.github/workflows/pr-checks.yml`

Validates pull requests: label requirements, conventional commit title format, PR size warnings, and auto-assigns reviewers.

### Inputs

| Name | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `require-labels` | boolean | no | `true` | Require labels |
| `required-label-pattern` | string | no | `''` | Regex for required label |
| `check-conventional-commits` | boolean | no | `true` | Validate PR title format |
| `max-file-changes` | number | no | `50` | Max changed files before warning |

### Outputs

| Name | Description |
| --- | --- |
| `validation-result` | `pass` or `fail` |

### Example

```yaml
on:
  pull_request:
    types: [opened, synchronize, labeled, unlabeled, edited]

jobs:
  pr-checks:
    uses: myorg/reusable-workflows/.github/workflows/pr-checks.yml@main
    with:
      require-labels: true
      required-label-pattern: '^(bug|feature|docs|chore)'
      check-conventional-commits: true
      max-file-changes: 50
    permissions:
      pull-requests: write
      contents: read
```

---

## Full Pipeline Example

Combine multiple reusable workflows into a complete CI/CD pipeline:

```yaml
name: Full Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  # 1. Validate PRs
  pr-checks:
    if: github.event_name == 'pull_request'
    uses: myorg/reusable-workflows/.github/workflows/pr-checks.yml@main
    with:
      check-conventional-commits: true

  # 2. Run CI
  ci:
    uses: myorg/reusable-workflows/.github/workflows/ci.yml@main
    with:
      node-version: '20'
    secrets:
      npm-token: ${{ secrets.NPM_TOKEN }}

  # 3. Security scan
  security:
    uses: myorg/reusable-workflows/.github/workflows/security-scan.yml@main
    with:
      scan-type: 'full'
    permissions:
      security-events: write
      contents: read

  # 4. Build Docker image
  docker:
    needs: [ci, security]
    uses: myorg/reusable-workflows/.github/workflows/docker-build-push.yml@main
    with:
      image-name: 'ghcr.io/myorg/myapp'
      push: ${{ github.ref == 'refs/heads/main' }}
      platforms: 'linux/amd64,linux/arm64'
    secrets:
      registry-username: ${{ github.actor }}
      registry-password: ${{ secrets.GITHUB_TOKEN }}

  # 5. Deploy to staging
  deploy-staging:
    if: github.ref == 'refs/heads/main'
    needs: [docker]
    uses: myorg/reusable-workflows/.github/workflows/deploy.yml@main
    with:
      environment: 'staging'
      version: ${{ needs.docker.outputs.image-digest }}
      app-name: 'myapp'
    secrets:
      deploy-token: ${{ secrets.DEPLOY_TOKEN }}

  # 6. Notify
  notify:
    if: always()
    needs: [ci, docker, deploy-staging]
    uses: myorg/reusable-workflows/.github/workflows/notify.yml@main
    with:
      status: ${{ needs.deploy-staging.result || needs.docker.result }}
      environment: 'staging'
      channels: 'slack'
    secrets:
      slack-webhook: ${{ secrets.SLACK_WEBHOOK }}
```

---

## Contributing

1. All workflows must use `on: workflow_call` with typed inputs, outputs, and secrets.
2. Pin third-party actions to major version tags (e.g. `@v4`).
3. Include a header comment with usage example in every workflow file.
4. Set minimal `permissions` at job or workflow level.
5. Test changes in a feature branch before merging to `main`.
