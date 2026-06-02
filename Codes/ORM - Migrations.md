
**Setup:**

```shell
# Install and initialise
pip install alembic
alembic init alembic

# alembic/env.py — point to your models
from myapp.models import Base
target_metadata = Base.metadata

# alembic.ini — set your DB URL
sqlalchemy.url = postgresql+psycopg2://user:pass@localhost/db
# or for async use synchronous URL here (alembic runs sync)
```

**Common Commands:**

```shell
# auto-generate migration from model diff
alembic revision --autogenerate -m "add users table"

# apply all pending migrations
alembic upgrade head

# roll back one step
alembic downgrade -1

# see current revision
alembic current

# view history
alembic history --verbose
```

**Async Alembic env.py:**

```python
from alembic import context
import asyncio
from sqlalchemy.ext.asyncio import create_async_engine

def run_migrations_online():
    connectable = create_async_engine(DB_URL)

    async def do_run():
        async with connectable.connect() as conn:
            await conn.run_sync(
                context.run_migrations
            )

    asyncio.run(do_run())
```

