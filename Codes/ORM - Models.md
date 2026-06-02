
**Declarative Base:**


```python
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship
from sqlalchemy import String, Integer, ForeignKey, DateTime, func
from datetime import datetime

class Base(DeclarativeBase):
    pass
```


**Models:**

```python
class User(Base):
    __tablename__ = "users"

    id: Mapped[int]          = mapped_column(primary_key=True)
    name: Mapped[str]        = mapped_column(String(100))
    email: Mapped[str]       = mapped_column(String(255), unique=True)
    age: Mapped[int | None]  = mapped_column(Integer, nullable=True)
    created_at: Mapped[datetime] = mapped_column(
        DateTime, server_default=func.now()
    )

    posts: Mapped[list["Post"]] = relationship(
        "Post", back_populates="author", cascade="all, delete-orphan"
    )

class Post(Base):
    __tablename__ = "posts"

    id: Mapped[int]    = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    user_id: Mapped[int] = mapped_column(ForeignKey("users.id"))

    author: Mapped["User"] = relationship("User", back_populates="posts")
```


**Create Tables:**

```python
# Sync
Base.metadata.create_all(engine)

# Async
async def init_db():
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
```






