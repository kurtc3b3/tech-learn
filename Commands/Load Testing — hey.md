## What & When

**hey** is a tiny Go CLI that sends load to an HTTP endpoint and prints latency statistics. It is **minimal and fast** — good for ad-hoc checks when you do not need oha’s TUI or JSON schema.

Overview: [[Load Testing]]. For CI JSON and latency correction, prefer [[Commands/Load Testing — oha]].

---

## Install

```bash
# macOS
brew install hey

hey -version
```

Alternative: `go install github.com/rakyll/hey@latest`

---

## Basic Usage

```bash
# 200 requests, 50 workers (concurrent)
hey -n 200 -c 50 http://localhost:8000/health

# Duration-based (run for 30 seconds)
hey -z 30s -c 50 http://localhost:8000/health

# QPS cap (approximate)
hey -q 100 -z 1m http://localhost:8000/health
```

---

## HTTP Method and Body

```bash
# POST JSON
hey -n 500 -c 20 -m POST \
  -H "Content-Type: application/json" \
  -d '{"name":"test"}' \
  http://localhost:8000/api/users

# Custom headers
hey -n 100 -H "Authorization: Bearer $TOKEN" \
  http://localhost:8000/api/me
```

---

## Headless / CI

hey is **always non-interactive** — output is text to stdout. Suitable for quick pipeline steps without a GUI.

```bash
hey -n 1000 -c 50 -z 30s http://staging.example.com/health \
  | tee hey-results.txt
```

For **machine-readable JSON**, use [[Commands/Load Testing — oha]] or [[Load Testing — k6]] with thresholds.

---

## TLS (Dev)

```bash
hey -n 200 -c 10 -disable-keepalive \
  http://localhost:8000/health
```

For HTTPS with self-signed certs, use a reverse proxy or test HTTP locally — hey has limited TLS tuning compared to oha.

---

## Common Flags

| Flag | Purpose |
| --- | --- |
| `-n` | Total requests |
| `-c` | Concurrent workers |
| `-z` | Max duration (`30s`, `2m`) |
| `-q` | Rate limit (requests per second) |
| `-m` | HTTP method |
| `-H` | Header (repeatable) |
| `-d` | Request body |
| `-t` | Timeout per request |

---

## hey vs oha vs k6

| Need | Tool |
| --- | --- |
| Fastest one-liner | hey |
| JSON + p99 in CI | oha |
| Thresholds as code | k6 |

---

## Troubleshooting

| Problem | Fix |
| --- | --- |
| All errors / 0 RPS | Wrong URL, firewall, or service down |
| High latency only under load | DB pool, [[API - FastAPI — Rate Limiting (SlowAPI)]] |
| `-n` ignored with `-z` | Duration mode stops by time — check hey docs |

---

## Quick Reference

```bash
brew install hey
hey -n 200 -c 50 http://localhost:8000/health
hey -z 60s -c 100 -q 50 http://localhost:8000/health
hey -m POST -d '{}' -H "Content-Type: application/json" URL
```

---

## Related Notes

- [[Load Testing]]
- [[Commands/Load Testing — oha]]
- [[Load Testing — k6]]
- [[API - FastAPI]]
- [[Commands/CLI — Docker & Compose]]

---

## Tags

#load-testing #hey #cli #http #benchmark #brew
