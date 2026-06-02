
## Sub-Applications & Mounting

FastAPI apps are ASGI applications. Any ASGI-compatible app can be **mounted** onto a parent FastAPI instance at a given path prefix. This enables:

- Splitting a large API into independent FastAPI apps
- Serving static files
- Mounting third-party ASGI apps (Starlette, Django, etc.)
- Independent OpenAPI docs per sub-application

---

## `mount` vs `include_router`

|                     | `include_router`            | `mount`                     |
| ------------------- | --------------------------- | --------------------------- |
| Type                | FastAPI routers             | Any ASGI app                |
| Shared middleware   | ✅ Yes                       | ❌ No — isolated             |
| Shared OpenAPI docs | ✅ Yes                       | ❌ Separate docs             |
| Use case            | Splitting routes in one app | Truly separate applications |

> [!tip] Prefer `include_router` for most cases. Use `mount` only when you need full isolation or are mounting a non-FastAPI app.

---

## `include_router` — Splitting Routes (Most Common)

```python
# routers/items.py
from fastapi import APIRouter

router = APIRouter(prefix="/items", tags=["items"])

@router.get("/")
async def list_items(): ...

@router.get("/{id}")
async def get_item(id: int): ...
```

```python
# main.py
from fastapi import FastAPI
from routers import items, users, orders

app = FastAPI()

app.include_router(items.router)
app.include_router(users.router)
app.include_router(orders.router)
```

All routes share the same middleware, dependencies, and OpenAPI docs.

---

## Mounting a Sub-Application

A mounted app is completely independent — its own middleware, its own docs, its own lifespan.

```python
from fastapi import FastAPI

# Main app
app = FastAPI()

# Sub-application
api_v1 = FastAPI(title="API v1", version="1.0")
api_v2 = FastAPI(title="API v2", version="2.0")

@api_v1.get("/users")
async def v1_users(): return {"version": "v1"}

@api_v2.get("/users")
async def v2_users(): return {"version": "v2"}

# Mount at a path prefix
app.mount("/v1", api_v1)
app.mount("/v2", api_v2)
```

Each sub-app gets its own docs:

- `/v1/docs` → Swagger for v1
- `/v2/docs` → Swagger for v2
- `/docs` → Main app docs (v1 and v2 routes won't appear here)

> [!warning] Routes on a mounted app are invisible to the parent's OpenAPI schema. If you need unified docs, use `include_router` instead.

---

## Mounting Static Files

```python
from fastapi import FastAPI
from fastapi.staticfiles import StaticFiles

app = FastAPI()

app.mount("/static", StaticFiles(directory="static"), name="static")
# Files in ./static/ are served at http://localhost:8000/static/...
```

For a Single Page Application (SPA) where all routes should serve `index.html`:

```python
app.mount("/", StaticFiles(directory="dist", html=True), name="spa")
```

> [!warning] Mount static files **last**. Static file mounts are path-greedy — mount them after all API routes or they will swallow API requests.

---

## API Versioning Pattern

A common real-world pattern — independent versioned sub-apps with shared utilities.

```python
from fastapi import FastAPI
from routers.v1 import router as v1_router
from routers.v2 import router as v2_router

app = FastAPI(title="My Service")

v1 = FastAPI(title="My Service — v1")
v2 = FastAPI(title="My Service — v2")

v1.include_router(v1_router)
v2.include_router(v2_router)

app.mount("/v1", v1)
app.mount("/v2", v2)
```

---

## CORS — Cross-Origin Resource Sharing

**CORS** is a browser security mechanism. When a frontend at `http://localhost:3000` makes a fetch to `http://localhost:8000`, the browser blocks it unless the server explicitly permits it via CORS headers.

FastAPI handles CORS via Starlette's `CORSMiddleware`.

---

## Adding CORS Middleware

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000", "https://myapp.com"],
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"],
    allow_headers=["Authorization", "Content-Type"],
)
```

---

## CORS Options Explained

```python
app.add_middleware(
    CORSMiddleware,

    # Which origins are allowed to make requests
    allow_origins=["https://myapp.com"],

    # Allow cookies / Authorization headers to be sent cross-origin
    allow_credentials=True,

    # Which HTTP methods are permitted
    allow_methods=["*"],

    # Which request headers the browser is allowed to send
    allow_headers=["*"],

    # How long (seconds) the browser can cache the preflight response
    expose_headers=["X-Custom-Header"],
    max_age=600,
)
```

|Option|Description|
|---|---|
|`allow_origins`|List of permitted origins, or `["*"]` for all|
|`allow_credentials`|Allow cookies and auth headers cross-origin|
|`allow_methods`|Permitted HTTP verbs, or `["*"]`|
|`allow_headers`|Permitted request headers, or `["*"]`|
|`expose_headers`|Response headers the browser can read|
|`max_age`|Preflight cache duration in seconds|

> [!warning] `allow_origins=["*"]` with `allow_credentials=True` is invalid. Browsers reject this combination. If you need credentials, you must list origins explicitly.

---

## CORS for Development vs Production

```python
import os
from fastapi.middleware.cors import CORSMiddleware

ENVIRONMENT = os.getenv("ENVIRONMENT", "development")

if ENVIRONMENT == "development":
    origins = ["*"]                         # permissive in dev
else:
    origins = [                             # strict in production
        "https://myapp.com",
        "https://www.myapp.com",
    ]

app.add_middleware(
    CORSMiddleware,
    allow_origins=origins,
    allow_credentials=ENVIRONMENT != "development",
    allow_methods=["*"],
    allow_headers=["*"],
)
```

---

## Middleware Order Matters

FastAPI processes middleware in **reverse registration order** (last added = first to run). CORS must run before anything that might reject a request, so add it first.

```python
app.add_middleware(CORSMiddleware, ...)       # added first → runs first
app.add_middleware(AuthMiddleware, ...)      # added second → runs second
app.add_middleware(LoggingMiddleware, ...)   # added last → runs last
```

> [!warning] If auth middleware runs before CORS, preflight `OPTIONS` requests from the browser will be rejected before CORS headers are added — breaking the browser's preflight check entirely.

---

## CORS + Mounted Sub-Apps

CORS middleware on the parent app does **not** apply to mounted sub-apps. Each mounted app needs its own middleware.

```python
app = FastAPI()
app.add_middleware(CORSMiddleware, allow_origins=["*"], allow_methods=["*"], allow_headers=["*"])

v1 = FastAPI()
v1.add_middleware(CORSMiddleware, allow_origins=["*"], allow_methods=["*"], allow_headers=["*"])
# ↑ required separately — parent middleware doesn't cascade

app.mount("/v1", v1)
```

---

## Full Pattern — Versioned App with CORS

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from fastapi.staticfiles import StaticFiles

def make_cors(app: FastAPI, origins: list[str]):
    app.add_middleware(
        CORSMiddleware,
        allow_origins=origins,
        allow_credentials=True,
        allow_methods=["*"],
        allow_headers=["*"],
    )

ORIGINS = ["https://myapp.com"]

# Sub-apps
v1 = FastAPI(title="API v1")
v2 = FastAPI(title="API v2")
make_cors(v1, ORIGINS)
make_cors(v2, ORIGINS)

# Main app
app = FastAPI(title="My Service")
make_cors(app, ORIGINS)

app.mount("/v1", v1)
app.mount("/v2", v2)
app.mount("/static", StaticFiles(directory="static"), name="static")  # always last
```

---
## Tags

#fastapi #python #cors #mount #sub-applications #middleware #backend #api-versioning