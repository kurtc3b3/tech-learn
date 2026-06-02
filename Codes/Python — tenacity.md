## What & When

**tenacity** is a general-purpose retrying library for Python. It wraps functions with configurable retry logic — handling transient failures from network calls, database connections, external APIs, or any flaky operation — without cluttering your code with manual retry loops.

Use tenacity when:

- Calling external APIs that may rate-limit or timeout
- Connecting to databases or queues that may be temporarily unavailable
- Handling transient network errors gracefully
- Implementing exponential backoff with jitter
- Retrying async FastAPI service calls

```bash
pip install tenacity
```

---

## Basic Usage

```python
from tenacity import retry, stop_after_attempt, wait_fixed

@retry(stop=stop_after_attempt(3), wait=wait_fixed(2))
def call_api():
    response = requests.get("https://api.example.com/data")
    response.raise_for_status()
    return response.json()

# Retries up to 3 times, waiting 2 seconds between each attempt
```

---

## `stop` — When to Stop Retrying

```python
from tenacity import (
    stop_after_attempt,     # stop after N attempts
    stop_after_delay,       # stop after N seconds total
    stop_never,             # retry forever (use with caution)
)

# Stop after 5 attempts
@retry(stop=stop_after_attempt(5))

# Stop after 30 seconds total
@retry(stop=stop_after_delay(30))

# Stop after 5 attempts OR 30 seconds — whichever comes first
from tenacity import stop_any
@retry(stop=stop_any(stop_after_attempt(5), stop_after_delay(30)))

# Stop after 5 attempts AND 30 seconds — both must be exceeded
from tenacity import stop_all
@retry(stop=stop_all(stop_after_attempt(5), stop_after_delay(30)))

# Retry forever
@retry(stop=stop_never)
```

---

## `wait` — How Long to Wait Between Attempts

```python
from tenacity import (
    wait_fixed,             # fixed delay
    wait_random,            # random delay between min and max
    wait_exponential,       # exponential backoff
    wait_random_exponential,# exponential backoff + jitter (recommended)
    wait_none,              # no delay
    wait_incrementing,      # incrementing delay
    wait_chain,             # different waits per attempt
)

# Fixed — wait 2 seconds between every attempt
@retry(wait=wait_fixed(2))

# Random — wait between 1 and 5 seconds
@retry(wait=wait_random(min=1, max=5))

# Exponential — 1s, 2s, 4s, 8s, ...
@retry(wait=wait_exponential(multiplier=1, min=1, max=60))

# Exponential with jitter — recommended for distributed systems
# avoids thundering herd when many clients retry simultaneously
@retry(wait=wait_random_exponential(multiplier=1, min=1, max=60))

# No delay
@retry(wait=wait_none())

# Incrementing — 0s, 5s, 10s, 15s, ...
@retry(wait=wait_incrementing(start=0, increment=5, max=30))

# Chain — different waits for each attempt
@retry(wait=wait_chain(
    wait_none(),            # attempt 1 — no wait
    wait_fixed(1),          # attempt 2 — wait 1s
    wait_fixed(5),          # attempt 3+ — wait 5s
))

# Combine — exponential + random jitter
from tenacity import wait_combine
@retry(wait=wait_combine(wait_exponential(max=10), wait_random(0, 2)))
```

---

## `retry` — What Errors to Retry On

```python
from tenacity import (
    retry_if_exception_type,
    retry_if_not_exception_type,
    retry_if_exception_message,
    retry_if_result,
    retry_if_not_result,
    retry_any,
    retry_all,
)
import requests
from requests.exceptions import ConnectionError, Timeout

# Retry only on specific exceptions
@retry(retry=retry_if_exception_type(ConnectionError))

# Retry on multiple exception types
@retry(retry=retry_if_exception_type((ConnectionError, Timeout)))

# Retry on any exception EXCEPT a specific type
@retry(retry=retry_if_not_exception_type(ValueError))

# Retry based on exception message
@retry(retry=retry_if_exception_message(match="rate limit"))

# Retry based on return value — retry if result is None
@retry(retry=retry_if_result(lambda r: r is None))

# Retry if result does NOT satisfy predicate
@retry(retry=retry_if_not_result(lambda r: r.get("status") == "success"))

# Combine — retry on connection error OR if result is None
@retry(retry=retry_any(
    retry_if_exception_type(ConnectionError),
    retry_if_result(lambda r: r is None),
))

# Combine — retry only if BOTH conditions are true
@retry(retry=retry_all(
    retry_if_exception_type(Exception),
    retry_if_exception_message(match="timeout"),
))
```

---

## `reraise` — Re-raise the Final Exception

By default, tenacity raises `RetryError` when all attempts are exhausted. Use `reraise=True` to get the original exception instead.

```python
from tenacity import retry, stop_after_attempt, wait_fixed

@retry(
    stop=stop_after_attempt(3),
    wait=wait_fixed(1),
    reraise=True,           # raise original exception, not RetryError
)
def fetch_data():
    response = requests.get("https://api.example.com/data")
    response.raise_for_status()
    return response.json()

try:
    data = fetch_data()
except requests.HTTPError as e:
    print(f"Final failure: {e}")    # original exception, not RetryError
```

---

## Callbacks — `before`, `after`, `before_sleep`

```python
import logging
from tenacity import (
    retry, stop_after_attempt, wait_exponential,
    before_log, after_log, before_sleep_log,
    RetryCallState,
)

logger = logging.getLogger(__name__)

# Built-in logging callbacks
@retry(
    stop=stop_after_attempt(5),
    wait=wait_exponential(multiplier=1, max=10),
    before=before_log(logger, logging.DEBUG),           # log before each attempt
    after=after_log(logger, logging.DEBUG),             # log after each attempt
    before_sleep=before_sleep_log(logger, logging.WARNING),  # log before sleeping
)
def flaky_call():
    ...

# Custom callbacks
def log_attempt(retry_state: RetryCallState) -> None:
    logger.warning(
        "Attempt %d failed: %s",
        retry_state.attempt_number,
        retry_state.outcome.exception(),
    )

def log_final_failure(retry_state: RetryCallState) -> None:
    logger.error(
        "All %d attempts failed. Giving up.",
        retry_state.attempt_number,
    )

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(max=10),
    after=log_attempt,
    retry_error_callback=log_final_failure,
)
def call_service():
    ...
```

---

## `retry_error_callback` — Return a Fallback Value

Instead of raising on exhaustion, return a default value.

```python
from tenacity import retry, stop_after_attempt, RetryCallState

def fallback(retry_state: RetryCallState):
    return []   # return empty list instead of raising

@retry(
    stop=stop_after_attempt(3),
    retry_error_callback=fallback,
)
def get_items() -> list:
    response = requests.get("https://api.example.com/items")
    response.raise_for_status()
    return response.json()

items = get_items()     # returns [] if all attempts fail
```

---

## Async Support

tenacity works natively with `async def` functions.

```python
import asyncio
import httpx
from tenacity import retry, stop_after_attempt, wait_exponential, retry_if_exception_type

@retry(
    stop=stop_after_attempt(5),
    wait=wait_random_exponential(multiplier=1, min=1, max=30),
    retry=retry_if_exception_type(httpx.HTTPStatusError),
    reraise=True,
)
async def fetch_async(url: str) -> dict:
    async with httpx.AsyncClient() as client:
        response = await client.get(url)
        response.raise_for_status()
        return response.json()

async def main():
    data = await fetch_async("https://api.example.com/data")

asyncio.run(main())
```

---

## Using with httpx in FastAPI

```python
import httpx
from tenacity import (
    retry, stop_after_attempt, wait_random_exponential,
    retry_if_exception_type, before_sleep_log,
)
from fastapi import FastAPI, Depends
from contextlib import asynccontextmanager
import logging

logger = logging.getLogger(__name__)

def is_retryable(exc: Exception) -> bool:
    if isinstance(exc, httpx.HTTPStatusError):
        return exc.response.status_code in {429, 500, 502, 503, 504}
    return isinstance(exc, (httpx.ConnectError, httpx.TimeoutException))

@retry(
    stop=stop_after_attempt(4),
    wait=wait_random_exponential(multiplier=1, min=1, max=30),
    retry=retry_if_exception_type((httpx.ConnectError, httpx.TimeoutException, httpx.HTTPStatusError)),
    before_sleep=before_sleep_log(logger, logging.WARNING),
    reraise=True,
)
async def resilient_get(client: httpx.AsyncClient, url: str) -> dict:
    response = await client.get(url)
    response.raise_for_status()
    return response.json()

@asynccontextmanager
async def lifespan(app: FastAPI):
    async with httpx.AsyncClient(
        base_url="https://api.partner.com",
        timeout=10.0,
    ) as client:
        app.state.http_client = client
        yield

app = FastAPI(lifespan=lifespan)

@app.get("/partner/data")
async def get_partner_data(request):
    client = request.app.state.http_client
    return await resilient_get(client, "/data")
```

---

## `Retrying` — Programmatic / Context Manager

Use `Retrying` directly when you can't use a decorator (e.g. inside a loop or when the function isn't yours to decorate).

```python
from tenacity import Retrying, stop_after_attempt, wait_fixed, RetryError

# Context manager style
try:
    for attempt in Retrying(stop=stop_after_attempt(3), wait=wait_fixed(1)):
        with attempt:
            result = call_external_service()
except RetryError:
    print("All attempts failed")

# Check attempt number
for attempt in Retrying(stop=stop_after_attempt(5)):
    with attempt:
        print(f"Attempt {attempt.retry_state.attempt_number}")
        result = flaky_function()
```

---

## Async `AsyncRetrying`

```python
from tenacity import AsyncRetrying, stop_after_attempt, wait_exponential, RetryError

async def main():
    try:
        async for attempt in AsyncRetrying(
            stop=stop_after_attempt(3),
            wait=wait_exponential(max=10),
        ):
            with attempt:
                result = await async_call()
    except RetryError:
        print("All attempts exhausted")
```

---

## `RetryCallState` — Inspect Retry State

```python
from tenacity import RetryCallState

def on_retry(retry_state: RetryCallState) -> None:
    print(f"Attempt:   {retry_state.attempt_number}")
    print(f"Elapsed:   {retry_state.outcome_timestamp - retry_state.start_time:.2f}s")
    print(f"Exception: {retry_state.outcome.exception()}")
    print(f"Args:      {retry_state.args}")
    print(f"Kwargs:    {retry_state.kwargs}")
```

---

## Combining with `urllib3` / `requests`

```python
import requests
from requests.exceptions import ConnectionError, Timeout, HTTPError
from tenacity import (
    retry, stop_after_attempt, wait_random_exponential,
    retry_if_exception_type, before_sleep_log,
)
import logging

logger = logging.getLogger(__name__)

@retry(
    stop=stop_after_attempt(5),
    wait=wait_random_exponential(multiplier=1, min=2, max=60),
    retry=retry_if_exception_type((ConnectionError, Timeout)),
    before_sleep=before_sleep_log(logger, logging.WARNING),
    reraise=True,
)
def get_with_retry(session: requests.Session, url: str) -> dict:
    response = session.get(url, timeout=(3, 10))
    response.raise_for_status()
    return response.json()
```

---

## Common Patterns

### Retry Only on 5xx — Not 4xx

```python
import httpx
from tenacity import retry, stop_after_attempt, wait_exponential, retry_if_exception

def is_server_error(exc: Exception) -> bool:
    if isinstance(exc, httpx.HTTPStatusError):
        return exc.response.status_code >= 500
    return True     # retry on all other exceptions

@retry(
    stop=stop_after_attempt(4),
    wait=wait_exponential(multiplier=1, max=30),
    retry=retry_if_exception(is_server_error),
    reraise=True,
)
async def call(client: httpx.AsyncClient, url: str) -> dict:
    r = await client.get(url)
    r.raise_for_status()
    return r.json()
```

### Rate Limit — Respect `Retry-After` Header

```python
import httpx, time
from tenacity import retry, stop_after_attempt, wait_base

class WaitRetryAfter(wait_base):
    def __call__(self, retry_state: "RetryCallState") -> float:
        exc = retry_state.outcome.exception()
        if isinstance(exc, httpx.HTTPStatusError) and exc.response.status_code == 429:
            retry_after = exc.response.headers.get("Retry-After", "5")
            return float(retry_after)
        return 2.0      # default wait

@retry(
    stop=stop_after_attempt(5),
    wait=WaitRetryAfter(),
    reraise=True,
)
async def rate_limited_call(client, url):
    r = await client.get(url)
    r.raise_for_status()
    return r.json()
```

### Database Connection Retry

```python
from sqlalchemy.exc import OperationalError
from tenacity import retry, stop_after_attempt, wait_exponential, retry_if_exception_type
from contextlib import asynccontextmanager
from fastapi import FastAPI

@retry(
    stop=stop_after_attempt(10),
    wait=wait_exponential(multiplier=1, min=1, max=10),
    retry=retry_if_exception_type(OperationalError),
    before_sleep=before_sleep_log(logger, logging.WARNING),
)
async def connect_db():
    engine = create_async_engine(DATABASE_URL)
    async with engine.connect() as conn:
        await conn.execute(text("SELECT 1"))
    return engine

@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.engine = await connect_db()   # retry on startup
    yield
    await app.state.engine.dispose()
```

---

## Quick Reference

|Task|Code|
|---|---|
|Basic retry|`@retry(stop=stop_after_attempt(3))`|
|Fixed wait|`wait=wait_fixed(2)`|
|Exponential backoff|`wait=wait_exponential(multiplier=1, max=60)`|
|Backoff + jitter|`wait=wait_random_exponential(min=1, max=60)`|
|Stop after time|`stop=stop_after_delay(30)`|
|Stop after attempts|`stop=stop_after_attempt(5)`|
|Stop either|`stop=stop_any(attempts, delay)`|
|Retry on exception|`retry=retry_if_exception_type(MyError)`|
|Retry on result|`retry=retry_if_result(lambda r: r is None)`|
|Re-raise original|`reraise=True`|
|Fallback value|`retry_error_callback=lambda s: default`|
|Log before sleep|`before_sleep=before_sleep_log(logger, logging.WARNING)`|
|Async function|Works natively with `async def`|
|No decorator|`for attempt in Retrying(...): with attempt:`|
|Async no decorator|`async for attempt in AsyncRetrying(...)`|

---

## Tags

#python #tenacity #retry #resilience #backoff #networking #backend