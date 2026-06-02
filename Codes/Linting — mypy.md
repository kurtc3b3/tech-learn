## What & When

**mypy** is Python's standard **static type checker**. It analyzes type hints ([[Python — typing]]) without running code — catching wrong arguments, missing `None` checks, and broken interfaces before tests or production.

Use mypy when:

- The project uses type hints (required for [[Python — Pydantic]], [[API - FastAPI]])
- Refactoring large codebases safely
- Enforcing contracts on [[Python — abc]] implementations
- CI must fail on type errors

```bash
pip install mypy
# common plugins
pip install pydantic mypy          # pydantic.mypy plugin
```

Ruff catches some typing **syntax** issues; mypy performs **semantic** analysis. Use both: [[Linting — Ruff]] + mypy. Overview: [[Linting]].

---

## mypy vs Ruff vs Runtime

| Layer | Tool | Example catch |
| --- | --- | --- |
| Style / imports | [[Linting — Ruff]] | Unused import |
| Static types | **mypy** | `str` passed where `int` expected |
| Runtime validation | [[Python — Pydantic]] | Invalid JSON body |

mypy is optional at runtime — hints are ignored unless you use a checker or Pydantic.

---

## Basic Usage

```bash
# Check packages
mypy src

# Include tests
mypy src tests

# Specific file
mypy src/app/main.py

# Verbose error codes
mypy src --show-error-codes

# Install missing stub packages hint
mypy src --install-types --non-interactive
```

With [[Python — uv]]:

```bash
uv run mypy src tests
```

---

## `pyproject.toml` Configuration

```toml
[tool.mypy]
python_version = "3.11"
strict = true
warn_unreachable = true
warn_unused_ignores = true
disallow_untyped_defs = true
disallow_any_generics = true
plugins = ["pydantic.mypy"]

[[tool.mypy.overrides]]
module = "tests.*"
disallow_untyped_defs = false

[[tool.mypy.overrides]]
module = "third_party.*"
ignore_missing_imports = true
```

### Strict mode (`strict = true`) enables:

- `disallow_untyped_defs`
- `disallow_any_generics`
- `no_implicit_optional`
- `warn_return_any`
- and more — good default for greenfield backend services

---

## Common Error Codes

| Code | Meaning | Fix |
| --- | --- | --- |
| `[arg-type]` | Wrong argument type | Fix call or widen hint |
| `[return-value]` | Return type mismatch | Fix return or annotation |
| `[assignment]` | Incompatible assignment | Correct type or use `Union` |
| `[union-attr]` | Attribute on optional | Narrow with `if x is not None` |
| `[override]` | Subclass method signature drift | Match base class |
| `[import-untyped]` | Library has no stubs | `types-*` package or override |

```python
# error: Argument 1 has incompatible type "str"; expected "int"
def add(x: int, y: int) -> int:
    return x + y

add("1", 2)
```

---

## Pydantic Plugin

Essential for [[Python — Pydantic]] v2 models:

```toml
[tool.mypy]
plugins = ["pydantic.mypy"]

[tool.pydantic-mypy]
init_forbid_extra = true
init_typed = true
warn_required_dynamic_aliases = true
```

```python
from pydantic import BaseModel

class UserCreate(BaseModel):
    name: str
    age: int

# mypy understands model fields and validators
def greet(u: UserCreate) -> str:
    return f"{u.name} is {u.age}"
```

---

## FastAPI Patterns

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class Item(BaseModel):
    name: str
    price: float

@app.post("/items")
async def create_item(item: Item) -> Item:
    return item    # mypy verifies return matches Item
```

For untyped third-party libs, add overrides instead of `# type: ignore` everywhere:

```toml
[[tool.mypy.overrides]]
module = "some_untyped_lib.*"
ignore_missing_imports = true
```

See [[API - FastAPI — Pydantic Models]].

---

## TYPE_CHECKING & Forward References

```python
from __future__ import annotations
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from app.models import User        # import only for type checkers

def get_user_id(user: "User") -> int:
    return user.id
```

Ruff rule `TCH` helps move imports into `TYPE_CHECKING` blocks. See [[Python — typing]].

---

## Gradual Typing on Legacy Code

```toml
# Start loose, tighten over time
[tool.mypy]
python_version = "3.11"
check_untyped_defs = true
disallow_untyped_defs = false

# Per-module strict
[[tool.mypy.overrides]]
module = "app.services.*"
strict = true
```

```bash
# Count errors baseline; fail CI only on new errors (advanced)
mypy src --html-report reports/mypy
```

---

## Stub Packages (`types-*`)

```bash
pip install types-requests types-redis types-PyYAML
```

```toml
[[tool.mypy.overrides]]
module = "requests.*"
# prefer types-requests over ignore_missing_imports
```

---

## Inline Suppressions

```python
x = expensive()  # type: ignore[no-untyped-call]  # narrow code only

# Prefer file/module overrides over blanket ignores
```

Use `# type: ignore[error-code]` with **specific codes** — required in strict teams.

---

## pre-commit Hook

```yaml
repos:
  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.13.0
    hooks:
      - id: mypy
        additional_dependencies:
          - pydantic>=2
        args: [--config-file=pyproject.toml]
        files: ^src/
```

See [[Linting — pre-commit]]. Ensure hook uses same Python version and deps as project.

---

## CI

```yaml
- run: uv sync --group dev
- run: uv run mypy src tests
```

---

## Common Mistakes

| Mistake | Fix |
| --- | --- |
| Hints added but mypy never run | Add to pre-commit and CI |
| `Any` everywhere | Enable `disallow_any_generics` incrementally |
| Ignoring all imports | Install `types-*` stubs |
| mypy vs Pydantic confusion | Pydantic validates at runtime; mypy at compile time |
| Different deps in mypy hook | Mirror `additional_dependencies` in pre-commit |

---

## Quick Reference

| Task | Command / config |
| --- | --- |
| Run checker | `mypy src tests` |
| Strict config | `strict = true` in `[tool.mypy]` |
| Pydantic | `plugins = ["pydantic.mypy"]` |
| Relax tests | `[[tool.mypy.overrides]]` on `tests.*` |
| Missing stubs | `pip install types-<pkg>` |
| Ignore import | `ignore_missing_imports = true` (last resort) |
| Error codes | `mypy --show-error-codes` |

---

## Tags

#python #mypy #typing #type-checking #linting #pydantic #code-quality #backend
