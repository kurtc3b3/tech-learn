## What & When

**requests** is the most widely used HTTP client library in Python. It wraps Python's `urllib` in a clean, human-friendly API and remains the standard choice for **synchronous** HTTP in scripts, CLI tools, and non-async services.

Use requests when:

- Writing scripts, CLI tools, or data pipelines
- Working in a synchronous codebase (Django, Flask, plain Python)
- Consuming REST APIs, webhooks, or scraping
- Simplicity is more important than async performance

For async code (FastAPI, asyncio), use `httpx` instead. See [[Python — httpx Package]]

```bash
pip install requests
```

---

## requests vs httpx

|                 | requests           | httpx                 |
| --------------- | ------------------ | --------------------- |
| Async support   | ❌ Sync only        | ✅ Both                |
| HTTP/2          | ❌ No               | ✅ Optional            |
| FastAPI testing | ❌ No               | ✅ Native              |
| Maturity        | ✅ Very mature      | ✅ Actively maintained |
| Ecosystem       | ✅ Huge             | Growing               |
| Best for        | Scripts, sync code | Async / FastAPI       |

---

## Quick Requests (One-Shot)

```python
import requests

r = requests.get("https://api.example.com/items")
r = requests.post("https://api.example.com/items", json={"name": "Widget"})
r = requests.put("https://api.example.com/items/1", json={"name": "Widget"})
r = requests.patch("https://api.example.com/items/1", json={"price": 9.99})
r = requests.delete("https://api.example.com/items/1")
r = requests.head("https://api.example.com/items/1")
r = requests.options("https://api.example.com/items")
```

> [!tip] Use `Session` for more than one request One-shot calls open and close a new connection every time. `Session` reuses connections and lets you set shared headers/auth once.

---

## Session — Persistent Connection Pool

```python
import requests

with requests.Session() as session:
    session.base_url = "https://api.example.com"    # not built-in, set manually
    session.headers.update({
        "Authorization": "Bearer my-token",
        "Accept":        "application/json",
    })

    r = session.get("https://api.example.com/items")
    r = session.post("https://api.example.com/orders", json={"item_id": 1})
```

---

## Request Parameters

```python
with requests.Session() as s:

    # Query parameters
    r = s.get("/items", params={"skip": 0, "limit": 10, "tag": ["a", "b"]})
    # → /items?skip=0&limit=10&tag=a&tag=b

    # Headers
    r = s.get("/items", headers={"X-API-Key": "secret"})

    # JSON body
    r = s.post("/items", json={"name": "Widget", "price": 9.99})

    # Form data
    r = s.post("/login", data={"username": "alice", "password": "secret"})

    # File upload
    with open("report.pdf", "rb") as f:
        r = s.post("/upload", files={"file": f})

    # Multipart with fields and file
    r = s.post("/upload", files={
        "file":        ("report.pdf", open("report.pdf", "rb"), "application/pdf"),
        "description": (None, "Monthly report"),
    })

    # Raw bytes
    r = s.post("/data", data=b"\x00\x01\x02",
               headers={"Content-Type": "application/octet-stream"})
```

---

## Response Object

```python
r = requests.get("https://api.example.com/items/1")

# Status
r.status_code                   # 200
r.ok                            # True if status < 400
r.raise_for_status()            # raises HTTPError for 4xx / 5xx

# Body
r.text                          # string (decoded)
r.content                       # raw bytes
r.json()                        # parsed JSON → dict/list

# Headers
r.headers["content-type"]
r.headers.get("x-request-id")

# URL & redirects
r.url                           # final URL after redirects
r.history                       # list of redirect responses

# Encoding
r.encoding                      # detected or declared encoding
r.apparent_encoding             # chardet detection
```

---

## Error Handling

```python
import requests
from requests.exceptions import (
    Timeout,
    ConnectionError,
    HTTPError,
    RequestException,
)

def safe_get(url: str):
    try:
        r = requests.get(url, timeout=10)
        r.raise_for_status()
        return r.json()

    except Timeout:
        print("Request timed out")

    except ConnectionError:
        print("Could not connect to server")

    except HTTPError as e:
        print(f"HTTP error {e.response.status_code}: {e.response.text}")

    except RequestException as e:
        print(f"Request failed: {e}")
```

### Exception Hierarchy

```
requests.RequestException
├── requests.ConnectionError
│   └── requests.ProxyError
├── requests.HTTPError           (raised by raise_for_status())
├── requests.URLRequired
├── requests.TooManyRedirects
└── requests.Timeout
    ├── requests.ConnectTimeout
    └── requests.ReadTimeout
```

---

## Timeouts

```python
# Single timeout for connect + read
r = requests.get(url, timeout=10)

# Separate connect and read timeouts
r = requests.get(url, timeout=(3.05, 10))
#                              ↑ connect  ↑ read

# Never time out (use with caution)
r = requests.get(url, timeout=None)
```

---

## Authentication

```python
from requests.auth import HTTPBasicAuth, HTTPDigestAuth

# Basic auth
r = requests.get(url, auth=("username", "password"))
r = requests.get(url, auth=HTTPBasicAuth("username", "password"))

# Digest auth
r = requests.get(url, auth=HTTPDigestAuth("username", "password"))

# Bearer token
session.headers.update({"Authorization": "Bearer eyJ..."})

# API key in header
session.headers.update({"X-API-Key": "my-key"})

# API key in query param
r = session.get("/items", params={"api_key": "my-key"})

# Custom auth class
class BearerAuth(requests.auth.AuthBase):
    def __init__(self, token: str):
        self.token = token

    def __call__(self, r):
        r.headers["Authorization"] = f"Bearer {self.token}"
        return r

session.auth = BearerAuth("my-token")
```

---

## SSL & Certificates

```python
# Verify SSL (default: True)
r = requests.get(url, verify=True)

# Disable SSL verification (not recommended in production)
import urllib3
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)
r = requests.get(url, verify=False)

# Custom CA bundle
r = requests.get(url, verify="/path/to/ca-bundle.crt")

# Client certificate (mutual TLS)
r = requests.get(url, cert=("/path/to/client.crt", "/path/to/client.key"))
r = requests.get(url, cert="/path/to/client.pem")   # combined cert+key
```

---

## Redirects

```python
# Follow redirects (default: True for GET, OPTIONS, HEAD — False for others)
r = requests.get(url, allow_redirects=True)

# Disable redirects
r = requests.get(url, allow_redirects=False)
print(r.status_code)    # 301 / 302
print(r.headers["Location"])

# Inspect redirect history
r = requests.get(url)
for redirect in r.history:
    print(f"{redirect.status_code} → {redirect.url}")
print(f"Final: {r.url}")
```

---

## Streaming Responses

For large files — download without loading the entire body into memory.

```python
import requests

url = "https://example.com/large-file.zip"

with requests.get(url, stream=True) as r:
    r.raise_for_status()
    total = int(r.headers.get("content-length", 0))
    downloaded = 0

    with open("large-file.zip", "wb") as f:
        for chunk in r.iter_content(chunk_size=8192):
            f.write(chunk)
            downloaded += len(chunk)
            if total:
                print(f"{downloaded / total:.0%}", end="\r")

# Stream text lines
with requests.get(url, stream=True) as r:
    for line in r.iter_lines():
        if line:
            print(line.decode("utf-8"))
```

---

## Retries with `urllib3`

```python
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

retry_strategy = Retry(
    total=3,                        # total retry attempts
    backoff_factor=1,               # wait 1, 2, 4 seconds between retries
    status_forcelist=[429, 500, 502, 503, 504],  # retry on these status codes
    allowed_methods=["GET", "POST"],
    raise_on_status=False,
)

adapter = HTTPAdapter(max_retries=retry_strategy)

session = requests.Session()
session.mount("https://", adapter)
session.mount("http://",  adapter)

r = session.get("https://api.example.com/items")
```

---

## Connection Pooling

```python
from requests.adapters import HTTPAdapter

adapter = HTTPAdapter(
    pool_connections=10,    # number of connection pools (per host)
    pool_maxsize=20,        # max connections per pool
    max_retries=3,
)

session = requests.Session()
session.mount("https://", adapter)
session.mount("http://",  adapter)
```

---

## Proxies

```python
proxies = {
    "http":  "http://proxy.example.com:8080",
    "https": "http://proxy.example.com:8080",
}

r = requests.get(url, proxies=proxies)

# With auth
proxies = {
    "https": "http://user:pass@proxy.example.com:8080",
}

# Session-wide
session.proxies.update(proxies)

# SOCKS proxy
# pip install requests[socks]
proxies = {"https": "socks5://user:pass@host:port"}
```

---

## Prepared Requests

Inspect or modify a request before sending — useful for signing, logging, or debugging.

```python
from requests import Request, Session

req = Request(
    method="POST",
    url="https://api.example.com/items",
    json={"name": "Widget"},
    headers={"Authorization": "Bearer token"},
)

prepared = req.prepare()

# Inspect before sending
print(prepared.url)
print(prepared.headers)
print(prepared.body)

# Send
session = Session()
r = session.send(prepared, timeout=10)
```

---

## Hooks — Request/Response Callbacks

```python
def log_response(r, *args, **kwargs):
    print(f"{r.request.method} {r.url} → {r.status_code} ({r.elapsed.total_seconds():.2f}s)")

def raise_on_4xx(r, *args, **kwargs):
    if 400 <= r.status_code < 500:
        r.raise_for_status()

session = requests.Session()
session.hooks["response"].append(log_response)
session.hooks["response"].append(raise_on_4xx)

# Per-request
r = requests.get(url, hooks={"response": [log_response]})
```

---

## Cookies

```python
# Send cookies
r = requests.get(url, cookies={"session": "abc123"})

# Read cookies from response
r = requests.get(url)
print(r.cookies["session"])
print(dict(r.cookies))

# Session-wide cookies (persisted across requests)
session = requests.Session()
session.get("https://example.com/login", data={"user": "alice", "pass": "secret"})
# session.cookies now contains any Set-Cookie from the response

r = session.get("https://example.com/profile")
# cookies sent automatically
```

---

## Mocking in Tests

```python
# pip install responses
import responses
import requests

@responses.activate
def test_get_items():
    responses.add(
        responses.GET,
        "https://api.example.com/items",
        json=[{"id": 1, "name": "Widget"}],
        status=200,
    )

    r = requests.get("https://api.example.com/items")
    assert r.status_code == 200
    assert r.json()[0]["name"] == "Widget"
    assert len(responses.calls) == 1
```

---

## Common Patterns

### Pagination

```python
def fetch_all(session, url: str) -> list:
    results = []
    params = {"page": 1, "per_page": 100}

    while True:
        r = session.get(url, params=params)
        r.raise_for_status()
        data = r.json()

        if not data:
            break

        results.extend(data)
        params["page"] += 1

    return results
```

### Rate-Limited API

```python
import time

def rate_limited_get(session, urls: list, calls_per_second: float = 5):
    interval = 1.0 / calls_per_second
    results = []

    for url in urls:
        r = session.get(url)
        r.raise_for_status()
        results.append(r.json())
        time.sleep(interval)

    return results
```

---

## Quick Reference

|Task|Code|
|---|---|
|GET|`requests.get(url)`|
|POST JSON|`requests.post(url, json={...})`|
|Session|`with requests.Session() as s`|
|Query params|`s.get(url, params={"key": "val"})`|
|Headers|`s.headers.update({...})`|
|Auth|`s.auth = ("user", "pass")`|
|Bearer token|`s.headers["Authorization"] = "Bearer ..."`|
|Timeout|`s.get(url, timeout=(3, 10))`|
|Raise on error|`r.raise_for_status()`|
|Check success|`r.ok`|
|Parse JSON|`r.json()`|
|Stream download|`s.get(url, stream=True)` then `r.iter_content()`|
|Retry|`HTTPAdapter(max_retries=Retry(...))`|
|Disable SSL|`verify=False`|
|Proxy|`s.get(url, proxies={...})`|
|Mock in tests|`@responses.activate`|
|Log responses|`session.hooks["response"].append(fn)`|

---

## Tags

#python #requests #http #client #backend #scripting