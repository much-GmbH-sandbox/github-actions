# Odoo CI/CD Setup Guide

## Prerequisites

### Organization Level

| Requirement | Details |
|-------------|---------|
| GitHub Plan | Team or higher |
| Organization Secrets | `CONFIG_TOKEN` - PAT with `repo` scope (for private central repo) |
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
│       └── odoo-quality.yml      # Reusable workflow
├── configs/
│   ├── .flake8                   # Flake8 config
│   ├── .bandit                   # Bandit config
│   └── .pylintrc-odoo{14-18}     # Pylint configs per Odoo version
├── dev-requirements/
│   ├── base.txt                  # Common tools (black, flake8, radon, bandit)
│   └── odoo{14-18}.txt           # Version-specific pylint-odoo
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

```yaml
name: CI

on:
  push:
    branches: [main]        # Adjust to your branches
  pull_request:
    branches: [main]
  workflow_dispatch:

concurrency:
  group: ${{ github.workflow }}-${{ github.event.pull_request.number || github.ref }}
  cancel-in-progress: true

permissions:
  contents: read
  pull-requests: write      # Required for PR comments

jobs:
  quality:
    uses: YOUR_ORG/github-actions/.github/workflows/odoo-quality.yml@main
    with:
      odoo-version: '17'    # 14, 15, 16, 17, or 18
      python-version: '3.10'
      # skip-sonar: true    # Optional: skip SonarQube
      # strict-mode: true   # Optional: fail on any check failure
    secrets: inherit
```

### Configuration Options

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `odoo-version` | Yes | - | Odoo version: 14, 15, 16, 17, 18 |
| `python-version` | No | 3.10 | Python version |
| `skip-sonar` | No | false | Skip SonarQube analysis |
| `strict-mode` | No | false | Fail workflow if any check fails |

### What Happens

1. **On Push/PR**: Workflow runs automatically
2. **Quality Checks**: Black, Flake8, Pylint-Odoo, Radon, Bandit
3. **PR Comment**: Summary table posted showing pass/fail per tool
4. **Status Check**: Shows on PR (green even if checks fail, unless `strict-mode: true`)

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

## Quick Start Checklist

### One-Time Setup (Organization)

- [ ] Create PAT with `repo` scope
- [ ] Set `CONFIG_TOKEN` org secret
- [ ] Set `access_level=organization` on central repo

### Per-Repository Setup

- [ ] Create `.github/workflows/ci.yml`
- [ ] Set correct `odoo-version`
- [ ] Adjust branch triggers as needed
- [ ] (Optional) Enable `strict-mode` for blocking checks

---

## Troubleshooting

| Error | Cause | Fix |
|-------|-------|-----|
| `startup_failure` | Missing permissions | Add `permissions` block to ci.yml |
| `HTTP 404` on config download | CONFIG_TOKEN missing/invalid | Check org secret |
| `Resource not accessible` | Missing PR write permission | Add `pull-requests: write` |
| Workflow not triggered | access_level not set | Run `gh api` command above |
