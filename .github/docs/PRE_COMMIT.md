# Pre-commit Setup Guide

This guide explains how to set up pre-commit hooks locally to automatically format your code before committing.

## Quick Start

```bash
# Install pre-commit
pip install pre-commit

# Install the hooks (run from repo root)
pre-commit install

# Done! Hooks will now run automatically on every commit
```

## Manual Run

To format all files manually:

```bash
pre-commit run --all-files
```

To format only staged files:

```bash
pre-commit run
```

## What It Does

The pre-commit hooks will automatically:

| Hook | What it does |
|------|--------------|
| **Black** | Formats Python code |
| **Flake8** | Checks for linting issues |
| **Trailing whitespace** | Removes trailing spaces |
| **End of file** | Ensures files end with newline |
| **Check YAML** | Validates YAML syntax |
| **Debug statements** | Catches leftover `pdb`/`breakpoint()` |

## Configuration

Copy the `.pre-commit-config.yaml` from the templates:

```bash
curl -o .pre-commit-config.yaml \
  https://raw.githubusercontent.com/much-GmbH-sandbox/github-actions/main/templates/.pre-commit-config.yaml
```

Or create it manually:

```yaml
repos:
  - repo: https://github.com/psf/black
    rev: 24.10.0
    hooks:
      - id: black

  - repo: https://github.com/pycqa/flake8
    rev: 7.1.1
    hooks:
      - id: flake8
        args: [--max-line-length=120]

  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v5.0.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: debug-statements
```

## Fixing CI Failures

If CI fails due to formatting issues:

```bash
# Pull latest changes
git pull

# Run formatters
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

⚠️ **Warning**: Only use this in emergencies. CI will still fail if code isn't formatted.

## Troubleshooting

### "Black would make changes"
Run `black .` to auto-fix, then commit.

### "Flake8 found issues"
Check the output for line numbers and fix manually, or use `# noqa` for false positives.

### Hooks not running
Make sure you ran `pre-commit install` in the repo root.
