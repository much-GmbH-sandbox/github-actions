# GitHub Actions - Odoo Quality Workflows

Central repository for reusable GitHub Actions workflows for Odoo addon development.

**Organization**: much-GmbH-sandbox (sandbox for testing)

## Getting Started

### New Repository (Recommended)

Use [base-repo](https://github.com/much-GmbH-sandbox/base-repo) as a template:
1. Click "Use this template" on the base-repo page
2. Create your new repository
3. Update `pyproject.toml` with your Odoo version
4. Copy the appropriate `.pylintrc` for your version

base-repo includes:
- CI workflow using these centralized workflows
- All linter configs (`.flake8`, `.bandit`, `.pylintrc`)
- Pre-commit hooks configuration
- Editor and Git configs

### Existing Repository

Add the CI workflow to `.github/workflows/ci.yml`:

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
    uses: much-GmbH-sandbox/github-actions/.github/workflows/odoo-quality.yml@v1
    with:
      odoo-version: '17'  # Change to your Odoo version
    secrets: inherit
```

Then copy config files from [base-repo](https://github.com/much-GmbH-sandbox/base-repo).

## Workflows

### `odoo-quality.yml` - Quality Checks

Consolidated quality checks running all tools in a single job (cost-optimized).

**Tools included**:
| Tool | Purpose |
|------|---------|
| Dependencies | Check for unreleased dependencies |
| Black | Code formatting |
| Flake8 | Python linting |
| Pylint-Odoo | Odoo-specific linting |
| Radon | Cyclomatic complexity |
| Bandit | Security scanning |
| SonarQube | Code quality analysis (optional) |

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

## Repository Structure

```
github-actions/
├── .github/
│   ├── workflows/
│   │   ├── odoo-quality.yml      # Quality checks workflow
│   │   └── jenkins-trigger.yml   # Jenkins integration
│   └── docs/
│       └── PRE_COMMIT.md         # Pre-commit setup guide
├── dev-requirements/
│   ├── base.txt                  # Core tools
│   └── odoo{14-18}.txt           # Version-specific tools
├── workflow-templates/
│   └── odoo-addon-ci.yml         # GitHub workflow starter template
└── README.md
```

**Note**: Linter configs (`.flake8`, `.bandit`, `.pylintrc`) are in [base-repo](https://github.com/much-GmbH-sandbox/base-repo).

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

```bash
gh secret set SONAR_TOKEN --org much-GmbH-sandbox --visibility private
gh secret set SONAR_HOST_URL --org much-GmbH-sandbox --visibility private
```

### Repository-Level

```bash
gh secret set SONAR_TOKEN -R much-GmbH-sandbox/my-addon
```

## Versioning

Use version tags for stability. **Never use `@main` in production**.

```yaml
# Major version (recommended)
uses: much-GmbH-sandbox/github-actions/.github/workflows/odoo-quality.yml@v1

# Exact version (most stable)
uses: much-GmbH-sandbox/github-actions/.github/workflows/odoo-quality.yml@v1.0.0
```

## Code Formatting

Format your code locally before committing using pre-commit hooks.

```bash
pip install pre-commit
pre-commit install
```

See the [Pre-commit Setup Guide](.github/docs/PRE_COMMIT.md) or [base-repo](https://github.com/much-GmbH-sandbox/base-repo) for config files.

## Related Repositories

- [base-repo](https://github.com/much-GmbH-sandbox/base-repo) - Template with linter configs and CI workflow
