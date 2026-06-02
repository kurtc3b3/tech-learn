
## What & When

**urllib3** is the low-level HTTP client library that powers both `requests` and `httpx` under the hood. It handles connection pooling, retries, SSL, and proxies at the transport layer. Most Python developers never use it directly — but understanding it explains the behaviour of the higher-level libraries built on top of it.

Use urllib3 directly when:

- You need fine-grained control over connection pooling
- Building a custom HTTP transport layer
- Working in environments where `requests`/`httpx` are unavailable
- Debugging low-level HTTP behaviour
- Writing a library that shouldn't pull in `requests` as a dependency

For application code, prefer `requests` (sync) or `httpx` (async/sync). See [[Python — requests Package]] and [[Python — httpx Package]].

```bash
pip install urllib3
```

---

## urllib3 vs requests vs httpx


|                    | urllib3               | requests      | httpx      |
| ------------------ | --------------------- | ------------- | ---------- |
| Level              | Low-level transport   | High-level    | High-level |
| API friendliness   | Verbose               | Simple        | Simple     |
| Async support      | ❌                     | ❌             | ✅          |
| Connection pooling | ✅ Core feature        | Via urllib3   | Built-in   |
| Used by            | requests, httpx, pip  | End users     | End users  |
| Direct use case    | Libraries, transports | Scripts, apps | Async apps |

---

## PoolManager — The Main Entry Point

`PoolManager` manages a pool of `HTTPConnectionPool` objects — one per host. It is the equivalent of `requests.Session` at the urllib3 level.

```python
import urllib3

http = urllib3.PoolManager()

# GET
r = http.request("GET", "https://api.example.com/items")

# POST with JSON
import json
r = http.request(
    "POST",
    "https://api.example.com/items",
    headers={"Content-Type": "application/json"},
    body=json.dumps({"name": "Widget", "price": 9.99}).encode(),
)

print(r.status)         # 200
print(r.data)           # raw bytes
print(json.loads(r.data))
```

---

## HTTP Methods

```python
http = urllib3.PoolManager()

r = http.request("GET",     "https://api.example.com/items")
r = http.request("POST",    "https://api.example.com/items",   body=b"...")
r = http.request("PUT",     "https://api.example.com/items/1", body=b"...")
r = http.request("PATCH",   "https://api.example.com/items/1", body=b"...")
r = http.request("DELETE",  "https://api.example.com/items/1")
r = http.request("HEAD",    "https://api.example.com/items/1")
r = http.request("OPTIONS", "https://api.example.com/items")
```

---

## Request Parameters

```python
http = urllib3.PoolManager()

# Query parameters — encode manually or use urlencode
from urllib.parse import urlencode

params = urlencode({"skip": 0, "limit": 10})
r = http.request("GET", f"https://api.example.com/items?{params}")

# Headers
r = http.request("GET", url, headers={
    "Authorization": "Bearer my-token",
    "Accept":        "application/json",
})

# JSON body
r = http.request(
    "POST", url,
    headers={"Content-Type": "application/json"},
    body=json.dumps({"name": "Widget"}).encode("utf-8"),
)

# Form data — use fields= for urlencoded
r = http.request("POST", url, fields={"username": "alice", "password": "secret"})

# File upload — multipart via fields=
with open("report.pdf", "rb") as f:
    r = http.request("POST", url, fields={
        "file": ("report.pdf", f.read(), "application/pdf"),
        "description": "Monthly report",
    })
```

---

## Response Object

```python
r = http.request("GET", "https://api.example.com/items")

r.status            # 200
r.reason            # "OK"
r.data              # raw bytes
r.headers           # HTTPHeaderDict
r.headers["content-type"]
r.headers.get("x-request-id")

# Decode body
text = r.data.decode("utf-8")
data = json.loads(r.data)
```

---

## Error Handling

```python
import urllib3
from urllib3.exceptions import (
    MaxRetryError,
    NewConnectionError,
    ConnectTimeoutError,
    ReadTimeoutError,
    SSLError,
    HTTPError,
)

http = urllib3.PoolManager()

try:
    r = http.request("GET", url, timeout=10.0)
    if r.status >= 400:
        raise Exception(f"HTTP {r.status}: {r.data.decode()}")

except ConnectTimeoutError:
    print("Connection timed out")

except ReadTimeoutError:
    print("Read timed out")

except MaxRetryError as e:
    print(f"Max retries exceeded: {e.reason}")

except NewConnectionError:
    print("Could not establish connection")

except SSLError as e:
    print(f"SSL error: {e}")

except HTTPError as e:
    print(f"HTTP error: {e}")
```

### Exception Hierarchy

```
urllib3.exceptions.HTTPError
├── urllib3.exceptions.RequestError
│   ├── urllib3.exceptions.ConnectionError
│   │   ├── urllib3.exceptions.NewConnectionError
│   │   └── urllib3.exceptions.ConnectTimeoutError
│   └── urllib3.exceptions.TimeoutError
│       ├── urllib3.exceptions.ConnectTimeoutError
│       └── urllib3.exceptions.ReadTimeoutError
├── urllib3.exceptions.SSLError
├── urllib3.exceptions.MaxRetryError
├── urllib3.exceptions.ResponseError
└── urllib3.exceptions.ProxyError
```

---

## Timeouts

```python
import urllib3

# Single float — applies to both connect and read
r = http.request("GET", url, timeout=10.0)

# Granular via Timeout object
timeout = urllib3.Timeout(connect=3.0, read=10.0)
r = http.request("GET", url, timeout=timeout)

# Pool-wide default timeout
http = urllib3.PoolManager(timeout=urllib3.Timeout(connect=3.0, read=10.0))

# Disable timeout
r = http.request("GET", url, timeout=urllib3.Timeout(read=None))
```

---

## Retries

urllib3's `Retry` object is what `requests`'s `HTTPAdapter` wraps internally.

```python
import urllib3
from urllib3.util.retry import Retry

retry = Retry(
    total=3,                                    # total retry attempts
    backoff_factor=1,                           # wait 1s, 2s, 4s between retries
    status_forcelist=[429, 500, 502, 503, 504], # retry on these HTTP codes
    allowed_methods=["GET", "POST"],            # only retry these methods
    raise_on_status=False,                      # don't raise MaxRetryError on status
    raise_on_redirect=False,
)

http = urllib3.PoolManager(retries=retry)
r = http.request("GET", url)

# Or per-request
r = http.request("GET", url, retries=retry)

# Disable retries
r = http.request("GET", url, retries=False)
```

---

## Connection Pooling

urllib3's core strength — maintains a pool of reusable connections per host.

```python
# PoolManager — multiple hosts
http = urllib3.PoolManager(
    num_pools=10,           # number of connection pools (one per host)
    maxsize=20,             # max connections per pool
    block=False,            # False = create new connection if pool exhausted
                            # True  = block until a connection is available
)

# HTTPConnectionPool / HTTPSConnectionPool — single host
pool = urllib3.HTTPSConnectionPool(
    host="api.example.com",
    port=443,
    maxsize=10,
    block=True,
)

r = pool.request("GET", "/items")
```

---

## SSL & HTTPS

```python
import urllib3
import certifi

# Use certifi's CA bundle (recommended)
http = urllib3.PoolManager(
    ca_certs=certifi.where(),
    cert_reqs="REQUIRED",
)

# Custom CA bundle
http = urllib3.PoolManager(ca_certs="/path/to/ca-bundle.crt")

# Disable SSL verification (not recommended in production)
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)
http = urllib3.PoolManager(cert_reqs="NONE")

# Mutual TLS (client certificate)
http = urllib3.PoolManager(
    ca_certs=certifi.where(),
    cert_file="/path/to/client.crt",
    key_file="/path/to/client.key",
)
```

---

## Proxies

```python
import urllib3

# HTTP proxy
proxy = urllib3.ProxyManager(
    "http://proxy.example.com:8080",
    num_pools=10,
    headers={"Proxy-Authorization": "Basic dXNlcjpwYXNz"},
)

r = proxy.request("GET", "https://api.example.com/items")

# HTTPS proxy
proxy = urllib3.ProxyManager("https://proxy.example.com:8080")

# SOCKs proxy — needs pip install urllib3[socks]
from urllib3.contrib.socks import SOCKSProxyManager
proxy = SOCKSProxyManager("socks5://user:pass@host:port")
```

---

## Streaming Responses

```python
import urllib3

http = urllib3.PoolManager()

# Stream in chunks — preload_content=False
r = http.request("GET", url, preload_content=False)

with open("large-file.zip", "wb") as f:
    for chunk in r.stream(amt=8192):    # amt = bytes per chunk
        f.write(chunk)

r.release_conn()    # return connection to pool

# Stream text lines
r = http.request("GET", url, preload_content=False)
for line in r.stream():
    print(line.decode("utf-8"))
r.release_conn()
```

> [!warning] Always call `r.release_conn()` after streaming Without it the connection is not returned to the pool, causing pool exhaustion under load.

---

## Redirects

```python
# Follow redirects (default: True)
r = http.request("GET", url, redirect=True)

# Disable redirects
r = http.request("GET", url, redirect=False)
print(r.status)                 # 301 / 302
print(r.headers["location"])

# Limit redirect hops (default: 10 via Retry)
retry = urllib3.Retry(redirect=3)
r = http.request("GET", url, retries=retry)
```

---

## HTTPConnectionPool Directly

Use a single-host pool when all requests go to the same server — avoids the overhead of `PoolManager`'s host lookup.

```python
import urllib3

pool = urllib3.HTTPSConnectionPool(
    host="api.example.com",
    port=443,
    maxsize=20,
    timeout=urllib3.Timeout(connect=3, read=10),
    retries=urllib3.Retry(total=3, backoff_factor=1),
    headers={"Authorization": "Bearer my-token"},
)

r = pool.request("GET", "/items")
r = pool.request("POST", "/items", body=json.dumps({"name": "Widget"}).encode(),
                 headers={"Content-Type": "application/json"})
```

---

## Custom Headers on PoolManager

```python
http = urllib3.PoolManager(
    headers={
        "User-Agent":    "my-service/1.0",
        "Accept":        "application/json",
        "Authorization": "Bearer my-token",
    }
)

# Override per-request
r = http.request("GET", url, headers={"X-Request-ID": "abc-123"})
```

---

## Low-Level: `urlopen`

The lowest level — used internally by `request()`.

```python
import urllib3

http = urllib3.PoolManager()

r = http.urlopen("GET", "https://api.example.com/items")
print(r.status)
print(r.data)
```

---

## How requests Uses urllib3

Understanding the relationship explains `requests` behaviour:

```
requests.Session.get(url)
    └── requests.adapters.HTTPAdapter.send(prepared_request)
            └── urllib3.PoolManager.urlopen(method, url, ...)
                    └── urllib3.HTTPSConnectionPool.urlopen(...)
                            └── http.client.HTTPSConnection.request(...)
```

`HTTPAdapter` wraps urllib3's `PoolManager` and translates between the `requests` and urllib3 interfaces. When you configure `HTTPAdapter(max_retries=Retry(...))`, you are passing a urllib3 `Retry` object directly.

---

## Useful Utilities

```python
from urllib3.util.url import parse_url

url = parse_url("https://user:pass@api.example.com:443/items?skip=0#top")
url.scheme      # "https"
url.host        # "api.example.com"
url.port        # 443
url.path        # "/items"
url.query       # "skip=0"
url.fragment    # "top"
url.auth        # "user:pass"


from urllib3.util.retry import Retry
from urllib3.util.timeout import Timeout

# Check if status should be retried
retry = Retry(status_forcelist=[500, 502])
retry.is_retry("GET", 500)   # True
retry.is_retry("GET", 200)   # False
```

---

## Quick Reference

|Task|Code|
|---|---|
|Pool manager|`http = urllib3.PoolManager()`|
|GET|`http.request("GET", url)`|
|POST JSON|`http.request("POST", url, body=json.dumps({}).encode(), headers={"Content-Type": "application/json"})`|
|Form fields|`http.request("POST", url, fields={"key": "val"})`|
|File upload|`http.request("POST", url, fields={"file": ("name", data, "mime")})`|
|Timeout|`http.request("GET", url, timeout=10.0)`|
|Granular timeout|`urllib3.Timeout(connect=3, read=10)`|
|Retries|`urllib3.Retry(total=3, backoff_factor=1, status_forcelist=[500])`|
|Disable retries|`retries=False`|
|SSL with certifi|`PoolManager(ca_certs=certifi.where())`|
|Disable SSL|`PoolManager(cert_reqs="NONE")`|
|Proxy|`urllib3.ProxyManager("http://proxy:8080")`|
|Stream|`http.request("GET", url, preload_content=False)`|
|Release conn|`r.release_conn()`|
|Single host pool|`urllib3.HTTPSConnectionPool(host, port)`|

---
## Tags

#python #urllib3 #http #networking #connection-pooling #backend #low-level