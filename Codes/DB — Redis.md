## What & When

**Redis** is an in-memory **key-value data store** with rich structures (strings, hashes, lists, sets, sorted sets, streams). Use it for **cache**, **session store**, **rate limiting**, **distributed locks**, **pub/sub**, and as **Celery broker/result backend** — not as the system of record for relational data ([[ORM - SQLAlchemy]]).

Use Redis when:

- **Sub-millisecond reads** for hot keys (API response cache)
- **Ephemeral state** — sessions, OTP codes, idempotency keys
- **Coordination** — locks, leader election, simple queues
- **Celery** broker or result backend — [[Processing — Celery]]
- **Real-time** pub/sub fan-out

```bash
pip install redis
# Server: docker run -d -p 6379:6379 redis:7-alpine
# or Compose — [[Commands/CLI — Docker & Compose]]
```

Overview: [[DB]].

---

## Redis vs Related Tools

| Need | Use | Notes |
| --- | --- | --- |
| Cache / sessions | **Redis** | TTL keys |
| Task queue (Python) | Redis or [[DB — RabbitMQ]] | Celery |
| Event log / replay | [[DB — Kafka]] | Not Redis streams alone at scale |
| Persistent OLTP | PostgreSQL + [[ORM - SQLAlchemy]] | Source of truth |
| Document JSON | [[DB — MongoDB]] | Different model |
| Metrics time series | [[DB — InfluxDB]] / Prometheus | Not Redis |

---

## Python Client (redis-py)

```python
import redis

r = redis.Redis(host="localhost", port=6379, db=0, decode_responses=True)

r.set("user:42:session", "abc123", ex=3600)   # TTL 1 hour
r.get("user:42:session")

r.hset("user:42", mapping={"name": "Ada", "plan": "pro"})
r.hgetall("user:42")

r.incr("rate:ip:1.2.3.4")
r.expire("rate:ip:1.2.3.4", 60)
```

Async (FastAPI):

```python
import redis.asyncio as redis

async def get_redis():
    return redis.from_url("redis://localhost:6379/0", decode_responses=True)

# In route / lifespan — [[API - FastAPI — Lifespan]]
```

---

## Cache-Aside Pattern

```python
import json

async def get_user(user_id: int, db, cache):
    key = f"user:{user_id}"
    cached = await cache.get(key)
    if cached:
        return json.loads(cached)
    user = await db.get_user(user_id)  # [[ORM - Queries]]
    if user:
        await cache.set(key, json.dumps(user), ex=300)
    return user
```

Invalidate on update: `await cache.delete(f"user:{user_id}")`.

---

## Rate Limiting (Simple)

```python
async def allow_request(cache, key: str, limit: int, window: int) -> bool:
    count = await cache.incr(key)
    if count == 1:
        await cache.expire(key, window)
    return count <= limit
```

Production HTTP APIs: [[API - FastAPI — Rate Limiting (SlowAPI)]] with `storage_uri=redis://...`. Other patterns: sliding window or token bucket libraries; Redis Cell module.

---

## Celery Integration

```python
# broker=redis://localhost:6379/0
# backend=redis://localhost:6379/1
```

See [[Processing — Celery]], [[DB — RabbitMQ]] for AMQP alternative.

---

## Pub/Sub

```python
pubsub = r.pubsub()
pubsub.subscribe("notifications")
for message in pubsub.listen():
    if message["type"] == "message":
        print(message["data"])
```

Pub/sub is fire-and-forget — subscribers offline miss messages. Use **Redis Streams** or **Kafka** for durable fan-out.

---

## Docker Compose Snippet

```yaml
services:
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redisdata:/data
volumes:
  redisdata:
```

Persistence: RDB/AOF configurable; still treat as cache tier unless explicitly designed otherwise.

---

## Operations Notes

| Topic | Guidance |
| --- | --- |
| Memory | Set `maxmemory` + eviction policy (`allkeys-lru`) |
| TTL | Always set on cache keys |
| Cluster | Redis Cluster for horizontal scale |
| Security | Password, private network, no public 6379 |
| Serialization | JSON for values; avoid pickle across services |

---

## Quick Reference

| Task | Command / API |
| --- | --- |
| Set with TTL | `SET key val EX 3600` / `r.set(..., ex=3600)` |
| Hash | `HSET`, `HGETALL` |
| List queue | `LPUSH` / `BRPOP` |
| Atomic incr | `INCR` |
| Delete | `DEL key` |
| CLI ping | `redis-cli ping` |

---

## Related Notes

- [[DB]]
- [[DB — RabbitMQ]]
- [[Processing — Celery]]
- [[ORM - SQLAlchemy]]
- [[API - FastAPI — Lifespan]]
- [[Commands/CLI — Docker & Compose]]

---

## Tags

#database #redis #cache #celery #python #backend #key-value
