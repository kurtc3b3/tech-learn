## What & When

**Tornado** is a Python **web framework and async HTTP server** built around a single-threaded **I/O event loop**. It pioneered non-blocking request handlers and **WebSocket** support before ASGI and [[Python — asyncio]] became standard. Use it mainly to **maintain existing Tornado codebases**; new projects in this vault prefer [[API - FastAPI]].

Use Tornado when:

- You inherit a **Tornado monolith** (long-polling, WebSocket fan-out)
- You need a **standalone async HTTP server** without Gunicorn+Uvicorn split
- You want **minimal dependencies** for async HTTP + templates in one package

For new APIs, WebSockets, and SSE: [[API - FastAPI]], [[API - FastAPI — WebSockets]], [[API - FastAPI — Server-Sent Events (SSE)]]. Overview: [[Web]].

```bash
pip install tornado
```

---

## Tornado vs Related Tools

| Need | Use | Notes |
| --- | --- | --- |
| New async JSON API | [[API - FastAPI]] | OpenAPI, Pydantic, ASGI ecosystem |
| Simple sync app | [[Web — Flask]] | WSGI, easier mental model |
| Full monolith | [[Web — Django]] | Admin, ORM |
| **Legacy async HTTP/WS** | **Tornado** | `IOLoop`, `@gen.coroutine` / native async |
| Low-level WS client | [[Python — websockets Package]] | Outside Tornado server |
| Many concurrent HTTP clients | [[Python — asyncio]] + [[Python — httpx Package]] | Not a web framework |

---

## Minimal Application

```python
import tornado.ioloop
import tornado.web

class MainHandler(tornado.web.RequestHandler):
    def get(self):
        self.write({"message": "hello"})

def make_app():
    return tornado.web.Application([
        (r"/", MainHandler),
    ])

if __name__ == "__main__":
    app = make_app()
    app.listen(8888)
    tornado.ioloop.IOLoop.current().start()
```

```bash
python app.py
# curl http://localhost:8888/
```

---

## Async Handlers (Modern Style)

```python
import asyncio
import tornado.web

class AsyncHandler(tornado.web.RequestHandler):
    async def get(self):
        await asyncio.sleep(0.1)  # non-blocking I/O placeholder
        self.write({"ok": True})
```

Tornado 6+ supports native `async def` on `RequestHandler`. Older code uses `@tornado.gen.coroutine` and `yield` — refactor to `async/await` when touching files.

---

## WebSocket Handler

```python
import tornado.websocket
import tornado.ioloop
import tornado.web

clients: set[tornado.websocket.WebSocketHandler] = set()

class ChatSocket(tornado.websocket.WebSocketHandler):
    def open(self):
        clients.add(self)

    def on_close(self):
        clients.discard(self)

    def on_message(self, message):
        for client in clients:
            if client is not self:
                client.write_message(message)

def make_app():
    return tornado.web.Application([
        (r"/ws", ChatSocket),
    ])

if __name__ == "__main__":
    make_app().listen(8888)
    tornado.ioloop.IOLoop.current().start()
```

Compare patterns: [[API - FastAPI — WebSockets]] (connection manager, `WebSocketDisconnect`, Pydantic message validation).

---

## Routing & Application Settings

```python
import tornado.web

class ItemHandler(tornado.web.RequestHandler):
    def get(self, item_id):
        self.write({"id": item_id})

def make_app():
    return tornado.web.Application(
        [
            (r"/items/([0-9]+)", ItemHandler),
        ],
        debug=True,
        template_path="templates",
        static_path="static",
    )
```

| Setting | Purpose |
| --- | --- |
| `template_path` | Tornado template directory |
| `static_path` | Serve `/static/...` |
| `debug` | Auto-reload (dev only) |
| `autoreload` | Watch file changes |

Templates use Tornado's syntax (`{% %}`, `{{ }}`) — similar to Jinja2 but not identical. See [[Python — Jinja2 Package]] for Flask/FastAPI.

---

## Templates

```python
class HomeHandler(tornado.web.RequestHandler):
    def get(self):
        self.render("home.html", title="Home", items=["a", "b"])
```

```html
<!-- templates/home.html -->
<h1>{{ title }}</h1>
<ul>
  {% for item in items %}
    <li>{{ item }}</li>
  {% end %}
</ul>
```

---

## HTTP Client (Async)

Tornado includes `tornado.httpclient` for outbound requests:

```python
from tornado.httpclient import AsyncHTTPClient
import tornado.ioloop

async def fetch():
    client = AsyncHTTPClient()
    resp = await client.fetch("https://httpbin.org/get")
    print(resp.body.decode())

tornado.ioloop.IOLoop.current().run_sync(fetch)
```

Prefer [[Python — httpx Package]] in new code shared with FastAPI services.

---

## Periodic Callbacks & Long-Running Loops

```python
from tornado.ioloop import PeriodicCallback

def tick():
    print("tick")

pc = PeriodicCallback(tick, 1000)  # ms
pc.start()
```

Use [[Processing — Celery]] Beat for cron across machines; Tornado timers suit single-process housekeeping.

---

## Testing

```python
from tornado.testing import AsyncHTTPTestCase, gen_test
import tornado.web

class HelloHandler(tornado.web.RequestHandler):
    def get(self):
        self.write("hello")

class MainTest(AsyncHTTPTestCase):
    def get_app(self):
        return tornado.web.Application([(r"/", HelloHandler)])

    def test_home(self):
        response = self.fetch("/")
        self.assertEqual(response.code, 200)
        self.assertEqual(response.body, b"hello")
```

Run with `python -m tornado.testing test_module` or pytest wrappers.

---

## Production Deployment

Historically Tornado ran as **its own process** listening on a port:

```bash
python app.py  # binds 8888 inside script
```

Behind nginx reverse proxy, one or more Tornado processes scale horizontally. Unlike WSGI (Gunicorn workers) or ASGI (Uvicorn), Tornado **is** the server.

| Approach | Notes |
| --- | --- |
| Multiple processes | Supervisor/systemd, one Tornado per port or SO_REUSEPORT |
| Static files | nginx or Tornado `static_path` |
| Migrate to FastAPI | Reimplement routes; map WebSocket handlers to Starlette |

---

## Migration Path to FastAPI

For teams retiring Tornado:

1. **Inventory** — HTTP routes, WebSocket endpoints, periodic tasks
2. **Extract services** — move business logic out of `RequestHandler`
3. **Port HTTP** — FastAPI routes + Pydantic models
4. **Port WebSocket** — [[API - FastAPI — WebSockets]] connection manager
5. **Run Uvicorn** — replace `IOLoop.current().start()`

Keep Tornado running until parity tests pass ([[Unit Testing - pytest]]).

---

## When Not to Choose Tornado

- Greenfield **REST API** — [[API - FastAPI]]
- Need **OpenAPI** docs — FastAPI auto-generates
- Team standard is **SQLAlchemy + DI** — [[ORM - SQLAlchemy]], [[API - FastAPI — Dependency Injection & User Management]]
- Heavy **CPU work** in handlers — offload to [[Processing — Celery]] or [[Processing — Ray]]

---

## Quick Reference

| Task | Pattern |
| --- | --- |
| Start server | `app.listen(port); IOLoop.current().start()` |
| GET handler | `def get(self): self.write(...)` |
| Async handler | `async def get(self): ...` |
| WebSocket | Subclass `tornado.websocket.WebSocketHandler` |
| Path param | Regex in URL pattern `([0-9]+)` |
| Template | `self.render("t.html", **ctx)` |
| Test | `AsyncHTTPTestCase` + `self.fetch()` |

---

## Related Notes

- [[Web]]
- [[Web — Flask]]
- [[Web — Django]]
- [[API - FastAPI]]
- [[API - FastAPI — WebSockets]]
- [[Python — asyncio]]
- [[Python — websockets Package]]

---

## Tags

#web #tornado #async #websocket #python #backend #ioloop #legacy
