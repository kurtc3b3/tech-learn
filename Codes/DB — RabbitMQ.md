## What & When

**RabbitMQ** is a **message broker** implementing AMQP — producers publish to **exchanges** that **route** messages to **queues**; consumers **acknowledge** delivery. Primary broker choice for [[Processing — Celery]] when you need durable queues, routing keys, and dead-letter handling.

Use RabbitMQ when:

- **Celery** task broker with routing (`task_routes`, priority queues)
- **Work queues** — many workers, competing consumers
- **Pub/sub patterns** — fanout, topic, headers exchanges
- **RPC-style** request/reply over AMQP (less common today vs HTTP)

```bash
pip install pika
# Celery: broker=amqp://guest:guest@localhost:5672//
# Server: docker run -d -p 5672:5672 -p 15672:15672 rabbitmq:3-management
```

Overview: [[DB]]. Celery patterns: [[Processing — Celery]].

---

## RabbitMQ vs Related Tools

| Need | Use | Notes |
| --- | --- | --- |
| Celery + routing | **RabbitMQ** | AMQP exchanges |
| Simple Celery broker | [[DB — Redis]] | Easier ops |
| Event log / replay | [[DB — Kafka]] | Not Rabbit primary |
| Fire-and-forget in API | FastAPI BackgroundTasks | No broker |
| Task queue Python | [[Processing — Celery]] | Broker agnostic |

---

## Mental Model

```text
Producer → Exchange → (bindings) → Queue → Consumer
                routing key: order.created
```

| Exchange type | Routing |
| --- | --- |
| **direct** | Exact routing key match |
| **topic** | Pattern `order.*` |
| **fanout** | All bound queues |
| **headers** | Match headers |

Interactive demo concept: producer → queue → consumer (see lagaluga RabbitMQ demo).

---

## Celery Configuration

```python
app = Celery(
    "myapp",
    broker="amqp://guest:guest@localhost:5672//",
    backend="redis://localhost:6379/1",  # results often Redis
)
```

```bash
celery -A celery_app worker -l info
celery -A celery_app beat -l info      # periodic tasks
```

Queue routing:

```python
app.conf.task_routes = {
    "tasks.heavy.*": {"queue": "heavy"},
    "tasks.email.*": {"queue": "email"},
}
```

---

## pika — Raw Producer/Consumer

```python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters("localhost"))
channel = connection.channel()
channel.queue_declare(queue="tasks", durable=True)

channel.basic_publish(
    exchange="",
    routing_key="tasks",
    body="process:123",
    properties=pika.BasicProperties(delivery_mode=2),  # persistent
)
connection.close()
```

```python
def callback(ch, method, properties, body):
    print(body.decode())
    ch.basic_ack(delivery_tag=method.delivery_tag)

channel.basic_qos(prefetch_count=1)
channel.basic_consume(queue="tasks", on_message_callback=callback)
channel.start_consuming()
```

Prefer Celery over raw pika for Python apps unless you need low-level control.

---

## Management UI

Port **15672** — `http://localhost:15672` (default guest/guest).

Inspect queues, rates, unacked messages, bindings.

---

## Docker Compose

```yaml
services:
  rabbitmq:
    image: rabbitmq:3-management
    ports:
      - "5672:5672"
      - "15672:15672"
    environment:
      RABBITMQ_DEFAULT_USER: app
      RABBITMQ_DEFAULT_PASS: secret
```

Celery URL: `amqp://app:secret@rabbitmq:5672//` (service name in Compose).

---

## Reliability Patterns

| Pattern | How |
| --- | --- |
| Durable queue | `durable=True` |
| Persistent messages | `delivery_mode=2` |
| Consumer ACK | `basic_ack` after success |
| Dead letter | DLX exchange + DLQ queue |
| Prefetch | `basic_qos(prefetch_count=N)` |

Celery: `task_acks_late=True`, `task_reject_on_worker_lost=True` for safer processing.

---

## RabbitMQ vs Redis as Celery Broker

| | RabbitMQ | Redis |
| --- | --- | --- |
| Routing | Exchanges, bindings | Lists |
| Durability | Strong AMQP story | AOF/RDB dependent |
| Ops | More moving parts | Simpler |
| Throughput | High | Very high for simple queues |

---

## Quick Reference

| Task | Command / config |
| --- | --- |
| Celery broker URL | `amqp://user:pass@host:5672//` |
| List queues | Management UI or `rabbitmqctl list_queues` |
| Purge queue | Management UI (dev only) |
| Worker | `celery -A app worker -l info -Q email` |
| Health | `rabbitmq-diagnostics ping` |

---

## Related Notes

- [[DB]]
- [[DB — Redis]]
- [[DB — Kafka]]
- [[Processing — Celery]]
- [[Processing]]
- [[Commands/CLI — Docker & Compose]]

---

## Tags

#database #rabbitmq #amqp #celery #message-broker #python #queue
