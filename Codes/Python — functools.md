## What & When

**functools** is Python's standard library module for **higher-order functions** and **decorator utilities**. It helps you write reusable wrappers, memoize expensive calls, pre-bind arguments, and dispatch behavior by type — without third-party dependencies.

Use functools when:

- Writing custom decorators and preserving `__name__` / docstrings (`@wraps`)
- Caching pure function results (`@lru_cache`, `@cache`)
- Lazy-computing expensive instance attributes (`@cached_property`)
- Pre-filling callback arguments (`partial`, `partialmethod`)
- Overloading a function by argument type (`@singledispatch`)
- Filling in comparison methods from `__eq__` (`@total_ordering`)
- Folding iterables into a single value (`reduce`)

```python
from functools import (
    wraps, lru_cache, cache, cached_property,
    partial, partialmethod, reduce,
    singledispatch, singledispatchmethod, total_ordering,
)
```

No installation needed — `functools` is part of the standard library.

For retry-specific decorators, prefer [[Python — tenacity]]. For structural typing without inheritance, see [[Python — typing]].

---

## functools vs Related Tools

| Need | Use | Notes |
| --- | --- | --- |
| Preserve decorator metadata | `functools.wraps` | Always use on custom decorators |
| Cache sync pure functions | `@lru_cache` / `@cache` | Not for async; not for mutable args |
| Lazy instance attribute | `@cached_property` | Per-instance, computed once |
| Pre-bind arguments | `partial` | DI, callbacks, CLI defaults |
| Type-based dispatch | `@singledispatch` | Cleaner than long `if isinstance` chains |
| Retry on failure | [[Python — tenacity]] | Stateful, backoff, async support |
| Immutable data classes | [[Python — dataclasses]] | Different concern |

---

## `@wraps` — Correct Decorators

Without `wraps`, decorated functions lose their name and docstring — breaking introspection, logging, and FastAPI route names.

```python
from functools import wraps
import time

def timed(func):
    @wraps(func)                    # copies __name__, __doc__, __annotations__, etc.
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = func(*args, **kwargs)
        print(f"{func.__name__} took {time.perf_counter() - start:.3f}s")
        return result
    return wrapper

@timed
def fetch_user(user_id: int) -> dict:
    """Load user from database."""
    return {"id": user_id, "name": "Alice"}

print(fetch_user.__name__)   # fetch_user  (not "wrapper")
print(fetch_user.__doc__)    # Load user from database.
```

```python
# Manual equivalent
from functools import update_wrapper

def timed(func):
    def wrapper(*args, **kwargs):
        ...
    return update_wrapper(wrapper, func)
```

> [!tip] Always `@wraps` custom decorators Any decorator that wraps a function should use `@wraps` unless you intentionally want to hide the wrapped callable.

---

## `@lru_cache` & `@cache` — Memoization

Cache results of **pure** functions with hashable arguments.

```python
from functools import lru_cache

@lru_cache(maxsize=128)
def fibonacci(n: int) -> int:
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)

fibonacci(100)    # fast after first calls
```

```python
from functools import cache    # Python 3.9+ — unbounded cache (same as lru_cache(maxsize=None))

@cache
def load_config(path: str) -> dict:
    with open(path) as f:
        return json.load(f)
```

```python
# Inspect / clear cache
fibonacci.cache_info()     # CacheInfo(hits=..., misses=..., maxsize=128, currsize=...)
fibonacci.cache_clear()    # invalidate all entries

# Per-call override
@lru_cache(maxsize=32)
def get_settings(env: str):
    ...

get_settings.cache_parameters()   # {'maxsize': 32, 'typed': False}
```

| Parameter | Description |
| --- | --- |
| `maxsize` | Max entries; `None` = unlimited |
| `typed` | `True` — `f(3)` and `f(3.0)` cached separately |

**When to use:**

- Parsing config files, loading schemas, building lookup tables
- Expensive pure transforms on stable inputs

**When NOT to use:**

- Functions with side effects (HTTP calls, DB writes)
- Unhashable arguments (`dict`, `list`) — unless converted to `tuple` / `frozenset`
- Async functions — use `async-lru` or manual caching instead

```python
# Pattern: cache on hashable key derived from dict
@lru_cache(maxsize=256)
def _normalize_cached(key: tuple) -> str:
    ...

def normalize(data: dict) -> str:
    return _normalize_cached(tuple(sorted(data.items())))
```

---

## `@cached_property` — Lazy Instance Attributes

Compute a value once per instance, then store it on `__dict__`.

```python
from functools import cached_property

class UserService:
    def __init__(self, repo):
        self.repo = repo

    @cached_property
    def role_index(self) -> dict[str, int]:
        """Built once on first access — expensive scan."""
        roles = self.repo.list_all_roles()
        return {r.name: r.id for r in roles}

svc = UserService(repo)
svc.role_index          # computed here
svc.role_index          # returned from cache
del svc.role_index      # delete to force recompute on next access
```

```python
# Class-level cached property (Python 3.12+)
from functools import cached_property

class Settings:
    @classmethod
    @cached_property
    def defaults(cls) -> dict:
        return load_defaults()
```

> [!tip] Prefer `@cached_property` over manual `if not hasattr` Keeps lazy-init logic declarative and thread-safe for read-after-first-compute in CPython.

---

## `partial` & `partialmethod` — Pre-Bound Arguments

Create a new callable with some arguments fixed — useful for callbacks, DI, and CLI defaults.

```python
from functools import partial

def connect(host: str, port: int, timeout: float, ssl: bool) -> None:
    ...

# Pre-bind connection defaults
connect_prod = partial(connect, "db.example.com", 5432, timeout=5.0, ssl=True)
connect_prod(ssl=False)     # only override ssl

# Callbacks
from functools import partial

def on_event(event: str, *, logger, level: str = "info") -> None:
    getattr(logger, level)(event)

handler = partial(on_event, logger=app_logger, level="warning")
handler("disk full")        # logger.warning("disk full")
```

```python
# FastAPI / service factory pattern
def get_user_service(session, settings) -> UserService:
    return UserService(session=session, settings=settings)

# In tests — bind mock session
service = partial(get_user_service, session=mock_session)(settings=test_settings)
```

```python
from functools import partialmethod

class Window:
    def resize(self, width: int, height: int) -> None:
        ...

    maximize = partialmethod(resize, width=1920, height=1080)
```

---

## `@singledispatch` — Function Overloading by Type

Register different implementations based on the **first argument's type**.

```python
from functools import singledispatch
import json

@singledispatch
def serialize(obj) -> str:
    raise TypeError(f"Unsupported type: {type(obj)!r}")

@serialize.register
def _(obj: dict) -> str:
    return json.dumps(obj)

@serialize.register
def _(obj: list) -> str:
    return json.dumps(obj)

@serialize.register
def _(obj: str) -> str:
    return obj

@serialize.register(int)
@serialize.register(float)
def _(obj) -> str:
    return str(obj)

serialize({"a": 1})     # '{"a": 1}'
serialize(42)           # "42"
```

```python
# Register with explicit type
from datetime import datetime

@serialize.register(datetime)
def _(obj: datetime) -> str:
    return obj.isoformat()
```

Use for: serializers, formatters, validators, event handlers — anywhere `isinstance` chains grow unwieldy.

---

## `@singledispatchmethod` — Method Overloading

Same idea for methods (Python 3.8+).

```python
from functools import singledispatchmethod

class Renderer:
    @singledispatchmethod
    def render(self, value) -> str:
        raise TypeError(type(value))

    @render.register
    def _(self, value: str) -> str:
        return value

    @render.register
    def _(self, value: int) -> str:
        return f"{value:,}"

Renderer().render(1000)     # "1,000"
```

---

## `@total_ordering` — Comparison Methods from `__eq__`

If you define `__eq__` and one ordering method, `@total_ordering` generates the rest.

```python
from functools import total_ordering

@total_ordering
class Priority:
    def __init__(self, level: int):
        self.level = level

    def __eq__(self, other):
        if not isinstance(other, Priority):
            return NotImplemented
        return self.level == other.level

    def __lt__(self, other):
        if not isinstance(other, Priority):
            return NotImplemented
        return self.level < other.level

Priority(1) < Priority(3)     # True
Priority(3) >= Priority(1)    # True
```

Requires `__eq__` and at least one of `__lt__`, `__le__`, `__gt__`, `__ge__`.

---

## `reduce` — Fold an Iterable

Apply a two-argument function cumulatively — less common now that comprehensions exist, but useful for custom aggregation.

```python
from functools import reduce

reduce(lambda acc, x: acc + x, [1, 2, 3, 4])          # 10
reduce(lambda acc, x: acc * x, [1, 2, 3, 4], 1)       # 24 — with initial value

# Merge dicts
configs = [{"debug": True}, {"timeout": 30}, {"retries": 3}]
merged = reduce(lambda a, b: {**a, **b}, configs, {})
```

Prefer `sum()`, `any()`, `all()`, or a simple loop when readable — use `reduce` when the accumulator is non-trivial.

---

## `cmp_to_key` — Legacy Comparators for `sorted`

Convert an old-style comparator to a `key` function for `sorted()` / `min()` / `max()`.

```python
from functools import cmp_to_key

def compare_length(a: str, b: str) -> int:
    return (len(a) > len(b)) - (len(a) < len(b))

sorted(["apple", "pi", "banana"], key=cmp_to_key(compare_length))
# ['pi', 'apple', 'banana']
```

Prefer `key=len` or `key=lambda x: ...` in new code — `cmp_to_key` is mainly for porting or special comparators.

---

## Common Backend Patterns

### Logging decorator with `@wraps`

```python
from functools import wraps
import logging

logger = logging.getLogger(__name__)

def log_call(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        logger.debug("calling %s", func.__name__)
        try:
            return func(*args, **kwargs)
        except Exception:
            logger.exception("error in %s", func.__name__)
            raise
    return wrapper
```

See [[Python — logging]].

### Cache settings / schema parsing

```python
from functools import lru_cache
from pathlib import Path
import json

@lru_cache(maxsize=1)
def load_openapi_spec(path: str) -> dict:
    return json.loads(Path(path).read_text(encoding="utf-8"))
```

### `partial` for FastAPI dependencies

```python
from functools import partial
from fastapi import Depends

def get_service(session=Depends(get_session), settings=Depends(get_settings)):
    return UserService(session, settings)

# Test override
app.dependency_overrides[get_service] = lambda: UserService(mock_session, test_settings)
```

See [[API - FastAPI — Dependency Injection & User Management]].

### Type-based event serialization

```python
from functools import singledispatch
from dataclasses import asdict
from datetime import datetime

@singledispatch
def to_jsonable(obj):
    raise TypeError(type(obj))

@to_jsonable.register
def _(obj: datetime) -> str:
    return obj.isoformat()

@to_jsonable.register
def _(obj: dict) -> dict:
    return obj

@to_jsonable.register
def _(obj) -> dict:
    return asdict(obj) if hasattr(obj, "__dataclass_fields__") else str(obj)
```

See [[Python — dataclasses]].

### Combine with tenacity (not functools)

Use [[Python — tenacity]] for retries; use `functools.wraps` if you write your own thin wrapper around it:

```python
from functools import wraps
from tenacity import retry, stop_after_attempt, wait_exponential

def retryable(**kwargs):
    def decorator(func):
        retried = retry(**kwargs)(func)
        @wraps(func)
        def wrapper(*args, **kw):
            return retried(*args, **kw)
        return wrapper
    return decorator
```

---

## Testing Cached Functions

```python
from functools import lru_cache

@lru_cache
def expensive(n: int) -> int:
    return n * n

def test_expensive():
    expensive.cache_clear()
    assert expensive(3) == 9
    assert expensive.cache_info().hits == 0
    expensive(3)
    assert expensive.cache_info().hits == 1
```

Always `cache_clear()` in tests when cache state could leak between cases.

---

## Quick Reference

| Task | Code |
| --- | --- |
| Decorator metadata | `@wraps(func)` on inner wrapper |
| Memoize function | `@lru_cache(maxsize=128)` or `@cache` |
| Cache stats / clear | `func.cache_info()`, `func.cache_clear()` |
| Lazy instance attr | `@cached_property` on method |
| Pre-bind args | `partial(func, arg1, kw=value)` |
| Overload by type | `@singledispatch` + `@func.register` |
| Overload method | `@singledispatchmethod` |
| Full ordering | `@total_ordering` + `__eq__` + `__lt__` |
| Fold iterable | `reduce(fn, items, initial)` |
| Sort with comparator | `sorted(items, key=cmp_to_key(cmp))` |

---

## Tags

#python #functools #stdlib #decorators #cache #patterns #backend
