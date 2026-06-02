## What & When

**Poetry** is a Python dependency management and packaging tool that provides a unified workflow through `pyproject.toml`. It handles dependency resolution, virtual environment management, package building, and publishing to PyPI — all from a single CLI.

Use Poetry when:

- Managing dependencies for an existing Poetry-based project
- Publishing packages to PyPI with a polished workflow
- Working in teams with an established Poetry workflow
- Needing dependency groups (dev, docs, test) with lockfile reproducibility

For new projects, **uv** is the faster modern alternative. See [[Python - uv]].

```bash
# Official installer (recommended — isolated environment)
curl -sSL https://install.python-poetry.org | python3 -

# Homebrew
brew install poetry

# pipx
pipx install poetry

# Verify
poetry --version
```

---

## Poetry vs uv vs pip

|                           | Poetry        | uv        | pip + pip-tools    |
| ------------------------- | ------------- | --------- | ------------------ |
| Dependency resolution     | ✅             | ✅ Faster  | ✅                  |
| Lockfile                  | `poetry.lock` | `uv.lock` | `requirements.txt` |
| Virtual env management    | ✅ Auto        | ✅ Auto    | Manual             |
| Python version management | ❌             | ✅         | ❌                  |
| Package publishing        | ✅             | ✅         | Via twine          |
| PEP 621 support           | ✅ v2.0+       | ✅         | N/A                |
| Speed                     | Moderate      | Very fast | Slow               |
| Workspaces / monorepo     | Limited       | ✅         | ❌                  |

> [!info] Poetry 2.0 — PEP 621 Poetry 2.0 (released January 2025) added support for the standard `[project]` table in `pyproject.toml`. Before 2.0, all metadata lived in `[tool.poetry]`. Both formats are still supported. New projects should use `[project]`.

---

## Creating a New Project

```bash
# New project in a new directory
poetry new my-project

# New project with src/ layout
poetry new --src my-project

# Init in existing directory (interactive)
cd existing-project
poetry init
```

Generated structure:

```
my-project/
├── pyproject.toml      ← project metadata + dependencies
├── poetry.lock         ← lockfile (commit this)
├── README.md
├── my_project/
│   └── __init__.py
└── tests/
    └── __init__.py
```

---

## `pyproject.toml` — Modern Format (Poetry 2.0+)

```toml
[project]
name        = "my-project"
version     = "0.1.0"
description = "My FastAPI service"
authors     = [{name = "Alice Smith", email = "alice@example.com"}]
license     = {text = "MIT"}
readme      = "README.md"
requires-python = ">=3.12"

dependencies = [
    "fastapi>=0.100",
    "uvicorn[standard]>=0.20",
    "pydantic-settings>=2.0",
    "sqlmodel>=0.0.14",
]

[project.optional-dependencies]
dev  = ["pytest>=8", "ruff>=0.4", "mypy>=1.10"]
docs = ["mkdocs", "mkdocs-material"]

[project.scripts]
myapp = "myapp.cli:app"

[build-system]
requires      = ["poetry-core>=2.0.0"]
build-backend = "poetry.core.masonry.api"

[tool.poetry]
packages = [{include = "myapp", from = "src"}]
```

---

## `pyproject.toml` — Legacy Format (Poetry 1.x)

Still widely found in existing projects.

```toml
[tool.poetry]
name        = "my-project"
version     = "0.1.0"
description = "My FastAPI service"
authors     = ["Alice Smith <alice@example.com>"]
license     = "MIT"
readme      = "README.md"
packages    = [{include = "myapp"}]

[tool.poetry.dependencies]
python       = "^3.12"
fastapi      = "^0.100"
uvicorn      = {extras = ["standard"], version = "^0.20"}
pydantic-settings = "^2.0"
sqlmodel     = "^0.0.14"

[tool.poetry.group.dev.dependencies]
pytest       = "^8.0"
ruff         = "^0.4"
mypy         = "^1.10"

[tool.poetry.group.docs.dependencies]
mkdocs          = "^1.5"
mkdocs-material = "^9.0"

[tool.poetry.scripts]
myapp = "myapp.cli:app"

[build-system]
requires      = ["poetry-core"]
build-backend = "poetry.core.masonry.api"
```

---

## Dependency Version Constraints

```toml
[tool.poetry.dependencies]
# Caret — compatible release (^1.2.3 means >=1.2.3, <2.0.0)
fastapi     = "^0.100"

# Tilde — patch-level updates (~1.2.3 means >=1.2.3, <1.3.0)
requests    = "~2.31.0"

# Exact version
pydantic    = "2.5.0"

# Inequality
httpx       = ">=0.25,<1.0"

# Wildcard
boto3       = "1.28.*"

# Latest (no constraint)
rich        = "*"

# Extras
uvicorn     = {extras = ["standard"], version = "^0.20"}

# Git source
mylib       = {git = "https://github.com/org/mylib.git", branch = "main"}
mylib       = {git = "https://github.com/org/mylib.git", tag = "v1.2.0"}

# Local path
mylib       = {path = "../my-local-library", develop = true}

# Optional dependency
psycopg2    = {version = "^2.9", optional = true}

# Platform-specific
pywin32     = {version = "^306", markers = "sys_platform == 'win32'"}

# Python version specific
dataclasses = {version = "^0.6", python = "<3.7"}
```

---

## Core Commands

### Installing Dependencies

```bash
# Install all dependencies (creates venv if needed)
poetry install

# Install without dev dependencies
poetry install --only main

# Install a specific group
poetry install --only dev
poetry install --with docs

# Install without a group
poetry install --without docs

# Install and sync — remove packages not in lockfile
poetry install --sync

# Frozen — fail if lockfile is out of date (CI)
poetry install --frozen-lockfile       # Poetry 2.0+
```

### Adding & Removing

```bash
# Add a dependency
poetry add fastapi
poetry add "fastapi>=0.100"
poetry add httpx sqlmodel pydantic-settings

# Add dev dependency
poetry add --group dev pytest ruff mypy

# Add to a custom group
poetry add --group docs mkdocs mkdocs-material

# Add with extras
poetry add "uvicorn[standard]"

# Add from git
poetry add git+https://github.com/org/repo.git
poetry add git+https://github.com/org/repo.git#v1.2.0

# Add local package (editable)
poetry add --editable ./my-local-lib

# Remove a dependency
poetry remove fastapi
poetry remove --group dev pytest
```

### Updating

```bash
# Update all dependencies (within constraints)
poetry update

# Update specific packages
poetry update fastapi httpx

# Show outdated packages
poetry show --outdated

# Update Poetry itself
poetry self update
```

---

## Virtual Environment Management

```bash
# Poetry creates and manages venvs automatically
# Default: ~/.cache/pypoetry/virtualenvs/

# Run command inside venv
poetry run python main.py
poetry run pytest
poetry run uvicorn myapp.main:app --reload

# Activate the venv in current shell
poetry shell

# Show venv info
poetry env info
poetry env info --path         # just the path

# List all venvs for current project
poetry env list

# Create venv with specific Python
poetry env use python3.12
poetry env use 3.12

# Remove a venv
poetry env remove python3.12
poetry env remove --all

# Configure venv location — inside project (like uv)
poetry config virtualenvs.in-project true
```

> [!tip] `virtualenvs.in-project true` Setting this puts the venv at `.venv/` inside the project directory — the same convention as uv and most modern tools. Easier to find, easier for IDEs to detect.

---

## Showing & Inspecting Dependencies

```bash
# List all installed packages
poetry show

# Show dependency tree
poetry show --tree

# Show a specific package
poetry show fastapi

# Show outdated packages
poetry show --outdated

# Show only top-level dependencies
poetry show --top-level       # Poetry 1.8+

# Check for issues
poetry check
```

---

## Lockfile

```bash
# Generate / refresh poetry.lock
poetry lock

# Update lockfile without installing
poetry lock --no-update        # regenerate without upgrading

# Check if lockfile is up to date
poetry check

# Export to requirements.txt (for deployment / compatibility)
poetry export -f requirements.txt --output requirements.txt
poetry export -f requirements.txt --without-hashes --output requirements.txt
poetry export --only main -f requirements.txt --output requirements.txt
```

> [!tip] Commit `poetry.lock` Always commit the lockfile for applications — it ensures reproducible installs across machines and CI. For libraries, it's optional.

---

## Building & Publishing

```bash
# Build wheel + sdist
poetry build

# Build only wheel
poetry build --format wheel

# Publish to PyPI
poetry publish

# Build and publish in one step
poetry publish --build

# Publish to a private registry
poetry publish --repository my-private-repo

# Configure PyPI token
poetry config pypi-token.pypi my-token-here

# Configure private repository
poetry config repositories.my-repo https://my-private-repo.example.com/simple/
poetry config http-basic.my-repo username password
```

---

## Configuration

```bash
# Show all config
poetry config --list

# Set config globally
poetry config virtualenvs.in-project true
poetry config virtualenvs.create true

# Set config locally (project-only)
poetry config virtualenvs.in-project true --local

# Common settings
poetry config virtualenvs.in-project true        # .venv/ inside project
poetry config installer.max-workers 10           # parallel installs
poetry config installer.no-binary :all:          # always build from source
poetry config cache-dir ~/.cache/poetry          # custom cache dir
```

```toml
# Or in pyproject.toml (local config, Poetry 2.0+)
[tool.poetry.config]
virtualenvs.in-project = true
```

---

## Dependency Groups

Groups let you separate dev, test, and docs dependencies cleanly.

```toml
[tool.poetry.group.dev.dependencies]
pytest     = "^8.0"
pytest-cov = "^5.0"
ruff       = "^0.4"
mypy       = "^1.10"

[tool.poetry.group.test.dependencies]
pytest         = "^8.0"
pytest-asyncio = "^0.23"
httpx          = "^0.27"
factory-boy    = "^3.3"

[tool.poetry.group.docs.dependencies]
mkdocs          = "^1.5"
mkdocs-material = "^9.0"
```

```bash
poetry install --with dev,test         # include multiple groups
poetry install --without docs          # exclude a group
poetry install --only main             # only main dependencies
```

---

## Plugins

```bash
# Install a plugin
poetry self add poetry-plugin-export
poetry self add poetry-dynamic-versioning

# List installed plugins
poetry self show plugins

# Remove a plugin
poetry self remove poetry-plugin-export
```

Common plugins:

|Plugin|Purpose|
|---|---|
|`poetry-plugin-export`|Export to requirements.txt|
|`poetry-dynamic-versioning`|Version from git tags|
|`poetry-plugin-bundle`|Bundle venv for deployment|
|`poetry-multiproject-plugin`|Monorepo support|

---

## Scripts & Entry Points

```toml
# pyproject.toml

[project.scripts]
myapp        = "myapp.cli:app"         # CLI entry point
myapp-worker = "myapp.worker:main"

# Or legacy format
[tool.poetry.scripts]
myapp = "myapp.cli:app"
```

```bash
poetry run myapp                       # run the script
poetry install && myapp               # after install, use directly
```

---

## CI — GitHub Actions

```yaml
# .github/workflows/ci.yml
name: CI

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install Poetry
        uses: snok/install-poetry@v1
        with:
          version: latest
          virtualenvs-in-project: true

      - name: Cache venv
        uses: actions/cache@v4
        with:
          path: .venv
          key: venv-${{ hashFiles('poetry.lock') }}

      - name: Install dependencies
        run: poetry install --with dev

      - name: Run tests
        run: poetry run pytest

      - name: Run linter
        run: poetry run ruff check .

      - name: Type check
        run: poetry run mypy .
```

---

## Docker Integration

```dockerfile
FROM python:3.12-slim

# Install Poetry
RUN pip install poetry==1.8.0

WORKDIR /app

# Copy dependency files first (layer caching)
COPY pyproject.toml poetry.lock ./

# Install dependencies only (no dev, no project code yet)
RUN poetry config virtualenvs.create false \
    && poetry install --only main --no-interaction --no-ansi

# Copy application code
COPY . .

CMD ["uvicorn", "myapp.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

```dockerfile
# Alternative — export to requirements.txt then use pip
FROM python:3.12-slim AS builder
RUN pip install poetry
COPY pyproject.toml poetry.lock ./
RUN poetry export --only main -f requirements.txt -o /requirements.txt

FROM python:3.12-slim
COPY --from=builder /requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["uvicorn", "myapp.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

> [!tip] `virtualenvs.create false` in Docker In containers you don't need a venv — install directly into the system Python. Use `poetry config virtualenvs.create false` before installing.

---

## Migrating to uv

```bash
# uv reads Poetry's pyproject.toml directly
uv sync                        # works immediately with legacy format

# For full migration (recommended)
# 1. Convert [tool.poetry.dependencies] → [project.dependencies]
# 2. Convert [tool.poetry.group.*.dependencies] → [project.optional-dependencies]
# 3. Replace poetry.lock with uv.lock
uv lock
uv sync
```

---

## Common Issues

> [!warning] `poetry.lock` out of sync If `pyproject.toml` changes without running `poetry lock`, Poetry warns on `poetry install`. Always run `poetry lock` after manual edits to `pyproject.toml`.

> [!warning] Python version not found Poetry uses the system Python by default. If the required Python version isn't found, install it with pyenv or uv first: `uv python install 3.12 && poetry env use 3.12`

> [!tip] Slow dependency resolution Poetry's resolver can be slow on complex dependency graphs. Use `poetry config installer.max-workers 10` to parallelise installs, or consider migrating to uv for significantly faster resolution.

---

## Quick Reference

|Task|Command|
|---|---|
|New project|`poetry new my-project`|
|Init in existing dir|`poetry init`|
|Install all deps|`poetry install`|
|Install (no dev)|`poetry install --only main`|
|Install frozen (CI)|`poetry install --frozen-lockfile`|
|Add dependency|`poetry add fastapi`|
|Add dev dep|`poetry add --group dev pytest`|
|Remove dep|`poetry remove fastapi`|
|Update all|`poetry update`|
|Update specific|`poetry update fastapi`|
|Show outdated|`poetry show --outdated`|
|Show tree|`poetry show --tree`|
|Lock deps|`poetry lock`|
|Export requirements|`poetry export -f requirements.txt -o requirements.txt`|
|Run command|`poetry run pytest`|
|Activate venv|`poetry shell`|
|Show venv info|`poetry env info`|
|Use Python version|`poetry env use 3.12`|
|venv in project|`poetry config virtualenvs.in-project true`|
|Build package|`poetry build`|
|Publish|`poetry publish --build`|
|Check config|`poetry check`|
|Update Poetry|`poetry self update`|

---

## Tags

#python #poetry #packaging #dependency-management #virtualenv #pypi #backend #devtools