## What & When

**Postman** is a GUI for designing, saving, and sharing **HTTP API collections** — requests, environments, tests, and docs. **Newman** is Postman's **CLI runner** — execute collections headless in CI against [[API - FastAPI]] (or any REST) services.

Use when:

- **Manual exploration** of APIs during development ([[API - FastAPI]] `/docs` complements this)
- **Shared collections** for QA and frontend teams
- **CI smoke / contract tests** — `newman run` after deploy
- **Regression** on staging before merge

Code-first alternative: [[Unit Testing - pytest]] + [[Python — httpx Package]]. Overview: [[CLI]].

---

## Install

```bash
# Newman (CLI)
npm install -g newman
# or: npm install -g newman newman-reporter-htmlextra

newman --version
```

Postman desktop: https://www.postman.com/downloads/

---

## Postman — Core Concepts

| Concept | Purpose |
| --- | --- |
| **Collection** | Folder of related requests |
| **Request** | Method, URL, headers, body |
| **Environment** | Variables (`baseUrl`, `token`) per stage |
| **Tests** | JavaScript assertions on response (in Postman sandbox) |
| **Pre-request script** | Set variables, sign requests before send |

---

## Collection Design (FastAPI)

Typical layout for [[API - FastAPI]] service:

```text
My API/
├── Auth/
│   └── POST login
├── Users/
│   ├── GET list users
│   ├── GET user by id
│   └── POST create user
└── Health/
    └── GET health
```

Environment variables:

| Variable | Example |
| --- | --- |
| `baseUrl` | `http://localhost:8000` |
| `accessToken` | set by login test script |

Request URL: `{{baseUrl}}/users`

---

## Tests Tab (Postman JavaScript)

```javascript
// Status
pm.test("Status is 200", () => {
  pm.response.to.have.status(200);
});

// JSON body
pm.test("Has id", () => {
  const json = pm.response.json();
  pm.expect(json).to.have.property("id");
});

// Save token from login
const json = pm.response.json();
pm.environment.set("accessToken", json.access_token);
```

Auth header on subsequent requests: `Authorization: Bearer {{accessToken}}`

---

## Export Collection

Postman UI:

1. Collection → `...` → **Export**
2. Choose **Collection v2.1**
3. Save as `postman/collection.json`

Commit JSON to repo for Newman CI. Export environments separately (strip secrets or use CI secrets).

---

## Newman — Run Locally

```bash
newman run postman/collection.json \
  --environment postman/local.environment.json \
  --reporters cli
```

With env vars inline:

```bash
newman run postman/collection.json \
  --env-var baseUrl=http://localhost:8000 \
  --env-var accessToken=test-token
```

Against Docker Compose stack — [[Commands/CLI — Docker & Compose]]:

```bash
docker compose up -d
newman run postman/collection.json --env-var baseUrl=http://localhost:8000
```

---

## Newman — Options

```bash
newman run collection.json \
  --folder "Users" \
  --bail \
  --timeout-request 10000 \
  --delay-request 200 \
  --iteration-count 3 \
  --reporters cli,json \
  --reporter-json-export results.json
```

| Flag | Use |
| --- | --- |
| `--folder` | Run subset |
| `--bail` | Stop on first failure |
| `--env-var` | Override environment |
| `--global-var` | Collection-level vars |
| `--insecure` | Skip TLS verify (dev only) |

---

## CI Example (GitHub Actions)

```yaml
# .github/workflows/api-smoke.yml
name: API smoke
on: [push, pull_request]

jobs:
  newman:
    runs-on: ubuntu-latest
    services:
      api:
        image: ghcr.io/myorg/myapi:${{ github.sha }}
        ports:
          - 8000:8000
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "20"
      - run: npm install -g newman
      - run: |
          newman run postman/collection.json \
            --env-var baseUrl=http://localhost:8000 \
            --bail
```

Pair with `gh pr checks` — [[Commands/CLI — Git & GitHub]].

---

## Newman vs pytest + httpx

| | Newman | pytest + httpx |
| --- | --- | --- |
| Authoring | Postman GUI | Python code |
| CI | `newman run` | `pytest tests/api/` |
| Non-dev users | ✅ Collections | ❌ Code required |
| Fixtures / DB | Limited | ✅ Full [[Unit Testing - pytest]] |
| OpenAPI sync | Manual | Can generate from schema |

Use **both**: pytest for deep integration tests; Newman for portable smoke collections.

---

## OpenAPI / Swagger Link

FastAPI serves OpenAPI at `/openapi.json` — import into Postman:

- Postman → **Import** → Link → `http://localhost:8000/openapi.json`

Keeps collection aligned with [[API - FastAPI — OpenAPI Specification]]; still add tests for business rules.

---

## Environment Files

`local.environment.json` (example structure — no secrets in git):

```json
{
  "name": "local",
  "values": [
    { "key": "baseUrl", "value": "http://localhost:8000", "enabled": true }
  ]
}
```

Staging/prod URLs and tokens via CI `--env-var` or secret-backed env files.

---

## Reporting

```bash
npm install -g newman-reporter-htmlextra

newman run collection.json \
  --reporters cli,htmlextra \
  --reporter-htmlextra-export report.html
```

Upload `report.html` as CI artifact for failed PR debugging.

---

## Troubleshooting

| Problem | Fix |
| --- | --- |
| Connection refused | API not up; check Compose / port |
| 401 on protected routes | Run login request first; set `accessToken` |
| SSL errors | `--insecure` dev only; fix certs in prod |
| Flaky tests | Add `--delay-request`; avoid race with async startup |
| Wrong host | Verify `baseUrl` env |

---

## Quick Reference

```bash
newman run postman/collection.json --env-var baseUrl=http://localhost:8000
newman run collection.json --folder Health --bail
newman run collection.json -e postman/staging.environment.json
```

---

## Related Notes

- [[CLI]]
- [[API - FastAPI]]
- [[API - FastAPI — OpenAPI Specification]]
- [[API - FastAPI — REST Principles & HTTP Methods]]
- [[Commands/CLI — Docker & Compose]]
- [[Commands/CLI — Git & GitHub]]
- [[Unit Testing - pytest]]
- [[Python — httpx Package]]

---

## Tags

#cli #postman #newman #api-testing #ci #rest #http #smoke-tests
