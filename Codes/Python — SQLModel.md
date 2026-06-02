## What & When

**SQLModel** is a library for interacting with SQL databases using Python type hints. Built by the creator of FastAPI, it combines **SQLAlchemy** (the ORM) and **Pydantic** (the validation layer) into a single model class — eliminating the need to maintain separate DB models and API schemas.

Use SQLModel when:

- Building FastAPI applications with a SQL database
- Wanting one model class for DB table + API schema + validation
- Preferring a clean, type-hint-driven ORM over raw SQLAlchemy
- Working with SQLite, PostgreSQL, or MySQL

```bash
pip install sqlmodel
# Async support
pip install "sqlmodel[asyncio]"
pip install asyncpg          # PostgreSQL async driver
pip install aiosqlite        # SQLite async driver
```

---

## SQLModel vs SQLAlchemy vs Pydantic

|                      | SQLModel           | SQLAlchemy | Pydantic |
| -------------------- | ------------------ | ---------- | -------- |
| DB ORM               | ✅ (via SQLAlchemy) | ✅          | ❌        |
| Data validation      | ✅ (via Pydantic)   | ❌          | ✅        |
| API schema           | ✅                  | ❌          | ✅        |
| Type hints           | ✅ Native           | Partial    | ✅ Native |
| Async support        | ✅                  | ✅          | N/A      |
| FastAPI integration  | ✅ First-class      | Partial    | ✅        |
| Single model for all | ✅                  | ❌          | ❌        |

---

## Defining Models

```python
from sqlmodel import SQLModel, Field
from typing import Optional
from datetime import datetime

# Table model — maps to a DB table
class Hero(SQLModel, table=True):
    id:          Optional[int] = Field(default=None, primary_key=True)
    name:        str           = Field(index=True)
    secret_name: str
    age:         Optional[int] = Field(default=None, index=True)
    created_at:  datetime      = Field(default_factory=datetime.utcnow)

# Non-table model — Pydantic model only (for API schemas)
class HeroCreate(SQLModel):
    name:        str
    secret_name: str
    age:         Optional[int] = None

class HeroRead(SQLModel):
    id:   int
    name: str
    age:  Optional[int] = None

class HeroUpdate(SQLModel):
    name:        Optional[str] = None
    secret_name: Optional[str] = None
    age:         Optional[int] = None
```

> [!info] `table=True` vs no `table` `table=True` creates a SQLAlchemy mapped class — a DB table. Without it, the class is a plain Pydantic model used for API schemas. This is the core pattern: one base, multiple views.

---

## Base Model Pattern — DRY Models

```python
from sqlmodel import SQLModel, Field
from typing import Optional
from datetime import datetime

# Shared fields
class ItemBase(SQLModel):
    name:        str           = Field(index=True)
    description: Optional[str] = None
    price:       float         = Field(gt=0)
    in_stock:    bool          = True

# DB table
class Item(ItemBase, table=True):
    id:         Optional[int] = Field(default=None, primary_key=True)
    created_at: datetime      = Field(default_factory=datetime.utcnow)
    updated_at: datetime      = Field(default_factory=datetime.utcnow)

# API schemas
class ItemCreate(ItemBase):
    pass                                # same as ItemBase

class ItemRead(ItemBase):
    id:         int
    created_at: datetime

class ItemUpdate(SQLModel):
    name:        Optional[str]   = None
    description: Optional[str]   = None
    price:       Optional[float] = Field(default=None, gt=0)
    in_stock:    Optional[bool]  = None
```

---

## Field Options

```python
from sqlmodel import Field, SQLModel
from typing import Optional

class User(SQLModel, table=True):
    id:       Optional[int] = Field(default=None, primary_key=True)
    email:    str           = Field(unique=True, index=True)
    username: str           = Field(min_length=3, max_length=50, index=True)
    age:      Optional[int] = Field(default=None, ge=0, le=150)
    bio:      Optional[str] = Field(default=None, max_length=500)
    role:     str           = Field(default="user", sa_column_kwargs={"server_default": "user"})

    # Foreign key
    team_id:  Optional[int] = Field(default=None, foreign_key="team.id")

    # Custom column name
    email_address: str = Field(sa_column_kwargs={"name": "email_addr"})

    # Exclude from table (Pydantic-only field)
    password_plain: Optional[str] = Field(default=None, exclude=True)
```

---

## Relationships

```python
from sqlmodel import SQLModel, Field, Relationship
from typing import Optional, List

class Team(SQLModel, table=True):
    id:   Optional[int] = Field(default=None, primary_key=True)
    name: str           = Field(index=True)
    city: str

    # One-to-many: Team has many Heroes
    heroes: List["Hero"] = Relationship(back_populates="team")

class Hero(SQLModel, table=True):
    id:          Optional[int] = Field(default=None, primary_key=True)
    name:        str
    secret_name: str
    age:         Optional[int] = None

    # Foreign key
    team_id: Optional[int] = Field(default=None, foreign_key="team.id")

    # Many-to-one: Hero belongs to one Team
    team: Optional[Team] = Relationship(back_populates="heroes")
```

### Many-to-Many with Link Table

```python
from sqlmodel import SQLModel, Field, Relationship
from typing import List, Optional

class HeroTeamLink(SQLModel, table=True):
    hero_id: Optional[int] = Field(
        default=None, foreign_key="hero.id", primary_key=True
    )
    team_id: Optional[int] = Field(
        default=None, foreign_key="team.id", primary_key=True
    )

class Hero(SQLModel, table=True):
    id:    Optional[int] = Field(default=None, primary_key=True)
    name:  str
    teams: List["Team"] = Relationship(
        back_populates="heroes", link_model=HeroTeamLink
    )

class Team(SQLModel, table=True):
    id:     Optional[int] = Field(default=None, primary_key=True)
    name:   str
    heroes: List[Hero] = Relationship(
        back_populates="teams", link_model=HeroTeamLink
    )
```

---

## Engine & Session — Sync

```python
from sqlmodel import create_engine, SQLModel, Session

DATABASE_URL = "sqlite:///./database.db"
# DATABASE_URL = "postgresql://user:pass@localhost/dbname"

engine = create_engine(DATABASE_URL, echo=True)

# Create all tables
def create_db_and_tables():
    SQLModel.metadata.create_all(engine)

# Session dependency
def get_session():
    with Session(engine) as session:
        yield session
```

---

## Engine & Session — Async

```python
from sqlmodel.ext.asyncio.session import AsyncSession
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker
from sqlmodel import SQLModel

DATABASE_URL = "postgresql+asyncpg://user:pass@localhost/dbname"
# DATABASE_URL = "sqlite+aiosqlite:///./database.db"

engine = create_async_engine(DATABASE_URL, echo=True)
AsyncSessionLocal = async_sessionmaker(engine, expire_on_commit=False)

async def create_db_and_tables():
    async with engine.begin() as conn:
        await conn.run_sync(SQLModel.metadata.create_all)

async def get_session() -> AsyncSession:
    async with AsyncSessionLocal() as session:
        yield session
```

---

## CRUD Operations — Sync

```python
from sqlmodel import Session, select

# CREATE
def create_hero(session: Session, hero: HeroCreate) -> Hero:
    db_hero = Hero.model_validate(hero)
    session.add(db_hero)
    session.commit()
    session.refresh(db_hero)
    return db_hero

# READ — by id
def get_hero(session: Session, hero_id: int) -> Hero | None:
    return session.get(Hero, hero_id)

# READ — query
def get_heroes(session: Session, skip: int = 0, limit: int = 20) -> list[Hero]:
    statement = select(Hero).offset(skip).limit(limit)
    return session.exec(statement).all()

# READ — with filter
def get_hero_by_name(session: Session, name: str) -> Hero | None:
    statement = select(Hero).where(Hero.name == name)
    return session.exec(statement).first()

# UPDATE
def update_hero(session: Session, hero_id: int, updates: HeroUpdate) -> Hero | None:
    hero = session.get(Hero, hero_id)
    if not hero:
        return None
    patch = updates.model_dump(exclude_unset=True)
    hero.sqlmodel_update(patch)
    session.add(hero)
    session.commit()
    session.refresh(hero)
    return hero

# DELETE
def delete_hero(session: Session, hero_id: int) -> bool:
    hero = session.get(Hero, hero_id)
    if not hero:
        return False
    session.delete(hero)
    session.commit()
    return True
```

---

## CRUD Operations — Async

```python
from sqlmodel.ext.asyncio.session import AsyncSession
from sqlmodel import select

# CREATE
async def create_hero(session: AsyncSession, hero: HeroCreate) -> Hero:
    db_hero = Hero.model_validate(hero)
    session.add(db_hero)
    await session.commit()
    await session.refresh(db_hero)
    return db_hero

# READ — by id
async def get_hero(session: AsyncSession, hero_id: int) -> Hero | None:
    return await session.get(Hero, hero_id)

# READ — query
async def get_heroes(session: AsyncSession, skip: int = 0, limit: int = 20) -> list[Hero]:
    statement = select(Hero).offset(skip).limit(limit)
    result = await session.exec(statement)
    return result.all()

# READ — with filter and order
async def search_heroes(session: AsyncSession, name: str) -> list[Hero]:
    statement = (
        select(Hero)
        .where(Hero.name.contains(name))
        .order_by(Hero.name)
        .limit(10)
    )
    result = await session.exec(statement)
    return result.all()

# UPDATE
async def update_hero(session: AsyncSession, hero_id: int, updates: HeroUpdate) -> Hero | None:
    hero = await session.get(Hero, hero_id)
    if not hero:
        return None
    patch = updates.model_dump(exclude_unset=True)
    hero.sqlmodel_update(patch)
    session.add(hero)
    await session.commit()
    await session.refresh(hero)
    return hero

# DELETE
async def delete_hero(session: AsyncSession, hero_id: int) -> bool:
    hero = await session.get(Hero, hero_id)
    if not hero:
        return False
    await session.delete(hero)
    await session.commit()
    return True
```

---

## Querying — `select`

```python
from sqlmodel import select
from sqlalchemy import func

# Basic select
statement = select(Hero)

# WHERE
statement = select(Hero).where(Hero.name == "Spider-Boy")
statement = select(Hero).where(Hero.age >= 18, Hero.age <= 60)    # AND
statement = select(Hero).where((Hero.age < 18) | (Hero.age > 60)) # OR

# LIKE / ILIKE
statement = select(Hero).where(Hero.name.contains("man"))
statement = select(Hero).where(Hero.name.startswith("Spider"))
statement = select(Hero).where(Hero.name.ilike("%man%"))           # case-insensitive

# IS NULL / IS NOT NULL
statement = select(Hero).where(Hero.age == None)
statement = select(Hero).where(Hero.age != None)

# IN
statement = select(Hero).where(Hero.name.in_(["Alice", "Bob"]))

# ORDER BY
statement = select(Hero).order_by(Hero.name)
statement = select(Hero).order_by(Hero.age.desc())

# LIMIT / OFFSET
statement = select(Hero).offset(20).limit(10)

# JOIN
statement = (
    select(Hero, Team)
    .join(Team, Hero.team_id == Team.id)
    .where(Team.city == "London")
)

# COUNT
statement = select(func.count(Hero.id))

# Execute and get results
result  = session.exec(statement)
heroes  = result.all()          # list
hero    = result.first()        # first or None
hero    = result.one()          # exactly one (raises if 0 or >1)
hero    = result.one_or_none()  # one or None (raises if >1)
```

---

## FastAPI Integration — Full Example

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI, Depends, HTTPException, status
from sqlmodel import SQLModel, Field, select
from sqlmodel.ext.asyncio.session import AsyncSession
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker
from typing import Optional

DATABASE_URL = "postgresql+asyncpg://user:pass@localhost/heroesdb"

engine           = create_async_engine(DATABASE_URL)
AsyncSessionLocal = async_sessionmaker(engine, expire_on_commit=False)

# Models
class HeroBase(SQLModel):
    name:        str           = Field(index=True)
    secret_name: str
    age:         Optional[int] = None

class Hero(HeroBase, table=True):
    id: Optional[int] = Field(default=None, primary_key=True)

class HeroCreate(HeroBase): pass
class HeroRead(HeroBase):
    id: int
class HeroUpdate(SQLModel):
    name:        Optional[str] = None
    secret_name: Optional[str] = None
    age:         Optional[int] = None

# Lifespan
@asynccontextmanager
async def lifespan(app: FastAPI):
    async with engine.begin() as conn:
        await conn.run_sync(SQLModel.metadata.create_all)
    yield
    await engine.dispose()

app = FastAPI(lifespan=lifespan)

# Dependency
async def get_session() -> AsyncSession:
    async with AsyncSessionLocal() as session:
        yield session

# Routes
@app.get("/heroes", response_model=list[HeroRead])
async def list_heroes(
    skip:    int = 0,
    limit:   int = 20,
    session: AsyncSession = Depends(get_session),
):
    result = await session.exec(select(Hero).offset(skip).limit(limit))
    return result.all()

@app.get("/heroes/{hero_id}", response_model=HeroRead)
async def get_hero(hero_id: int, session: AsyncSession = Depends(get_session)):
    hero = await session.get(Hero, hero_id)
    if not hero:
        raise HTTPException(status_code=404, detail="Hero not found")
    return hero

@app.post("/heroes", response_model=HeroRead, status_code=status.HTTP_201_CREATED)
async def create_hero(hero: HeroCreate, session: AsyncSession = Depends(get_session)):
    db_hero = Hero.model_validate(hero)
    session.add(db_hero)
    await session.commit()
    await session.refresh(db_hero)
    return db_hero

@app.patch("/heroes/{hero_id}", response_model=HeroRead)
async def update_hero(
    hero_id: int,
    updates: HeroUpdate,
    session: AsyncSession = Depends(get_session),
):
    hero = await session.get(Hero, hero_id)
    if not hero:
        raise HTTPException(status_code=404, detail="Hero not found")
    patch = updates.model_dump(exclude_unset=True)
    hero.sqlmodel_update(patch)
    session.add(hero)
    await session.commit()
    await session.refresh(hero)
    return hero

@app.delete("/heroes/{hero_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_hero(hero_id: int, session: AsyncSession = Depends(get_session)):
    hero = await session.get(Hero, hero_id)
    if not hero:
        raise HTTPException(status_code=404, detail="Hero not found")
    await session.delete(hero)
    await session.commit()
```

---

## Alembic Migrations

SQLModel uses SQLAlchemy under the hood — Alembic works directly.

```bash
pip install alembic
alembic init alembic
```

```python
# alembic/env.py
from sqlmodel import SQLModel
from myapp.models import *          # import all models to register metadata

target_metadata = SQLModel.metadata
```

```bash
# Generate a migration
alembic revision --autogenerate -m "add heroes table"

# Apply migrations
alembic upgrade head

# Rollback one step
alembic downgrade -1

# Show current revision
alembic current

# Show history
alembic history
```

---

## Testing with SQLModel

```python
import pytest
from sqlmodel import SQLModel, create_engine, Session
from sqlmodel.pool import StaticPool
from fastapi.testclient import TestClient
from myapp.main import app
from myapp.db import get_session

@pytest.fixture(name="session")
def session_fixture():
    # In-memory SQLite for tests
    engine = create_engine(
        "sqlite://",
        connect_args={"check_same_thread": False},
        poolclass=StaticPool,
    )
    SQLModel.metadata.create_all(engine)
    with Session(engine) as session:
        yield session

@pytest.fixture(name="client")
def client_fixture(session: Session):
    def override_get_session():
        yield session

    app.dependency_overrides[get_session] = override_get_session
    with TestClient(app) as client:
        yield client
    app.dependency_overrides.clear()

def test_create_hero(client: TestClient):
    response = client.post("/heroes", json={"name": "Deadpond", "secret_name": "Dive Wilson"})
    assert response.status_code == 201
    data = response.json()
    assert data["name"] == "Deadpond"
    assert data["id"] is not None

def test_get_hero_not_found(client: TestClient):
    response = client.get("/heroes/999")
    assert response.status_code == 404
```

---

## Common Patterns

### Pagination Response Model

```python
from sqlmodel import SQLModel
from typing import Generic, TypeVar, List

T = TypeVar("T")

class Page(SQLModel, Generic[T]):
    items: List[T]
    total: int
    skip:  int
    limit: int

@app.get("/heroes", response_model=Page[HeroRead])
async def list_heroes(
    skip:    int = 0,
    limit:   int = 20,
    session: AsyncSession = Depends(get_session),
):
    from sqlalchemy import func
    count_stmt = select(func.count(Hero.id))
    total = (await session.exec(count_stmt)).one()

    heroes_stmt = select(Hero).offset(skip).limit(limit)
    heroes = (await session.exec(heroes_stmt)).all()

    return Page(items=heroes, total=total, skip=skip, limit=limit)
```

### Soft Delete

```python
from datetime import datetime
from typing import Optional
from sqlmodel import SQLModel, Field, select

class SoftDeleteMixin(SQLModel):
    deleted_at: Optional[datetime] = Field(default=None, index=True)

    @property
    def is_deleted(self) -> bool:
        return self.deleted_at is not None

class Item(SoftDeleteMixin, table=True):
    id:   Optional[int] = Field(default=None, primary_key=True)
    name: str

# Query only non-deleted
statement = select(Item).where(Item.deleted_at == None)

# Soft delete
item.deleted_at = datetime.utcnow()
session.add(item)
await session.commit()
```

---

## Quick Reference

|Task|Code|
|---|---|
|Table model|`class M(SQLModel, table=True)`|
|Schema model|`class M(SQLModel)`|
|Primary key|`id: Optional[int] = Field(default=None, primary_key=True)`|
|Index|`Field(index=True)`|
|Unique|`Field(unique=True)`|
|Foreign key|`Field(foreign_key="table.id")`|
|Relationship|`Relationship(back_populates="...")`|
|Create engine sync|`create_engine(url)`|
|Create engine async|`create_async_engine(url)`|
|Create tables|`SQLModel.metadata.create_all(engine)`|
|Session sync|`with Session(engine) as session`|
|Session async|`async with AsyncSessionLocal() as session`|
|Add|`session.add(obj)`|
|Commit|`await session.commit()`|
|Refresh|`await session.refresh(obj)`|
|Get by id|`session.get(Model, id)`|
|Query|`select(Model).where(...)`|
|Execute|`session.exec(statement)`|
|All results|`.all()`|
|First result|`.first()`|
|Patch update|`obj.sqlmodel_update(dict)`|
|Delete|`session.delete(obj)`|
|Migrations|`alembic revision --autogenerate`|

---

## Tags

#python #sqlmodel #sqlalchemy #pydantic #orm #database #fastapi #backend #async