
**sqlite sync:**

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

engine = create_engine(
    "sqlite:///app.db",
    echo=True,       # log SQL
    connect_args={
        "check_same_thread": False
    },
)

Session = sessionmaker(bind=engine)

with Session() as session:
    ...  # use session here
```


**sqlite async:**

```python
from sqlalchemy.ext.asyncio import (
    create_async_engine,
    AsyncSession,
)
from sqlalchemy.orm import sessionmaker

engine = create_async_engine(
    "sqlite+aiosqlite:///app.db",
    echo=True,
)

AsyncSessionLocal = sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False,
)
```

**postgres sync:**

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

engine = create_engine(
    "postgresql+psycopg2://"
    "user:pass@host:5432/db",
    pool_size=5,
    max_overflow=10,
    pool_pre_ping=True,
)

Session = sessionmaker(bind=engine)

with Session() as session:
    ...  # use session here
```

**postgres async:**

```python
from sqlalchemy.ext.asyncio import (
    create_async_engine,
    AsyncSession,
)
from sqlalchemy.orm import sessionmaker

engine = create_async_engine(
    "postgresql+asyncpg://"
    "user:pass@host:5432/db",
    pool_size=10,
)

AsyncSessionLocal = sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False,
)
```

