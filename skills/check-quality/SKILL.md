---
name: check-code-quality
description: |
  Check the quality of modified Python code in the current project. ALWAYS run this skill automatically
  after ANY edit to a Python file, without waiting for the user to ask. Also run it when the user
  types /check-quality, asks to "check my code", "verify quality", "lint my changes", or wants to
  know if their modifications are clean before committing.
---

# Check Quality

Run quality checks on the files changed in this project, report issues as `file:line`,
and end with a clear verdict. Mirrors the project's `.pre-commit-config.yaml`.

## Steps

1. **Find changed files**
   ```bash
   git diff --name-only HEAD          # unstaged + staged vs HEAD
   git diff --name-only --cached      # staged
   ```
   Merge and dedupe both lists. If empty, fall back to `git diff --name-only HEAD~1 HEAD`
   (last commit). If still empty, tell the user there's nothing to check and stop.

2. **Run pre-commit on the changed files — primary path**
   Pass ALL changed files (not just `.py`): pre-commit routes each file to the hooks
   that apply to it, and this is the only way `uv-lock`, `check-toml`, `gitleaks`, etc. fire.
   ```bash
   uv run pre-commit run --files <all changed files>
   ```
   > Include `pyproject.toml` in the list when it changed, so the **`uv-lock`** hook
   > re-syncs `uv.lock`. No separate `uv lock` call is needed — the hook handles it.

   One pass runs every configured hook:
   - **Security**: `detect-private-key`, `gitleaks` (~160 secret-detection rules)
   - **Syntax**: `check-json`, `check-yaml`, `check-toml`
   - **Whitespace / EOL**: `trailing-whitespace`, `end-of-file-fixer`, `mixed-line-ending`
   - **Git hygiene**: `check-added-large-files` (50 MB), `check-merge-conflict`, `check-case-conflict`
   - **Python**: `debug-statements`
   - **JSON format**: `pretty-format-json`
   - **Lint / format**: `ruff-check`, `ruff-format` (config from `pyproject.toml`)
   - **Docstrings**: `interrogate` (fail-under = 100%)
   - **Deps**: `uv-lock` (keeps `uv.lock` in sync with `pyproject.toml`)

3. **Fallback — pre-commit not installed**
   Use the Makefile targets (they read `pyproject.toml`):
   ```bash
   make lint-format    # ruff check + ruff format
   make interrogate    # docstring coverage
   ```
   To scope to the changed `.py` files only:
   ```bash
   uv run ruff check <py files> --config pyproject.toml --no-fix
   uv run ruff format --check <py files> --config pyproject.toml
   uv run interrogate <py files> --config pyproject.toml -v
   ```
   `gitleaks` and the pre-commit-hooks set aren't covered by the fallback — say so in the report.

4. **Run tests**
   ```bash
   make test           # == uv run pytest
   ```
   Report failures with test name, `file:line`, and the error.

5. **Present results**
   - Per check: number of issues, each as `file:line` + a one-line fix suggestion.
   - Note anything auto-fixed by a hook (ruff, formatters) so the user re-stages it.
   - End with a **verdict**:
     - **OK** — clean across all checks.
     - **À corriger** — issues grouped by file, with concrete fixes.

   Keep it concise. Reference `file:line`, never dump full file contents.
