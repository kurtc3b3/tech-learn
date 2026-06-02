
**Async Session Dependency (FastAPI Pattern):**

```python
from sqlalchemy.ext.asyncio import AsyncSession
from typing import AsyncGenerator

async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with AsyncSessionLocal() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise

# FastAPI route usage
from fastapi import Depends

@app.get("/users/{user_id}")
async def get_user(user_id: int, db: AsyncSession = Depends(get_db)):
    user = await db.get(User, user_id)
    return user
```


**Async Relationship Loading:**

```python
# lazy loading is NOT supported in async — always eager load
from sqlalchemy.orm import selectinload

async with AsyncSessionLocal() as s:
    result = await s.execute(
        select(User)
        .options(selectinload(User.posts))
        .where(User.id == 1)
    )
    user = result.scalar_one()
    print(user.posts)  # safe — already loaded

# explicit async_lazy on model definition
posts: Mapped[list["Post"]] = relationship(
    lazy="selectin"  # auto eager-load every time
)
```

**Async Transactions:**

```python
async with AsyncSessionLocal() as s:
    async with s.begin():
        # everything inside is one transaction
        s.add(User(name="Alice", email="a@a.com"))
        s.add(Post(title="Hi", user_id=1))
    # auto-committed or rolled back on error
```

**Async Engine Disposal:**

```python
# Always dispose on app shutdown
import asyncio

async def shutdown():
    await engine.dispose()

# lifespan (FastAPI)
from contextlib import asynccontextmanager

@asynccontextmanager
async def lifespan(app):
    yield
    await engine.dispose()
```

## Related Notes

[[Python — asyncio]]
