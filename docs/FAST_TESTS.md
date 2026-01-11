# Fast Tests (Unit Testing via Jenkins)

The `fast_tests` flag controls whether unit tests run for your Odoo module. Tests execute in **Jenkins**, not GitHub Actions.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           GitHub Actions                                 │
│                                                                          │
│   ┌──────────────┐         ┌──────────────────────────────────────┐     │
│   │   Quality    │         │         Jenkins Trigger              │     │
│   │   Checks     │────────►│  Reads pyproject.toml:               │     │
│   │  (CI.md)     │  needs  │    - fast_tests = true/false         │     │
│   └──────────────┘         │    - build_docker = true/false       │     │
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

## Configuration

### pyproject.toml

```toml
[odoo]
version = 17.0
fast_tests = true        # Enable unit tests (default: true)
build_docker = false     # Enable Docker build on deploy branches (default: false)
edition = "enterprise"   # enterprise or community
git_hosts = ""           # Extra git hosts for Docker build (optional)
```

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `fast_tests` | boolean | `true` | Run unit tests via Jenkins |
| `build_docker` | boolean | `false` | Build Docker image on deploy branches |
| `version` | float | - | Odoo version (e.g., `17.0`) |
| `edition` | string | `enterprise` | Odoo edition |
| `git_hosts` | string | `""` | Additional git hosts for private dependencies |

### Workflow Input (ci.yml)

```yaml
jobs:
  quality:
    uses: YOUR_ORG/github-actions/.github/workflows/odoo-quality.yml@main
    with:
      odoo-version: '17'
    secrets: inherit

  jenkins:
    needs: quality                    # <-- Depends on quality passing
    uses: YOUR_ORG/github-actions/.github/workflows/jenkins-trigger.yml@main
    with:
      deploy-branches: 'main,production'  # Branches that trigger Docker deploy
    secrets: inherit
```

| Input | Type | Default | Description |
|-------|------|---------|-------------|
| `deploy-branches` | string | `main` | Comma-separated branches that trigger Docker build |
| `dry-run` | boolean | `false` | Log without triggering Jenkins |

## Execution Flow

### Step 1: Quality Checks (GitHub Actions)

The `quality` job runs first (see [CI.md](CI.md)). Jenkins trigger **waits** for quality to complete.

### Step 2: Read Configuration

Jenkins trigger reads `pyproject.toml` from your repo:

```bash
# Extracts these values:
fast_tests = true/false    # Controls unit tests
build_docker = true/false  # Controls Docker build
version = 17.0             # Odoo version
edition = "enterprise"     # Odoo edition
```

### Step 3: Determine Mode

The workflow decides what to do based on:
- Event type (PR or push)
- Current branch vs `deploy-branches`
- `fast_tests` and `build_docker` flags

### Step 4: Trigger Jenkins (or Skip)

If there's work to do, Jenkins receives a webhook with all configuration.

## Decision Matrix

### Pull Requests

PRs **never deploy**. They only run tests if `fast_tests=true`.

| fast_tests | build_docker | Result |
|------------|--------------|--------|
| `true` | `true` | Unit Tests Only |
| `true` | `false` | Unit Tests Only |
| `false` | `true` | **Jenkins Skipped** |
| `false` | `false` | **Jenkins Skipped** |

### Push to Deploy Branch

A "deploy branch" is any branch listed in `deploy-branches` input (e.g., `main,production`).

| fast_tests | build_docker | Result |
|------------|--------------|--------|
| `true` | `true` | Unit Tests + Docker Build |
| `true` | `false` | Unit Tests Only |
| `false` | `true` | Docker Build Only |
| `false` | `false` | **Jenkins Skipped** |

### Push to Non-Deploy Branch

| fast_tests | build_docker | Result |
|------------|--------------|--------|
| `true` | `true` | Unit Tests Only |
| `true` | `false` | Unit Tests Only |
| `false` | `true` | **Jenkins Skipped** |
| `false` | `false` | **Jenkins Skipped** |

## Jenkins Payload

When triggered, Jenkins receives this JSON payload:

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
  "fast_tests": true,
  "github_run_id": "123456789",
  "github_run_url": "https://github.com/your-org/your-repo/actions/runs/123456789",
  "event": "push",
  "pr_number": ""
}
```

| Field | Description |
|-------|-------------|
| `repository` | Full repo name (org/repo) |
| `branch` | Branch that triggered the workflow |
| `commit` | Full commit SHA |
| `odoo_version` | Odoo version (14, 15, 16, 17, 18) |
| `odoo_edition` | `enterprise` or `community` |
| `git_hosts` | Extra git hosts for Docker build |
| `deploy` | `true` if this is a deploy (Docker build) |
| `build_docker` | `true` if Docker build is enabled |
| `fast_tests` | `true` if unit tests should run |
| `github_run_id` | GitHub Actions run ID |
| `github_run_url` | Link back to GitHub Actions |
| `event` | `push` or `pull_request` |
| `pr_number` | PR number (if applicable) |

## Secrets Required

| Secret | Required | Description |
|--------|----------|-------------|
| `JENKINS_URL` | Yes | Jenkins server URL (e.g., `https://jenkins.example.com`) |
| `JENKINS_USER` | Yes | Jenkins username with API access |
| `JENKINS_TOKEN` | Yes | Jenkins API token (not password) |

### Setting Up Jenkins Secrets

```bash
# Set at organization level
gh secret set JENKINS_URL --org YOUR_ORG --visibility all
gh secret set JENKINS_USER --org YOUR_ORG --visibility all
gh secret set JENKINS_TOKEN --org YOUR_ORG --visibility all
```

If `JENKINS_URL` is not set, the workflow runs in **dry-run mode** (logs what would happen but doesn't trigger Jenkins).

## Jenkins Requirements

Jenkins must have:

1. **Generic Webhook Trigger Plugin** installed
2. **Pipeline** configured to accept the JSON payload above
3. **User** with API access and build permissions

## Workflow Outputs

The Jenkins trigger workflow exposes these outputs:

| Output | Type | Description |
|--------|------|-------------|
| `triggered` | boolean | Whether Jenkins was actually triggered |
| `skipped` | boolean | Whether Jenkins was skipped |
| `odoo-version` | string | Odoo version from pyproject.toml |
| `build-docker` | boolean | Whether Docker build is enabled |
| `fast-tests` | boolean | Whether fast tests are enabled |

## GitHub Actions Summary

Each run produces a summary in the Actions UI:

**When Triggered:**
```
## Jenkins Triggered

**Mode:** Unit Tests + Docker Build

### Configuration from pyproject.toml

| Setting | Value |
|---------|-------|
| Odoo Version | `17` |
| Fast Tests | `true` |
| Build Docker | `true` |
| Edition | `enterprise` |

### Trigger Parameters

| Parameter | Value |
|-----------|-------|
| Repository | `your-org/your-repo` |
| Branch | `main` |
| Commit | `abc123...` |
| Deploy | `true` |

🚀 Jenkins job started!
```

**When Skipped:**
```
## Jenkins Skipped

**Reason:** Skipped (fast_tests=false for PR)

### Configuration from pyproject.toml

| Setting | Value |
|---------|-------|
| Fast Tests | `false` |
| Build Docker | `true` |

ℹ️ No Jenkins job was triggered.
```

## Common Scenarios

### 1. Development Workflow (Tests on Every PR)

```toml
[odoo]
version = 17.0
fast_tests = true
build_docker = false
```

- PRs: Run unit tests
- Push to main: Run unit tests only (no deploy)

### 2. Full CI/CD (Tests + Deploy)

```toml
[odoo]
version = 17.0
fast_tests = true
build_docker = true
```

- PRs: Run unit tests
- Push to main: Run unit tests + Docker build

### 3. Deploy Only (Skip Tests)

```toml
[odoo]
version = 17.0
fast_tests = false
build_docker = true
```

- PRs: Skip Jenkins entirely
- Push to main: Docker build only (no tests)

### 4. Quality Checks Only (No Jenkins)

```toml
[odoo]
version = 17.0
fast_tests = false
build_docker = false
```

- PRs: Skip Jenkins entirely
- Push to main: Skip Jenkins entirely
- Only quality checks (Black, Flake8, etc.) run

## Troubleshooting

| Issue | Cause | Fix |
|-------|-------|-----|
| Jenkins in dry-run mode | `JENKINS_URL` not set | Set Jenkins secrets at org level |
| Jenkins HTTP 401 | Invalid credentials | Check `JENKINS_USER` and `JENKINS_TOKEN` |
| Jenkins HTTP 403 | User lacks permissions | Grant build permissions in Jenkins |
| Tests not running | `fast_tests = false` in pyproject.toml | Set `fast_tests = true` |
| Docker not building | Not a deploy branch | Add branch to `deploy-branches` input |
| Docker not building | `build_docker = false` | Set `build_docker = true` in pyproject.toml |
