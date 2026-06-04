## What & When

**Apache JMeter** is a Java-based load testing tool with a **GUI test planner** and **headless CLI** (`jmeter -n`). It supports HTTP, JDBC, JMS, FTP, and rich reporting — common in enterprise QA teams.

Overview: [[Load Testing]]. For code-first HTTP load, see [[Load Testing — k6]] or [[Load Testing — Locust]].

---

## Install

```bash
# macOS
brew install jmeter

# Verify
jmeter --version
```

Requires **Java** (JMeter bundles or uses system JRE depending on install). GUI:

```bash
jmeter
# or on macOS with brew:
open /opt/homebrew/opt/jmeter/libexec/bin/jmeter
```

---

## Project Layout (Typical)

```text
load/jmeter/
├── test-plan.jmx          # GUI-authored test plan
├── users.csv              # CSV Data Set Config
├── results/               # .jtl + HTML report (gitignore)
└── run.sh
```

---

## GUI Mode (Design)

1. Open JMeter → **Test Plan**
2. Add **Thread Group** (users, ramp-up, loops)
3. Add **HTTP Request Defaults** (host, port, protocol)
4. Add **HTTP Request** samplers per endpoint
5. Add listeners (View Results Tree — dev only; **Summary Report** / **Aggregate Report** for load)
6. Save as `test-plan.jmx`

Thread Group fields:

| Field | Meaning |
| --- | --- |
| Number of Threads | Virtual users |
| Ramp-Up Period | Seconds to start all threads |
| Loop Count | Iterations per user (`∞` with scheduler) |

For [[API - FastAPI]] JSON APIs: set **Content-Type: application/json** and **Body Data** on POST samplers.

---

## Headless / Non-GUI Mode (CI)

**Always use `-n` in CI** — no GUI on servers.

```bash
# Run test, write JTL results
jmeter -n \
  -t load/jmeter/test-plan.jmx \
  -l load/jmeter/results/run.jtl \
  -j load/jmeter/results/jmeter.log

# Generate HTML dashboard from JTL
jmeter -g load/jmeter/results/run.jtl \
  -o load/jmeter/results/html-report
```

Single command (run + report):

```bash
jmeter -n -t load/jmeter/test-plan.jmx -l results.jtl -e -o html-report/
```

| Flag | Meaning |
| --- | --- |
| `-n` | Non-GUI (headless) |
| `-t` | Test plan `.jmx` |
| `-l` | Results file `.jtl` (CSV/XML) |
| `-j` | JMeter log file |
| `-e` | Generate report dashboard after test |
| `-o` | Output folder for HTML report (must be empty or missing) |

---

## Properties and Overrides

Pass properties without editing the JMX:

```bash
jmeter -n -t test-plan.jmx -l results.jtl \
  -Jthreads=100 \
  -Jrampup=60 \
  -Jhost=staging.example.com \
  -Jprotocol=https
```

In the plan, reference `${__P(threads,10)}` style properties.

---

## CSV Data Set (Many Users)

```text
# users.csv
user,password
alice,secret1
bob,secret2
```

JMeter: **CSV Data Set Config** → Filename `users.csv`, Variable names `user,password`, Sharing mode **All threads**.

---

## Assertions

Add **Response Assertion** (status 200) and **JSON Assertion** (JMESPath / JSON Path) on critical samplers so failed responses mark the sample as error.

---

## Distributed Load (Concept)

For load beyond one machine:

- **Remote engines** — start `jmeter-server` on workers, control from controller
- Firewall and RMI ports must be open (operational overhead)

Most teams on [[K8S]] prefer **horizontal k6/Locust workers** or cloud load services instead.

---

## CI Example (GitHub Actions)

```yaml
name: JMeter load
on: workflow_dispatch

jobs:
  jmeter:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: brew install jmeter
      - name: Headless test
        run: |
          jmeter -n -t load/jmeter/test-plan.jmx -l results.jtl -j jmeter.log \
            -Jhost=${{ vars.STAGING_HOST }} -Jthreads=50 -Jrampup=30
      - name: HTML report
        run: jmeter -g results.jtl -o report/
      - uses: actions/upload-artifact@v4
        with:
          name: jmeter-report
          path: report/
```

---

## JMeter vs k6 / Locust

| | JMeter | k6 | Locust |
| --- | --- | --- | --- |
| Authoring | GUI + XML | JavaScript | Python |
| CI | `jmeter -n` | `k6 run` | `--headless` |
| Learning curve | GUI-friendly | Dev-friendly | Python-friendly |
| Protocols | Many | HTTP primary | HTTP + custom |

---

## Troubleshooting

| Problem | Fix |
| --- | --- |
| Out of memory | Increase `HEAP` in `jmeter` script; reduce threads |
| `-o` folder not empty | Delete report dir before `-e -o` |
| SSL errors | Add **HTTP Request** keystore or test HTTP locally |
| 429 / 503 under load | App limits — [[API - FastAPI — Rate Limiting (SlowAPI)]]; scale [[K8S]] |

---

## Quick Reference

```bash
brew install jmeter
jmeter                                    # GUI
jmeter -n -t plan.jmx -l results.jtl      # headless
jmeter -g results.jtl -o report/          # HTML dashboard
jmeter -n -t plan.jmx -l results.jtl -e -o report/
```

---

## Related Notes

- [[Load Testing]]
- [[Load Testing — k6]]
- [[Load Testing — Locust]]
- [[Commands/Load Testing — oha]]
- [[API - FastAPI]]
- [[Commands/CLI — Newman & Postman]]

---

## Tags

#load-testing #jmeter #cli #gui #headless #ci #brew #java
