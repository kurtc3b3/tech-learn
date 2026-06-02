## What & When

**InfluxDB** is a **time-series database (TSDB)** optimized for **timestamped metrics** — high write throughput, retention policies, downsampling. Use for **IoT**, **application metrics** (when not using Prometheus pull model), **custom dashboards**, and **operational time series** — distinct from relational OLTP ([[ORM - SQLAlchemy]]).

Use InfluxDB when:

- **Many writes per second** with tags + fields model
- **Retention tiers** — raw 7d, aggregated 90d
- **Flux** or **InfluxQL** queries over time windows
- Storing sensor / billing / usage series

```bash
pip install influxdb-client
# InfluxDB 2.x local: docker run -d -p 8086:8086 influxdb:2
```

Overview: [[DB]]. Service metrics often use [[DB — Prometheus & Grafana]] instead — see comparison below.

---

## InfluxDB vs Related Tools

| Need | Use | Notes |
| --- | --- | --- |
| App/service metrics (pull) | [[DB — Prometheus & Grafana]] | De facto K8s standard |
| Custom high-write TSDB | **InfluxDB** | Push API |
| Logs | [[DB — ELK]] | Full text |
| Relational reports | PostgreSQL | Not TSDB |
| Cache | [[DB — Redis]] | Ephemeral |

---

## Data Model (InfluxDB 2.x)

```text
measurement,tag1=v1,tag2=v2 field1=1.0,field2=42 1234567890000000000
```

| Piece | Role |
| --- | --- |
| **Measurement** | Like table name |
| **Tags** | Indexed metadata (host, region) — low cardinality |
| **Fields** | Values (cpu, temp) — not indexed |
| **Timestamp** | Nanosecond precision |

Keep tag cardinality low — high cardinality kills performance.

---

## Python Client (2.x)

```python
from influxdb_client import InfluxDBClient, Point, WritePrecision
from influxdb_client.client.write_api import SYNCHRONOUS

client = InfluxDBClient(
    url="http://localhost:8086",
    token="my-token",
    org="myorg",
)
write_api = client.write_api(write_options=SYNCHRONOUS)

point = (
    Point("cpu_usage")
    .tag("host", "api-1")
    .tag("region", "eu")
    .field("percent", 72.5)
)
write_api.write(bucket="metrics", org="myorg", record=point)
```

Query with Flux:

```python
query = '''
from(bucket: "metrics")
  |> range(start: -1h)
  |> filter(fn: (r) => r._measurement == "cpu_usage")
'''
tables = client.query_api().query(query, org="myorg")
```

---

## FastAPI Instrumentation (Push)

```python
async def record_request_duration(path: str, ms: float, write_api):
    point = Point("http_request").tag("path", path).field("duration_ms", ms)
    write_api.write(bucket="metrics", org="myorg", record=point)
```

Prefer **Prometheus middleware** for standard HTTP metrics unless you already standardize on Influx.

---

## Retention & Downsampling

```text
Bucket "metrics_raw"   — 7d retention
Bucket "metrics_hourly" — task downsamples from raw
```

Configure in Influx UI or API — policy per bucket.

---

## Docker Compose

```yaml
services:
  influxdb:
    image: influxdb:2
    ports:
      - "8086:8086"
    volumes:
      - influxdata:/var/lib/influxdb2
volumes:
  influxdata:
```

Setup: open UI, create org, bucket, API token.

---

## Influx vs Prometheus

| | InfluxDB | Prometheus |
| --- | --- | --- |
| Model | Push (primary) | Pull scrape |
| Query | Flux / InfluxQL | PromQL |
| Ecosystem | Influx dashboards | Grafana native |
| K8s default | Less common | Very common |

Many teams: **Prometheus for services**, **Influx for IoT/custom push**.

---

## Quick Reference

| Task | API / UI |
| --- | --- |
| Write point | `Point(...).tag().field()` |
| Query | Flux `from(bucket:)` |
| Org/bucket/token | Influx UI setup |
| CLI | `influx write`, `influx query` |

---

## Related Notes

- [[DB]]
- [[DB — Prometheus & Grafana]]
- [[DB — ELK]]
- [[API - FastAPI]]
- [[K8S]]

---

## Tags

#database #influxdb #timeseries #metrics #flux #python #iot #monitoring
