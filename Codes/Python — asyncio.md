
## What & When

**asyncio** is Python's built-in library for writing **concurrent code** using the `async`/`await` syntax. It runs on a single thread using an **event loop** that switches between tasks whenever one is waiting on I/O — network, disk, database, timers.

Use asyncio when:

- Making many network requests (HTTP, WebSocket, DB queries)
- Building servers that handle many concurrent connections
- Running tasks concurrently without threads or processes
- Working with FastAPI, aiohttp, SQLAlchemy async, websockets

Do NOT use asyncio for:

- CPU-bound work (use `multiprocessing` or `ProcessPoolExecutor`)
- Blocking libraries that don't support async (use `run_in_executor`)

---

## Sync vs Async Mental Model

```python
# Synchronous — blocks the whole program while waiting
import time

def fetch(url):
    time.sleep(2)           # nothing else can run during this
    return "data"

# Asynchronous — yields control while waiting
import asyncio

async def fetch(url):
    await asyncio.sleep(2)  # event loop runs other tasks during this
    return "data"
```

---

## Event Loop

The event loop is the engine that drives asyncio. It schedules and runs coroutines, handles I/O callbacks, and manages tasks.

```python
import asyncio

async def main():
    print("Hello from asyncio")

# Run the event loop
asyncio.run(main())         # creates loop, runs main(), closes loop
```

> [!info] `asyncio.run()` is the entry point Always use `asyncio.run(main())` at the top level. Never call `loop.run_until_complete()` directly in new code — it's the old pattern.

---

## Coroutines

A **coroutine** is a function defined with `async def`. Calling it returns a coroutine object — it does not execute until awaited.

```python
async def greet(name: str) -> str:
    await asyncio.sleep(1)          # simulate async work
    return f"Hello, {name}!"

async def main():
    result = await greet("Alice")   # execute the coroutine
    print(result)

asyncio.run(main())
```

> [!warning] Calling without `await` does nothing `greet("Alice")` returns a coroutine object but never runs it. Always `await` coroutines or schedule them as tasks.

---

## Tasks — Concurrent Execution

A **Task** wraps a coroutine and schedules it to run concurrently on the event loop. This is how you run things in parallel with asyncio.

```python
import asyncio

async def fetch(url: str) -> str:
    await asyncio.sleep(1)          # simulate network call
    return f"data from {url}"

async def main():
    # Create tasks — they start immediately
    task1 = asyncio.create_task(fetch("https://api.example.com/users"))
    task2 = asyncio.create_task(fetch("https://api.example.com/orders"))
    task3 = asyncio.create_task(fetch("https://api.example.com/products"))

    # Await them — total time ≈ 1s, not 3s
    result1 = await task1
    result2 = await task2
    result3 = await task3

asyncio.run(main())
```

---

## `asyncio.gather` — Run Many Coroutines

The most common way to run multiple coroutines concurrently and collect results.

```python
async def main():
    results = await asyncio.gather(
        fetch("https://api.example.com/users"),
        fetch("https://api.example.com/orders"),
        fetch("https://api.example.com/products"),
    )
    # results = ["data from /users", "data from /orders", "data from /products"]
    print(results)
```

### Handling Errors in `gather`

```python
# Default — one failure cancels all, raises the exception
results = await asyncio.gather(task1, task2, task3)

# return_exceptions=True — collect errors as results instead of raising
results = await asyncio.gather(task1, task2, task3, return_exceptions=True)

for result in results:
    if isinstance(result, Exception):
        print(f"Task failed: {result}")
    else:
        print(f"Result: {result}")
```

---

## `asyncio.wait` — More Control

```python
import asyncio

async def main():
    tasks = {
        asyncio.create_task(fetch(url))
        for url in ["url1", "url2", "url3"]
    }

    # Wait for all to complete
    done, pending = await asyncio.wait(tasks, return_when=asyncio.ALL_COMPLETED)

    # Wait for first to complete
    done, pending = await asyncio.wait(tasks, return_when=asyncio.FIRST_COMPLETED)

    # Wait for first to fail
    done, pending = await asyncio.wait(tasks, return_when=asyncio.FIRST_EXCEPTION)

    for task in done:
        print(task.result())

    for task in pending:
        task.cancel()           # cancel remaining tasks
```

---

## `asyncio.wait_for` — Timeout

```python
async def slow_operation():
    await asyncio.sleep(10)
    return "done"

async def main():
    try:
        result = await asyncio.wait_for(slow_operation(), timeout=3.0)
    except asyncio.TimeoutError:
        print("Operation timed out")
```

---

## `asyncio.as_completed` — Process as They Finish

```python
async def main():
    coroutines = [fetch(url) for url in urls]

    async for result in asyncio.as_completed(coroutines):  # py 3.13+
        print(f"Got: {await result}")

    # Pre 3.13
    for future in asyncio.as_completed(coroutines):
        result = await future
        print(f"Got: {result}")
```

---

## Task Management

```python
# Create a task
task = asyncio.create_task(my_coroutine(), name="my-task")

# Cancel a task
task.cancel()
try:
    await task
except asyncio.CancelledError:
    print("Task was cancelled")

# Check task state
task.done()         # True if finished, cancelled, or errored
task.cancelled()    # True if cancelled
task.result()       # result value (raises if not done or cancelled)
task.exception()    # exception if one was raised

# Get all running tasks
tasks = asyncio.all_tasks()

# Get the current task
current = asyncio.current_task()
```

---

## Semaphore — Limit Concurrency

Prevent hammering a service by limiting how many coroutines run at once.

```python
import asyncio
import httpx

async def fetch(client, url, semaphore):
    async with semaphore:               # only N tasks enter at a time
        response = await client.get(url)
        return response.json()

async def main():
    semaphore = asyncio.Semaphore(10)   # max 10 concurrent requests
    urls = [f"https://api.example.com/items/{i}" for i in range(100)]

    async with httpx.AsyncClient() as client:
        tasks = [fetch(client, url, semaphore) for url in urls]
        results = await asyncio.gather(*tasks)
```

---

## Lock — Mutual Exclusion

Prevent race conditions when coroutines share mutable state.

```python
import asyncio

counter = 0
lock = asyncio.Lock()

async def increment():
    global counter
    async with lock:                    # only one coroutine at a time
        value = counter
        await asyncio.sleep(0)          # yield — without lock this would race
        counter = value + 1

async def main():
    await asyncio.gather(*[increment() for _ in range(100)])
    print(counter)                      # 100, not a random lower number
```

---

## Queue — Producer / Consumer

```python
import asyncio

async def producer(queue: asyncio.Queue):
    for i in range(10):
        await queue.put(f"item-{i}")
        await asyncio.sleep(0.1)
    await queue.put(None)               # sentinel — signal done

async def consumer(queue: asyncio.Queue):
    while True:
        item = await queue.get()
        if item is None:
            break
        print(f"Processing: {item}")
        queue.task_done()

async def main():
    queue = asyncio.Queue(maxsize=5)    # buffer of 5 items
    await asyncio.gather(
        producer(queue),
        consumer(queue),
    )

asyncio.run(main())
```

---

## Event — Notify Between Coroutines

```python
import asyncio

async def waiter(event: asyncio.Event):
    print("Waiting for event...")
    await event.wait()
    print("Event received!")

async def trigger(event: asyncio.Event):
    await asyncio.sleep(2)
    event.set()                         # wake all waiters

async def main():
    event = asyncio.Event()
    await asyncio.gather(waiter(event), trigger(event))
```

---

## Running Blocking Code — `run_in_executor`

Use `run_in_executor` to run synchronous (blocking) code without blocking the event loop.

```python
import asyncio
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor

def blocking_io(path: str) -> str:
    with open(path) as f:
        return f.read()

def cpu_heavy(n: int) -> int:
    return sum(i * i for i in range(n))

async def main():
    loop = asyncio.get_event_loop()

    # Thread pool — for blocking I/O (file, legacy DB drivers, requests)
    result = await loop.run_in_executor(None, blocking_io, "/etc/hosts")

    # Custom thread pool
    with ThreadPoolExecutor(max_workers=4) as pool:
        result = await loop.run_in_executor(pool, blocking_io, "/etc/hosts")

    # Process pool — for CPU-bound work
    with ProcessPoolExecutor() as pool:
        result = await loop.run_in_executor(pool, cpu_heavy, 10_000_000)
```

> [!warning] Never call blocking functions directly in async code Calling `time.sleep()`, `requests.get()`, or any blocking I/O directly inside an `async def` freezes the entire event loop for every connected client.

---

## `asyncio.sleep` — Yield Control

```python
await asyncio.sleep(1.0)    # sleep for 1 second, other tasks run
await asyncio.sleep(0)      # yield to event loop immediately, no delay
```

`await asyncio.sleep(0)` is useful inside long CPU loops to keep the event loop responsive without actually sleeping.

---

## Context Variables — `contextvars`

Share request-scoped data across coroutines without passing arguments. Used by FastAPI internally for request context.

```python
from contextvars import ContextVar

request_id: ContextVar[str] = ContextVar("request_id", default="none")

async def handler():
    request_id.set("req-abc-123")
    await process()

async def process():
    print(request_id.get())     # "req-abc-123" — inherited from handler
```

---

## Async Context Managers

```python
class ManagedResource:
    async def __aenter__(self):
        await self.connect()
        return self

    async def __aexit__(self, *args):
        await self.disconnect()

async def main():
    async with ManagedResource() as resource:
        await resource.do_work()
```

```python
# Using contextlib
from contextlib import asynccontextmanager

@asynccontextmanager
async def managed_db():
    db = await connect()
    try:
        yield db
    finally:
        await db.close()

async def main():
    async with managed_db() as db:
        await db.query("SELECT 1")
```

---

## Async Iterators & Generators

```python
# Async generator — yield values over time
async def ticker(interval: float, count: int):
    for i in range(count):
        yield i
        await asyncio.sleep(interval)

async def main():
    async for tick in ticker(0.5, 10):
        print(tick)
```

```python
# Async comprehension
results = [result async for result in ticker(0.1, 5)]
results = {r async for r in ticker(0.1, 5)}
```

---

## Common asyncio Patterns in FastAPI

```python
# Run multiple DB queries concurrently
async def get_dashboard(db: AsyncSession = Depends(get_db)):
    users, orders, revenue = await asyncio.gather(
        db.execute(select(func.count(User.id))),
        db.execute(select(func.count(Order.id))),
        db.execute(select(func.sum(Order.total))),
    )
    return {
        "users":   users.scalar(),
        "orders":  orders.scalar(),
        "revenue": revenue.scalar(),
    }

# Fire-and-forget background task
async def notify_user(user_id: int):
    await asyncio.sleep(0)          # yield first
    await send_email(user_id)

@app.post("/orders")
async def create_order(payload: OrderCreate):
    order = await save_order(payload)
    asyncio.create_task(notify_user(order.user_id))   # don't await
    return order
```

---

## Debugging asyncio

```python
# Enable debug mode — logs slow callbacks, unawaited coroutines
import asyncio
asyncio.run(main(), debug=True)

# Or via environment variable
# PYTHONASYNCIODEBUG=1 python main.py

# Detect blocking calls (threshold in seconds)
loop = asyncio.get_event_loop()
loop.slow_callback_duration = 0.1   # warn if callback takes > 100ms
```

---

## Quick Reference

|Task|Code|
|---|---|
|Run event loop|`asyncio.run(main())`|
|Define coroutine|`async def fn(): ...`|
|Await coroutine|`await fn()`|
|Run concurrently|`await asyncio.gather(c1, c2, c3)`|
|Create task|`asyncio.create_task(coro)`|
|Timeout|`await asyncio.wait_for(coro, timeout=5)`|
|First completed|`asyncio.wait(..., return_when=FIRST_COMPLETED)`|
|Limit concurrency|`asyncio.Semaphore(n)`|
|Mutual exclusion|`asyncio.Lock()`|
|Producer/consumer|`asyncio.Queue()`|
|Signal between tasks|`asyncio.Event()`|
|Run blocking code|`loop.run_in_executor(None, fn, args)`|
|Yield to loop|`await asyncio.sleep(0)`|
|Async context mgr|`async with` / `@asynccontextmanager`|
|Async iteration|`async for x in gen`|
|Async generator|`async def gen(): yield value`|

---

## Tags

#python #asyncio #async #concurrency #event-loop #backend