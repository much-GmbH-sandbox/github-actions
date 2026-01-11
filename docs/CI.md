# CI Quality Checks

The CI runs **one job** called `odoo-quality.yml` that executes 6 tools sequentially. It runs on **every push and PR** to check code quality.

```
┌─────────────────────────────────────────────────────────────┐
│                    GitHub Actions Runner                     │
│                                                              │
│   1. Black ──► 2. Flake8 ──► 3. Pylint-Odoo ──►             │
│   4. Radon ──► 5. Bandit ──► 6. SonarQube (optional)        │
│                                                              │
│   Output: PR comment + workflow summary                      │
└─────────────────────────────────────────────────────────────┘
```

## Input Parameters

```yaml
uses: YOUR_ORG/github-actions/.github/workflows/odoo-quality.yml@main
with:
  odoo-version: '17'       # REQUIRED: 14, 15, 16, 17, 18
  python-version: '3.10'   # Optional (default: 3.10)
  skip-sonar: false        # Optional: skip SonarQube
  strict-mode: false       # Optional: fail workflow if any check fails
secrets: inherit           # Passes CONFIG_TOKEN, SONAR_TOKEN, SONAR_HOST_URL
```

## The 6 Quality Tools

| Tool | What it does | Pass criteria |
|------|-------------|---------------|
| **Black** | Code formatting (PEP 8 style) | All files formatted correctly |
| **Flake8** | Linting (syntax, style errors) | No errors (max line: 120 chars) |
| **Pylint-Odoo** | Odoo-specific checks (manifest, API usage) | Score >= 5.0/10 |
| **Radon** | Cyclomatic complexity | Average grade A-C (D/E = warning) |
| **Bandit** | Security vulnerabilities | No issues found |
| **SonarQube** | Deep code analysis | Analyzed (quality gate separate) |

## Config Files (per Odoo version)

```
github-actions/
├── configs/
│   ├── .flake8              # Shared: max-line-length=120, ignores W503/E203
│   ├── .bandit              # Security scan config
│   ├── .pylintrc-odoo14     # Version-specific pylint configs
│   ├── .pylintrc-odoo15
│   ├── .pylintrc-odoo16
│   ├── .pylintrc-odoo17     # Uses pylint_odoo plugin, valid_odoo_versions=17.0
│   └── .pylintrc-odoo18
└── dev-requirements/
    ├── base.txt             # black, flake8, radon, bandit
    └── odoo17.txt           # pylint, pylint-odoo (version-specific)
```

## Key Behaviors

1. **All checks run with `continue-on-error: true`** - one failure doesn't stop others
2. **PR Comment** - Automatic comment with results table (updates on re-run)
3. **Workflow Summary** - Results visible in GitHub Actions UI
4. **Strict Mode** - If enabled, workflow fails when any check fails

## Secrets Required

| Secret | Required | Purpose |
|--------|----------|---------|
| `CONFIG_TOKEN` | If private repo | Access to download configs from github-actions repo |
| `SONAR_TOKEN` | For SonarQube | Authentication to SonarQube server |
| `SONAR_HOST_URL` | For SonarQube | Your SonarQube server URL |

## Example PR Comment Output

```
## Quality Check Results

| Tool | Status |
|------|--------|
| Black (formatting) | ✅ Passed |
| Flake8 (linting) | ✅ Passed |
| Pylint-Odoo | ❌ Failed |
| Radon (complexity) | ⚠️ Warning |
| Bandit (security) | ✅ Passed |

### ⚠️ Some checks need attention
```

## Summary

**CI = Quality checks only (no tests, no deploy)**
- Runs: Black, Flake8, Pylint-Odoo, Radon, Bandit, SonarQube
- Configs: Centralized in `github-actions` repo, versioned per Odoo version
- Output: PR comment + workflow summary

**Jenkins = Tests + Deploy** (separate, triggered after quality passes)
