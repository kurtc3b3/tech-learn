## Why Modular Structure?

Putting everything in `main.py` breaks down fast. FastAPI's `APIRouter` lets you split routes into separate modules that are composed together at startup — each with its own prefix, tags, dependencies, and responses.

Benefits:

- Each domain (users, orders, products) lives in its own file
- Dependencies and middleware can be scoped per router
- Teams can work on separate routers without conflicts
- Easier to test routes in isolation

---

## Project Structure

A clean, scalable FastAPI project layout:

```
myapp/
├── main.py
├── core/
│   ├── config.py          # Settings / env vars
│   └── dependencies.py    # Shared dependencies (get_db, get_settings)
├── routers/
│   ├── __init__.py
│   ├── users.py
│   ├── items.py
│   └── orders.py
├── models/
│   ├── user.py            # SQLAlchemy models
│   └── item.py
├── schemas/
│   ├── user.py            # Pydantic schemas
│   └── item.py
└── db.py                  # DB engine & session
```

---

## Defining a Router

```python
# routers/items.py
from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy.ext.asyncio import AsyncSession
from core.dependencies import get_db
from schemas.item import ItemCreate, ItemResponse, ItemUpdate

router = APIRouter(
    prefix="/items",            # all routes prefixed with /items
    tags=["items"],             # grouped under "items" in Swagger
    responses={404: {"description": "Item not found"}},  # shared response docs
)

@router.get("/", response_model=list[ItemResponse])
async def list_items(db: AsyncSession = Depends(get_db)):
    ...

@router.get("/{item_id}", response_model=ItemResponse)
async def get_item(item_id: int, db: AsyncSession = Depends(get_db)):
    ...

@router.post("/", response_model=ItemResponse, status_code=status.HTTP_201_CREATED)
async def create_item(payload: ItemCreate, db: AsyncSession = Depends(get_db)):
    ...

@router.patch("/{item_id}", response_model=ItemResponse)
async def update_item(item_id: int, payload: ItemUpdate, db: AsyncSession = Depends(get_db)):
    ...

@router.delete("/{item_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_item(item_id: int, db: AsyncSession = Depends(get_db)):
    ...
```

---

## Shared Dependencies on a Router

Apply a dependency to **every route** in a router without repeating `Depends` on each handler.

```python
# routers/admin.py
from fastapi import APIRouter, Depends
from core.dependencies import require_admin

router = APIRouter(
    prefix="/admin",
    tags=["admin"],
    dependencies=[Depends(require_admin)],   # every route requires admin
)

@router.get("/stats")
async def get_stats(): ...         # require_admin runs automatically

@router.delete("/users/{id}")
async def delete_user(id: int): ...  # require_admin runs automatically
```

---

## Registering Routers in `main.py`

```python
# main.py
from contextlib import asynccontextmanager
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from routers import items, users, orders, admin

@asynccontextmanager
async def lifespan(app: FastAPI):
    # startup / shutdown
    yield

app = FastAPI(title="My Service", lifespan=lifespan)

app.add_middleware(CORSMiddleware, allow_origins=["*"], allow_methods=["*"], allow_headers=["*"])

app.include_router(users.router)
app.include_router(items.router)
app.include_router(orders.router)
app.include_router(admin.router)
```

Resulting routes:

```
GET     /users/
POST    /users/
GET     /users/{id}
GET     /items/
POST    /items/
GET     /items/{id}
PATCH   /items/{id}
DELETE  /items/{id}
GET     /orders/
...
GET     /admin/stats
DELETE  /admin/users/{id}
```

---

## Adding a Prefix at Registration Time

You can set or override the prefix when including the router — useful when the router itself doesn't declare one.

```python
app.include_router(items.router, prefix="/api/v1/items", tags=["items"])
app.include_router(users.router, prefix="/api/v1/users", tags=["users"])
```

Or version the whole API by nesting routers:

```python
# routers/__init__.py
from fastapi import APIRouter
from routers import items, users, orders

v1_router = APIRouter(prefix="/api/v1")
v1_router.include_router(items.router)
v1_router.include_router(users.router)
v1_router.include_router(orders.router)
```

```python
# main.py
from routers import v1_router
app.include_router(v1_router)
```

---

## Shared Dependencies Module

Centralise reusable dependencies so routers import from one place.

```python
# core/dependencies.py
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from sqlalchemy.ext.asyncio import AsyncSession
from db import AsyncSessionLocal

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/auth/login")

async def get_db() -> AsyncSession:
    async with AsyncSessionLocal() as session:
        yield session

async def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: AsyncSession = Depends(get_db),
) -> User:
    # decode JWT, look up user
    ...

async def require_admin(user: User = Depends(get_current_user)) -> User:
    if not user.is_superuser:
        raise HTTPException(status_code=403, detail="Admins only")
    return user
```

---

## Router-Specific Response Codes & Docs

Document expected responses per router — these appear in Swagger for every route in the router.

```python
router = APIRouter(
    prefix="/orders",
    tags=["orders"],
    responses={
        401: {"description": "Not authenticated"},
        403: {"description": "Not authorised"},
        404: {"description": "Order not found"},
        422: {"description": "Validation error"},
    },
)
```

---

## Nested Routers

Routers can include other routers — useful for grouping sub-resources.

```python
# routers/users.py
from fastapi import APIRouter
from routers import user_addresses, user_orders

router = APIRouter(prefix="/users", tags=["users"])

router.include_router(
    user_addresses.router,
    prefix="/{user_id}/addresses",
    tags=["user addresses"],
)
router.include_router(
    user_orders.router,
    prefix="/{user_id}/orders",
    tags=["user orders"],
)

# Resulting routes:
# GET /users/
# GET /users/{user_id}
# GET /users/{user_id}/addresses/
# GET /users/{user_id}/orders/
```

---

## Separating Schemas by Domain

Keep Pydantic schemas co-located with their router domain.

```python
# schemas/item.py
from pydantic import BaseModel, Field
from typing import Optional
from datetime import datetime

class ItemBase(BaseModel):
    name: str = Field(..., min_length=1)
    price: float = Field(..., gt=0)

class ItemCreate(ItemBase):
    pass

class ItemUpdate(BaseModel):
    name: Optional[str] = None
    price: Optional[float] = Field(default=None, gt=0)

class ItemResponse(ItemBase):
    id: int
    created_at: datetime
    model_config = ConfigDict(from_attributes=True)
```

---

## Full Modular Example

```
GET  /api/v1/users/              → list users
POST /api/v1/users/              → create user
GET  /api/v1/users/me            → current user profile
GET  /api/v1/users/{id}          → get user
GET  /api/v1/users/{id}/orders/  → orders for a user

GET  /api/v1/items/              → list items
POST /api/v1/items/              → create item
GET  /api/v1/items/{id}          → get item
PATCH /api/v1/items/{id}         → update item
DELETE /api/v1/items/{id}        → delete item

GET  /admin/stats                → admin only
DELETE /admin/users/{id}         → admin only
```

Each domain in its own file. Each file testable independently. Swagger groups them cleanly by tag.

---

## Router Checklist

- [ ] `prefix` set on router or at `include_router` time
- [ ] `tags` set for Swagger grouping
- [ ] Shared `dependencies` declared at router level, not per-handler
- [ ] `responses` documented for common error codes
- [ ] Schemas imported from `schemas/` not defined inline
- [ ] `get_db` and auth dependencies imported from `core/dependencies.py`
- [ ] Router registered in `main.py` via `app.include_router`

---
## Tags

#fastapi #python #routers #modular #project-structure #backend #api-design