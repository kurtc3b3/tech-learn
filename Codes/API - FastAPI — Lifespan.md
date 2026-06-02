## What is Lifespan?

**Lifespan** is FastAPI's hook for running code at application **startup** and **shutdown**. It replaced the older `@app.on_event("startup")` / `@app.on_event("shutdown")` decorators (deprecated since FastAPI 0.93).

Common uses:

- Initialising a DB connection pool
- Loading an ML model into memory
- Warming up a cache
- Connecting to a message broker (Redis, RabbitMQ)
- Gracefully closing all of the above on shutdown

---

## Basic Pattern

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI

@asynccontextmanager
async def lifespan(app: FastAPI):
    # --- STARTUP: runs before the app starts accepting requests ---
    print("Starting up...")
    yield
    # --- SHUTDOWN: runs after the last request is handled ---
    print("Shutting down...")

app = FastAPI(lifespan=lifespan)
```

Everything **before** `yield` = startup. Everything **after** `yield` = shutdown. The `yield` itself is where the app runs.

---

## Real-World Example — DB Pool + Redis

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI
import redis.asyncio as redis
from sqlalchemy.ext.asyncio import create_async_engine

engine = None
redis_client = None

@asynccontextmanager
async def lifespan(app: FastAPI):
    global engine, redis_client

    # Startup
    engine = create_async_engine("postgresql+asyncpg://user:pass@localhost/db")
    redis_client = await redis.from_url("redis://localhost")
    print("DB pool and Redis ready")

    yield  # app is live here

    # Shutdown
    await engine.dispose()
    await redis_client.close()
    print("Connections closed cleanly")

app = FastAPI(lifespan=lifespan)
```

---

## Sharing State via `app.state`

Resources created in lifespan can be attached to `app.state` and accessed anywhere — including inside dependencies.

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.db_engine = create_async_engine(DATABASE_URL)
    app.state.redis     = await redis.from_url(REDIS_URL)
    yield
    await app.state.db_engine.dispose()
    await app.state.redis.close()

app = FastAPI(lifespan=lifespan)
```

```python
# Access in a dependency via the Request object
from fastapi import Request

async def get_db(request: Request):
    async with AsyncSession(request.app.state.db_engine) as session:
        yield session
```

---

## Lifespan with fastapi-users

When using `fastapi-users` with SQLAlchemy, lifespan is the right place to create DB tables on startup.

```python
from app.db import Base, engine

@asynccontextmanager
async def lifespan(app: FastAPI):
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)  # create tables if missing
    yield

app = FastAPI(lifespan=lifespan)
```

---

## Why Not `@app.on_event`?

```python
# Old pattern — deprecated, avoid in new projects
@app.on_event("startup")
async def startup(): ...

@app.on_event("shutdown")
async def shutdown(): ...
```

Lifespan is preferred because:

- Startup and shutdown logic live **together** — easier to reason about
- Works naturally with `asynccontextmanager`
- Compatible with testing via `TestClient` (the lifespan runs inside `with TestClient(app)`)
- Follows the same `yield`-based pattern as FastAPI dependencies

---

## Testing with Lifespan

`TestClient` as a context manager triggers the full lifespan.

```python
from fastapi.testclient import TestClient

def test_app():
    with TestClient(app) as client:
        # lifespan startup has run here
        response = client.get("/health")
        assert response.status_code == 200
    # lifespan shutdown has run here
```

---

## Tags

#fastapi #python #lifespan #startup #shutdown #backend