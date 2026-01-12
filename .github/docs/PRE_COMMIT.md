# Pre-commit Setup Guide

This guide explains how to set up pre-commit hooks locally to automatically format and lint your code before committing.

## Quick Start

```bash
# Install pre-commit
pip install pre-commit

# Install the hooks (run from repo root)
pre-commit install

# Done! Hooks will now run automatically on every commit
```

## Required Config Files

Pre-commit hooks use config files in your repo. Copy these from `base-repo` or create them:

| File | Purpose | Source |
|------|---------|--------|
| `.pre-commit-config.yaml` | Pre-commit hook definitions | [templates/.pre-commit-config.yaml](../../templates/.pre-commit-config.yaml) |
| `.flake8` | Flake8 linting rules | [base-repo/.flake8](https://github.com/much-GmbH/base-repo/blob/main/.flake8) |
| `.bandit` | Bandit security config | [base-repo/.bandit](https://github.com/much-GmbH/base-repo/blob/main/.bandit) |
| `.pylintrc` | Pylint-Odoo config | Copy from `configs/.pylintrc-odoo{VERSION}` |
| `pyproject.toml` | Black formatter config | [base-repo/pyproject.toml](https://github.com/much-GmbH/base-repo/blob/main/pyproject.toml) |
| `.isort.cfg` | Import sorting config | [base-repo/.isort.cfg](https://github.com/much-GmbH/base-repo/blob/main/.isort.cfg) |

### Pylint Config (Version-Specific)

Copy the appropriate pylint config for your Odoo version:

```bash
# For Odoo 17
curl -o .pylintrc \
  https://raw.githubusercontent.com/much-GmbH/github-actions/main/configs/.pylintrc-odoo17

# For Odoo 18
curl -o .pylintrc \
  https://raw.githubusercontent.com/much-GmbH/github-actions/main/configs/.pylintrc-odoo18
```

## Manual Run

To run all hooks on all files:

```bash
pre-commit run --all-files
```

To run only on staged files:

```bash
pre-commit run
```

To run a specific hook:

```bash
pre-commit run black --all-files
pre-commit run flake8 --all-files
```

## What It Does

| Hook | What it does |
|------|--------------|
| **Autoflake** | Removes unused imports and variables |
| **Black** | Formats Python code |
| **isort** | Sorts imports |
| **Flake8** | Linting (uses `.flake8`) |
| **Pylint-Odoo** | Odoo-specific checks (uses `.pylintrc`) |
| **Bandit** | Security checks (uses `.bandit`) |
| **Xenon** | Code complexity analysis |
| **OCA hooks** | Odoo manifest and PO file checks |
| **Trailing whitespace** | Removes trailing spaces |
| **End of file** | Ensures files end with newline |
| **Debug statements** | Catches leftover `pdb`/`breakpoint()` |

## Fixing CI Failures

If CI fails due to formatting issues:

```bash
# Pull latest changes
git pull

# Run all formatters
pre-commit run --all-files

# Or just Black
black .

# Commit the fixes
git add -A
git commit -m "style: fix formatting"
git push
```

## Skipping Hooks (Emergency Only)

If you need to bypass hooks temporarily:

```bash
git commit --no-verify -m "your message"
```

⚠️ **Warning**: Only use this in emergencies. CI will still fail if code isn't properly formatted.

## Troubleshooting

### "Black would make changes"
Run `black .` to auto-fix, then commit.

### "Flake8 found issues"
Check the output for line numbers and fix manually, or use `# noqa` for false positives.

### "Pylint errors"
Check `.pylintrc` for enabled checks. You can disable specific checks with `# pylint: disable=check-name`.

### Hooks not running
Make sure you ran `pre-commit install` in the repo root.

### Missing config file errors
Copy the required config files from base-repo (see Required Config Files section above).
