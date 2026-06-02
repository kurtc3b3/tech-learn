## What & When

**Server-Sent Events (SSE)** is a one-way, server-to-client streaming protocol over a regular HTTP connection. The client opens a connection and the server pushes events down it indefinitely — no client messages, no handshake upgrade.

Use SSE when:

- Server needs to push updates to the client (notifications, progress, logs)
- Communication is **one-way** — client only listens
- You want simplicity over the full duplex of WebSockets
- You need automatic browser reconnection for free

---

## SSE vs WebSockets vs Polling


|                   | SSE                            | WebSocket           | Polling         |
| ----------------- | ------------------------------ | ------------------- | --------------- |
| Direction         | Server → Client only           | Bi-directional      | Client → Server |
| Protocol          | HTTP                           | WS / WSS            | HTTP            |
| Browser reconnect | ✅ Automatic                    | ❌ Manual            | N/A             |
| Complexity        | Low                            | Medium              | Low             |
| Overhead          | Low                            | Lowest              | High            |
| Use case          | Notifications, feeds, progress | Chat, collaboration | Simple updates  |


> [!tip] SSE is often the right choice If you don't need the client to send messages over the same connection, SSE is simpler than WebSockets — no connection manager, no upgrade, built-in reconnection, and works over standard HTTP/2.

---

## Setup

```bash
pip install sse-starlette
```

```python
from fastapi import FastAPI
from sse_starlette.sse import EventSourceResponse
import asyncio

app = FastAPI()
```

---

## Basic SSE Endpoint

```python
from sse_starlette.sse import EventSourceResponse
import asyncio

async def event_generator():
    for i in range(10):
        yield {"data": f"Message {i}"}
        await asyncio.sleep(1)

@app.get("/stream")
async def stream():
    return EventSourceResponse(event_generator())
```

### Connecting from the Browser

```javascript
const source = new EventSource("http://localhost:8000/stream");

source.onmessage = (event) => console.log(event.data);
source.onerror   = () => source.close();
```

> [!info] `EventSource` is a native browser API — no library needed on the client.

---

## Event Structure

SSE events are plain text with specific fields:

```
data: hello world\n\n          ← minimal event

id: 42\n
event: notification\n
data: {"message": "You have mail"}\n
retry: 3000\n
\n                              ← blank line = end of event
```

```python
async def event_generator():
    yield {
        "id":    "1",                           # client uses for reconnect resume
        "event": "notification",                # custom event type
        "data":  '{"message": "You have mail"}',
        "retry": 3000,                          # reconnect delay in ms
    }
```

```javascript
// Listen for a specific event type
source.addEventListener("notification", (event) => {
    const payload = JSON.parse(event.data);
    console.log(payload.message);
});
```

---

## Streaming JSON

```python
import json
from sse_starlette.sse import EventSourceResponse

async def price_feed():
    while True:
        price = await fetch_latest_price()
        yield {
            "event": "price",
            "data":  json.dumps({"symbol": "BTC", "price": price}),
        }
        await asyncio.sleep(1)

@app.get("/prices/stream")
async def stream_prices():
    return EventSourceResponse(price_feed())
```

---

## Detecting Client Disconnect

Without disconnect detection, the generator keeps running after the client leaves — wasting resources.

```python
from fastapi import Request

async def event_generator(request: Request):
    while True:
        if await request.is_disconnected():
            print("Client disconnected — stopping generator")
            break

        yield {"data": f"tick {datetime.utcnow().isoformat()}"}
        await asyncio.sleep(1)

@app.get("/stream")
async def stream(request: Request):
    return EventSourceResponse(event_generator(request))
```

> [!warning] Always check `request.is_disconnected()` Without it, generators for departed clients accumulate silently, leaking memory and CPU over time.

---

## Progress Streaming

A natural SSE use case — push job progress without polling.

```python
from fastapi import Request
from sse_starlette.sse import EventSourceResponse
import asyncio, json

async def run_job(request: Request):
    steps = ["Fetching data", "Processing", "Saving results", "Done"]
    for i, step in enumerate(steps):
        if await request.is_disconnected():
            break
        yield {
            "event": "progress",
            "data": json.dumps({
                "step":    step,
                "percent": int((i + 1) / len(steps) * 100),
            }),
        }
        await asyncio.sleep(2)   # simulate work

@app.get("/jobs/{job_id}/stream")
async def stream_job(job_id: str, request: Request):
    return EventSourceResponse(run_job(request))
```

```javascript
const source = new EventSource(`/jobs/${jobId}/stream`);

source.addEventListener("progress", (event) => {
    const { step, percent } = JSON.parse(event.data);
    progressBar.style.width = `${percent}%`;
    statusLabel.textContent = step;
    if (percent === 100) source.close();
});
```

---

## Notification Feed with Authentication

Pass auth via query parameter (same limitation as WebSockets — `EventSource` does not support custom headers in the browser).

```python
from fastapi import Request, Query, HTTPException
from core.dependencies import decode_jwt_token

async def notification_stream(request: Request, user):
    while True:
        if await request.is_disconnected():
            break
        notifications = await fetch_unread(user.id)
        for n in notifications:
            yield {
                "id":    str(n.id),
                "event": "notification",
                "data":  json.dumps({"title": n.title, "body": n.body}),
            }
        await asyncio.sleep(5)

@app.get("/notifications/stream")
async def stream_notifications(
    request: Request,
    token: str = Query(...),
):
    user = decode_jwt_token(token)
    if not user:
        raise HTTPException(status_code=401, detail="Invalid token")
    return EventSourceResponse(notification_stream(request, user))
```

---

## SSE with a Queue — Push from Anywhere

Instead of polling inside the generator, use an `asyncio.Queue` so any part of the app can push an event.

```python
from asyncio import Queue
from typing import AsyncGenerator

# One queue per connected client
class EventBus:
    def __init__(self):
        self.subscribers: list[Queue] = []

    def subscribe(self) -> Queue:
        q = Queue()
        self.subscribers.append(q)
        return q

    def unsubscribe(self, q: Queue):
        self.subscribers.remove(q)

    async def publish(self, event: dict):
        for q in self.subscribers:
            await q.put(event)


bus = EventBus()

async def listen(request: Request, queue: Queue) -> AsyncGenerator:
    try:
        while True:
            if await request.is_disconnected():
                break
            event = await asyncio.wait_for(queue.get(), timeout=30)
            yield event
    finally:
        bus.unsubscribe(queue)

@app.get("/events/stream")
async def stream_events(request: Request):
    queue = bus.subscribe()
    return EventSourceResponse(listen(request, queue))

# Trigger from any route or background task
@app.post("/events/publish")
async def publish_event(message: str):
    await bus.publish({"event": "update", "data": message})
    return {"published": True}
```

---

## SSE in a Router

```python
# routers/stream.py
from fastapi import APIRouter, Request
from sse_starlette.sse import EventSourceResponse

router = APIRouter(prefix="/stream", tags=["streaming"], include_in_schema=True)

@router.get("/logs")
async def stream_logs(request: Request): ...

@router.get("/notifications")
async def stream_notifications(request: Request): ...
```

---

## Browser Reconnection & `Last-Event-ID`

The browser automatically reconnects after a dropped SSE connection and sends the last received event ID in a `Last-Event-ID` header — use this to resume from where you left off.

```python
from fastapi import Request, Header
from typing import Optional

@app.get("/stream/resumable")
async def resumable_stream(
    request: Request,
    last_event_id: Optional[str] = Header(None),
):
    start_from = int(last_event_id) + 1 if last_event_id else 0

    async def generator():
        for i in range(start_from, start_from + 100):
            if await request.is_disconnected():
                break
            yield {"id": str(i), "data": f"Event {i}"}
            await asyncio.sleep(1)

    return EventSourceResponse(generator())
```

---

## Quick Reference

|Task|Code|
|---|---|
|Basic stream|`EventSourceResponse(async_generator())`|
|Send plain text|`yield {"data": "hello"}`|
|Send JSON|`yield {"data": json.dumps({...})}`|
|Named event|`yield {"event": "update", "data": "..."}`|
|Event ID|`yield {"id": "42", "data": "..."}`|
|Reconnect delay|`yield {"retry": 3000, "data": "..."}`|
|Detect disconnect|`await request.is_disconnected()`|
|Resume after reconnect|`last_event_id: str = Header(None)`|
|Auth|`token: str = Query(...)`|

---
## Tags

#fastapi #python #sse #server-sent-events #streaming #realtime #backend