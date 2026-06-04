## What & When

**SlowAPI** is a rate-limiting library for **FastAPI** and **Starlette** (adapted from Flask-Limiter). It caps how often clients can call endpoints — by IP, API key, user id, or custom keys — and returns **429 Too Many Requests** when limits are exceeded.

Use SlowAPI when:

- You need **per-route** limits (`/login` stricter than `/health`)
- You want a **global default** on all routes with selective exemptions
- You run **multiple Uvicorn workers** and need a **shared backend** ([[DB — Redis]])
- You prefer **decorators** over hand-rolled Redis counters in every handler

```bash
pip install slowapi
# Distributed limits across workers
pip install slowapi redis
```

Overview: [[API - FastAPI]]. For raw Redis counters without SlowAPI, see [[DB — Redis#Rate Limiting (Simple)]]. Heavy async job throttling → [[Processing — Celery]].

---

## SlowAPI vs Alternatives

| Approach | Best when |
| --- | --- |
| **SlowAPI** | FastAPI-native decorators, OpenAPI routes, shared limits |
| **Redis INCR** | Custom logic, non-HTTP limits | [[DB — Redis]] |
| **API gateway** | Central limits before app (Cloud Armor, Kong) | [[GCP]], infra |
| **Depends() only** | One-off limit in a single route | [[API - FastAPI — Dependency Injection & User Management]] |

---

## Minimal Setup

```python
from fastapi import FastAPI, Request
from fastapi.responses import PlainTextResponse
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.errors import RateLimitExceeded
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)
app = FastAPI()
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)

# Route decorator FIRST, then @limiter.limit (order matters)
@app.get("/home")
@limiter.limit("5/minute")
async def homepage(request: Request):
    return PlainTextResponse("test")
```

**Rules:**

1. Handlers with `@limiter.limit` must include **`request: Request`** (SlowAPI hooks into it).
2. Put **`@app.get` / `@app.post` above** `@limiter.limit`, not below.
3. For dict/JSON responses where headers should include `X-RateLimit-*`, also pass **`response: Response`** when `headers_enabled=True`.

---

## Per-Route Limits

```python
from fastapi import FastAPI, Request
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.errors import RateLimitExceeded
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)
app = FastAPI()
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)

@app.get("/login")
@limiter.limit("3/minute")
async def login(request: Request):
    return {"message": "Login attempts capped"}

@app.get("/data")
@limiter.limit("100/hour")
async def data(request: Request):
    return {"message": "Higher quota endpoint"}

@app.get("/health")
@limiter.exempt
async def health(request: Request):
    return {"status": "ok"}
```

Supported limit strings: `"5/second"`, `"10/minute"`, `"100/hour"`, `"1000/day"`, etc.

---

## Global Default + Middleware

Apply a baseline limit to every route; exempt health/docs as needed.

```python
from fastapi import FastAPI, Request
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.errors import RateLimitExceeded
from slowapi.middleware import SlowAPIMiddleware
from slowapi.util import get_remote_address

limiter = Limiter(
    key_func=get_remote_address,
    default_limits=["60/minute"],
)
app = FastAPI()
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)
app.add_middleware(SlowAPIMiddleware)

@app.get("/api/items")
async def list_items(request: Request):
    return {"items": []}

@app.get("/health")
@limiter.exempt
async def health(request: Request):
    return {"status": "ok"}
```

For ASGI-first stacks, docs also mention `SlowAPIASGIMiddleware` — prefer when recommended in your SlowAPI version for performance.

---

## Custom Key — API Key or User

Rate limit by header or authenticated user instead of IP.

```python
from fastapi import FastAPI, Request, Header, HTTPException

def get_api_key(request: Request) -> str:
    key = request.headers.get("X-API-Key")
    if not key:
        raise HTTPException(status_code=401, detail="Missing X-API-Key")
    return key

limiter = Limiter(key_func=get_api_key)
app = FastAPI()
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)

@app.get("/v1/resource")
@limiter.limit("1000/hour")
async def resource(request: Request):
    return {"ok": True}
```

Combine with JWT deps from [[API - FastAPI — Dependency Injection & User Management]]:

```python
def get_user_id(request: Request) -> str:
    # After auth middleware sets request.state.user_id
    return getattr(request.state, "user_id", None) or get_remote_address(request)

limiter = Limiter(key_func=get_user_id)
```

---

## Dynamic Limits (Tiered Plans)

```python
def tiered_limit(request: Request) -> str:
    if request.headers.get("X-API-Plan") == "premium":
        return "100/minute"
    return "10/minute"

@app.get("/resource")
@limiter.limit(tiered_limit)
async def resource(request: Request):
    return {"message": "Plan-based limit"}
```

---

## Redis Backend (Multi-Worker / Production)

In-memory storage breaks across processes — use Redis when running Gunicorn + multiple Uvicorn workers.

```python
import os
from fastapi import FastAPI, Request
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.errors import RateLimitExceeded
from slowapi.util import get_remote_address

REDIS_URL = os.getenv("REDIS_URL", "redis://localhost:6379/0")

limiter = Limiter(
    key_func=get_remote_address,
    storage_uri=REDIS_URL,
    default_limits=["120/minute"],
)
app = FastAPI()
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)
```

Docker Compose: run Redis alongside the API — [[Commands/CLI — Docker & Compose]], [[DB — Redis]].

---

## APIRouter (Modular Apps)

Attach the same `limiter` instance to routers — [[API - FastAPI — Routers & Modular Applications]].

```python
from fastapi import APIRouter, Request

router = APIRouter(prefix="/users")

@router.get("/")
@limiter.limit("30/minute")
async def list_users(request: Request):
    return []

# In main.py:
# app.include_router(router)
# app.state.limiter = limiter  # set once on app
```

Ensure `app.state.limiter` is set on the **parent** `FastAPI` app before mounting routers.

---

## Rate Limit Headers

```python
from fastapi import Response

limiter = Limiter(
    key_func=get_remote_address,
    headers_enabled=True,
)

@app.get("/mars")
@limiter.limit("5/minute")
async def mars(request: Request, response: Response):
    return {"key": "value"}
```

Clients can read `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset` when enabled.

---

## Shared Limit Across Routes

```python
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)

@app.get("/a")
@limiter.shared_limit("10/minute", scope="reports")
async def report_a(request: Request):
    return {"route": "a"}

@app.get("/b")
@limiter.shared_limit("10/minute", scope="reports")
async def report_b(request: Request):
    return {"route": "b"}
```

Both routes share one counter for scope `"reports"`.

---

## Custom 429 Response

```python
from fastapi import Request
from fastapi.responses import JSONResponse
from slowapi.errors import RateLimitExceeded

def rate_limit_handler(request: Request, exc: RateLimitExceeded):
    return JSONResponse(
        status_code=429,
        content={
            "error": "rate_limit_exceeded",
            "detail": str(exc.detail),
        },
    )

app.add_exception_handler(RateLimitExceeded, rate_limit_handler)
```

Or use built-in `_rate_limit_exceeded_handler` for default behavior.

---

## Lifespan + Limiter

Create Redis-backed limiter after config loads — [[API - FastAPI — Lifespan]].

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.errors import RateLimitExceeded
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)  # storage_uri set in lifespan if needed

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Optional: re-init with Redis URL from settings
    app.state.limiter = limiter
    yield

app = FastAPI(lifespan=lifespan)
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)
```

---

## Common Mistakes

| Mistake | Fix |
| --- | --- |
| `@limiter.limit` above `@app.get` | Route decorator must be **first** |
| Missing `request: Request` | Add explicit `request` parameter |
| Limits not shared across workers | `storage_uri="redis://..."` |
| Health check returns 429 | `@limiter.exempt` on `/health` |
| Login not throttled | Stricter limit on auth routes |
| Testing flakes | `limiter.enabled = False` in test app factory |
| Load test returns 429 | Expected — tune test rate or use dedicated test env — [[Load Testing]] |

```python
# Disable in tests
limiter = Limiter(key_func=get_remote_address, enabled=False)
```

---

## Quick Reference

| Task | Code |
| --- | --- |
| Install | `pip install slowapi` |
| Key by IP | `key_func=get_remote_address` |
| Per route | `@limiter.limit("5/minute")` |
| Global default | `default_limits=["60/minute"]` + `SlowAPIMiddleware` |
| Exempt | `@limiter.exempt` |
| Redis | `storage_uri="redis://localhost:6379/0"` |
| 429 handler | `app.add_exception_handler(RateLimitExceeded, ...)` |
| Headers | `headers_enabled=True` + `response: Response` |

---

## Tags

#fastapi #slowapi #rate-limiting #redis #security #backend #api-design
