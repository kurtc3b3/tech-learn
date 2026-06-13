## What & When

**pre-commit** is a framework for managing **Git hook scripts** — it runs linters, formatters, and tests automatically before each commit (and optionally in CI). One `.pre-commit-config.yaml` defines the same checks for every developer.

Use pre-commit when:

- The team should not rely on "remember to run Ruff"
- You want [[Linting — Ruff]], [[Linting — mypy]], **Gitleaks**, **Bandit**, and [[Unit Testing - pytest]] gated on `git commit`
- Multiple languages/tools need orchestration (Python, YAML, Docker)
- CI should mirror local hooks exactly

```bash
pip install pre-commit
# Optional standalone tools (if not using Ruff for imports/security):
# pip install isort bandit
# gitleaks — installed by pre-commit hook env, or: brew install gitleaks
# or
uv add --dev pre-commit
```

Overview: [[Linting]]. Typical stack: Gitleaks → Ruff (or isort) → Bandit → mypy → pytest hooks.

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

  # Secrets — scan staged files before commit
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.24.2
    hooks:
      - id: gitleaks

  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.8.4
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format

  # Security — Python SAST (complements Ruff S* rules; see Bandit section)
  - repo: https://github.com/PyCQA/bandit
    rev: 1.8.3
    hooks:
      - id: bandit
        args: ["-c", "pyproject.toml"]
        additional_dependencies: ["bandit[toml]"]

  # Import sort — optional if Ruff I rules are enabled; see isort section
  # - repo: https://github.com/PyCQA/isort
  #   rev: 6.0.1
  #   hooks:
  #     - id: isort

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

Add Bandit config to `pyproject.toml`:

```toml
[tool.bandit]
exclude_dirs = ["tests", ".venv"]
skips = ["B101"]   # assert_used — allow in tests via exclude_dirs
```

See [[Linting — Ruff]], [[Linting — mypy]], [[Unit Testing - pytest]].

---

## Gitleaks (Secret Scanning)

**Gitleaks** scans staged files for API keys, tokens, passwords, and private keys before they reach Git. Run it **early** in the hook list so commits fail fast on accidental secrets.

```yaml
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.24.2
    hooks:
      - id: gitleaks
```

Manual scan (including history):

```bash
gitleaks detect --source . --verbose
gitleaks detect --log-opts="HEAD~10..HEAD"   # recent commits
pre-commit run gitleaks --all-files
```

| Practice | Why |
| --- | --- |
| `.gitignore` `.env`, `*.pem` | Never stage secrets |
| [[Python — python-dotenv]] locally | Env vars, not hardcoded keys |
| CI + GitHub secret scanning | Second line after pre-commit |
| Custom rules | `.gitleaks.toml` for project-specific patterns |

> [!warning] Gitleaks blocks the commit Hook runs on **staged** content; rotate any key that was ever pushed even if later removed.

Pair with [[Commands/CLI — Git & GitHub]] — never use `--no-verify` to bypass secret scans unless you are certain the finding is a false positive.

---

## Bandit (Python Security)

**Bandit** is a Python **SAST** linter — finds `eval`, hardcoded passwords, unsafe YAML load, weak crypto, etc. Complements [[Linting — Ruff]] (optional `S` rules) with deeper security-focused checks.

```yaml
  - repo: https://github.com/PyCQA/bandit
    rev: 1.8.3
    hooks:
      - id: bandit
        args: ["-c", "pyproject.toml"]
        additional_dependencies: ["bandit[toml]"]
```

```toml
# pyproject.toml
[tool.bandit]
exclude_dirs = ["tests", "migrations", ".venv"]
skips = ["B101"]   # assert in tests — or exclude tests dir only
```

CLI:

```bash
bandit -r src -c pyproject.toml
uv run bandit -r src
```

| Ruff `S` rules | Bandit hook |
| --- | --- |
| Fast; same config as lint | Dedicated security report |
| Good default in one tool | Stricter / more rules for security reviews |
| `--fix` on some issues | Report-only; fix manually |

Use **both** on sensitive backends ([[API - FastAPI]], [[ORM - SQLAlchemy]] with raw SQL) or **Ruff `S` only** for minimal overhead.

---

## isort (Import Sorting)

**isort** sorts and groups imports (stdlib → third-party → first-party). In this vault, **Ruff `I` rules** usually replace a separate isort hook — enable in [[Linting — Ruff]]:

```toml
[tool.ruff.lint]
select = ["E", "F", "I", "UP"]

[tool.ruff.lint.isort]
known-first-party = ["myapp"]
```

Add a **standalone isort hook** only when:

- The project does not use Ruff yet
- You need isort-specific options not mirrored in Ruff
- A legacy repo already standardizes on isort config

```yaml
  - repo: https://github.com/PyCQA/isort
    rev: 6.0.1
    hooks:
      - id: isort
        args: ["--profile", "black"]
```

```toml
# pyproject.toml (isort-only projects)
[tool.isort]
profile = "black"
known_first_party = ["myapp"]
line_length = 88
```

```bash
isort .
isort . --check --diff
pre-commit run isort --all-files
```

> [!tip] Do not run isort and Ruff import fix on the same commit Pick one — duplicate hooks fight each other. Default here: **Ruff only**.

---

## Hook Stages

| Stage | When | Typical hooks |
| --- | --- | --- |
| `pre-commit` (default) | `git commit` | Gitleaks, Ruff, Bandit, isort, yaml |
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
| isort + Ruff both fix imports | Use Ruff `I` only, or isort only |
| Secret false positive | `.gitleaks.toml` allowlist; never blanket `--no-verify` |
| Bandit noisy on tests | `exclude_dirs = ["tests"]` in `[tool.bandit]` |

---

## Quick Reference

| Task | Command |
| --- | --- |
| Install hooks | `pre-commit install` |
| Run all hooks | `pre-commit run --all-files` |
| Run one hook | `pre-commit run ruff --all-files` |
| Run Gitleaks only | `pre-commit run gitleaks --all-files` |
| Run Bandit only | `pre-commit run bandit --all-files` |
| Update versions | `pre-commit autoupdate` |
| Skip hook | `SKIP=mypy git commit ...` |
| Config file | `.pre-commit-config.yaml` |
| CI | `pre-commit/action` or `pre-commit run --all-files` |
| Uninstall | `pre-commit uninstall` |

---

## Tags

#python #pre-commit #git-hooks #linting #ci #code-quality #ruff #mypy #gitleaks #bandit #isort #security #backend
