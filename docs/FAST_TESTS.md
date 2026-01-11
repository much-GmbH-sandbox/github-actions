# Unit Tests & Deploy (Jenkins)

Unit tests run in **Jenkins**, not GitHub Actions. Tests are triggered automatically when the workflow runs.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           GitHub Actions                                 │
│                                                                          │
│   ┌──────────────┐         ┌──────────────────────────────────────┐     │
│   │   Quality    │         │         Jenkins Trigger              │     │
│   │   Checks     │────────►│  Always runs tests                   │     │
│   │  (CI.md)     │  needs  │  Deploy if: push + deploy branch     │     │
│   └──────────────┘         │            + build_docker=true       │     │
│                            └──────────────┬─────────────────────────┘     │
└───────────────────────────────────────────┼─────────────────────────────┘
                                            │
                                            ▼
                              ┌─────────────────────────┐
                              │        Jenkins          │
                              │  - Runs unit tests      │
                              │  - Builds Docker image  │
                              └─────────────────────────┘
```

## Behavior

| Event | Condition | Result |
|-------|-----------|--------|
| PR | To any configured branch | **Unit Tests** |
| Push | To deploy branch, `build_docker=true` | **Unit Tests + Docker Build** |
| Push | To deploy branch, `build_docker=false` | **Unit Tests** |
| Push | To non-deploy branch | **Unit Tests** |

**Key points:**
- Tests **always run** when the workflow triggers
- Deploy only happens on push to deploy branches with `build_docker=true`
- Branch configuration is done in your repo's `ci.yml` using standard GitHub Actions syntax

## Configuration

### Your ci.yml (controls which branches trigger)

```yaml
name: CI/CD

on:
  push:
    branches: [main, production]     # These trigger tests (+ deploy if configured)
  pull_request:
    branches: [main, develop]        # PRs to these branches trigger tests
  workflow_dispatch:

jobs:
  quality:
    uses: YOUR_ORG/github-actions/.github/workflows/odoo-quality.yml@main
    with:
      odoo-version: '17'
    secrets: inherit

  jenkins:
    needs: quality
    uses: YOUR_ORG/github-actions/.github/workflows/jenkins-trigger.yml@main
    with:
      deploy-branches: 'main,production'  # Which branches deploy Docker
    secrets: inherit
```

### pyproject.toml (controls Docker build)

```toml
[odoo]
version = 17.0
build_docker = true      # Enable Docker build on deploy branches
edition = "enterprise"
git_hosts = ""
```

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `build_docker` | boolean | `false` | Build Docker image on deploy branches |
| `version` | float | - | Odoo version (e.g., `17.0`) |
| `edition` | string | `enterprise` | Odoo edition |
| `git_hosts` | string | `""` | Additional git hosts for private dependencies |

### Workflow Input

| Input | Type | Default | Description |
|-------|------|---------|-------------|
| `deploy-branches` | string | `main` | Comma-separated branches that trigger Docker build |
| `dry-run` | boolean | `false` | Log without triggering Jenkins |

## Jenkins Payload

When triggered, Jenkins receives:

```json
{
  "repository": "your-org/your-repo",
  "branch": "main",
  "commit": "abc123def456...",
  "odoo_version": "17",
  "odoo_edition": "enterprise",
  "git_hosts": "",
  "deploy": true,
  "build_docker": true,
  "github_run_id": "123456789",
  "github_run_url": "https://github.com/...",
  "event": "push",
  "pr_number": ""
}
```

## Secrets Required

| Secret | Required | Description |
|--------|----------|-------------|
| `JENKINS_URL` | Yes | Jenkins server URL |
| `JENKINS_USER` | Yes | Jenkins username with API access |
| `JENKINS_TOKEN` | Yes | Jenkins API token |

```bash
# Set at organization level
gh secret set JENKINS_URL --org YOUR_ORG --visibility all
gh secret set JENKINS_USER --org YOUR_ORG --visibility all
gh secret set JENKINS_TOKEN --org YOUR_ORG --visibility all
```

If `JENKINS_URL` is not set, the workflow runs in **dry-run mode**.

## Common Scenarios

### 1. Tests on Every PR, Deploy on Main

```yaml
# ci.yml
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
```

```toml
# pyproject.toml
[odoo]
version = 17.0
build_docker = true
```

- PRs to main → Tests
- Push to main → Tests + Docker Build

### 2. Tests Only (No Deploy)

```yaml
# ci.yml
on:
  push:
    branches: [main]
  pull_request:
    branches: [main, develop]
```

```toml
# pyproject.toml
[odoo]
version = 17.0
build_docker = false
```

- PRs to main/develop → Tests
- Push to main → Tests only

### 3. Multiple Deploy Branches

```yaml
# ci.yml
on:
  push:
    branches: [main, staging, production]
  pull_request:
    branches: [main]

jobs:
  jenkins:
    uses: YOUR_ORG/github-actions/.github/workflows/jenkins-trigger.yml@main
    with:
      deploy-branches: 'main,staging,production'
```

- Push to main/staging/production → Tests + Docker Build

## Troubleshooting

| Issue | Cause | Fix |
|-------|-------|-----|
| Jenkins in dry-run mode | `JENKINS_URL` not set | Set Jenkins secrets at org level |
| Jenkins HTTP 401 | Invalid credentials | Check `JENKINS_USER` and `JENKINS_TOKEN` |
| Jenkins HTTP 403 | User lacks permissions | Grant build permissions in Jenkins |
| Docker not building | Not a deploy branch | Add branch to `deploy-branches` input |
| Docker not building | `build_docker = false` | Set `build_docker = true` in pyproject.toml |
| Workflow not triggering | Branch not in `on:` | Add branch to push/pull_request triggers |
