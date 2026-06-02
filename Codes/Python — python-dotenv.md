## What & When

**python-dotenv** loads environment variables from a `.env` file into `os.environ` (or a dict) at runtime. It bridges the gap between local development — where you don't want secrets in code — and production — where environment variables are set by the platform (Docker, Kubernetes, Heroku, etc.).

Use python-dotenv when:

- Storing secrets and config outside of source code
- Managing different settings per environment (dev, staging, prod)
- Following the [12-Factor App](https://12factor.net/config) config principle
- Working locally without setting system-wide environment variables

```bash
pip install python-dotenv
```

---

## The `.env` File Format

```ini
# Comments start with #
APP_NAME=My Service
DEBUG=true
PORT=8000

# Quoted values — quotes are stripped
SECRET_KEY="supersecretkey1234567890abcdef"
DATABASE_URL='postgresql://user:pass@localhost/db'

# Empty value
OPTIONAL_FEATURE=

# Multiline value (use quotes)
PRIVATE_KEY="-----BEGIN RSA PRIVATE KEY-----
MIIEowIBAAKCAQEA...
-----END RSA PRIVATE KEY-----"

# Export prefix (optional — ignored by python-dotenv)
export AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE

# Variable expansion
BASE_URL=https://api.example.com
USERS_URL=${BASE_URL}/users
ORDERS_URL=${BASE_URL}/orders
```

> [!warning] Never commit `.env` to version control Add `.env` to `.gitignore` immediately. Commit a `.env.example` with placeholder values instead so other developers know what variables are needed.

```gitignore
# .gitignore
.env
.env.local
.env.*.local
```

---

## Basic Usage — `load_dotenv`

```python
from dotenv import load_dotenv
import os

# Load .env from current directory (or search upward)
load_dotenv()

# Access variables
db_url  = os.getenv("DATABASE_URL")
debug   = os.getenv("DEBUG", "false").lower() == "true"
port    = int(os.getenv("PORT", "8000"))

print(db_url)   # "postgresql://user:pass@localhost/db"
```

---

## Load Options

```python
from dotenv import load_dotenv
from pathlib import Path

# Explicit path
load_dotenv("/path/to/.env")
load_dotenv(Path(__file__).parent / ".env")

# Override existing env vars (default: False — existing vars win)
load_dotenv(override=True)

# Don't raise if file not found (default: no error anyway)
load_dotenv(dotenv_path=".env", verbose=True)   # print what was loaded

# Search parent directories for .env (like git does for .gitignore)
load_dotenv(find_dotenv())

# Encoding
load_dotenv(encoding="utf-8")
```

---

## `find_dotenv` — Locate the File Automatically

```python
from dotenv import load_dotenv, find_dotenv

# Finds .env by walking up directory tree from current file
dotenv_path = find_dotenv()
load_dotenv(dotenv_path)

# Raises if not found
dotenv_path = find_dotenv(raise_error_if_not_found=True)

# Start search from a specific directory
dotenv_path = find_dotenv(filename=".env.local", usecwd=True)
```

---

## `dotenv_values` — Load into Dict (No Side Effects)

Loads variables into a dict without touching `os.environ`. Useful when you want to inspect values without polluting the process environment.

```python
from dotenv import dotenv_values

config = dotenv_values(".env")
print(config["DATABASE_URL"])
print(config.get("OPTIONAL_KEY", "default"))

# All values are strings
print(type(config["PORT"]))     # <class 'str'>

# Merge multiple files (later files override earlier ones)
config = {
    **dotenv_values(".env"),             # base config
    **dotenv_values(".env.local"),       # local overrides
    **os.environ,                        # actual env vars win
}
```

---

## Multiple Environment Files

A clean pattern for layered config — base → environment-specific → local overrides.

```
.env                  ← shared defaults (committed)
.env.development      ← dev-specific (committed)
.env.production       ← prod-specific (committed, no secrets)
.env.local            ← local machine overrides (gitignored)
.env.test             ← test environment (committed)
```

```python
import os
from dotenv import load_dotenv
from pathlib import Path

BASE_DIR = Path(__file__).parent
ENV      = os.getenv("ENVIRONMENT", "development")

# Load in order — later calls override earlier ones if override=True
load_dotenv(BASE_DIR / ".env",                  override=False)
load_dotenv(BASE_DIR / f".env.{ENV}",           override=True)
load_dotenv(BASE_DIR / ".env.local",            override=True)
```

---

## Variable Expansion

Variables can reference other variables in the same file.

```ini
# .env
HOST=api.example.com
PORT=8000
BASE_URL=http://${HOST}:${PORT}
HEALTH_URL=${BASE_URL}/health
```

```python
from dotenv import load_dotenv
import os

load_dotenv()
print(os.getenv("BASE_URL"))    # http://api.example.com:8000
print(os.getenv("HEALTH_URL"))  # http://api.example.com:8000/health
```

Variable expansion is enabled by default. Disable with:

```python
load_dotenv(interpolate=False)
```

---

## Programmatic Read / Write / Unset

```python
from dotenv import get_key, set_key, unset_key, dotenv_values

dotenv_file = ".env"

# Read a single key
value = get_key(dotenv_file, "DATABASE_URL")

# Write / update a key
set_key(dotenv_file, "PORT", "9000")
set_key(dotenv_file, "DEBUG", "false", quote_mode="never")

# Remove a key
unset_key(dotenv_file, "OLD_API_KEY")

# List all key-value pairs
for key, value in dotenv_values(dotenv_file).items():
    print(f"{key}={value}")
```

---

## Type Coercion — Parsing String Values

All `.env` values are strings. Parse them explicitly:

```python
import os
from dotenv import load_dotenv

load_dotenv()

# Boolean
DEBUG = os.getenv("DEBUG", "false").lower() in ("true", "1", "yes")

# Integer
PORT = int(os.getenv("PORT", "8000"))

# Float
RATE_LIMIT = float(os.getenv("RATE_LIMIT", "100.0"))

# List (comma-separated)
ALLOWED_HOSTS = os.getenv("ALLOWED_HOSTS", "localhost").split(",")

# Optional
SENTRY_DSN = os.getenv("SENTRY_DSN") or None

# Required — raise if missing
DATABASE_URL = os.environ["DATABASE_URL"]   # KeyError if not set
```

> [!tip] Use `pydantic-settings` to avoid manual coercion `BaseSettings` handles all type parsing automatically from env vars. See [[Python - Pydantic]] and [[FastAPI - Dependency Injection & User Management]].

---

## With pydantic-settings (Recommended Pattern)

`pydantic-settings` integrates directly with `.env` files via python-dotenv under the hood — no manual `load_dotenv()` needed.

```python
from pydantic_settings import BaseSettings, SettingsConfigDict
from pydantic import Field
from typing import Optional
from functools import lru_cache

class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        extra="ignore",
    )

    app_name:     str          = "My Service"
    debug:        bool         = False
    port:         int          = 8000
    database_url: str          = Field(...)
    secret_key:   str          = Field(..., min_length=32)
    allowed_hosts: list[str]   = ["localhost"]
    sentry_dsn:   Optional[str]= None

@lru_cache
def get_settings() -> Settings:
    return Settings()
```

---

## With FastAPI

```python
from fastapi import FastAPI, Depends
from functools import lru_cache
from dotenv import load_dotenv
import os

load_dotenv()                   # load before app creation

from config import get_settings, Settings
from contextlib import asynccontextmanager

@asynccontextmanager
async def lifespan(app: FastAPI):
    settings = get_settings()
    print(f"Starting {settings.app_name} in {settings.environment} mode")
    yield

app = FastAPI(lifespan=lifespan)

@app.get("/config")
async def show_config(settings: Settings = Depends(get_settings)):
    return {
        "app_name":    settings.app_name,
        "environment": settings.environment,
        "debug":       settings.debug,
        # Never expose secrets here
    }
```

---

## Testing — Overriding Variables

```python
import pytest
from unittest.mock import patch

# Patch individual env vars
def test_debug_mode():
    with patch.dict("os.environ", {"DEBUG": "true", "PORT": "9000"}):
        from config import get_settings
        settings = get_settings()
        assert settings.debug is True
        assert settings.port == 9000


# Load a test-specific .env file
from dotenv import load_dotenv

def test_with_test_env():
    load_dotenv(".env.test", override=True)
    # test code here


# pytest fixture
import pytest
from dotenv import dotenv_values

@pytest.fixture(autouse=True)
def test_env(monkeypatch):
    config = dotenv_values(".env.test")
    for key, value in config.items():
        monkeypatch.setenv(key, value)
```

---

## CLI Usage

python-dotenv ships with a CLI for reading and writing `.env` files.

```bash
# Install
pip install "python-dotenv[cli]"

# Get a value
dotenv get DATABASE_URL

# Set a value
dotenv set PORT 9000
dotenv set SECRET_KEY "mynewsecret"

# Unset a value
dotenv unset OLD_KEY

# List all values
dotenv list
dotenv list --format=json

# Run a command with .env loaded
dotenv run python main.py
dotenv run -- uvicorn main:app --reload

# Specify a file
dotenv -f .env.staging get DATABASE_URL
```

---

## `.env.example` — Template for Collaborators

Always commit a documented example file with placeholder values.

```ini
# .env.example — copy to .env and fill in values

# Application
APP_NAME=My Service
ENVIRONMENT=development
DEBUG=false
PORT=8000

# Database (required)
DATABASE_URL=postgresql+asyncpg://user:password@localhost:5432/dbname

# Auth (required — generate with: openssl rand -hex 32)
SECRET_KEY=changeme_generate_with_openssl_rand_hex_32
JWT_EXPIRE_SECONDS=3600

# Redis
REDIS_URL=redis://localhost:6379

# Optional integrations
SENTRY_DSN=
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_REGION=eu-west-2
```

---

## Docker Integration

In Docker, `.env` files are passed via `--env-file` — python-dotenv is not needed inside the container.

```yaml
# docker-compose.yml
services:
  api:
    image: my-service
    env_file:
      - .env                  # loaded by Docker, not python-dotenv
    environment:
      - ENVIRONMENT=production  # override specific vars
```

```dockerfile
# Dockerfile — do NOT copy .env into the image
COPY . .
# RUN pip install ... etc
# ENV variables come from docker-compose or platform
```

> [!warning] Never `COPY .env` into a Docker image It gets baked into every layer and is visible in `docker history`. Pass secrets at runtime via `env_file` or a secrets manager.

---

## Quick Reference

|Task|Code|
|---|---|
|Load `.env`|`load_dotenv()`|
|Load explicit path|`load_dotenv("/path/.env")`|
|Override existing|`load_dotenv(override=True)`|
|Find `.env` upward|`load_dotenv(find_dotenv())`|
|Load to dict|`dotenv_values(".env")`|
|Get single key|`get_key(".env", "KEY")`|
|Set single key|`set_key(".env", "KEY", "value")`|
|Remove key|`unset_key(".env", "KEY")`|
|Read string|`os.getenv("KEY", "default")`|
|Read required|`os.environ["KEY"]`|
|Parse bool|`os.getenv("DEBUG","false").lower() == "true"`|
|Parse int|`int(os.getenv("PORT", "8000"))`|
|Parse list|`os.getenv("HOSTS","").split(",")`|
|CLI run with env|`dotenv run -- uvicorn main:app`|

---
## Tags

#python #dotenv #environment #config #secrets #backend #12factor