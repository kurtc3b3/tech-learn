## What is Dependency Injection?

FastAPI's DI system lets you declare shared logic — DB sessions, auth, config, pagination — as **dependencies** and inject them into route handlers automatically. Dependencies are:

- Declared once, reused everywhere
- Composable — dependencies can depend on other dependencies
- Overridable — easily swapped in tests
- Async-friendly — can be `async def` or plain `def`

---

## Basic Dependency — Function

```python
from fastapi import FastAPI, Depends

app = FastAPI()

def get_query_params(skip: int = 0, limit: int = 10):
    return {"skip": skip, "limit": limit}

@app.get("/items")
async def list_items(params: dict = Depends(get_query_params)):
    # params == {"skip": 0, "limit": 10}
    return params
```

FastAPI resolves `get_query_params` before calling `list_items`, injecting query string values automatically.

---

## Class-Based Dependencies

Classes are cleaner when a dependency carries state or multiple methods.

```python
class Paginator:
    def __init__(self, skip: int = 0, limit: int = 20):
        self.skip = skip
        self.limit = limit

@app.get("/products")
async def list_products(page: Paginator = Depends(Paginator)):
    return {"skip": page.skip, "limit": page.limit}
```

---

## Database Session Dependency

The most common real-world use — a per-request DB session that is always closed, even if the handler raises an exception.

```python
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine, async_sessionmaker
from contextlib import asynccontextmanager

DATABASE_URL = "postgresql+asyncpg://user:pass@localhost/db"

engine = create_async_engine(DATABASE_URL)
AsyncSessionLocal = async_sessionmaker(engine, expire_on_commit=False)

async def get_db() -> AsyncSession:
    async with AsyncSessionLocal() as session:
        yield session          # yield hands control to the route handler
                               # code after yield runs as cleanup (teardown)

@app.get("/items/{item_id}")
async def get_item(item_id: int, db: AsyncSession = Depends(get_db)):
    result = await db.get(Item, item_id)
    if not result:
        raise HTTPException(status_code=404, detail="Not found")
    return result
```

> [!info] `yield` Dependencies Using `yield` instead of `return` turns the dependency into a context manager. Code after `yield` always runs — ideal for session cleanup, releasing locks, or closing connections.

---

## Chained / Nested Dependencies

Dependencies can themselves declare dependencies — FastAPI builds and resolves the full graph automatically.

```python
def get_db() -> AsyncSession:
    ...  # as above

def get_current_user(db: AsyncSession = Depends(get_db)) -> User:
    ...  # uses db to look up user from token

def require_admin(user: User = Depends(get_current_user)) -> User:
    if not user.is_admin:
        raise HTTPException(status_code=403, detail="Admins only")
    return user

@app.delete("/users/{user_id}")
async def delete_user(user_id: int, admin: User = Depends(require_admin)):
    # Only reachable by admins
    ...
```

Resolution order: `get_db` → `get_current_user` → `require_admin` → handler.

---

## Router-Level Dependencies

Apply a dependency to every route in a router — no need to repeat `Depends` on each handler.

```python
from fastapi import APIRouter

router = APIRouter(
    prefix="/admin",
    dependencies=[Depends(require_admin)]   # applied to all routes below
)

@router.get("/stats")
async def get_stats(): ...

@router.delete("/users/{id}")
async def remove_user(id: int): ...
```

---

## App-Level Dependencies

```python
app = FastAPI(dependencies=[Depends(verify_api_key)])
# Every single route requires a valid API key
```

---

## Overriding Dependencies in Tests

DI makes testing clean — swap real implementations for mocks without touching production code.

```python
from fastapi.testclient import TestClient

def mock_db():
    yield fake_session  # in-memory or SQLite session

app.dependency_overrides[get_db] = mock_db

client = TestClient(app)
```

---

## Settings / Config as a Dependency

Inject typed config from environment variables using `pydantic-settings`.

```python
from functools import lru_cache
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    database_url: str
    secret_key: str
    debug: bool = False

    class Config:
        env_file = ".env"

@lru_cache          # instantiated once, cached for app lifetime
def get_settings() -> Settings:
    return Settings()

@app.get("/info")
async def info(settings: Settings = Depends(get_settings)):
    return {"debug": settings.debug}
```

> [!tip] `@lru_cache` on `get_settings` Without it, Pydantic re-reads the `.env` file on every request. `lru_cache` ensures it's parsed once at startup.

---

## fastapi-users

`fastapi-users` is a complete user management library built entirely on FastAPI's DI system. It provides:

- Registration, login, logout, password reset, email verification routes
- JWT and Cookie auth backends
- Pluggable DB adapters (SQLAlchemy, Beanie/MongoDB)
- A `current_user` dependency that slots directly into your own DI graph

```bash
pip install fastapi-users[sqlalchemy]
```

---

### User Model & Schemas

```python
from fastapi_users.db import SQLAlchemyBaseUserTableUUID
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    pass

# DB table — inherits id, email, hashed_password, is_active, is_verified, is_superuser
class User(SQLAlchemyBaseUserTableUUID, Base):
    pass  # add your own columns here
```

```python
import uuid
from fastapi_users import schemas

class UserRead(schemas.BaseUser[uuid.UUID]):
    pass                        # fields exposed in API responses

class UserCreate(schemas.BaseUserCreate):
    pass                        # fields accepted on registration

class UserUpdate(schemas.BaseUserUpdate):
    pass                        # fields accepted on profile update
```

---

### Auth Backend — JWT

```python
from fastapi_users.authentication import (
    AuthenticationBackend,
    BearerTransport,
    JWTStrategy,
)

bearer_transport = BearerTransport(tokenUrl="auth/jwt/login")

def get_jwt_strategy() -> JWTStrategy:
    return JWTStrategy(secret="YOUR_SECRET", lifetime_seconds=3600)

auth_backend = AuthenticationBackend(
    name="jwt",
    transport=bearer_transport,
    get_strategy=get_jwt_strategy,
)
```

---

### FastAPIUsers Instance & Routers

```python
from fastapi_users import FastAPIUsers

fastapi_users = FastAPIUsers[User, uuid.UUID](
    get_user_manager,       # see below
    [auth_backend],
)

app.include_router(
    fastapi_users.get_auth_router(auth_backend),
    prefix="/auth/jwt",
    tags=["auth"],
)
app.include_router(
    fastapi_users.get_register_router(UserRead, UserCreate),
    prefix="/auth",
    tags=["auth"],
)
app.include_router(
    fastapi_users.get_users_router(UserRead, UserUpdate),
    prefix="/users",
    tags=["users"],
)
```

This gives you these routes automatically:

|Method|Path|Description|
|---|---|---|
|POST|`/auth/jwt/login`|Login → JWT token|
|POST|`/auth/jwt/logout`|Logout|
|POST|`/auth/register`|Register new user|
|POST|`/auth/forgot-password`|Request reset email|
|POST|`/auth/reset-password`|Confirm reset|
|GET|`/users/me`|Current user profile|
|PATCH|`/users/me`|Update own profile|
|GET|`/users/{id}`|Get user (superuser only)|
|PATCH|`/users/{id}`|Update user (superuser only)|
|DELETE|`/users/{id}`|Delete user (superuser only)|

---

### UserManager — Custom Logic Hook

```python
from fastapi_users import BaseUserManager, UUIDIDMixin

class UserManager(UUIDIDMixin, BaseUserManager[User, uuid.UUID]):
    reset_password_token_secret = "RESET_SECRET"
    verification_token_secret = "VERIFY_SECRET"

    async def on_after_register(self, user: User, request=None):
        print(f"New user registered: {user.email}")
        # send welcome email, create default profile, etc.

    async def on_after_login(self, user: User, request=None, response=None):
        print(f"User logged in: {user.email}")

    async def on_after_forgot_password(self, user: User, token: str, request=None):
        print(f"Password reset token for {user.email}: {token}")
        # send email with reset link
```

---

### `current_user` — Where DI Meets Auth

`fastapi-users` exposes ready-made dependencies you inject exactly like any other FastAPI dependency.

```python
current_active_user    = fastapi_users.current_user(active=True)
current_verified_user  = fastapi_users.current_user(active=True, verified=True)
current_superuser      = fastapi_users.current_user(active=True, superuser=True)
current_optional_user  = fastapi_users.current_user(optional=True)  # None if not logged in
```

```python
from fastapi import Depends
from fastapi_users import FastAPIUsers

@app.get("/profile")
async def get_profile(user: User = Depends(current_active_user)):
    return {"email": user.email, "id": str(user.id)}

@app.get("/dashboard")
async def dashboard(user: User = Depends(current_verified_user)):
    # Only verified users reach here
    return {"message": f"Welcome, {user.email}"}

@app.delete("/admin/users/{user_id}")
async def admin_delete(user_id: uuid.UUID, admin: User = Depends(current_superuser)):
    # Only superusers reach here
    ...
```

---

### Combining fastapi-users with Your Own DI Graph

`current_user` composes with your own dependencies cleanly.

```python
async def get_user_items(
    user: User = Depends(current_active_user),
    db: AsyncSession = Depends(get_db),
):
    return await db.execute(select(Item).where(Item.owner_id == user.id))

@app.get("/my-items")
async def my_items(items=Depends(get_user_items)):
    return items.scalars().all()
```

FastAPI resolves `current_active_user` and `get_db` in parallel where possible, then passes both results into `get_user_items`, then its result into the handler.

---

## DI at a Glance

```
get_db ──────────────────────────────┐
                                     ▼
get_user_manager ──► current_user ──► get_user_items ──► route handler
                                     ▲
get_settings ────────────────────────┘
```

FastAPI builds this graph from type hints alone — no wiring or containers needed.

---

## Tags

#fastapi #python #dependency-injection #auth #fastapi-users #jwt #backend