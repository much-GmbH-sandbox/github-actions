# Odoo CI/CD Setup Guide

## Prerequisites

### Organization Level

| Requirement | Details |
|-------------|---------|
| GitHub Plan | Team or higher |
| Organization Secrets | `CONFIG_TOKEN` - PAT with `repo` scope (for private central repo) |
| Jenkins Secrets | `JENKINS_URL`, `JENKINS_USER`, `JENKINS_TOKEN` (for CD pipeline) |
| Optional Secrets | `SONAR_TOKEN`, `SONAR_HOST_URL` (for SonarQube integration) |

### Creating CONFIG_TOKEN

1. Go to https://github.com/settings/tokens/new
2. Set:
   - **Note**: `github-actions-config-reader`
   - **Expiration**: 90 days
   - **Scopes**: `repo` only
3. Create organization secret:
   ```bash
   gh secret set CONFIG_TOKEN --org YOUR_ORG --visibility all
   ```

---

## Central Repository (`github-actions`)

### Structure

```
github-actions/
├── .github/
│   └── workflows/
│       ├── odoo-quality.yml      # Quality checks (CI)
│       └── jenkins-trigger.yml   # Jenkins trigger (CD)
├── configs/
│   ├── .flake8                   # Flake8 config
│   ├── .bandit                   # Bandit config
│   └── .pylintrc-odoo{14-18}     # Pylint configs per Odoo version
├── dev-requirements/
│   ├── base.txt                  # Common tools (black, flake8, radon, bandit)
│   └── odoo{14-18}.txt           # Version-specific pylint-odoo
├── templates/
│   ├── ci.yml                    # Full CI/CD workflow template
│   └── .pre-commit-config.yaml   # Pre-commit hooks template
└── docs/
    └── SETUP.md                  # This file
```

### Repository Settings

```bash
# Enable workflow access for organization repos
gh api repos/YOUR_ORG/github-actions/actions/permissions/access \
  -X PUT -f access_level=organization
```

---

## Consuming Repositories (Odoo Addons)

### Required File: `.github/workflows/ci.yml`

Copy from `templates/ci.yml` and adjust:

```yaml
name: CI/CD

on:
  push:
    branches: [main, staging, production]  # Adjust to your branches
  pull_request:
    branches: [main, staging]              # PRs to these branches
  workflow_dispatch:

concurrency:
  group: ${{ github.workflow }}-${{ github.event.pull_request.number || github.ref }}
  cancel-in-progress: true

permissions:
  contents: read
  pull-requests: write

jobs:
  # Stage 1: Quality Checks (GitHub Actions)
  quality:
    uses: YOUR_ORG/github-actions/.github/workflows/odoo-quality.yml@main
    with:
      odoo-version: '17'    # 14, 15, 16, 17, or 18
      python-version: '3.10'
    secrets: inherit

  # Stage 2: Unit Tests + Deploy (Jenkins)
  # Tests always run; deploy on push to deploy-branches with build_docker=true
  jenkins:
    needs: quality
    uses: YOUR_ORG/github-actions/.github/workflows/jenkins-trigger.yml@main
    with:
      deploy-branches: 'main,production'  # Comma-separated list
    secrets: inherit
```

### Configuration Options

**Quality Workflow:**

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `odoo-version` | Yes | - | Odoo version: 14, 15, 16, 17, 18 |
| `python-version` | No | 3.10 | Python version |
| `skip-sonar` | No | false | Skip SonarQube analysis |
| `strict-mode` | No | false | Fail workflow if any check fails |

**Jenkins Workflow:**

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `deploy-branches` | No | main | Comma-separated branches that trigger Docker deploy (e.g., `'main,production'`) |
| `dry-run` | No | false | Log without triggering Jenkins |

**Note:** Jenkins workflow reads `pyproject.toml` automatically for:
- `build_docker` - controls Docker build on deploy branches (default: false)
- `version`, `edition` - Odoo configuration

**Behavior:** Tests always run. Deploy (Docker build) only on push to deploy branches with `build_docker=true`.

### pyproject.toml Configuration

Each repo must have a `pyproject.toml` with an `[odoo]` section:

```toml
[odoo]
version = 17.0           # Odoo version (required)
build_docker = true      # Enable Docker build on deploy branches (default: false)
edition = "enterprise"   # enterprise or community (default: enterprise)
git_hosts = ""           # Extra git hosts for Docker build (optional)
```

| Setting | Default | Description |
|---------|---------|-------------|
| `version` | - | Odoo version (required) |
| `build_docker` | false | Build/push Docker image on deploy branches |
| `edition` | enterprise | Odoo edition |
| `git_hosts` | "" | Additional git hosts for Docker build |

### What Happens

```
PR opened/updated:
  └─▶ Quality Checks (GHA) ─▶ Unit Tests (Jenkins)

Push to deploy branch (build_docker=true):
  └─▶ Quality Checks (GHA) ─▶ Unit Tests + Docker Build (Jenkins)

Push to deploy branch (build_docker=false):
  └─▶ Quality Checks (GHA) ─▶ Unit Tests (Jenkins)

Push to non-deploy branch:
  └─▶ Quality Checks (GHA) ─▶ Unit Tests (Jenkins)
```

1. **Quality Checks**: Black, Flake8, Pylint-Odoo, Radon, Bandit, SonarQube
2. **PR Comment**: Summary table posted showing pass/fail per tool
3. **Jenkins Trigger**: Tests always run; deploy based on branch + `build_docker` flag

---

## Quality Tools

| Tool | Purpose | Fails On |
|------|---------|----------|
| Black | Code formatting | Unformatted code |
| Flake8 | Linting | Style violations |
| Pylint-Odoo | Odoo-specific checks | Odoo anti-patterns |
| Radon | Complexity | Grade D or E (warning only) |
| Bandit | Security | Security vulnerabilities |
| SonarQube | Deep analysis | Optional, requires setup |

---

## Pre-commit Hooks (Local Development)

Pre-commit runs the same checks locally before each commit, catching issues early.

### Setup (Per Developer)

```bash
# Install pre-commit
pip install pre-commit

# Copy config to your repo (from templates/.pre-commit-config.yaml)
# Then install hooks
pre-commit install
```

### Usage

```bash
# Runs automatically on git commit (only on staged files)
git commit -m "my changes"

# Run manually on all files
pre-commit run --all-files

# Skip hooks temporarily (not recommended)
git commit --no-verify -m "emergency fix"
```

### What It Checks

| Hook | Purpose |
|------|---------|
| Black | Auto-formats code |
| Flake8 | Linting errors |
| Bandit | Security issues |
| trailing-whitespace | Removes trailing spaces |
| end-of-file-fixer | Ensures newline at EOF |
| check-yaml | Validates YAML syntax |
| check-merge-conflict | Catches unresolved conflicts |
| debug-statements | Finds leftover pdb/breakpoint |

---

## Jenkins Integration

GitHub Actions triggers Jenkins for unit tests and Docker builds.

### Required Secrets

```bash
# Set Jenkins secrets at organization level
gh secret set JENKINS_URL --org YOUR_ORG --visibility all
# Example: https://jenkins.yourcompany.com

gh secret set JENKINS_USER --org YOUR_ORG --visibility all
# Jenkins username with API access

gh secret set JENKINS_TOKEN --org YOUR_ORG --visibility all
# Jenkins API token (not password)
```

### Jenkins Requirements

1. **Generic Webhook Trigger Plugin** installed
2. **Pipeline configured** to accept JSON payload with:
   - `repository`, `branch`, `commit`
   - `odoo_version`, `odoo_edition`, `git_hosts`
   - `deploy`, `build_docker`
   - `github_run_id`, `github_run_url`, `event`, `pr_number`

### Payload Example

```json
{
  "repository": "your-org/your-repo",
  "branch": "main",
  "commit": "abc123...",
  "odoo_version": "17",
  "odoo_edition": "enterprise",
  "git_hosts": "",
  "deploy": true,
  "build_docker": true,
  "github_run_id": "123456",
  "github_run_url": "https://github.com/...",
  "event": "push",
  "pr_number": ""
}
```

---

## Quick Start Checklist

### One-Time Setup (Organization)

- [ ] Create PAT with `repo` scope
- [ ] Set `CONFIG_TOKEN` org secret
- [ ] Set `access_level=organization` on central repo
- [ ] Set `JENKINS_URL`, `JENKINS_USER`, `JENKINS_TOKEN` secrets
- [ ] Configure Jenkins Generic Webhook Trigger

### Per-Repository Setup

- [ ] Create `.github/workflows/ci.yml` (copy from templates)
- [ ] Set correct `odoo-version`
- [ ] Adjust branch triggers as needed
- [ ] Copy `.pre-commit-config.yaml` from templates
- [ ] (Optional) Enable `strict-mode` for blocking checks

---

## Troubleshooting

| Error | Cause | Fix |
|-------|-------|-----|
| `startup_failure` | Missing permissions | Add `permissions` block to ci.yml |
| `HTTP 404` on config download | CONFIG_TOKEN missing/invalid | Check org secret |
| `Resource not accessible` | Missing PR write permission | Add `pull-requests: write` |
| Workflow not triggered | access_level not set | Run `gh api` command above |
| Jenkins dry-run mode | JENKINS_URL not set | Set Jenkins secrets |
| Jenkins HTTP 401 | Invalid credentials | Check JENKINS_USER and JENKINS_TOKEN |
| Jenkins HTTP 403 | User lacks permissions | Grant user build permissions in Jenkins |
