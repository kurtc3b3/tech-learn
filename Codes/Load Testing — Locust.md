## What & When

**Locust** is a **Python** load testing framework — you define user behavior in code (`HttpUser` tasks) and scale **virtual users** from a **web UI** or **headless CLI**. Fits teams already on [[API - FastAPI]] / [[Python Development]].

```bash
pip install locust
# optional: web UI extras
pip install locust[web]
```

Overview: [[Load Testing]]. JavaScript CI alternative: [[Load Testing — k6]].

---

## Locust vs Related Tools

| Need | Use |
| --- | --- |
| Python scenarios, custom logic | **Locust** |
| JS thresholds in CI | [[Load Testing — k6]] |
| One-liner bench | [[Commands/Load Testing — oha]] |
| GUI for non-devs | [[Commands/Load Testing — JMeter]] |

---

## Minimal `locustfile.py`

```python
from locust import HttpUser, task, between

class ApiUser(HttpUser):
    wait_time = between(1, 3)

    @task
    def health(self):
        self.client.get("/health")

    @task(3)
    def list_items(self):
        self.client.get("/api/items", name="/api/items")
```

Run with web UI (local dev):

```bash
cd load/locust
locust -f locustfile.py --host http://localhost:8000
# Open http://localhost:8089 — set users & spawn rate in UI
```

---

## Headless Mode (CI / Servers)

No browser UI — fixed users, spawn rate, duration:

```bash
locust -f locustfile.py \
  --headless \
  --host http://localhost:8000 \
  -u 50 \
  -r 5 \
  --run-time 5m \
  --html report.html \
  --csv results
```

| Flag | Meaning |
| --- | --- |
| `--headless` | No web UI |
| `-u` | Peak concurrent users |
| `-r` | Spawn rate (users per second) |
| `--run-time` | Stop after duration (`5m`, `1h`) |
| `--html` | HTML report path |
| `--csv` | CSV metrics prefix |

Exit code non-zero on failure with `--exit-code-on-error` (check Locust version docs).

---

## POST + JSON

```python
from locust import HttpUser, task

class ApiUser(HttpUser):
    def on_start(self):
        r = self.client.post(
            "/api/v1/login",
            json={"email": "load@test.com", "password": "secret"},
        )
        if r.ok:
            token = r.json().get("access_token")
            self.client.headers["Authorization"] = f"Bearer {token}"

    @task
    def me(self):
        self.client.get("/api/me")
```

---

## Task Weights and Tags

```python
from locust import HttpUser, task

class ApiUser(HttpUser):
    @task(5)
    def browse(self):
        self.client.get("/api/items")

    @task(1)
    def create(self):
        self.client.post("/api/items", json={"name": "load-item"})
```

Run subset: `locust -f locustfile.py --tags browse`

---

## Custom Load Shape (Stages)

```python
from locust import LoadTestShape

class StagesShape(LoadTestShape):
    stages = [
        {"duration": 60, "users": 10, "spawn_rate": 2},
        {"duration": 120, "users": 50, "spawn_rate": 5},
        {"duration": 60, "users": 0, "spawn_rate": 10},
    ]

    def tick(self):
        run_time = self.get_run_time()
        for stage in self.stages:
            if run_time < stage["duration"]:
                return stage["users"], stage["spawn_rate"]
            run_time -= stage["duration"]
        return None
```

---

## Distributed Locust (Concept)

```bash
# Master
locust -f locustfile.py --master --expect-workers 4

# Each worker machine
locust -f locustfile.py --worker --master-host=<master-ip>
```

Use when one process cannot generate enough load; coordinate network and target allowlists.

---

## Docker Compose Sidecar

```yaml
# compose.load.yml (snippet)
services:
  locust:
    image: locustio/locust
    volumes:
      - ./load/locust:/mnt/locust
    command: -f /mnt/locust/locustfile.py --host http://api:8000
    ports:
      - "8089:8089"
    depends_on:
      - api
```

See [[Commands/CLI — Docker & Compose]].

---

## CI Example (GitHub Actions)

```yaml
name: Locust headless
on: workflow_dispatch

jobs:
  locust:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install locust
      - run: docker compose up -d
      - name: Headless load
        run: |
          locust -f load/locust/locustfile.py --headless \
            --host http://localhost:8000 \
            -u 30 -r 3 --run-time 2m \
            --html locust-report.html
      - uses: actions/upload-artifact@v4
        with:
          name: locust-report
          path: locust-report.html
```

---

## Locust vs k6

| | Locust | k6 |
| --- | --- | --- |
| Language | Python | JavaScript |
| Web UI | Built-in | No (Grafana Cloud optional) |
| Custom code in tasks | Easy | Possible in `setup()` |
| CI thresholds | Custom / stats | Native `thresholds` |

---

## Common Mistakes

| Mistake | Fix |
| --- | --- |
| Forgot `--host` | Always pass API base URL |
| No `wait_time` | Unrealistic zero-delay users |
| Locust worker = API server | Separate machines or containers |
| 429 storm | Lower `-u` / `-r`; check [[API - FastAPI — Rate Limiting (SlowAPI)]] |

---

## Quick Reference

```bash
pip install locust
locust -f locustfile.py --host http://localhost:8000
locust -f locustfile.py --headless --host URL -u 50 -r 5 --run-time 5m
locust -f locustfile.py --master / --worker
```

---

## Related Notes

- [[Load Testing]]
- [[Load Testing — k6]]
- [[Commands/Load Testing — oha]]
- [[API - FastAPI]]
- [[Python Development]]
- [[Commands/CLI — Docker & Compose]]

---

## Tags

#load-testing #locust #python #headless #http #ci #fastapi
