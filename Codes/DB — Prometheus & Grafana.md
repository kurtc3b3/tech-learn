## What & When

**Prometheus** collects **time-series metrics** via HTTP **pull** scraping — counters, gauges, histograms. **Grafana** visualizes metrics and logs with dashboards and alerts. Together they are the default **observability** pair for [[K8S]] and [[API - FastAPI]] services.

Use when:

- **SLO dashboards** — latency, error rate, throughput
- **Alerting** — Alertmanager → PagerDuty / Slack
- **Kubernetes metrics** — node, pod, ingress (kube-prometheus-stack)
- Complementing [[DB — ELK]] (logs) with numeric signals

```bash
pip install prometheus-client
# Grafana: docker run -d -p 3000:3000 grafana/grafana
# Prometheus: docker run -d -p 9090:9090 prom/prometheus
```

Overview: [[DB]].

---

## Prometheus vs Related Tools

| Need | Use | Notes |
| --- | --- | --- |
| Service metrics (pull) | **Prometheus** | `/metrics` endpoint |
| Dashboards | **Grafana** | PromQL datasource |
| Logs | [[DB — ELK]] | Full text |
| Push metrics (IoT) | [[DB — InfluxDB]] | Different model |
| ML experiment metrics | [[ML — MLflow]] | Training runs |

---

## Architecture

```text
FastAPI / workers → expose /metrics
Prometheus → scrape every 15s → TSDB
Grafana → query Prometheus → dashboards
Alertmanager → routes firing alerts
```

On K8s: ServiceMonitor CRDs (Prometheus Operator).

---

## Python Instrumentation

```python
from prometheus_client import Counter, Histogram, generate_latest, CONTENT_TYPE_LATEST
from fastapi import FastAPI, Response

REQUEST_COUNT = Counter("http_requests_total", "Total HTTP requests", ["method", "path", "status"])
REQUEST_LATENCY = Histogram("http_request_duration_seconds", "Latency", ["method", "path"])

app = FastAPI()

@app.middleware("http")
async def metrics_middleware(request, call_next):
    with REQUEST_LATENCY.labels(request.method, request.url.path).time():
        response = await call_next(request)
    REQUEST_COUNT.labels(request.method, request.url.path, response.status_code).inc()
    return response

@app.get("/metrics")
def metrics():
    return Response(generate_latest(), media_type=CONTENT_TYPE_LATEST)
```

Exclude `/metrics` from high-cardinality path labels in production — use route templates.

---

## PromQL Examples

```promql
# Request rate
rate(http_requests_total[5m])

# Error ratio
sum(rate(http_requests_total{status=~"5.."}[5m]))
/
sum(rate(http_requests_total[5m]))

# p95 latency
histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))
```

---

## prometheus.yml (Scrape Config)

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: fastapi
    static_configs:
      - targets: ["host.docker.internal:8000"]

  - job_name: prometheus
    static_configs:
      - targets: ["localhost:9090"]
```

---

## Grafana Basics

1. Add **Prometheus** datasource → `http://prometheus:9090`
2. Import dashboard — e.g. ID **3662** (FastAPI) or **6417** (Kubernetes)
3. Create panels with PromQL
4. Configure alerts → contact points

Variables: `$namespace`, `$pod` for K8s dashboards.

---

## Docker Compose Sketch

```yaml
services:
  prometheus:
    image: prom/prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana
    ports:
      - "3000:3000"
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
    depends_on:
      - prometheus
```

FastAPI on host: use `host.docker.internal` as scrape target.

---

## Alertmanager (Brief)

```yaml
# alert rules in Prometheus
groups:
  - name: api
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m]))
          / sum(rate(http_requests_total[5m])) > 0.05
        for: 5m
        labels:
          severity: page
        annotations:
          summary: "API error rate above 5%"
```

Route `severity: page` to on-call; `warning` to Slack.

---

## Cardinality Warning

| Bad | Good |
| --- | --- |
| Label `user_id` on every request | Aggregate counters only |
| Label raw URL path | Template `/users/{id}` |
| Unbounded label values | Bounded enum labels |

High cardinality crashes Prometheus.

---

## ML / Celery Metrics

| Service | Metric examples |
| --- | --- |
| [[ML — Seldon]] | `seldon_prediction_latency` |
| [[Processing — Celery]] | Flower / exporter queue depth |
| [[ML — MLflow]] | Track server uptime separately |

---

## Quick Reference

| Task | Tool |
| --- | --- |
| Expose metrics | `prometheus_client` + `/metrics` |
| Query | PromQL in Grafana or Prometheus UI |
| Scrape config | `prometheus.yml` |
| K8s install | kube-prometheus-stack Helm |
| Default Grafana | `:3000` admin/admin (change password) |
| Prometheus UI | `:9090` |

---

## Related Notes

- [[DB]]
- [[DB — ELK]]
- [[DB — InfluxDB]]
- [[API - FastAPI]]
- [[K8S]]
- [[ML — Seldon]]
- [[Processing — Celery]]

---

## Tags

#database #prometheus #grafana #metrics #observability #promql #monitoring #devops #sre
