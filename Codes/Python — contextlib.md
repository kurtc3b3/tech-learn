## What & When

**contextlib** is Python's standard library module for working with **context managers** — the `with` / `async with` protocol that guarantees setup and teardown (open/close, connect/disconnect, lock/unlock) even when errors occur.

Use contextlib when:

- Writing reusable setup/teardown without a full class (`@contextmanager`, `@asynccontextmanager`)
- Composing multiple context managers dynamically (`ExitStack`, `AsyncExitStack`)
- Silencing expected exceptions during cleanup (`suppress`)
- Providing a no-op context for tests (`nullcontext`)
- FastAPI **lifespan** (startup/shutdown) — [[API - FastAPI — Lifespan]]
- Yielding DB sessions, HTTP clients, or locks from dependencies

```python
from contextlib import (
    contextmanager, asynccontextmanager,
    ExitStack, AsyncExitStack,
    suppress, nullcontext, closing,
    redirect_stdout, redirect_stderr,
    AbstractContextManager, AbstractAsyncContextManager,
)
```

No installation needed — `contextlib` is part of the standard library.

> **Note:** There is no separate `asynccontextlib` package in the stdlib. Async context managers use **`contextlib.asynccontextmanager`** and **`AsyncExitStack`** in the same module. Use with [[Python — asyncio]].

For decorator utilities on plain functions, see [[Python — functools]].

---

## Context Manager Protocol

```python
# Any object with __enter__/__exit__ (sync) or __aenter__/__aexit__ (async)

with open("file.txt") as f:       # sync
    data = f.read()

async with httpx.AsyncClient() as client:   # async
    r = await client.get("/")
```

| Phase | Sync | Async |
| --- | --- | --- |
| Enter | `__enter__()` → bound value | `await __aenter__()` |
| Body | run block | run block |
| Exit | `__exit__(exc_type, exc, tb)` | `await __aexit__(...)` |

If `__exit__` / `__aexit__` returns **True**, the exception is suppressed (usually return `None`/False).

---

## `@contextmanager` — Sync Generator Context Managers

Turn a generator with a single `yield` into a context manager.

```python
from contextlib import contextmanager

@contextmanager
def managed_resource(name: str):
    print(f"acquire {name}")
    try:
        yield name                    # value bound to `as` target
    finally:
        print(f"release {name}")

with managed_resource("db") as res:
    print(f"using {res}")
# acquire db → using db → release db
```

```python
# Timer / logging wrapper
import time
from contextlib import contextmanager

@contextmanager
def timed(label: str):
    start = time.perf_counter()
    try:
        yield
    finally:
        print(f"{label}: {time.perf_counter() - start:.3f}s")

with timed("query"):
    run_query()
```

```python
# Yield a resource — session pattern
@contextmanager
def get_session():
    session = SessionLocal()
    try:
        yield session
        session.commit()
    except Exception:
        session.rollback()
        raise
    finally:
        session.close()

with get_session() as session:
    session.add(User(name="Alice"))
```

See [[ORM - Setup]], [[ORM - CRUD]].

---

## `@asynccontextmanager` — Async Generator Context Managers

Same pattern for `async with` — **must** use `async def` + `yield`.

```python
from contextlib import asynccontextmanager

@asynccontextmanager
async def get_async_session():
    session = AsyncSessionLocal()
    try:
        yield session
        await session.commit()
    except Exception:
        await session.rollback()
        raise
    finally:
        await session.close()

async with get_async_session() as session:
    session.add(User(name="Alice"))
```

See [[ORM - Async]].

---

## FastAPI Lifespan

The canonical backend use — startup before `yield`, shutdown after.

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI
import httpx

@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.http = httpx.AsyncClient(base_url="https://api.example.com")
    yield
    await app.state.http.aclose()

app = FastAPI(lifespan=lifespan)
```

Full patterns: [[API - FastAPI — Lifespan]], [[Python — httpx Package]].

---

## `ExitStack` — Dynamic Sync Context Managers

Enter multiple context managers — including a **runtime-dependent** number — in one `with` block.

```python
from contextlib import ExitStack

files = ["a.txt", "b.txt", "c.txt"]

with ExitStack() as stack:
    handles = [stack.enter_context(open(f)) for f in files]
    for f in handles:
        process(f.read())
# all files closed automatically
```

```python
# Callback on exit — register cleanup without a CM class
from contextlib import ExitStack

with ExitStack() as stack:
    conn = open_connection()
    stack.callback(conn.close)
    stack.callback(log.info, "connection closed")
    do_work(conn)
```

```python
# Conditional resources
with ExitStack() as stack:
    if use_cache:
        cache = stack.enter_context(connect_redis())
    db = stack.enter_context(get_session())
    ...
```

---

## `AsyncExitStack` — Dynamic Async Context Managers

Async equivalent for `async with`.

```python
from contextlib import AsyncExitStack
import httpx

async def fetch_all(urls: list[str]) -> list[dict]:
    async with AsyncExitStack() as stack:
        client = await stack.enter_async_context(httpx.AsyncClient())
        tasks = [client.get(url) for url in urls]
        responses = await asyncio.gather(*tasks)
        return [r.json() for r in responses]
```

```python
async with AsyncExitStack() as stack:
    db = await stack.enter_async_context(get_async_session())
    redis = await stack.enter_async_context(connect_redis())
    stack.push_async_callback(redis.close)
    await run(db, redis)
```

---

## `suppress` — Ignore Expected Exceptions

Cleaner than empty `except` blocks during cleanup or parsing.

```python
from contextlib import suppress
import os

with suppress(FileNotFoundError):
    os.remove("/tmp/stale.lock")

# equivalent to:
# try:
#     os.remove(...)
# except FileNotFoundError:
#     pass
```

```python
# Parse optional JSON field
with suppress(json.JSONDecodeError, KeyError):
    value = json.loads(raw)["key"]
else:
    value = default
```

> [!tip] Only suppress **specific**, expected exceptions Never use bare `suppress(Exception)` in production paths.

---

## `nullcontext` — No-Op Context Manager

Returns a given value (or `None`) — useful when a context manager is optional.

```python
from contextlib import nullcontext

def process(use_lock: bool):
    lock = Lock() if use_lock else nullcontext()
    with lock:
        mutate_shared_state()
```

```python
# Tests — swap real CM for noop
@pytest.fixture
def db_session(use_real_db):
    cm = get_session if use_real_db else nullcontext(mock_session())
    with cm() as session:
        yield session
```

---

## `closing` — Context Manager for `.close()` Objects

Wraps any object with a `.close()` method.

```python
from contextlib import closing
from urllib.request import urlopen

with closing(urlopen("https://example.com")) as response:
    data = response.read()
# response.close() called
```

Prefer native context managers (`with open(...)`, `async with client`) when available.

---

## `redirect_stdout` / `redirect_stderr`

Temporarily redirect stdio — common in tests and CLI capture.

```python
from contextlib import redirect_stdout
import io

buffer = io.StringIO()
with redirect_stdout(buffer):
    print("hidden")

buffer.getvalue()    # "hidden\n"
```

```python
from contextlib import redirect_stderr

with redirect_stderr(open("errors.log", "w")):
    noisy_function()
```

---

## `contextlib.chdir` (Python 3.11+)

Temporarily change working directory.

```python
from contextlib import chdir
from pathlib import Path

with chdir(Path("/tmp")):
    run_tool_expecting_cwd_here()
# restored to previous cwd
```

For older Python, use [[Python — pathlib]] + manual `try/finally`.

---

## `ContextDecorator` — Context Manager as Decorator

Apply setup/teardown to a whole function.

```python
from contextlib import ContextDecorator

class debug(ContextDecorator):
    def __enter__(self):
        print(">>> enter")
        return self

    def __exit__(self, *exc):
        print("<<< exit")
        return False

@debug()
def compute():
    print("working")

compute()    # enter → working → exit
```

---

## Common Backend Patterns

### DB session dependency (FastAPI)

```python
from contextlib import asynccontextmanager
from fastapi import Depends

@asynccontextmanager
async def get_db():
    async with AsyncSessionLocal() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise

async def db_dep(session=Depends(get_db)):
    yield session   # FastAPI treats async generator deps as CM-like
```

See [[API - FastAPI — Dependency Injection & User Management]].

### Shared httpx client in lifespan

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    async with httpx.AsyncClient(timeout=10.0) as client:
        app.state.client = client
        yield
```

### Timed + logged block

```python
from contextlib import contextmanager
import logging

logger = logging.getLogger(__name__)

@contextmanager
def log_duration(operation: str):
    logger.info("start %s", operation)
    try:
        yield
    finally:
        logger.info("done %s", operation)
```

See [[Python — logging]].

### Nested resource cleanup with `AsyncExitStack`

```python
@asynccontextmanager
async def app_resources(settings):
    async with AsyncExitStack() as stack:
        engine = await stack.enter_async_context(create_engine_cm(settings.db_url))
        redis = await stack.enter_async_context(connect_redis_cm(settings.redis_url))
        yield {"engine": engine, "redis": redis}
```

### Retry test isolation with `suppress`

```python
from contextlib import suppress

def teardown_test_artifacts():
    with suppress(OSError):
        shutil.rmtree(temp_dir)
```

Combine with [[Python — tenacity]] for retry logic (different concern — tenacity retries calls; contextlib manages resource scope).

---

## Sync vs Async — Quick Comparison

| Task | Sync | Async |
| --- | --- | --- |
| Write CM from generator | `@contextmanager` | `@asynccontextmanager` |
| Dynamic stack | `ExitStack` | `AsyncExitStack` |
| Enter CM on stack | `stack.enter_context(cm)` | `await stack.enter_async_context(cm)` |
| Register cleanup | `stack.callback(fn)` | `stack.push_async_callback(fn)` |
| Usage | `with ...` | `async with ...` |

---

## Common Mistakes

| Mistake | Fix |
| --- | --- |
| `@contextmanager` not wrapping `yield` in `try/finally` | Always release resources in `finally` after `yield` |
| Using sync `@contextmanager` for async I/O | Use `@asynccontextmanager` + `async with` |
| Forgetting `await` on async exit | Use `async with`, not plain `with` |
| `yield` more than once | Single `yield` only — not a regular generator |
| Swallowing all errors with `suppress(Exception)` | List specific exception types |
| Storing async CM without entering | `async with` or `AsyncExitStack.enter_async_context` |

---

## Quick Reference

| Task | Code |
| --- | --- |
| Sync CM from generator | `@contextmanager` + `yield` |
| Async CM from generator | `@asynccontextmanager` + `async yield` |
| Multiple CMs dynamically | `ExitStack()` / `AsyncExitStack()` |
| Ignore exception | `with suppress(ExcType):` |
| Optional CM | `nullcontext(default)` |
| Object with `.close()` | `closing(obj)` |
| Capture stdout in tests | `redirect_stdout(buffer)` |
| FastAPI startup/shutdown | `@asynccontextmanager` lifespan |
| Register exit callback | `stack.callback(fn, *args)` |
| Temp cwd (3.11+) | `with chdir(path):` |

---

## Tags

#python #contextlib #asynccontextmanager #stdlib #context-managers #async #fastapi #backend
