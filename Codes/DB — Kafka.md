## What & When

**Apache Kafka** is a **distributed event streaming platform** — producers append records to **topics** partitioned for scale; consumers read at their own pace with offset tracking. Use for **domain events**, **audit logs**, **CDC pipelines**, and **many-subscriber** architectures — not as a simple Celery replacement ([[Processing — Celery]] + [[DB — RabbitMQ]]).

Use Kafka when:

- Multiple services must **read the same event stream**
- You need **replay** and **retention** (days/weeks)
- **High throughput** ordered streams per key (partition)
- Decoupling producers/consumers with schema evolution (Avro/JSON + Schema Registry)

```bash
pip install confluent-kafka
# or: pip install aiokafka
# Local: docker compose with bitnami/kafka or redpanda
```

Overview: [[DB]].

---

## Kafka vs Related Tools

| Need | Use | Notes |
| --- | --- | --- |
| Durable event log | **Kafka** | Partitions, retention |
| Python task queue | [[DB — RabbitMQ]] / Redis | [[Processing — Celery]] |
| Simple pub/sub | [[DB — Redis]] | No long retention |
| Request/reply RPC | RabbitMQ / HTTP | Not Kafka primary pattern |
| OLTP | [[ORM - SQLAlchemy]] | Different layer |

---

## Core Concepts

| Term | Meaning |
| --- | --- |
| **Topic** | Named stream of records |
| **Partition** | Ordered shard; scale unit |
| **Offset** | Position in partition |
| **Producer** | Writes records |
| **Consumer group** | Coordinated readers; one consumer per partition |
| **Broker** | Kafka server node |
| **ZooKeeper / KRaft** | Cluster metadata (modern: KRaft mode) |

---

## Producer (confluent-kafka)

```python
from confluent_kafka import Producer

def acked(err, msg):
    if err:
        print(f"Failed: {err}")

p = Producer({"bootstrap.servers": "localhost:9092"})
p.produce("orders.created", key="order-99", value='{"id": 99, "total": 42.0}')
p.flush()
```

Key = same key → same partition → ordering per entity.

---

## Consumer

```python
from confluent_kafka import Consumer

c = Consumer({
    "bootstrap.servers": "localhost:9092",
    "group.id": "billing-service",
    "auto.offset.reset": "earliest",
})
c.subscribe(["orders.created"])

while True:
    msg = c.poll(1.0)
    if msg is None:
        continue
    if msg.error():
        print(msg.error())
        continue
    print(msg.key(), msg.value().decode())
    c.commit(msg)  # at-least-once if commit after processing
```

---

## Async (aiokafka)

```python
from aiokafka import AIOKafkaProducer
import asyncio
import json

async def send_event():
    producer = AIOKafkaProducer(bootstrap_servers="localhost:9092")
    await producer.start()
    await producer.send_and_wait(
        "orders.created",
        json.dumps({"id": 1}).encode(),
    )
    await producer.stop()

asyncio.run(send_event())
```

Pair with [[Python — asyncio]] workers or FastAPI lifespan consumers.

---

## Topic Design

```text
orders.created      # domain event
orders.created.dlq  # dead letter
user.signups        # separate topic per event type (common pattern)
```

| Practice | Why |
| --- | --- |
| Key by entity id | Order events stay ordered per order |
| Small messages | Large blobs → S3 + reference in event |
| Idempotent consumers | At-least-once delivery |
| Schema registry | Safe evolution |

---

## Docker Compose (Redpanda — Kafka-compatible)

```yaml
services:
  redpanda:
    image: redpanda/redpanda:latest
    command:
      - redpanda start
      - --overprovisioned
      - --smp 1
      - --memory 1G
      - --kafka-addr internal://0.0.0.0:9092,external://0.0.0.0:19092
    ports:
      - "19092:19092"
```

Bootstrap: `localhost:19092`.

---

## FastAPI Integration Pattern

```text
API handler → validate → write PostgreSQL ([[ORM - CRUD]])
                      → produce Kafka event (async)
Downstream consumers → update read models / notify / analytics
```

Transactional outbox pattern: write event to outbox table in same DB transaction; separate relay publishes to Kafka.

---

## Operations

| Concern | Approach |
| --- | --- |
| Retention | `retention.ms` per topic |
| Rebalance | Consumer group join triggers partition assign |
| Monitoring | [[DB — Prometheus & Grafana]] + Kafka exporter |
| Local dev | Redpanda / single-node Kafka |

---

## Quick Reference

| Task | Tool |
| --- | --- |
| List topics | `kafka-topics --list --bootstrap-server localhost:9092` |
| Create topic | `kafka-topics --create --topic X --partitions 3` |
| Console consumer | `kafka-console-consumer --topic X --from-beginning` |
| Python produce | `confluent_kafka.Producer` |
| Consumer group lag | Burrow / Kafka exporter metrics |

---

## Related Notes

- [[DB]]
- [[DB — RabbitMQ]]
- [[DB — Redis]]
- [[Processing — Celery]]
- [[ORM - SQLAlchemy]]
- [[Python — asyncio]]

---

## Tags

#database #kafka #streaming #events #python #microservices #message-bus
