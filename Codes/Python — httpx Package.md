
## What & When

**httpx** is a fully featured HTTP client for Python that supports both **synchronous** and **asynchronous** usage. It is the natural successor to `requests` in async Python projects and the recommended HTTP client for FastAPI-based services.

Use httpx when:

- Making HTTP requests from async code (FastAPI, asyncio services)
- Calling external APIs, microservices, or webhooks
- Testing FastAPI endpoints via `AsyncClient`
- Needing HTTP/2 support
- Requiring connection pooling for high-throughput clients

```bash
pip install httpx
```

---

## httpx vs requests

|                    | httpx               | requests           |
| ------------------ | ------------------- | ------------------ |
| Async support      | ✅ `AsyncClient`     | ❌ Sync only        |
| HTTP/2             | ✅ Optional          | ❌ No               |
| Type hints         | ✅ Full              | Partial            |
| Connection pooling | ✅ Built-in          | ✅ Built-in         |
| FastAPI testing    | ✅ Native\|          | ❌ Needs `requests` |
| API compatibility  | Similar to requests | —                  |

---

## Synchronous Client

```python
import httpx

# One-shot request (creates/closes connection each time — avoid in loops)
response = httpx.get("https://api.example.com/items")
response = httpx.post("https://api.example.com/items", json={"name": "Widget"})

# Persistent client — reuses connection pool (preferred)
with httpx.Client() as client:
    response = client.get("https://api.example.com/items")
    print(response.status_code)
    print(response.json())
```

---

## Async Client

```python
import httpx
import asyncio

async def main():
    async with httpx.AsyncClient() as client:
        response = await client.get("https://api.example.com/items")
        print(response.json())

asyncio.run(main())
```

> [!tip] Always use a client context manager `httpx.get()` / `httpx.AsyncClient()` without a context manager do not reuse connections. Always use `with httpx.Client()` or `async with httpx.AsyncClient()` to benefit from connection pooling.

---

## HTTP Methods

```python
async with httpx.AsyncClient() as client:

    # GET
    r = await client.get("/items", params={"skip": 0, "limit": 10})

    # POST with JSON body
    r = await client.post("/items", json={"name": "Widget", "price": 9.99})

    # PUT — full replacement
    r = await client.put("/items/1", json={"name": "Widget", "price": 11.99})

    # PATCH — partial update
    r = await client.patch("/items/1", json={"price": 11.99})

    # DELETE
    r = await client.delete("/items/1")

    # HEAD
    r = await client.head("/items/1")

    # OPTIONS
    r = await client.options("/items")
```

---

## Request Parameters

```python
async with httpx.AsyncClient() as client:

    # Query parameters
    r = await client.get("/items", params={"skip": 0, "limit": 10, "tag": ["a", "b"]})
    # → /items?skip=0&limit=10&tag=a&tag=b

    # Headers
    r = await client.get("/items", headers={"X-API-Key": "secret"})

    # JSON body
    r = await client.post("/items", json={"name": "Widget"})

    # Form data
    r = await client.post("/login", data={"username": "alice", "password": "secret"})

    # File upload
    with open("report.pdf", "rb") as f:
        r = await client.post("/upload", files={"file": f})

    # Multipart with fields and file
    r = await client.post("/upload", files={
        "file":        ("report.pdf", open("report.pdf", "rb"), "application/pdf"),
        "description": (None, "Monthly report"),
    })

    # Raw bytes
    r = await client.post("/data", content=b"\x00\x01\x02", headers={"Content-Type": "application/octet-stream"})
```

---

## Response Object

```python
response = await client.get("/items/1")

# Status
response.status_code                    # 200
response.is_success                     # True if 2xx
response.is_redirect                    # True if 3xx
response.is_client_error               # True if 4xx
response.is_server_error               # True if 5xx

# Raise for non-2xx
response.raise_for_status()            # raises httpx.HTTPStatusError

# Body
response.text                           # string
response.content                        # bytes
response.json()                         # parsed JSON → dict/list

# Headers
response.headers["content-type"]
response.headers.get("x-request-id")

# URL & history
response.url                            # final URL (after redirects)
response.history                        # list of redirect responses

# Encoding
response.encoding                       # detected encoding
response.apparent_encoding             # charset-normalizer detection
```

---

## Error Handling

```python
import httpx

async def safe_request(url: str):
    async with httpx.AsyncClient() as client:
        try:
            response = await client.get(url, timeout=10.0)
            response.raise_for_status()
            return response.json()

        except httpx.TimeoutException:
            print("Request timed out")

        except httpx.HTTPStatusError as e:
            print(f"HTTP error {e.response.status_code}: {e.response.text}")

        except httpx.ConnectError:
            print("Could not connect to server")

        except httpx.RequestError as e:
            print(f"Request failed: {e}")
```

### Exception Hierarchy

```
httpx.RequestError
├── httpx.TransportError
│   ├── httpx.TimeoutException
│   │   ├── httpx.ConnectTimeout
│   │   ├── httpx.ReadTimeout
│   │   ├── httpx.WriteTimeout
│   │   └── httpx.PoolTimeout
│   ├── httpx.ConnectError
│   ├── httpx.ReadError
│   └── httpx.WriteError
└── httpx.HTTPStatusError     (raised by raise_for_status())
```

---

## Timeouts

```python
# Single timeout for all phases
client = httpx.AsyncClient(timeout=10.0)

# Granular timeouts
timeout = httpx.Timeout(
    connect=5.0,    # time to establish connection
    read=10.0,      # time to receive response body
    write=5.0,      # time to send request body
    pool=3.0,       # time to acquire connection from pool
)
client = httpx.AsyncClient(timeout=timeout)

# Disable timeout (use with caution)
client = httpx.AsyncClient(timeout=None)

# Per-request override
response = await client.get("/slow-endpoint", timeout=60.0)
```

---

## Authentication

```python
# Basic auth
client = httpx.AsyncClient(auth=("username", "password"))

# Bearer token
client = httpx.AsyncClient(headers={"Authorization": "Bearer eyJ..."})

# Custom auth class
class BearerAuth(httpx.Auth):
    def __init__(self, token: str):
        self.token = token

    def auth_flow(self, request):
        request.headers["Authorization"] = f"Bearer {self.token}"
        yield request

client = httpx.AsyncClient(auth=BearerAuth("my-token"))

# API key in header
client = httpx.AsyncClient(headers={"X-API-Key": "my-key"})

# API key in query param
response = await client.get("/items", params={"api_key": "my-key"})
```

---

## Base URL & Headers

```python
# Set base URL — all requests are relative to it
client = httpx.AsyncClient(
    base_url="https://api.example.com/v1",
    headers={
        "Authorization": "Bearer my-token",
        "Accept":        "application/json",
        "X-Client-ID":   "my-service",
    },
)

async with client:
    r = await client.get("/items")          # → https://api.example.com/v1/items
    r = await client.post("/orders")        # → https://api.example.com/v1/orders
```

---

## Connection Pooling & Limits

```python
limits = httpx.Limits(
    max_connections=100,        # total connections in pool
    max_keepalive_connections=20,  # idle keepalive connections
    keepalive_expiry=30.0,      # seconds before idle connection is closed
)

client = httpx.AsyncClient(limits=limits)
```

---

## Retries

httpx does not retry by default. Use `httpx` with a transport that supports retries:

```python
transport = httpx.AsyncHTTPTransport(retries=3)
client = httpx.AsyncClient(transport=transport)

# Or manual retry with tenacity
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(stop=stop_after_attempt(3), wait=wait_exponential(multiplier=1, min=1, max=10))
async def fetch_with_retry(url: str):
    async with httpx.AsyncClient() as client:
        response = await client.get(url)
        response.raise_for_status()
        return response.json()
```

---

## Streaming Responses

For large responses — download without loading the entire body into memory.

```python
async with httpx.AsyncClient() as client:

    # Stream response body
    async with client.stream("GET", "https://example.com/large-file") as response:
        async for chunk in response.aiter_bytes(chunk_size=8192):
            process(chunk)

    # Stream text lines
    async with client.stream("GET", "https://example.com/logs") as response:
        async for line in response.aiter_lines():
            print(line)

    # Stream raw bytes
    async with client.stream("GET", url) as response:
        total = int(response.headers["content-length"])
        downloaded = 0
        with open("file.bin", "wb") as f:
            async for chunk in response.aiter_bytes():
                f.write(chunk)
                downloaded += len(chunk)
                print(f"{downloaded / total:.0%}")
```

---

## HTTP/2

```bash
pip install httpx[http2]
```

```python
client = httpx.AsyncClient(http2=True)

async with client:
    response = await client.get("https://http2.example.com/")
    print(response.http_version)    # "HTTP/2"
```

---

## Concurrent Requests with asyncio

```python
import asyncio, httpx

async def fetch(client: httpx.AsyncClient, url: str) -> dict:
    response = await client.get(url)
    response.raise_for_status()
    return response.json()

async def main():
    urls = [
        "https://api.example.com/users",
        "https://api.example.com/orders",
        "https://api.example.com/products",
    ]

    async with httpx.AsyncClient(base_url="https://api.example.com") as client:
        results = await asyncio.gather(*[fetch(client, url) for url in urls])

    return results

asyncio.run(main())
```

---

## Using httpx as a Service Client in FastAPI

A clean pattern — create a shared client via lifespan and inject it as a dependency.

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI, Depends, Request
import httpx

@asynccontextmanager
async def lifespan(app: FastAPI):
    async with httpx.AsyncClient(
        base_url="https://api.partner.com",
        headers={"Authorization": f"Bearer {settings.partner_api_key}"},
        timeout=10.0,
    ) as client:
        app.state.http_client = client
        yield

app = FastAPI(lifespan=lifespan)

def get_http_client(request: Request) -> httpx.AsyncClient:
    return request.app.state.http_client

@app.get("/partner/items")
async def get_partner_items(client: httpx.AsyncClient = Depends(get_http_client)):
    response = await client.get("/items")
    response.raise_for_status()
    return response.json()
```

---

## Testing FastAPI with `AsyncClient`

httpx's `AsyncClient` with the `ASGITransport` is the standard way to test FastAPI endpoints without running a server.

```python
import pytest
import pytest_asyncio
from httpx import AsyncClient, ASGITransport
from main import app

@pytest.mark.anyio
async def test_create_item():
    async with AsyncClient(
        transport=ASGITransport(app=app),
        base_url="http://test",
    ) as client:
        response = await client.post("/items", json={"name": "Widget", "price": 9.99})
        assert response.status_code == 201
        assert response.json()["name"] == "Widget"


@pytest.mark.anyio
async def test_get_item():
    async with AsyncClient(
        transport=ASGITransport(app=app),
        base_url="http://test",
    ) as client:
        response = await client.get("/items/1")
        assert response.status_code == 200
```

```bash
pip install pytest pytest-asyncio anyio
```

```ini
# pytest.ini
[pytest]
asyncio_mode = auto
```

---

## Mocking httpx in Tests

```python
import pytest
import respx
import httpx

@pytest.mark.anyio
@respx.mock
async def test_external_call():
    respx.get("https://api.partner.com/items").mock(
        return_value=httpx.Response(200, json=[{"id": 1, "name": "Widget"}])
    )

    async with httpx.AsyncClient() as client:
        response = await client.get("https://api.partner.com/items")
        assert response.status_code == 200
        assert response.json()[0]["name"] == "Widget"
```

```bash
pip install respx
```

---

## Quick Reference

|Task|Code|
|---|---|
|Sync client|`with httpx.Client() as client`|
|Async client|`async with httpx.AsyncClient() as client`|
|Base URL|`AsyncClient(base_url="https://...")`|
|Auth header|`AsyncClient(headers={"Authorization": "Bearer ..."})`|
|Timeout|`AsyncClient(timeout=10.0)`|
|Raise on error|`response.raise_for_status()`|
|Check success|`response.is_success`|
|Parse JSON|`response.json()`|
|Stream response|`client.stream("GET", url)`|
|HTTP/2|`AsyncClient(http2=True)`|
|Concurrent|`asyncio.gather(*[client.get(u) for u in urls])`|
|Test FastAPI|`AsyncClient(transport=ASGITransport(app=app))`|
|Mock requests|`respx.mock` + `respx.get(url).mock(...)`|
|Shared client|`app.state.http_client` via lifespan|

---

## Tags

#python #httpx #http #async #client #testing #backend