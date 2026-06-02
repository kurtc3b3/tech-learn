## What & When

**Celery** is a distributed **task queue** for Python: producers enqueue work to a **broker** (Redis, RabbitMQ); **workers** pull messages, run `@task` functions, and optionally store results in a **result backend**. **Celery Beat** adds cron-style schedules.

Use Celery when:

- Work must survive process restarts and run **off the HTTP path** (see [[Processing]], [[API - FastAPI]])
- You need **retries**, **timeouts**, **rate limits**, or **periodic jobs** (ETL, emails, report generation)
- Multiple worker machines share one queue
- Pipeline stages compose as **chains**, **groups**, or **chords**
- Scrapers or batch jobs run on a schedule — pair with [[Browser Automation — Scrapy]] and Beat

```bash
pip install celery redis
# RabbitMQ broker: pip install celery[librabbitmq]  # or use amqp:// URL
```

For flaky **in-process** HTTP calls inside a task, wrap with [[Python — tenacity]]; Celery's own `autoretry_for` covers task-level retries. Brokers: [[DB — Redis]], [[DB — RabbitMQ]]. Overview: [[Processing]], [[DB]].

---

## Celery vs Related Tools

| Need | Use | Notes |
| --- | --- | --- |
| Fire-and-forget in same process | FastAPI `BackgroundTasks` | No broker; dies with worker |
| Async I/O, many connections | [[Python — asyncio]] | Not for CPU-bound fan-out |
| Multi-core, one machine | [[Python — multiprocessing]] | No queue, no remote workers |
| **Job queue + cron + retries** | **Celery** | Broker required |
| Parallel Python / ML cluster | [[Processing — Ray]] | Object store, not message queue |
| Sync retry decorator | [[Python — tenacity]] | Inside task body or API client |

---

## Celery App & Configuration

```python
# celery_app.py
from celery import Celery

app = Celery(
    "myapp",
    broker="redis://localhost:6379/0",
    backend="redis://localhost:6379/1",  # result backend (optional)
    include=["tasks"],  # module(s) with @app.task
)

app.conf.update(
    task_serializer="json",
    accept_content=["json"],
    result_serializer="json",
    timezone="UTC",
    enable_utc=True,
    task_track_started=True,
    task_time_limit=3600,
    worker_prefetch_multiplier=1,  # fair distribution for long tasks
)
```

### Broker & Result Backend Options

| Backend | URL example | Notes |
| --- | --- | --- |
| **Redis** | `redis://localhost:6379/0` | Simple; broker + results on same host |
| **RabbitMQ** | `amqp://guest:guest@localhost:5672//` | Durable queues, routing exchanges |
| Results only | `redis://.../1` or `db+postgresql://...` | Omit backend if fire-and-forget |

```python
# RabbitMQ broker, Redis results (common hybrid)
app = Celery(
    "myapp",
    broker="amqp://guest:guest@localhost:5672//",
    backend="redis://localhost:6379/1",
)
```

---

## Defining Tasks

```python
# tasks.py
from celery_app import app
import time

@app.task(bind=True, max_retries=3, default_retry_delay=60)
def process_order(self, order_id: str) -> dict:
    try:
        # business logic
        time.sleep(2)
        return {"order_id": order_id, "status": "shipped"}
    except ConnectionError as exc:
        raise self.retry(exc=exc)

# Call from API, script, or another task
result = process_order.delay("ord-123")
# result.id → task_id for polling
```

## Worker

```bash
celery -A celery_app worker --loglevel=info --concurrency=4
celery -A celery_app inspect active   # optional: pip install flower → celery flower
```

Task states: `PENDING` → `STARTED` → `SUCCESS` | `FAILURE` | `RETRY`. Tests: `app.conf.task_always_eager = True`.

---

## Celery Beat (Scheduler)

```python
# celery_app.py — periodic schedule
from celery.schedules import crontab

app.conf.beat_schedule = {
    "nightly-etl": {
        "task": "tasks.run_etl",
        "schedule": crontab(hour=2, minute=0),
        "args": (),
    },
    "scrape-books-hourly": {
        "task": "tasks.run_books_crawl",
        "schedule": crontab(minute=0),  # every hour
    },
}
```

```bash
celery -A celery_app beat --loglevel=info
# Production: run beat on ONE instance; workers scale horizontally
```

---

## Chains, Groups, Chords

```python
from celery import chain, group, chord
from tasks import fetch_row, transform_row, aggregate_results

chain(fetch_row.s(1), transform_row.s(), aggregate_results.s()).apply_async()
group(transform_row.s(i) for i in range(100)).apply_async()
chord((transform_row.s(i) for i in range(100)), aggregate_results.s()).apply_async()
```

**Chains** = sequential ETL; **groups** = parallel map; **chords** = group then one aggregator callback.

---

## Retries: Celery vs tenacity

| Layer | Tool | When |
| --- | --- | --- |
| Task failure → re-queue | `self.retry()`, `autoretry_for` | Transient DB/API in worker |
| Function call inside task | [[Python — tenacity]] `@retry` | HTTP client, small sync helper |

```python
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(stop=stop_after_attempt(5), wait=wait_exponential(multiplier=1, max=30))
def call_external_api(payload: dict) -> dict:
    ...

@app.task(bind=True, autoretry_for=(ConnectionError,), retry_backoff=True, max_retries=5)
def sync_inventory(self):
    data = call_external_api({"sku": "all"})
    ...
```

---

## FastAPI Integration — Enqueue & Poll `task_id`

Return **202 Accepted** with `task_id`; client polls a status route. Do not block the request on long Celery work.

```python
# api.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from tasks import process_order
from celery_app import app as celery_app
from celery.result import AsyncResult

api = FastAPI()

class OrderIn(BaseModel):
    order_id: str

class TaskAccepted(BaseModel):
    task_id: str
    status: str = "PENDING"

@api.post("/orders", status_code=202, response_model=TaskAccepted)
async def create_order(body: OrderIn):
    async_result = process_order.delay(body.order_id)
    return TaskAccepted(task_id=async_result.id)

@api.get("/tasks/{task_id}")
async def get_task(task_id: str):
    r = AsyncResult(task_id, app=celery_app)
    if r.state == "PENDING":
        return {"task_id": task_id, "state": r.state}
    if r.failed():
        raise HTTPException(500, detail=str(r.result))
    if r.successful():
        return {"task_id": task_id, "state": r.state, "result": r.result}
    return {"task_id": task_id, "state": r.state}
```

Auth, rate limits, OpenAPI: keep in [[API - FastAPI]]; workers remain plain importable modules.

---

## Scrapy + Celery Pattern

Scrapy runs on **Twisted**, not asyncio — trigger crawls from Celery (subprocess or `CrawlerProcess` in a dedicated worker). See [[Browser Automation — Scrapy]].

```python
# tasks.py
from celery import shared_task
import subprocess

@shared_task
def run_books_crawl():
    subprocess.run(
        ["scrapy", "crawl", "books", "-o", "/data/books.json"],
        cwd="/app/myproject",
        check=True,
    )
```

Schedule via Beat; store artifacts on shared volume or object storage.

---

## Backend Patterns

### Architecture

```text
Client → [[API - FastAPI]] → .delay() → Redis/RabbitMQ → Celery worker(s)
                ← task_id (202)     ← result backend ← poll GET /tasks/{id}
Beat → enqueue on cron ────────────────────────────────┘
```

**Production:** separate API / worker / beat (one beat leader); idempotent tasks; `task_track_started` or Flower; monitor `FAILURE` and queue depth; broker URL from env.

---

## Pitfalls

- **Running CPU-heavy work in the API process** — always `.delay()` for seconds+ jobs.
- **No result backend** — `AsyncResult.result` blocks or fails; configure Redis or disable result expectations.
- **Beat duplicated** — multiple beat instances → duplicate cron fires.
- **Large payloads in messages** — pass IDs/URLs, not multi-MB blobs.
- **JSON serializer** — bytes/datetimes need custom encoding or pickle (avoid pickle in untrusted envs).

---

## Quick Reference

| Task | Code |
| --- | --- |
| Create app | `Celery("name", broker=..., backend=...)` |
| Define task | `@app.task` / `@shared_task` |
| Enqueue | `my_task.delay(*args)` / `apply_async(kwargs={})` |
| Task id | `result.id` |
| Poll | `AsyncResult(task_id, app=app).state` |
| Chain | `chain(a.s(), b.s())()` |
| Group | `group(t.s(i) for i in xs)()` |
| Chord | `chord(group_tasks, callback.s())()` |
| Beat | `app.conf.beat_schedule` + `celery beat` |
| Worker | `celery -A celery_app worker -l info` |

---

## Related Notes

- [[Processing]]
- [[Processing — Ray]]
- [[API - FastAPI]]
- [[API - FastAPI — Lifespan]]
- [[Python — asyncio]]
- [[Python — multiprocessing]]
- [[Python — tenacity]]
- [[Browser Automation — Scrapy]]
- [[Python Development]]

---

## Tags

#python #celery #redis #rabbitmq #task-queue #distributed #backend #fastapi #beat #processing #scrapy #etl
