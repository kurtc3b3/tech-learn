## What & When

**Ruff** is an extremely fast Python linter and formatter written in Rust. It replaces **Flake8**, **isort**, **Black**, **pyupgrade**, **autoflake**, and dozens of plugins — with one tool and one `pyproject.toml` config.

Use Ruff when:

- Starting a new Python project (default linter in this vault)
- You want **lint + format** in one dependency
- CI and pre-commit must finish in seconds
- Migrating from Black + isort + Flake8

```bash
pip install ruff
# or
uv add --dev ruff
```

Pair with [[Linting — mypy]] for type inference and [[Linting — pre-commit]] to run on commit. Overview: [[Linting]].

---

## Ruff vs Legacy Stack

| Need | Ruff command | Old stack |
| --- | --- | --- |
| Lint | `ruff check` | Flake8 + plugins |
| Auto-fix | `ruff check --fix` | autoflake, pyupgrade |
| Import sort | `I` rules / `--fix` | isort |
| Format | `ruff format` | Black |
| Speed | ~10–100× faster | Multiple tools |

> [!tip] Use Ruff for format **and** lint Running Black and Ruff format together causes fighting configs — pick Ruff only.

---

## CLI Essentials

```bash
# Lint entire project
ruff check .

# Lint + apply safe fixes
ruff check . --fix

# Show what would change (fixable only)
ruff check . --diff

# Format (like Black)
ruff format .

# Check formatting without writing
ruff format . --check

# Lint specific paths
ruff check src tests

# Explain a rule code
ruff rule F401
```

With [[Python — uv]]:

```bash
uv run ruff check .
uv run ruff format .
uvx ruff check .              # no project install
```

---

## `pyproject.toml` Configuration

```toml
[tool.ruff]
target-version = "py311"
line-length = 88
src = ["src", "tests"]
exclude = [".venv", "migrations"]

[tool.ruff.lint]
select = [
    "E",      # pycodestyle errors
    "W",      # pycodestyle warnings
    "F",      # Pyflakes
    "I",      # isort
    "UP",     # pyupgrade
    "B",      # flake8-bugbear
    "SIM",    # flake8-simplify
    "ASYNC",  # async pitfalls
]
ignore = [
    "E501",   # line length (formatter handles)
]

[tool.ruff.lint.per-file-ignores]
"tests/**/*.py" = ["S101"]      # allow assert in tests
"migrations/**/*.py" = ["ALL"]

[tool.ruff.lint.isort]
known-first-party = ["myapp"]

[tool.ruff.format]
quote-style = "double"
indent-style = "space"
docstring-code-format = true
```

---

## Rule Selection Guide

| Prefix | Category | Examples |
| --- | --- | --- |
| `E`, `W` | Style (pycodestyle) | Indentation, whitespace |
| `F` | Pyflakes | Unused imports, undefined names |
| `I` | Import sort | isort-compatible |
| `UP` | pyupgrade | Modern Python syntax |
| `B` | bugbear | Mutable defaults, assert misuse |
| `SIM` | simplify | Redundant if/else |
| `ASYNC` | async | Blocking call in async def |
| `S` | bandit | Security (optional) |
| `TCH` | type-checking imports | TYPE_CHECKING blocks |

Enable gradually on brownfield repos: start with `F`, `E`, `I`, then add `UP`, `B`.

---

## Auto-Fix Workflow

```bash
# Safe fixes only
ruff check . --fix

# Include unsafe fixes (review diff carefully)
ruff check . --fix --unsafe-fixes

# Format after lint fixes
ruff format .
```

```bash
# Pre-commit style: fail if not formatted
ruff format --check .
ruff check .
```

---

## Editor Integration

**VS Code / Cursor** — install the Ruff extension; set as default formatter:

```json
{
  "[python]": {
    "editor.defaultFormatter": "charliermarsh.ruff",
    "editor.formatOnSave": true,
    "editor.codeActionsOnSave": {
      "source.fixAll.ruff": "explicit",
      "source.organizeImports.ruff": "explicit"
    }
  }
}
```

Ruff reads `pyproject.toml` automatically — same rules in editor, CLI, and CI.

---

## FastAPI / Backend Patterns

```toml
# Stricter async rules for API code
[tool.ruff.lint]
select = ["E", "F", "I", "UP", "B", "ASYNC"]

[tool.ruff.lint.per-file-ignores]
"src/migrations/*" = ["ALL"]
```

```python
# F401: unused import — Ruff removes with --fix
# B008: do not call function in default arg — common in FastAPI Depends
# Often ignored for Depends pattern in framework code:
```

```toml
[tool.ruff.lint.per-file-ignores]
"src/api/**/*.py" = ["B008"]    # FastAPI Depends() in defaults — use sparingly
```

See [[API - FastAPI]], [[Python — typing]].

---

## pre-commit Hook

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.8.4
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format
```

See [[Linting — pre-commit]].

---

## CI (GitHub Actions)

```yaml
- name: Ruff lint
  run: uv run ruff check .

- name: Ruff format
  run: uv run ruff format --check .
```

---

## Common Mistakes

| Mistake | Fix |
| --- | --- |
| Black + Ruff format both enabled | Remove Black; use `ruff format` only |
| Huge `--fix` on legacy codebase | Enable rule sets incrementally |
| Ignoring `ASYNC` in async APIs | Fix blocking `open()` / `requests` in async routes |
| Different config in CI vs local | Single `pyproject.toml`; run via `uv run` |

---

## Quick Reference

| Task | Command |
| --- | --- |
| Lint | `ruff check .` |
| Fix | `ruff check . --fix` |
| Format | `ruff format .` |
| Format check | `ruff format . --check` |
| Rule help | `ruff rule F401` |
| Config | `[tool.ruff]` in `pyproject.toml` |
| Linter only section | `[tool.ruff.lint]` |
| Formatter section | `[tool.ruff.format]` |

---

## Tags

#python #ruff #linting #formatter #isort #flake8 #code-quality #backend
