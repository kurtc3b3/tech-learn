
- **Setup** — engine and session factory for all four combos: SQLite/PostgreSQL × sync/async, plus the required driver packages
	- Dependencies: [[sqlalchemy]]
	- Setup: [[ORM - Setup]]
- **Models** — declarative base, typed columns with `Mapped[]`, relationships, and how to create tables (sync + async)
	- Declarative Base and Models: [[ORM - Models]]
- **CRUD** — side-by-side sync and async examples for create, read, update, and delete operations
	- CRUD Operations: [[ORM - CRUD]]
- **Queries** — filtering, joins, aggregates, eager loading to avoid N+1, and raw SQL with `text()`
	- Query Operations: [[ORM - Queries]]
- **Async** — FastAPI dependency pattern, why lazy loading doesn't work in async (and how to fix it), transactions, and engine disposal
	- Async Operations: [[ORM - Async]]
- **Migrations** — Alembic setup, common CLI commands, and async-compatible `env.py`
	- Migrations: [[ORM - Migrations]]

A few things worth highlighting that catch people out:

- In async SQLAlchemy, **lazy loading relationships will raise a `MissingGreenlet` error** — always use `selectinload()` or `joinedload()` in your queries.
- For async PostgreSQL use `asyncpg` as the driver (`postgresql+asyncpg://...`), and for async SQLite use `aiosqlite` (`sqlite+aiosqlite://...`).
- Alembic itself runs synchronously even if your app is async — use `conn.run_sync()` to bridge them in `env.py`.