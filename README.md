# GitHub Actions - Odoo Quality Workflows

Central repository for reusable GitHub Actions workflows for Odoo addon development.

**Organization**: much-GmbH-sandbox (sandbox for testing)

## Quick Start

Add this workflow to your Odoo addon repository at `.github/workflows/ci.yml`:

```yaml
name: CI

on:
  push:
    branches: [main, staging]
  pull_request:

concurrency:
  group: ${{ github.workflow }}-${{ github.event.pull_request.number || github.ref }}
  cancel-in-progress: true

jobs:
  quality:
    uses: much-GmbH-sandbox/github-actions/.github/workflows/odoo-quality.yml@main
    with:
      odoo-version: '17'  # Change to your Odoo version
    secrets: inherit
```

## Workflows

### `odoo-quality.yml` - Quality Checks

Consolidated quality checks running all tools in a single job (cost-optimized).

**Tools included**:
| Tool | Purpose |
|------|---------|
| Black | Code formatting |
| Flake8 | Python linting |
| Pylint-Odoo | Odoo-specific linting |
| Radon | Cyclomatic complexity |
| Bandit | Security scanning |
| SonarQube | Code quality analysis |

**Inputs**:
| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `odoo-version` | Yes | - | Odoo version (14, 15, 16, 17, 18) |
| `python-version` | No | 3.10 | Python version |
| `skip-sonar` | No | false | Skip SonarQube analysis |
| `strict-mode` | No | false | Fail on any check failure |

**Secrets** (optional):
- `SONAR_TOKEN` - SonarQube authentication token
- `SONAR_HOST_URL` - SonarQube server URL

**Example**:
```yaml
jobs:
  quality:
    uses: much-GmbH-sandbox/github-actions/.github/workflows/odoo-quality.yml@main
    with:
      odoo-version: '17'
      python-version: '3.10'
      strict-mode: true
    secrets: inherit
```

### `jenkins-trigger.yml` - Jenkins Integration

Triggers Jenkins Docker build jobs after quality checks pass.

**Inputs**:
| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `job-name` | No | odoo-docker-build | Jenkins job name |
| `branch` | No | current | Branch to build |
| `commit` | No | current | Commit SHA |
| `dry-run` | No | false | Log but don't trigger |

**Secrets** (required for real triggers):
- `JENKINS_URL` - Jenkins server URL
- `JENKINS_USER` - Jenkins API username
- `JENKINS_TOKEN` - Jenkins API token

**Example**:
```yaml
jobs:
  deploy:
    needs: quality
    if: github.ref == 'refs/heads/main'
    uses: much-GmbH-sandbox/github-actions/.github/workflows/jenkins-trigger.yml@main
    with:
      job-name: 'odoo-docker-build'
    secrets: inherit
```

## Repository Structure

```
github-actions/
├── .github/workflows/
│   ├── odoo-quality.yml      # Main quality workflow
│   └── jenkins-trigger.yml   # Jenkins integration
├── configs/
│   ├── .flake8               # Flake8 configuration
│   ├── .bandit               # Bandit security config
│   └── .pylintrc-odoo{14-18} # Version-specific pylint
├── dev-requirements/
│   ├── base.txt              # Core tools
│   └── odoo{14-18}.txt       # Version-specific tools
├── workflow-templates/
│   └── odoo-addon-ci.yml     # Template for new repos
└── README.md
```

## Supported Odoo Versions

| Odoo Version | Python Versions | Pylint-Odoo |
|--------------|-----------------|-------------|
| 14 | 3.6, 3.8 | 7.0.0 |
| 15 | 3.8, 3.10 | 8.0.0 |
| 16 | 3.8, 3.10 | 8.0.0 |
| 17 | 3.10, 3.11 | 9.3.7 |
| 18 | 3.10, 3.11, 3.12 | 9.3.7 |

## Cost Optimization

This workflow uses a **single consolidated job** instead of parallel jobs.

**Why?** GitHub bills each job rounded up to the nearest minute:
- 5 parallel 30-second jobs = **5 minutes billed**
- 1 sequential 3.5-minute job = **4 minutes billed**

This saves **20-40%** on GitHub Actions minutes.

## Secrets Configuration

### Organization-Level (Recommended)

Set secrets at the organization level for all repos:

```bash
gh secret set SONAR_TOKEN --org much-GmbH-sandbox --visibility private
gh secret set SONAR_HOST_URL --org much-GmbH-sandbox --visibility private
```

### Repository-Level

Or set per-repository:

```bash
gh secret set SONAR_TOKEN -R much-GmbH-sandbox/my-addon
gh secret set SONAR_HOST_URL -R much-GmbH-sandbox/my-addon
```

## Path Filtering

Workflows skip documentation-only changes to save minutes:

```yaml
paths-ignore:
  - '**.md'
  - 'docs/**'
  - 'LICENSE'
```

## Local Testing with `act`

Test workflows locally before pushing:

```bash
# Install act
brew install act

# Run quality checks locally
act push --secret-file .secrets

# Dry run
act -n push
```

Create `.secrets` file:
```
SONAR_TOKEN=your_token
SONAR_HOST_URL=https://qa.muchconsulting.dev
```

## Versioning

Use version tags for stability:

```yaml
# Latest (may change)
uses: much-GmbH-sandbox/github-actions/.github/workflows/odoo-quality.yml@main

# Pinned version (recommended for production)
uses: much-GmbH-sandbox/github-actions/.github/workflows/odoo-quality.yml@v1.0.0
```

## Troubleshooting

### Quality checks fail but code is fine

Check which tool failed in the workflow summary. Common issues:
- **Black**: Run `black .` locally to auto-fix formatting
- **Flake8**: Check for unused imports, line length
- **Pylint-Odoo**: Check `__manifest__.py` format

### SonarQube not running

Ensure secrets are configured:
1. `SONAR_TOKEN` must be set at org or repo level
2. `SONAR_HOST_URL` must be accessible from GitHub runners

### Workflow not triggering

Check:
1. Branch matches trigger pattern
2. Path filters aren't excluding your changes
3. Workflow file syntax is valid

## Contributing

1. Create a branch
2. Make changes
3. Test with `act` locally
4. Push and verify in sandbox repos
5. Create PR for review
