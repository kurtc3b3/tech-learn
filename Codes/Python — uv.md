## What & When

**uv** is a blazing-fast Python package and project manager written in Rust, built by Astral (the team behind Ruff). It replaces an entire toolchain — `pip`, `pip-tools`, `virtualenv`, `pyenv`, `pipx`, and parts of Poetry — with a single binary and a unified mental model.

Before uv, the same set of capabilities required combining pyenv for interpreter management, virtualenv for environment isolation, pip-tools for dependency locking, and pipx for CLI tools. uv does all of this, faster.

```bash
# macOS / Linux
curl -LsSf https://astral.sh/uv/install.sh | sh

# Windows (PowerShell)
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"

# Homebrew
brew install uv

# pip
pip install uv
```

---

## Why uv?

uv achieves large speedups over pip through aggressive caching, parallel downloads, and an optimized dependency resolver written in Rust.

|Operation|pip|uv|Speedup|
|---|---|---|---|
|Create virtual environment|1.95s|0.03s|**56x**|
|Install 23 packages (cold cache)|13.1s|0.87s|**15x**|
|Install 23 packages (warm cache)|6.6s|0.15s|**44x**|

uv's cache stores pre-built wheels and uses hard links to avoid copying files into the virtual environment, so "installing" cached packages is nearly instantaneous.

---

## uv Replaces

|Old tool|uv equivalent|
|---|---|
|`pip install`|`uv pip install` / `uv add`|
|`pip-compile`|`uv lock`|
|`pip-sync`|`uv sync`|
|`virtualenv` / `venv`|`uv venv`|
|`pyenv`|`uv python install`|
|`pipx`|`uvx` / `uv tool`|
|`poetry new`|`uv init`|
|`poetry add`|`uv add`|

---

## Python Version Management

```bash
# Install Python versions
uv python install 3.12
uv python install 3.11 3.12 3.13     # multiple at once

# List available versions
uv python list

# List installed versions
uv python list --only-installed

# Pin Python version for current directory
uv python pin 3.12                    # writes .python-version

# Find a specific Python
uv python find 3.12

# Uninstall
uv python uninstall 3.11
```

---

## Project Management

### Initialise a New Project

```bash
# New project in a new directory
uv init my-project
cd my-project

# New project in current directory
uv init

# Library (with src/ layout)
uv init --lib my-library

# Application (default)
uv init --app my-app

# Minimal — no README, no hello.py
uv init --bare my-project

# Specify Python version
uv init --python 3.12 my-project
```

Generated structure:

```
my-project/
├── .python-version     ← pins Python version
├── pyproject.toml      ← project metadata + dependencies
├── uv.lock             ← lockfile (commit this)
├── .venv/              ← virtual environment (gitignore this)
├── README.md
└── hello.py
```

### `pyproject.toml`

```toml
[project]
name = "my-project"
version = "0.1.0"
description = "My FastAPI service"
requires-python = ">=3.12"
dependencies = [
    "fastapi>=0.100",
    "uvicorn[standard]>=0.20",
]

[project.optional-dependencies]
dev  = ["pytest", "ruff", "mypy"]
docs = ["mkdocs", "mkdocs-material"]

[project.scripts]
myapp = "myapp.cli:app"             # CLI entry point

[build-system]
requires      = ["uv_build"]
build-backend = "uv_build"
```

---

## Adding & Removing Dependencies

```bash
# Add a dependency
uv add fastapi
uv add "fastapi>=0.100"
uv add httpx sqlmodel pydantic-settings

# Add dev dependency
uv add --dev pytest ruff mypy

# Add optional dependency group
uv add --optional docs mkdocs mkdocs-material

# Add from git
uv add "git+https://github.com/org/repo.git"
uv add "git+https://github.com/org/repo.git@v1.2.0"

# Add local package (editable)
uv add --editable ./my-local-package

# Remove a dependency
uv remove fastapi
uv remove --dev pytest

# Upgrade a dependency
uv add "fastapi>=0.110"              # update constraint
uv lock --upgrade-package fastapi   # upgrade to latest matching
uv lock --upgrade                   # upgrade all
```

---

## Installing & Syncing

```bash
# Install all dependencies (creates .venv if needed)
uv sync

# Install including dev dependencies (default)
uv sync --dev

# Install without dev dependencies (production)
uv sync --no-dev

# Install a specific group
uv sync --extra docs

# Install all extras
uv sync --all-extras

# Reinstall everything from scratch
uv sync --reinstall

# Frozen — fail if lockfile is out of date (CI)
uv sync --frozen
```

---

## Lockfile

```bash
# Generate / update uv.lock
uv lock

# Update all packages to latest allowed
uv lock --upgrade

# Update a specific package
uv lock --upgrade-package requests

# Check lockfile is up to date (CI)
uv lock --check

# Export to requirements.txt (for deployment / compatibility)
uv export --format requirements-txt > requirements.txt
uv export --no-dev > requirements.txt
```

> [!tip] Commit `uv.lock` The lockfile ensures reproducible installs across machines and CI. Always commit it for applications. For libraries, committing is optional but recommended.

---

## Running Commands

```bash
# Run in the project's virtual environment
uv run python main.py
uv run pytest
uv run uvicorn myapp.main:app --reload
uv run myapp                          # project script entry point

# Run with extra packages (without installing to project)
uv run --with rich python script.py

# Run a specific Python version
uv run --python 3.11 python main.py
```

> [!tip] `uv run` vs activating `.venv` `uv run cmd` is equivalent to `source .venv/bin/activate && cmd`. You never need to activate the virtual environment manually.

---

## Virtual Environments

```bash
# Create a virtual environment
uv venv                               # .venv in current dir
uv venv my-env                        # named environment
uv venv --python 3.12                 # specific Python version

# Activate (when needed — prefer uv run)
source .venv/bin/activate             # macOS / Linux
.venv\Scripts\activate                # Windows

# Create with seed packages
uv venv --seed                        # includes pip, setuptools, wheel
```

---

## `uvx` & Tool Management (pipx replacement)

Run CLI tools in isolated temporary environments — no installation needed.

```bash
# Run a tool without installing
uvx ruff check .
uvx black --check .
uvx httpie GET https://api.example.com
uvx copier copy gh:org/template .
uvx pytest                            # one-shot test run

# Install a tool globally (persistent)
uv tool install ruff
uv tool install black
uv tool install mypy

# Run installed tool
uv tool run ruff check .
ruff check .                          # also works — added to PATH

# List installed tools
uv tool list

# Upgrade a tool
uv tool upgrade ruff
uv tool upgrade --all

# Uninstall a tool
uv tool uninstall black

# Install with specific extras
uv tool install "copier[all]"

# Install from git
uv tool install "git+https://github.com/org/tool.git"
```

---

## pip-Compatible Interface

For projects not yet using `uv init` — drop-in pip replacement.

```bash
# Install packages
uv pip install fastapi
uv pip install -r requirements.txt
uv pip install -e .                   # editable install

# Uninstall
uv pip uninstall fastapi

# List installed packages
uv pip list
uv pip list --outdated

# Show package info
uv pip show fastapi

# Freeze — output installed packages
uv pip freeze > requirements.txt

# Compile requirements (pip-tools replacement)
uv pip compile requirements.in -o requirements.txt
uv pip compile pyproject.toml -o requirements.txt

# Sync environment to requirements
uv pip sync requirements.txt

# Check for dependency conflicts
uv pip check
```

---

## Workspaces — Monorepo Support

```toml
# pyproject.toml (workspace root)
[tool.uv.workspace]
members = [
    "packages/*",
    "services/api",
    "services/worker",
]
```

```bash
my-monorepo/
├── pyproject.toml          ← workspace root
├── uv.lock                 ← single lockfile for all packages
├── packages/
│   ├── shared-models/
│   │   └── pyproject.toml
│   └── utils/
│       └── pyproject.toml
└── services/
    ├── api/
    │   └── pyproject.toml
    └── worker/
        └── pyproject.toml
```

```bash
# Run command in a specific workspace member
uv run --package api uvicorn api.main:app

# Add dependency to a specific member
uv add --package api fastapi
```

---

## Inline Script Dependencies

Run scripts with dependencies declared inline — no project needed.

```python
# script.py
# /// script
# requires-python = ">=3.12"
# dependencies = [
#   "httpx",
#   "rich",
# ]
# ///

import httpx
from rich import print

response = httpx.get("https://api.github.com")
print(response.json())
```

```bash
uv run script.py             # installs httpx + rich automatically
```

---

## Configuration — `pyproject.toml` and `uv.toml`

```toml
# pyproject.toml
[tool.uv]
# Package index
index-url       = "https://pypi.org/simple"
extra-index-url = ["https://my-private-registry.example.com/simple"]

# Index strategy
index-strategy  = "unsafe-best-match"   # or "first-match" (default)

# Cache
cache-dir       = "~/.cache/uv"

# Python preference
python-preference = "managed"           # prefer uv-managed Python versions

# Constraints
constraint-dependencies = [
    "urllib3<2",
]

# Override versions (when you know better than the resolver)
override-dependencies = [
    "certifi>=2024.0",
]

# Compile bytecode on install
compile-bytecode = true

# Dev dependencies
dev-dependencies = [
    "pytest>=8",
    "ruff>=0.4",
    "mypy>=1.10",
]
```

---

## Docker Integration

`uv`'s standalone binary and fast installs make it well-suited for container builds. Because uv has no runtime dependency on Python, you can copy the binary into a Docker image and use it immediately.

```dockerfile
FROM python:3.12-slim

# Copy uv binary from official image
COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/

WORKDIR /app

# Copy dependency files first (layer caching)
COPY pyproject.toml uv.lock ./

# Install dependencies (no dev, frozen)
RUN uv sync --frozen --no-dev --no-install-project

# Copy application code
COPY . .

# Install the project itself
RUN uv sync --frozen --no-dev

CMD ["uv", "run", "uvicorn", "myapp.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

```dockerfile
# Alternative — multi-stage build
FROM ghcr.io/astral-sh/uv:python3.12-bookworm-slim AS builder

WORKDIR /app
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev

FROM python:3.12-slim AS runtime
COPY --from=builder /app/.venv /app/.venv
COPY --from=builder /app /app
ENV PATH="/app/.venv/bin:$PATH"
CMD ["uvicorn", "myapp.main:app", "--host", "0.0.0.0", "--port", "8000"]
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

      - name: Install uv
        uses: astral-sh/setup-uv@v4
        with:
          version: "latest"
          enable-cache: true             # cache uv's package cache

      - name: Set up Python
        run: uv python install

      - name: Install dependencies
        run: uv sync --frozen

      - name: Run tests
        run: uv run pytest

      - name: Run linter
        run: uv run ruff check .

      - name: Run type checker
        run: uv run mypy .
```

---

## Migrating from pip / requirements.txt

```bash
# 1. Initialise uv in existing project
uv init --bare                        # don't overwrite existing files

# 2. Import existing requirements
uv add $(cat requirements.txt)

# Or generate pyproject.toml from requirements.txt
# (manual — review and adjust)

# 3. Lock
uv lock

# 4. Sync
uv sync
```

## Migrating from Poetry

```bash
# Poetry's pyproject.toml is compatible — just start using uv
uv sync                               # reads [tool.poetry.dependencies]

# Or migrate to PEP 621 format (recommended)
# Move [tool.poetry.dependencies] → [project.dependencies]
# Replace poetry.lock with uv.lock
uv lock
```

---

## Shell Completion

```bash
# Bash
echo 'eval "$(uv generate-shell-completion bash)"' >> ~/.bashrc

# Zsh
echo 'eval "$(uv generate-shell-completion zsh)"' >> ~/.zshrc

# Fish
echo 'uv generate-shell-completion fish | source' >> ~/.config/fish/config.fish

# PowerShell
Add-Content $PROFILE '(& uv generate-shell-completion powershell) | Out-String | Invoke-Expression'
```

---

## Quick Reference

|Task|Command|
|---|---|
|Install uv|`curl -LsSf https://astral.sh/uv/install.sh \| sh`|
|New project|`uv init my-project`|
|Install Python|`uv python install 3.12`|
|Pin Python|`uv python pin 3.12`|
|Add dependency|`uv add fastapi`|
|Add dev dep|`uv add --dev pytest`|
|Remove dep|`uv remove fastapi`|
|Install all deps|`uv sync`|
|Install prod only|`uv sync --no-dev`|
|Install frozen (CI)|`uv sync --frozen`|
|Run command|`uv run pytest`|
|Run script|`uv run python script.py`|
|Update lockfile|`uv lock`|
|Upgrade all|`uv lock --upgrade`|
|Export requirements|`uv export > requirements.txt`|
|Run tool|`uvx ruff check .`|
|Install tool|`uv tool install ruff`|
|List tools|`uv tool list`|
|Create venv|`uv venv`|
|pip install|`uv pip install package`|
|pip compile|`uv pip compile requirements.in`|
|pip sync|`uv pip sync requirements.txt`|
|Shell completion|`uv generate-shell-completion zsh`|

---

## Tags

#python #uv #packaging #virtualenv #devtools #dependency-management #rust #backend