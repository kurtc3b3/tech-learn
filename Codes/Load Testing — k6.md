## What & When

**k6** (Grafana k6) is a **JavaScript** load testing tool — scripts define VUs (virtual users), stages, thresholds, and checks. It runs **headless** in CI and locally; ideal when load tests are **version-controlled** next to [[API - FastAPI]] code.

```bash
brew install k6
k6 version
```

Overview: [[Load Testing]]. CLI benches: [[Commands/Load Testing — oha]]. Python alternative: [[Load Testing — Locust]].

---

## k6 vs Related Tools

| Need | Use |
| --- | --- |
| JS test in CI with pass/fail thresholds | **k6** |
| Quick terminal bench | [[Commands/Load Testing — oha]] |
| Python user flows | [[Load Testing — Locust]] |
| GUI test plans | [[Commands/Load Testing — JMeter]] |

---

## Minimal Script

`load/k6/script.js`:

```javascript
import http from "k6/http";
import { check, sleep } from "k6";

export const options = {
  vus: 10,
  duration: "30s",
  thresholds: {
    http_req_failed: ["rate<0.01"],
    http_req_duration: ["p(95)<500"],
  },
};

const BASE_URL = __ENV.BASE_URL || "http://localhost:8000";

export default function () {
  const res = http.get(`${BASE_URL}/health`);
  check(res, {
    "status is 200": (r) => r.status === 200,
  });
  sleep(1);
}
```

**Headless run:**

```bash
k6 run load/k6/script.js
k6 run -e BASE_URL=http://localhost:8000 load/k6/script.js
```

---

## Staged Ramp (Realistic Load)

```javascript
export const options = {
  stages: [
    { duration: "1m", target: 20 },   // ramp to 20 VUs
    { duration: "3m", target: 20 },   // hold
    { duration: "1m", target: 50 },   // spike
    { duration: "2m", target: 50 },
    { duration: "1m", target: 0 },    // ramp down
  ],
  thresholds: {
    http_req_duration: ["p(99)<800"],
    http_req_failed: ["rate<0.05"],
  },
};
```

---

## POST + JSON (FastAPI)

```javascript
import http from "k6/http";
import { check } from "k6";

const BASE_URL = __ENV.BASE_URL || "http://localhost:8000";

export default function () {
  const payload = JSON.stringify({ email: "load@test.com", password: "secret" });
  const params = {
    headers: { "Content-Type": "application/json" },
  };
  const res = http.post(`${BASE_URL}/api/v1/login`, payload, params);
  check(res, { "login ok": (r) => r.status === 200 || r.status === 401 });
}
```

Pair with [[API - FastAPI — Pydantic Models]] request shapes.

---

## Auth Header from Environment

```javascript
const TOKEN = __ENV.API_TOKEN;

export default function () {
  const res = http.get(`${BASE_URL}/api/me`, {
    headers: { Authorization: `Bearer ${TOKEN}` },
  });
  check(res, { "authorized": (r) => r.status === 200 });
}
```

```bash
API_TOKEN=eyJ... k6 run load/k6/authenticated.js
```

---

## Thresholds Fail the Run (CI Gate)

```javascript
export const options = {
  vus: 50,
  duration: "2m",
  thresholds: {
    http_req_duration: ["p(95)<300", "p(99)<600"],
    http_req_failed: ["rate<0.01"],
    checks: ["rate>0.99"],
  },
};
```

Exit code non-zero when thresholds fail — use in GitHub Actions `if: success()`.

---

## JSON Output (Artifacts)

```bash
k6 run --out json=load/k6/results.json load/k6/script.js
k6 run --summary-export=summary.json load/k6/script.js
```

---

## CI Example (GitHub Actions)

```yaml
name: k6 load
on:
  workflow_dispatch:
  pull_request:
    paths: ["load/k6/**", "app/**"]

jobs:
  k6:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: brew install k6
      - name: Start API
        run: |
          docker compose up -d
          sleep 5
      - name: Run load test
        run: k6 run -e BASE_URL=http://localhost:8000 load/k6/script.js
```

---

## k6 Cloud (Optional)

Grafana Cloud k6 runs distributed load from the cloud — useful when a single laptop cannot generate enough RPS. Local `k6 run` remains the default for dev.

---

## Common Mistakes

| Mistake | Fix |
| --- | --- |
| No `sleep()` in loop | Unrealistic hammering; add think time |
| Testing prod URL | Use staging; respect rate limits |
| Ignoring 429 | Tune VUs or disable limits in test env only |
| Hard-coded secrets | `__ENV` / CI secrets |
| Thresholds too tight on shared CI | Calibrate on staging baseline |

---

## Quick Reference

```bash
brew install k6
k6 run script.js
k6 run -e BASE_URL=http://localhost:8000 script.js
k6 run --vus 50 --duration 2m script.js
k6 run --out json=results.json script.js
```

---

## Related Notes

- [[Load Testing]]
- [[Commands/Load Testing — oha]]
- [[Load Testing — Locust]]
- [[API - FastAPI]]
- [[Commands/CLI — Docker & Compose]]
- [[DB — Prometheus & Grafana]]

---

## Tags

#load-testing #k6 #javascript #ci #thresholds #headless #brew
