## What & When

**Copier** is a project templating and scaffolding tool. It generates new projects from templates — and crucially, unlike Cookiecutter, it can **update existing projects** when the template evolves. Templates are Git repositories with Jinja2-powered files and a `copier.yml` questionnaire.

Use Copier when:

- Scaffolding new Python packages, FastAPI services, or CLI tools consistently
- Keeping many projects in sync with a shared template over time
- Standardising boilerplate across a team (CI config, linting, Docker setup)
- Replacing Cookiecutter — especially when template updates matter

```bash
pip install copier
# or via uv (recommended)
uvx copier
```

Requirements: Python 3.10+, Git 2.27+

---

## Copier vs Cookiecutter vs Cruft


|                       | Copier                  | Cookiecutter | Cruft             |
| --------------------- | ----------------------- | ------------ | ----------------- |
| Template updates      | ✅ Built-in              | ❌            | ✅ (via git magic) |
| Jinja2 templating     | ✅                       | ✅            | ✅                 |
| Git URL templates     | ✅                       | ✅            | ✅                 |
| Interactive prompts   | ✅ Rich                  | ✅ Basic      | ✅ Basic           |
| Answers file          | ✅ `.copier-answers.yml` | ❌            | ✅ `.cruft.json`   |
| Conditional questions | ✅                       | Limited      | Limited           |
| Tasks / hooks         | ✅                       | ✅            | ✅                 |
| Python API            | ✅                       | ✅            | ✅                 |
| Active maintenance    | ✅                       | ✅            | Moderate          |

---

## Template Structure

```
my-template/                        ← Git repository root
├── copier.yml                      ← questions + settings
├── .copier-answers.yml.jinja       ← records answers for future updates
├── {{project_name}}/               ← Jinja2 in directory names
│   ├── __init__.py.jinja           ← .jinja suffix = rendered by Jinja2
│   ├── main.py.jinja
│   └── config.py.jinja
├── tests/
│   └── test_{{module_name}}.py.jinja
├── pyproject.toml.jinja
├── README.md.jinja
├── Dockerfile.jinja
└── .github/
    └── workflows/
        └── ci.yml.jinja
```

> [!info] `.jinja` suffix Only files ending in `.jinja` are rendered through Jinja2. All other files are copied verbatim. You can change this suffix in `copier.yml` with `_templates_suffix`.

---

## `copier.yml` — Questions & Settings

```yaml
# copier.yml

# ── Questions ──────────────────────────────────────────────────────────────
project_name:
    type: str
    help: "Project name (e.g. my-fastapi-service)"
    placeholder: "my-project"

module_name:
    type: str
    help: "Python module name (snake_case)"
    default: "{{ project_name | lower | replace('-', '_') }}"

author_name:
    type: str
    help: "Your full name"

author_email:
    type: str
    help: "Your email address"

python_version:
    type: str
    help: "Minimum Python version"
    choices:
        - "3.10"
        - "3.11"
        - "3.12"
    default: "3.12"

use_docker:
    type: bool
    help: "Include Dockerfile?"
    default: true

license:
    type: str
    help: "Project licence"
    choices:
        MIT: "MIT"
        Apache: "Apache-2.0"
        Private: "Proprietary"
    default: "MIT"

database:
    type: str
    help: "Database backend"
    choices:
        - none
        - postgres
        - sqlite
    default: none

database_url:
    type: str
    help: "Database connection URL"
    default: >-
        {%- if database == 'postgres' -%}
        postgresql+asyncpg://user:pass@localhost/{{ module_name }}
        {%- elif database == 'sqlite' -%}
        sqlite+aiosqlite:///./{{ module_name }}.db
        {%- else -%}
        {{ UNSET }}
        {%- endif -%}
    when: "{{ database != 'none' }}"

secret_key:
    type: str
    help: "Application secret key"
    secret: true
    default: "changeme"

# ── Settings ───────────────────────────────────────────────────────────────
_min_copier_version: "9.0"
_templates_suffix: ".jinja"
_answers_file: ".copier-answers.yml"

_message_before_copy: |
    Thanks for using this template!
    Make sure you have uv installed: https://docs.astral.sh/uv/

_message_after_copy: |
    ✅ Project created!
    Next steps:
      cd {{ project_name }}
      uv sync
      uvicorn {{ module_name }}.main:app --reload

_tasks:
    - "git init"
    - "git add ."
    - "uv sync"
```

---

## Question Types

```yaml
# str — text input
name:
    type: str
    help: "Project name"
    placeholder: "my-project"
    default: "my-project"

# int — integer
port:
    type: int
    help: "Port number"
    default: 8000

# float
version:
    type: float
    default: 0.1

# bool — yes/no
use_docker:
    type: bool
    default: true

# Single choice from list
python_version:
    type: str
    choices:
        - "3.10"
        - "3.11"
        - "3.12"

# Choice with display name and value
license:
    type: str
    choices:
        MIT License: "MIT"
        Apache License 2.0: "Apache-2.0"
        Proprietary: "proprietary"

# Multiselect — pick multiple
extras:
    type: str
    multiselect: true
    choices:
        - docker
        - github-actions
        - pre-commit
        - docs

# Secret — hidden input, not saved in answers file
db_password:
    type: str
    secret: true
    default: "changeme"
```

---

## Conditional Questions & Files

### Conditional questions with `when`

```yaml
use_database:
    type: bool
    default: false

database_url:
    type: str
    help: "Database URL"
    when: "{{ use_database }}"          # only asked if use_database is true
    default: "postgresql://localhost/db"

use_redis:
    type: bool
    default: false
    when: "{{ use_database }}"          # only asked if use_database is true
```

### Conditional files

```
# File only included when use_docker is true
{% if use_docker %}Dockerfile.jinja{% endif %}
```

Or use `_exclude` in `copier.yml`:

```yaml
_exclude:
    - "Dockerfile*"                     # exclude by default

# Then override conditionally in a task or use when
```

Better pattern — use an `if` block inside the file and rely on empty output, or name files with conditions handled via tasks.

---

## Jinja2 in Templates

All the power of Jinja2 is available in `.jinja` files, plus Ansible filters.

```python
# {{module_name}}/main.py.jinja

from fastapi import FastAPI

app = FastAPI(
    title="{{ project_name }}",
    version="{{ version | default('0.1.0') }}",
)

{% if use_database %}
from .database import engine, Base

@app.on_event("startup")
async def startup():
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
{% endif %}

@app.get("/health")
async def health():
    return {"status": "ok"}
```

```toml
# pyproject.toml.jinja

[project]
name = "{{ project_name }}"
version = "0.1.0"
description = "{{ project_description | default('') }}"
authors = [{name = "{{ author_name }}", email = "{{ author_email }}"}]
requires-python = ">={{ python_version }}"
license = {text = "{{ license }}"}

[project.dependencies]
fastapi = ">=0.100"
uvicorn = {extras = ["standard"], version = ">=0.20"}
{% if use_database %}
sqlalchemy = {extras = ["asyncio"], version = ">=2.0"}
{% if database == 'postgres' %}
asyncpg = ">=0.27"
{% elif database == 'sqlite' %}
aiosqlite = ">=0.19"
{% endif %}
{% endif %}
```

---

## `.copier-answers.yml.jinja` — Answers File

Always include this in your template. It records the answers used to generate the project — required for future `copier update` runs.

```yaml
# .copier-answers.yml.jinja
# Changes here will be overwritten by Copier — do not edit manually
{{ _copier_answers | to_nice_yaml -}}
```

The generated `.copier-answers.yml` looks like:

```yaml
# Changes here will be overwritten by Copier
_commit: "1.2.0"
_src_path: "gh:myorg/my-template"
project_name: my-fastapi-service
module_name: my_fastapi_service
author_name: Alice Smith
python_version: "3.12"
use_docker: true
database: postgres
```

---

## CLI — Generating a Project

```bash
# From a local template
copier copy path/to/template path/to/new-project

# From a Git repository
copier copy https://github.com/myorg/my-template.git my-project

# GitHub shorthand
copier copy gh:myorg/my-template my-project

# GitLab shorthand
copier copy gl:myorg/my-template my-project

# Specific git tag / branch / commit
copier copy --vcs-ref=v1.2.0 gh:myorg/my-template my-project
copier copy --vcs-ref=main   gh:myorg/my-template my-project

# Skip prompts — use defaults
copier copy --defaults gh:myorg/my-template my-project

# Pass answers via CLI (non-interactive)
copier copy \
    --data project_name=my-service \
    --data author_name="Alice Smith" \
    --defaults \
    gh:myorg/my-template my-project

# Pretend — dry run, no files written
copier copy --pretend gh:myorg/my-template my-project

# Overwrite existing files without asking
copier copy --overwrite gh:myorg/my-template my-project
```

---

## CLI — Updating a Project

This is Copier's killer feature — update a generated project when the template releases a new version.

```bash
# Inside the generated project directory
cd my-project

# Update to latest template version
copier update

# Update to a specific tag
copier update --vcs-ref=v2.0.0

# Preview what would change (dry run)
copier update --pretend

# Skip interactive prompts — use existing answers
copier update --defaults

# Force overwrite all files
copier update --overwrite

# Skip running tasks during update
copier update --skip-tasks
```

> [!tip] How update works Copier re-renders the template with your saved answers from `.copier-answers.yml`, then performs a 3-way merge between the old rendered output, your local changes, and the new rendered output — similar to `git merge`. Conflicts are shown inline.

---

## Tasks — Post-Generation Hooks

Tasks run shell commands after the project is generated or updated.

```yaml
# copier.yml
_tasks:
    - "git init"
    - "git add ."
    - "uv sync"
    - "pre-commit install"
    - "echo 'Done! Run: cd {{ project_name }} && uvicorn {{ module_name }}.main:app'"
```

Conditional tasks:

```yaml
_tasks:
    - command: "uv sync"
      when: "{{ python_version >= '3.11' }}"

    - command: "docker build -t {{ module_name }} ."
      when: "{{ use_docker }}"
```

---

## Python API

Use Copier programmatically — in scripts, CI pipelines, or testing.

```python
import copier

# Generate a project
copier.run_copy(
    src_path="gh:myorg/my-template",
    dst_path="./my-project",
    data={
        "project_name": "my-project",
        "author_name":  "Alice Smith",
        "author_email": "alice@example.com",
        "use_docker":   True,
        "database":     "postgres",
    },
    defaults=True,
    overwrite=True,
    vcs_ref="v1.2.0",
)

# Update an existing project
copier.run_update(
    dst_path="./my-project",
    defaults=True,
    overwrite=False,
    vcs_ref="v2.0.0",
)
```

---

## FastAPI Service Template Example

A minimal but practical Copier template for a FastAPI service.

```
fastapi-template/
├── copier.yml
├── .copier-answers.yml.jinja
├── {{project_name}}/
│   ├── __init__.py
│   ├── main.py.jinja
│   ├── config.py.jinja
│   └── routers/
│       └── __init__.py
├── tests/
│   ├── __init__.py
│   └── test_main.py.jinja
├── pyproject.toml.jinja
├── .env.example.jinja
├── Dockerfile.jinja
└── .github/
    └── workflows/
        └── ci.yml.jinja
```

```yaml
# copier.yml
project_name:
    type: str
    help: "Service name (kebab-case)"

module_name:
    type: str
    default: "{{ project_name | lower | replace('-', '_') }}"

port:
    type: int
    default: 8000

use_docker:
    type: bool
    default: true

use_github_actions:
    type: bool
    default: true

_tasks:
    - "git init && git add . && git commit -m 'Initial commit from template'"
    - "uv sync"
```

```python
# {{project_name}}/main.py.jinja
from fastapi import FastAPI
from .config import get_settings

app = FastAPI(
    title="{{ project_name }}",
    version="0.1.0",
)

@app.get("/health")
async def health():
    return {"status": "ok", "service": "{{ project_name }}"}
```

```python
# {{project_name}}/config.py.jinja
from pydantic_settings import BaseSettings, SettingsConfigDict
from functools import lru_cache

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env")

    app_name: str = "{{ project_name }}"
    debug:    bool = False
    port:     int = {{ port }}

@lru_cache
def get_settings() -> Settings:
    return Settings()
```

---

## Migrations — Template Version Changes

When a template variable is renamed or removed between versions, define a migration to keep existing projects in sync.

```yaml
# copier.yml
_migrations:
    - version: "2.0.0"
      before:
          - >-
              python -c "
              import yaml, pathlib;
              f = pathlib.Path('.copier-answers.yml');
              d = yaml.safe_load(f.read_text());
              d['module_name'] = d.pop('package_name', d.get('module_name'));
              f.write_text(yaml.dump(d))
              "
      after:
          - "uv sync"
```

---

## Best Practices

> [!tip] Version your template with Git tags Use semantic versioning (`v1.0.0`, `v1.1.0`) for template releases. Generated projects record the tag in `.copier-answers.yml` — `copier update` then upgrades between tagged versions cleanly.

> [!tip] Always include `.copier-answers.yml.jinja` Without it, `copier update` cannot function. This is the most important file in any Copier template.

> [!tip] Use `_message_after_copy` for next steps Print clear instructions so developers know what to do immediately after generation — `uv sync`, `cd project`, start commands.

> [!warning] Secret answers are not saved Questions marked `secret: true` are not written to `.copier-answers.yml`. On `copier update`, the user will be prompted again. Use this for passwords and tokens, not for structural choices that affect file generation.

> [!tip] Test your template Use `copier copy --pretend` to validate rendering, or write pytest tests using the Python API to assert generated file contents.

---

## Quick Reference

|Task|Code|
|---|---|
|Generate from local|`copier copy ./template ./project`|
|Generate from GitHub|`copier copy gh:org/template my-project`|
|Specific version|`copier copy --vcs-ref=v1.0.0 gh:org/template .`|
|Non-interactive|`copier copy --defaults --data key=val gh:org/template .`|
|Dry run|`copier copy --pretend gh:org/template .`|
|Update project|`copier update`|
|Update to tag|`copier update --vcs-ref=v2.0.0`|
|Update dry run|`copier update --pretend`|
|Required field|`key:\n type: str\n help: "..."`|
|Default value|`default: "my-value"`|
|Computed default|`default: "{{ other_var \| lower }}"`|
|Choices|`choices:\n - a\n - b`|
|Conditional question|`when: "{{ use_feature }}"`|
|Secret question|`secret: true`|
|Post-gen tasks|`_tasks:\n - "uv sync"`|
|Answers file|`.copier-answers.yml.jinja`|
|Python API copy|`copier.run_copy(src, dst, data={...})`|
|Python API update|`copier.run_update(dst, defaults=True)`|

---

## Related Notes

- [[Python — Jinja2 Package]]

## Tags

#python #copier #templating #scaffolding #project-template #devtools #backend