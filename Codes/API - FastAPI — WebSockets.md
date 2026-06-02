## What & When

**WebSockets** provide a persistent, full-duplex connection between client and server — both sides can send messages at any time without the client polling. Use WebSockets when:

- Real-time updates are needed (chat, notifications, live dashboards)
- Low-latency bi-directional communication is required
- Streaming data continuously to the client (logs, prices, progress)

For simple one-way server pushes, **Server-Sent Events (SSE)** may be enough. For request/response patterns, stick with REST.

---

## Setup

No extra dependencies — WebSocket support is built into FastAPI via Starlette.

```python
from fastapi import FastAPI, WebSocket

app = FastAPI()
```

---

## Basic WebSocket Endpoint

```python
@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await websocket.accept()            # complete the handshake

    try:
        while True:
            data = await websocket.receive_text()
            await websocket.send_text(f"Echo: {data}")
    except WebSocketDisconnect:
        print("Client disconnected")
```

### Connecting from the Browser

```javascript
const ws = new WebSocket("ws://localhost:8000/ws");

ws.onopen    = () => ws.send("Hello server");
ws.onmessage = (event) => console.log(event.data);
ws.onclose   = () => console.log("Disconnected");
ws.onerror   = (err) => console.error(err);
```

---

## Send & Receive Methods

```python
# Receiving
data = await websocket.receive_text()    # string
data = await websocket.receive_bytes()   # binary / file
data = await websocket.receive_json()    # parsed JSON → dict

# Sending
await websocket.send_text("hello")
await websocket.send_bytes(b"\x00\x01")
await websocket.send_json({"event": "update", "payload": {...}})

# Closing
await websocket.close(code=1000)         # 1000 = normal closure
```

---

## Handling Disconnection

```python
from fastapi import WebSocket, WebSocketDisconnect

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await websocket.accept()
    try:
        while True:
            data = await websocket.receive_text()
            await websocket.send_text(f"You said: {data}")
    except WebSocketDisconnect as e:
        print(f"Disconnected with code: {e.code}")
        # cleanup — remove from active connections, notify others, etc.
```

> [!warning] Always wrap the loop in `try/except WebSocketDisconnect` If the client closes the tab or loses connection, FastAPI raises `WebSocketDisconnect`. Without catching it, the server logs an unhandled exception for every normal disconnection.

---

## Path & Query Parameters

WebSocket endpoints support path and query parameters exactly like HTTP routes.

```python
from fastapi import WebSocket, Query

@app.websocket("/ws/{room_id}")
async def room_websocket(
    websocket: WebSocket,
    room_id: str,
    token: str = Query(...),            # ?token=abc
):
    await websocket.accept()
    print(f"Client joined room {room_id} with token {token}")
    ...
```

```javascript
const ws = new WebSocket("ws://localhost:8000/ws/general?token=abc123");
```

---

## Authentication via Query Parameter

The WebSocket handshake is an HTTP `GET` — standard `Authorization` headers can't be set from the browser's `WebSocket` API. Pass the JWT as a query param instead.

```python
from fastapi import WebSocket, Query, HTTPException
from core.dependencies import decode_jwt_token

@app.websocket("/ws")
async def authenticated_ws(
    websocket: WebSocket,
    token: str = Query(...),
):
    user = decode_jwt_token(token)
    if not user:
        await websocket.close(code=1008)   # 1008 = policy violation
        return

    await websocket.accept()
    try:
        while True:
            data = await websocket.receive_json()
            await websocket.send_json({"user": user.email, "data": data})
    except WebSocketDisconnect:
        pass
```

> [!tip] Token in query string vs header Browsers cannot set custom headers on WebSocket connections. Passing the JWT as `?token=` is the standard workaround. On mobile/native clients where you control the headers, use the `Sec-WebSocket-Protocol` header trick or a dedicated handshake message instead.

---

## Connection Manager — Broadcasting

The most common pattern: manage a pool of active connections and broadcast messages to all of them.

```python
from fastapi import FastAPI, WebSocket, WebSocketDisconnect

class ConnectionManager:
    def __init__(self):
        self.active: list[WebSocket] = []

    async def connect(self, websocket: WebSocket):
        await websocket.accept()
        self.active.append(websocket)

    def disconnect(self, websocket: WebSocket):
        self.active.remove(websocket)

    async def broadcast(self, message: str):
        for connection in self.active:
            await connection.send_text(message)

    async def send_personal(self, message: str, websocket: WebSocket):
        await websocket.send_text(message)


manager = ConnectionManager()

@app.websocket("/ws/chat")
async def chat(websocket: WebSocket):
    await manager.connect(websocket)
    try:
        while True:
            data = await websocket.receive_text()
            await manager.broadcast(f"Message: {data}")
    except WebSocketDisconnect:
        manager.disconnect(websocket)
        await manager.broadcast("A user left the chat")
```

---

## Room-Based Broadcasting

Extend the manager to support named rooms.

```python
from collections import defaultdict

class RoomManager:
    def __init__(self):
        self.rooms: dict[str, list[WebSocket]] = defaultdict(list)

    async def connect(self, room: str, websocket: WebSocket):
        await websocket.accept()
        self.rooms[room].append(websocket)

    def disconnect(self, room: str, websocket: WebSocket):
        self.rooms[room].remove(websocket)
        if not self.rooms[room]:
            del self.rooms[room]

    async def broadcast_to_room(self, room: str, message: str, exclude: WebSocket = None):
        for ws in self.rooms.get(room, []):
            if ws != exclude:
                await ws.send_text(message)


manager = RoomManager()

@app.websocket("/ws/{room_id}")
async def room_chat(websocket: WebSocket, room_id: str):
    await manager.connect(room_id, websocket)
    try:
        while True:
            data = await websocket.receive_text()
            await manager.broadcast_to_room(room_id, data, exclude=websocket)
    except WebSocketDisconnect:
        manager.disconnect(room_id, websocket)
```

---

## Structured Messages with Pydantic

Use Pydantic to validate WebSocket message payloads.

```python
from pydantic import BaseModel, ValidationError
from typing import Literal

class ChatMessage(BaseModel):
    type: Literal["message", "typing", "leave"]
    content: str = ""
    room: str

@app.websocket("/ws/chat")
async def chat(websocket: WebSocket):
    await websocket.accept()
    try:
        while True:
            raw = await websocket.receive_json()
            try:
                msg = ChatMessage(**raw)
            except ValidationError as e:
                await websocket.send_json({"error": e.errors()})
                continue

            if msg.type == "message":
                await manager.broadcast(msg.content)
            elif msg.type == "typing":
                await manager.broadcast(f"{msg.room}: someone is typing...")
    except WebSocketDisconnect:
        pass
```

---

## Server-Side Push — Streaming Data

Push data to the client continuously without waiting for a message.

```python
import asyncio

@app.websocket("/ws/live-prices")
async def live_prices(websocket: WebSocket):
    await websocket.accept()
    try:
        while True:
            price = await fetch_latest_price()   # your data source
            await websocket.send_json({"price": price, "ts": datetime.utcnow().isoformat()})
            await asyncio.sleep(1)               # push every second
    except WebSocketDisconnect:
        pass
```

---

## WebSockets in a Router

```python
# routers/ws.py
from fastapi import APIRouter, WebSocket, WebSocketDisconnect

router = APIRouter(prefix="/ws", tags=["websockets"])

@router.websocket("/chat")
async def chat(websocket: WebSocket): ...

@router.websocket("/notifications")
async def notifications(websocket: WebSocket): ...
```

```python
# main.py
app.include_router(ws.router)
```

---

## WebSocket Close Codes

|Code|Meaning|
|---|---|
|`1000`|Normal closure|
|`1001`|Going away (browser tab closed)|
|`1003`|Unsupported data type|
|`1008`|Policy violation (auth failure)|
|`1011`|Internal server error|
|`4000–4999`|Application-defined custom codes|

---

## REST vs WebSocket — When to Use Which

|Scenario|Use|
|---|---|
|CRUD operations|REST|
|Fetching data on demand|REST|
|One-way periodic updates|SSE or polling|
|Real-time chat|WebSocket|
|Live dashboard / prices|WebSocket|
|Collaborative editing|WebSocket|
|File upload progress|WebSocket or SSE|
|Long-running job status|WebSocket or SSE|

---

## Quick Reference

|Task|Code|
|---|---|
|Accept connection|`await websocket.accept()`|
|Receive text|`await websocket.receive_text()`|
|Receive JSON|`await websocket.receive_json()`|
|Send text|`await websocket.send_text("...")`|
|Send JSON|`await websocket.send_json({...})`|
|Detect disconnect|`except WebSocketDisconnect`|
|Close from server|`await websocket.close(code=1000)`|
|Auth via query|`token: str = Query(...)`|

---
## Related Notes

- [[Python — websockets Package]]
## Tags

#fastapi #python #websockets #realtime #chat #backend #async