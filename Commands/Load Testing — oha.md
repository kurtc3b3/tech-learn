## What & When

**oha** is a Rust HTTP load generator with a **real-time TUI** (inspired by hey). It supports HTTP/1.1, HTTP/2, JSON/CSV output, and **latency correction** for coordinated omission. Use it for **quick benches** and **CI-friendly JSON** summaries.

Overview: [[Load Testing]]. Compare [[Commands/Load Testing — hey]] (simpler, unmaintained upstream).

---

## Install

```bash
# macOS
brew install oha

# Verify
oha --version
```

Also available via `cargo install oha` or release binaries from [hatoo/oha](https://github.com/hatoo/oha).

---

## Basic Run (Interactive TUI)

```bash
# 200 requests, 50 concurrent workers
oha -n 200 -c 50 http://localhost:8000/health

# POST with JSON body
oha -n 1000 -c 100 -m POST \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com"}' \
  http://localhost:8000/api/v1/login
```

Against Compose stack — [[Commands/CLI — Docker & Compose]]:

```bash
docker compose up -d
oha -n 500 -c 25 http://localhost:8000/health
```

---

## Headless Mode (CI / Scripts)

Disable the TUI for stable, parseable output in pipelines:

```bash
oha --no-tui -n 1000 -c 50 http://localhost:8000/health
```

**JSON summary** (for `jq`, dashboards, gates):

```bash
oha --no-tui --output-format json -n 2000 -c 100 \
  http://localhost:8000/api/items \
  | jq '.summary'
```

Write to file:

```bash
oha --no-tui --output-format json -o results.json \
  -n 5000 -c 200 https://staging.example.com/health
```

**CSV** (per-request lines):

```bash
oha --no-tui --output-format csv -n 500 -c 10 -o requests.csv \
  http://localhost:8000/health
```

---

## Rate and Duration

| Flag | Meaning |
| --- | --- |
| `-n` | Total number of requests |
| `-c` | Number of concurrent connections |
| `-q` | Target queries per second (rate limit) |
| `-z` | Duration (e.g. `30s`, `2m`) — use with `-q` or `-c` |
| `-p` | HTTP/2 only (percent of connections) |

```bash
# Run for 60 seconds at ~200 RPS
oha --no-tui -z 60s -q 200 http://localhost:8000/health

# Burst pattern (see oha --help for burst-delay)
oha --no-tui -n 10000 -c 500 http://localhost:8000/health
```

---

## TLS and Auth

```bash
# Skip TLS verify (dev only)
oha --no-tui --insecure -n 500 https://localhost:8443/health

# Bearer token
oha --no-tui -n 200 \
  -H "Authorization: Bearer $TOKEN" \
  http://localhost:8000/api/me
```

---

## Debug Single Request

```bash
oha --debug http://localhost:8000/health
```

---

## CI Example (GitHub Actions)

```yaml
name: Load smoke
on: [workflow_dispatch]

jobs:
  oha:
    runs-on: ubuntu-latest
    steps:
      - run: brew install oha
      - name: Bench staging health
        run: |
          oha --no-tui --output-format json -n 500 -c 25 \
            https://staging.example.com/health \
            | tee results.json
          # Example gate: parse with jq (adjust field names to your oha version)
          p99=$(jq -r '.latency_percentiles.p99 // .summary.latency.p99' results.json)
          echo "p99=${p99}"
```

Pair with deploy workflow — [[Commands/CLI — Git & GitHub]].

---

## oha vs hey

| | oha | hey |
| --- | --- | --- |
| Maintenance | Active | Limited |
| TUI | ✅ | No |
| JSON output | ✅ | Basic |
| Latency correction | ✅ default | No |

Prefer **oha** for new work unless you need minimal hey compatibility.

---

## Troubleshooting

| Problem | Fix |
| --- | --- |
| `too many open files` | Raise `ulimit -n`; lower `-c` |
| 429 responses | [[API - FastAPI — Rate Limiting (SlowAPI)]] — lower rate or exempt test IP |
| Connection refused | API not listening; check Compose port |
| TUI vs `--no-tui` mismatch | Use same `-n`, `-c`, `-z`; prefer `--no-tui` in CI |

---

## Quick Reference

```bash
brew install oha
oha -n 200 -c 50 http://localhost:8000/health
oha --no-tui --output-format json -n 1000 -c 50 URL | jq .
oha --no-tui -z 30s -q 100 URL
oha --debug URL
```

---

## Related Notes

- [[Load Testing]]
- [[Commands/Load Testing — hey]]
- [[Load Testing — k6]]
- [[API - FastAPI]]
- [[Commands/CLI — Docker & Compose]]

---

## Tags

#load-testing #oha #cli #http #benchmark #ci #headless #brew
