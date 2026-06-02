
## What & When

**logging** is Python's built-in structured logging framework. It provides a hierarchical system of loggers, handlers, filters, and formatters that gives fine-grained control over what gets logged, where it goes, and how it looks — without `print()` statements scattered everywhere.

Use logging when:

- Recording application events, errors, and diagnostics
- Routing logs to different destinations (file, console, external service)
- Filtering log output per module or severity level
- Structuring logs for ingestion by log aggregators (Datadog, Loki, CloudWatch)
- Replacing `print()` in any code beyond a quick script

```python
import logging
```

No installation needed — logging is part of the standard library.

---

## Log Levels

```python
import logging

logging.debug("Detailed diagnostic info")       # level 10
logging.info("Normal application events")       # level 20
logging.warning("Something unexpected")         # level 30 — default threshold
logging.error("An error occurred")              # level 40
logging.critical("System is unusable")          # level 50

# Custom level
logging.log(25, "Custom level message")
```

|Level|Value|When to Use|
|---|---|---|
|`DEBUG`|10|Detailed diagnostic — dev only|
|`INFO`|20|Milestones, normal flow|
|`WARNING`|30|Unexpected but handled — default|
|`ERROR`|40|Failure that needs attention|
|`CRITICAL`|50|System-level failure|

---

## Basic Configuration

```python
import logging

# Quickstart — configure root logger
logging.basicConfig(
    level=logging.DEBUG,
    format="%(asctime)s [%(levelname)s] %(name)s: %(message)s",
    datefmt="%Y-%m-%d %H:%M:%S",
)

logging.info("App started")
logging.warning("Low disk space")
logging.error("Database connection failed")
```

> [!warning] `basicConfig` only works once It configures the **root logger** and is a no-op if any handler is already attached. Call it once at the very start of your application, before any other logging calls or imports that log.

---

## Loggers — Named & Hierarchical

```python
import logging

# Get a named logger — best practice in every module
logger = logging.getLogger(__name__)
# __name__ = "myapp.routers.users" in myapp/routers/users.py

logger.info("User created")
logger.warning("Rate limit approaching")
logger.error("Failed to send email", exc_info=True)

# Logger hierarchy
# "myapp.routers.users" → "myapp.routers" → "myapp" → root
# Each propagates up unless propagate=False
```

> [!tip] Always use `logging.getLogger(__name__)` Named loggers let you control verbosity per module. Never use the root logger directly in library or application module code.

---

## Handlers — Where Logs Go

```python
import logging
import logging.handlers

logger = logging.getLogger("myapp")
logger.setLevel(logging.DEBUG)      # logger threshold

# Console handler
console = logging.StreamHandler()
console.setLevel(logging.INFO)      # handler threshold

# File handler
file_handler = logging.FileHandler("app.log", encoding="utf-8")
file_handler.setLevel(logging.WARNING)

# Rotating file handler — max 10MB per file, keep 5 backups
rotating = logging.handlers.RotatingFileHandler(
    "app.log",
    maxBytes=10 * 1024 * 1024,
    backupCount=5,
    encoding="utf-8",
)

# Timed rotating handler — new file every midnight, keep 30 days
timed = logging.handlers.TimedRotatingFileHandler(
    "app.log",
    when="midnight",
    interval=1,
    backupCount=30,
    encoding="utf-8",
)

# HTTP handler — POST logs to a remote endpoint
http = logging.handlers.HTTPHandler(
    host="logs.example.com",
    url="/ingest",
    method="POST",
)

# Add handlers to logger
logger.addHandler(console)
logger.addHandler(file_handler)
```

---

## Formatters — How Logs Look

```python
import logging

formatter = logging.Formatter(
    fmt="%(asctime)s [%(levelname)-8s] %(name)s:%(lineno)d — %(message)s",
    datefmt="%Y-%m-%dT%H:%M:%S",
)

handler = logging.StreamHandler()
handler.setFormatter(formatter)
```

### Format String Variables

|Variable|Description|
|---|---|
|`%(asctime)s`|Human-readable timestamp|
|`%(created)f`|Unix timestamp|
|`%(levelname)s`|Level name (DEBUG, INFO, …)|
|`%(levelno)d`|Level number (10, 20, …)|
|`%(name)s`|Logger name|
|`%(module)s`|Module name|
|`%(filename)s`|Filename|
|`%(pathname)s`|Full path|
|`%(lineno)d`|Line number|
|`%(funcName)s`|Function name|
|`%(message)s`|Formatted message|
|`%(process)d`|Process ID|
|`%(thread)d`|Thread ID|
|`%(threadName)s`|Thread name|

---

## Logging Exceptions

```python
import logging

logger = logging.getLogger(__name__)

try:
    result = 1 / 0
except ZeroDivisionError:
    # Include full traceback
    logger.exception("Division failed")         # auto exc_info=True

    # Or explicitly
    logger.error("Division failed", exc_info=True)

    # Log and re-raise
    logger.error("Division failed", exc_info=True)
    raise
```

---

## Extra Context — Structured Fields

Pass additional fields to enrich log records.

```python
import logging

logger = logging.getLogger(__name__)

# Extra dict — fields available in formatter as %(key)s
logger.info("User logged in", extra={
    "user_id":    42,
    "ip_address": "192.168.1.1",
    "endpoint":   "/auth/login",
})

# LoggerAdapter — attach context once, use everywhere
adapter = logging.LoggerAdapter(logger, extra={
    "request_id": "req-abc-123",
    "user_id":    42,
})
adapter.info("Processing order")
adapter.error("Payment failed")
# Both include request_id and user_id automatically
```

---

## Dict Config — Production Setup

The recommended way to configure logging in applications — all config in one place, no code scattered across modules.

```python
import logging
import logging.config

LOGGING_CONFIG = {
    "version": 1,
    "disable_existing_loggers": False,      # keep third-party loggers

    "formatters": {
        "standard": {
            "format": "%(asctime)s [%(levelname)s] %(name)s: %(message)s",
            "datefmt": "%Y-%m-%dT%H:%M:%S",
        },
        "json": {
            "()": "pythonjsonlogger.jsonlogger.JsonFormatter",
            "format": "%(asctime)s %(levelname)s %(name)s %(message)s",
        },
    },

    "handlers": {
        "console": {
            "class":     "logging.StreamHandler",
            "level":     "INFO",
            "formatter": "standard",
            "stream":    "ext://sys.stdout",
        },
        "file": {
            "class":      "logging.handlers.RotatingFileHandler",
            "level":      "WARNING",
            "formatter":  "standard",
            "filename":   "logs/app.log",
            "maxBytes":   10485760,         # 10 MB
            "backupCount": 5,
            "encoding":   "utf-8",
        },
    },

    "loggers": {
        "myapp": {
            "level":     "DEBUG",
            "handlers":  ["console", "file"],
            "propagate": False,
        },
        "uvicorn": {
            "level":     "INFO",
            "handlers":  ["console"],
            "propagate": False,
        },
        "sqlalchemy.engine": {
            "level":     "WARNING",        # suppress SQL output in prod
            "handlers":  ["console"],
            "propagate": False,
        },
    },

    "root": {
        "level":    "WARNING",
        "handlers": ["console"],
    },
}

logging.config.dictConfig(LOGGING_CONFIG)
```

---

## JSON Logging — Structured for Log Aggregators

Plain text logs are hard to query in Datadog, Loki, or CloudWatch. JSON logs are machine-readable and filterable.

```bash
pip install python-json-logger
```

```python
import logging
from pythonjsonlogger import jsonlogger

logger = logging.getLogger(__name__)

handler = logging.StreamHandler()
handler.setFormatter(jsonlogger.JsonFormatter(
    fmt="%(asctime)s %(levelname)s %(name)s %(message)s",
    rename_fields={"levelname": "level", "asctime": "timestamp"},
))

logger.addHandler(handler)
logger.setLevel(logging.INFO)

logger.info("User created", extra={
    "user_id": 42,
    "email":   "alice@example.com",
    "event":   "user.created",
})
# Output:
# {"timestamp": "2026-05-31T12:00:00", "level": "INFO", "name": "myapp.users",
#  "message": "User created", "user_id": 42, "email": "alice@example.com", "event": "user.created"}
```

---

## `structlog` — Structured Logging Alternative

```bash
pip install structlog
```

```python
import structlog

log = structlog.get_logger()

log.info("user.created",  user_id=42, email="alice@example.com")
log.warning("rate.limit", user_id=42, endpoint="/api/items", remaining=5)
log.error("payment.failed", order_id=99, amount=29.99, reason="card_declined")

# Bind context — carries through all subsequent log calls
bound_log = log.bind(request_id="req-abc-123", user_id=42)
bound_log.info("order.created", order_id=1)
bound_log.info("email.sent",    template="order_confirmation")
```

`structlog` outputs clean key-value or JSON logs and is the recommended choice for greenfield projects over the stdlib `logging` module.

---

## Filters — Fine-Grained Control

```python
import logging

class SensitiveFilter(logging.Filter):
    """Redact sensitive fields from log records."""

    SENSITIVE = {"password", "token", "secret", "api_key"}

    def filter(self, record: logging.LogRecord) -> bool:
        if hasattr(record, "extra"):
            for key in self.SENSITIVE:
                if key in record.extra:
                    record.extra[key] = "***REDACTED***"
        return True     # True = keep the record, False = drop it

class HealthCheckFilter(logging.Filter):
    """Suppress noisy health check endpoint logs."""

    def filter(self, record: logging.LogRecord) -> bool:
        return "/health" not in record.getMessage()


handler = logging.StreamHandler()
handler.addFilter(SensitiveFilter())
handler.addFilter(HealthCheckFilter())
```

---

## Request-Scoped Logging in FastAPI

Use `contextvars` to attach a `request_id` to every log line within a request without passing a logger around.

```python
import logging
import uuid
from contextvars import ContextVar
from fastapi import FastAPI, Request
from fastapi.middleware.base import BaseHTTPMiddleware

request_id_var: ContextVar[str] = ContextVar("request_id", default="-")

class RequestIDFilter(logging.Filter):
    def filter(self, record: logging.LogRecord) -> bool:
        record.request_id = request_id_var.get()
        return True

# Attach filter to all handlers
logging.getLogger().addFilter(RequestIDFilter())

class LoggingMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        rid = str(uuid.uuid4())
        token = request_id_var.set(rid)
        response = await call_next(request)
        request_id_var.reset(token)
        response.headers["X-Request-ID"] = rid
        return response

app = FastAPI()
app.add_middleware(LoggingMiddleware)

logger = logging.getLogger(__name__)

@app.get("/items")
async def list_items():
    logger.info("Listing items")   # automatically includes request_id
    return []
```

---

## Logging in FastAPI with Uvicorn

```python
# main.py
import logging
import logging.config
from contextlib import asynccontextmanager
from fastapi import FastAPI

LOGGING = {
    "version": 1,
    "disable_existing_loggers": False,
    "formatters": {
        "default": {
            "format": "%(asctime)s [%(levelname)s] %(name)s: %(message)s",
        },
    },
    "handlers": {
        "console": {
            "class":     "logging.StreamHandler",
            "formatter": "default",
        },
    },
    "loggers": {
        "myapp":          {"level": "DEBUG", "handlers": ["console"], "propagate": False},
        "uvicorn":        {"level": "INFO",  "handlers": ["console"], "propagate": False},
        "uvicorn.error":  {"level": "INFO",  "handlers": ["console"], "propagate": False},
        "uvicorn.access": {"level": "WARNING","handlers": ["console"], "propagate": False},
    },
}

@asynccontextmanager
async def lifespan(app: FastAPI):
    logging.config.dictConfig(LOGGING)
    logger = logging.getLogger("myapp")
    logger.info("Application starting")
    yield
    logger.info("Application shutting down")

app = FastAPI(lifespan=lifespan)
```

```bash
# Suppress uvicorn's own log config (use ours instead)
uvicorn main:app --no-access-log --log-config /dev/null
```

---

## Suppressing Noisy Third-Party Loggers

```python
import logging

# Suppress SQLAlchemy query output
logging.getLogger("sqlalchemy.engine").setLevel(logging.WARNING)

# Suppress httpx request logs
logging.getLogger("httpx").setLevel(logging.WARNING)

# Suppress urllib3 retry noise
logging.getLogger("urllib3.connectionpool").setLevel(logging.ERROR)

# Suppress boto3 / AWS SDK verbosity
logging.getLogger("boto3").setLevel(logging.WARNING)
logging.getLogger("botocore").setLevel(logging.WARNING)
```

---

## Performance — Avoid String Formatting on Disabled Levels

```python
import logging

logger = logging.getLogger(__name__)

# Bad — string is formatted even if DEBUG is disabled
logger.debug(f"Processing {len(items)} items: {items}")

# Good — % formatting is lazy — only evaluated if level is active
logger.debug("Processing %d items: %s", len(items), items)

# Also good — guard expensive computation
if logger.isEnabledFor(logging.DEBUG):
    logger.debug("Complex debug info: %s", expensive_computation())
```

---

## Quick Reference

|Task|Code|
|---|---|
|Get logger|`logging.getLogger(__name__)`|
|Set level|`logger.setLevel(logging.DEBUG)`|
|Log message|`logger.info("message")`|
|Log exception|`logger.exception("msg")` or `logger.error("msg", exc_info=True)`|
|Extra fields|`logger.info("msg", extra={"key": "val"})`|
|Bind context|`logging.LoggerAdapter(logger, extra={...})`|
|Console handler|`logging.StreamHandler()`|
|File handler|`logging.FileHandler("app.log")`|
|Rotating handler|`RotatingFileHandler("app.log", maxBytes=..., backupCount=...)`|
|Set formatter|`handler.setFormatter(logging.Formatter(fmt=...))`|
|Dict config|`logging.config.dictConfig(LOGGING_CONFIG)`|
|Suppress logger|`logging.getLogger("lib").setLevel(logging.WARNING)`|
|Add filter|`handler.addFilter(MyFilter())`|
|Check level|`logger.isEnabledFor(logging.DEBUG)`|
|JSON logging|`pip install python-json-logger`|
|Structured alt|`pip install structlog`|

---

## Tags

#python #logging #observability #monitoring #backend #structlog #fastapi