## What & When

**pre-commit** is a framework for managing **Git hook scripts** — it runs linters, formatters, and tests automatically before each commit (and optionally in CI). One `.pre-commit-config.yaml` defines the same checks for every developer.

Use pre-commit when:

- The team should not rely on "remember to run Ruff"
- You want [[Linting — Ruff]], [[Linting — mypy]], and [[Unit Testing - pytest]] gated on `git commit`
- Multiple languages/tools need orchestration (Python, YAML, Docker)
- CI should mirror local hooks exactly

```bash
pip install pre-commit
# or
uv add --dev pre-commit
```

Overview: [[Linting]]. Typical stack: Ruff → mypy → pytest hooks.

---

## pre-commit vs Manual / CI-only

| Approach | Pros | Cons |
| --- | --- | --- |
| **pre-commit** | Fast local feedback; shared config | One-time `pre-commit install` |
| CI only | No local setup | Broken commits until push |
| Manual | Flexible | Inconsistent across team |

Best practice: **pre-commit locally** + `pre-commit run --all-files` in CI.

---

## Installation & Setup

```bash
# Add to dev dependencies
uv add --dev pre-commit

# Install git hook scripts
pre-commit install

# Optional: run on push too
pre-commit install --hook-type pre-push

# Run against all files (first time / CI)
pre-commit run --all-files

# Update hook versions to latest allowed by rev pins
pre-commit autoupdate
```

Uninstall hooks: `pre-commit uninstall`

---

## Full Python Quality Config

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v5.0.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-added-large-files
        args: [--maxkb=1000]
      - id: check-merge-conflict

  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.8.4
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format

  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.13.0
    hooks:
      - id: mypy
        additional_dependencies:
          - pydantic>=2
          - types-requests
        args: [--config-file=pyproject.toml]
        files: ^src/

  - repo: local
    hooks:
      - id: pytest
        name: pytest
        entry: uv run pytest -q
        language: system
        pass_filenames: false
        always_run: true
        stages: [pre-push]    # keep commits fast; test on push
```

See [[Linting — Ruff]], [[Linting — mypy]], [[Unit Testing - pytest]].

---

## Hook Stages

| Stage | When | Typical hooks |
| --- | --- | --- |
| `pre-commit` (default) | `git commit` | Ruff, whitespace, yaml |
| `pre-push` | `git push` | pytest, slow integration |
| `commit-msg` | After message entered | Conventional commit lint |
| `manual` | `pre-commit run --hook-stage manual` | On demand |

```yaml
- id: pytest
  stages: [pre-push]
```

```bash
pre-commit install --hook-type pre-push
```

---

## Local Hooks

Run project scripts without publishing a hook repo:

```yaml
repos:
  - repo: local
    hooks:
      - id: alembic-heads-check
        name: alembic single head
        entry: uv run alembic heads
        language: system
        pass_filenames: false

      - id: django-check
        name: django system checks
        entry: uv run python manage.py check
        language: system
        pass_filenames: false
        files: ^(app|config)/
```

`language: system` — uses your venv / `uv run` on PATH; no isolated hook env.

---

## `pass_filenames` & File Filters

```yaml
- id: ruff
  files: ^src/.*\.py$          # only src Python
  exclude: ^src/migrations/

- id: mypy
  files: ^src/
  pass_filenames: false         # mypy checks whole package

- id: ruff
  types: [python]               # only when Python files staged
```

---

## Skip Hooks (Emergency Only)

```bash
# Skip all hooks once — use rarely
git commit --no-verify -m "wip: emergency hotfix"

# Skip specific hook (pre-commit 3.2+)
SKIP=mypy git commit -m "docs: readme only"
```

> [!tip] Do not skip hooks in normal workflow CI may still fail; `--no-verify` bypasses local safety only.

---

## CI Integration

```yaml
# .github/workflows/ci.yml
jobs:
  pre-commit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v4
      - run: uv sync --group dev
      - uses: pre-commit/action@v3.0.1
```

The official action runs `pre-commit run --all-files` using `.pre-commit-config.yaml`.

Alternative — call tools directly (faster cache control):

```yaml
- run: uv run pre-commit run --all-files
```

---

## Version Pinning & autoupdate

```yaml
# Pin rev to tag or commit SHA — reproducible hooks
- repo: https://github.com/astral-sh/ruff-pre-commit
  rev: v0.8.4
```

```bash
pre-commit autoupdate              # bump revs within repos
pre-commit run --all-files         # verify after update
git add .pre-commit-config.yaml
```

Commit hook config changes with the code that satisfies new rules.

---

## Monorepo Patterns

```yaml
# Root config delegates per package
- repo: local
  hooks:
    - id: ruff-api
      name: ruff (api)
      entry: bash -c 'cd services/api && uv run ruff check .'
      language: system
      files: ^services/api/
```

Or use `exclude` / `files` regex per service.

---

## Bootstrap in New Projects

[[Python — Copier]] templates often include:

```bash
pre-commit install
```

Manual bootstrap after [[Python — uv]] init:

```bash
uv add --dev pre-commit ruff mypy pytest
# write .pre-commit-config.yaml
pre-commit install
pre-commit run --all-files
```

---

## Common Mistakes

| Mistake | Fix |
| --- | --- |
| Hooks not installed | Run `pre-commit install` after clone |
| mypy hook missing deps | `additional_dependencies` in hook |
| Slow commits | Move pytest to `pre-push` |
| Hook env ≠ project env | Use `language: system` + `uv run` |
| Unpinned `rev: master` | Pin tags/SHAs for reproducibility |

---

## Quick Reference

| Task | Command |
| --- | --- |
| Install hooks | `pre-commit install` |
| Run all hooks | `pre-commit run --all-files` |
| Run one hook | `pre-commit run ruff --all-files` |
| Update versions | `pre-commit autoupdate` |
| Skip hook | `SKIP=mypy git commit ...` |
| Config file | `.pre-commit-config.yaml` |
| CI | `pre-commit/action` or `pre-commit run --all-files` |
| Uninstall | `pre-commit uninstall` |

---

## Tags

#python #pre-commit #git-hooks #linting #ci #code-quality #ruff #mypy #backend
