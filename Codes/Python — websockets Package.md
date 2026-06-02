## What & When

The `websockets` package is a low-level, asyncio-based WebSocket library for Python. While FastAPI handles WebSocket **servers** via Starlette, `websockets` is the go-to for:

- **WebSocket clients** — connecting to external WS services from Python
- **Calling your FastAPI WS endpoints** from scripts, workers, or other services
- **Integration testing** FastAPI WebSocket routes
- **Standalone WS servers** outside of FastAPI
- Services that consume live feeds (crypto prices, trading, IoT, etc.)

```bash
pip install websockets
```

---

## FastAPI WebSocket vs `websockets` Package

||FastAPI WebSocket|`websockets` package|
|---|---|---|
|Role|Server-side endpoint|Client or standalone server|
|Integration|Built into FastAPI/Starlette|Standalone library|
|Use case|Handling browser/client connections|Connecting **to** a WS service|
|Auth, DI, routing|✅ Full FastAPI features|❌ Manual|

---

## Basic Client — Connect & Send

```python
import asyncio
import websockets

async def main():
    uri = "ws://localhost:8000/ws"

    async with websockets.connect(uri) as ws:
        await ws.send("Hello server")
        response = await ws.recv()
        print(f"Received: {response}")

asyncio.run(main())
```

---

## Sending & Receiving

```python
async with websockets.connect(uri) as ws:

    # Send
    await ws.send("plain text message")
    await ws.send(b"\x00\x01\x02")          # binary

    # Receive
    message = await ws.recv()               # blocks until a message arrives
    print(type(message))                    # str or bytes depending on server

    # Send JSON
    import json
    await ws.send(json.dumps({"action": "subscribe", "channel": "prices"}))

    # Receive JSON
    raw = await ws.recv()
    data = json.loads(raw)
```

---

## Continuous Listener

```python
import asyncio, json, websockets

async def listen():
    uri = "wss://stream.example.com/prices"

    async with websockets.connect(uri) as ws:
        await ws.send(json.dumps({"subscribe": "BTC-USD"}))

        async for message in ws:            # iterate indefinitely
            data = json.loads(message)
            print(f"Price: {data['price']}")

asyncio.run(listen())
```

> [!info] `async for message in ws` Iterating over the WebSocket connection yields messages as they arrive and exits cleanly when the connection closes.

---

## Sending Headers & Auth

```python
import websockets

async def connect_authenticated():
    headers = {
        "Authorization": "Bearer eyJhbGciOiJIUzI1NiJ9...",
        "X-Client-ID":   "service-worker-01",
    }

    # websockets package CAN set custom headers — unlike browser EventSource/WebSocket
    async with websockets.connect("wss://api.myapp.com/ws", additional_headers=headers) as ws:
        ...
```

> [!tip] Python clients can set headers — browsers cannot The `websockets` package sends headers during the HTTP upgrade handshake. This means Python clients can use `Authorization: Bearer` properly, unlike browser `WebSocket` or `EventSource` which cannot set custom headers.

---

## Connecting to a FastAPI Endpoint (Testing / Scripts)

```python
import asyncio, json, websockets
from fastapi.testclient import TestClient
from main import app

# Option 1 — httpx + anyio for async test
async def test_ws_chat():
    uri = "ws://localhost:8000/ws/chat?token=test-token"

    async with websockets.connect(uri) as ws:
        await ws.send("Hello")
        reply = await ws.recv()
        assert reply == "Echo: Hello"

asyncio.run(test_ws_chat())


# Option 2 — FastAPI TestClient (synchronous, no running server needed)
from fastapi.testclient import TestClient

def test_ws_with_test_client():
    client = TestClient(app)
    with client.websocket_connect("/ws") as ws:
        ws.send_text("Hello")
        data = ws.receive_text()
        assert data == "Echo: Hello"
```

---

## Reconnection with Backoff

The `websockets` package does not reconnect automatically. Implement it with exponential backoff:

```python
import asyncio, websockets, json
from websockets.exceptions import ConnectionClosed

async def reliable_listener(uri: str):
    delay = 1                               # initial retry delay in seconds

    while True:
        try:
            async with websockets.connect(uri) as ws:
                delay = 1                   # reset on successful connection
                print(f"Connected to {uri}")

                async for message in ws:
                    data = json.loads(message)
                    await handle_message(data)

        except ConnectionClosed as e:
            print(f"Connection closed: {e.code} {e.reason}")
        except OSError as e:
            print(f"Connection failed: {e}")

        print(f"Reconnecting in {delay}s...")
        await asyncio.sleep(delay)
        delay = min(delay * 2, 60)          # cap at 60 seconds

asyncio.run(reliable_listener("wss://stream.example.com/feed"))
```

---

## Ping / Pong — Keepalive

WebSocket connections drop silently on idle networks. `websockets` sends pings automatically by default — configure or disable as needed.

```python
async with websockets.connect(
    uri,
    ping_interval=20,       # send ping every 20 seconds (default)
    ping_timeout=10,        # close if pong not received within 10s (default)
    close_timeout=10,       # wait up to 10s for clean close handshake
) as ws:
    ...

# Disable keepalive (e.g. server handles it)
async with websockets.connect(uri, ping_interval=None) as ws:
    ...
```

---

## Handling Exceptions

```python
from websockets.exceptions import (
    ConnectionClosed,           # clean or unclean close
    ConnectionClosedOK,         # normal closure (code 1000)
    ConnectionClosedError,      # abnormal closure
    InvalidHandshake,           # server rejected the upgrade
    InvalidURI,                 # malformed URI
)

async def safe_connect(uri: str):
    try:
        async with websockets.connect(uri) as ws:
            async for message in ws:
                process(message)

    except ConnectionClosedOK:
        print("Server closed the connection cleanly")

    except ConnectionClosedError as e:
        print(f"Connection dropped: code={e.code} reason={e.reason}")

    except InvalidHandshake as e:
        print(f"Handshake failed — check URL and auth: {e}")

    except OSError as e:
        print(f"Network error: {e}")
```

---

## Bi-directional — Send and Receive Concurrently

Use `asyncio.gather` to send and receive on the same connection at the same time.

```python
import asyncio, websockets, json

async def sender(ws):
    for i in range(5):
        await ws.send(json.dumps({"seq": i, "msg": f"message {i}"}))
        await asyncio.sleep(1)
    await ws.close()

async def receiver(ws):
    async for message in ws:
        print(f"Got: {message}")

async def main():
    async with websockets.connect("ws://localhost:8000/ws") as ws:
        await asyncio.gather(sender(ws), receiver(ws))

asyncio.run(main())
```

---

## Standalone WebSocket Server (Without FastAPI)

When you need a lightweight WS server with no HTTP overhead.

```python
import asyncio, json, websockets

connected: set[websockets.WebSocketServerProtocol] = set()

async def handler(ws: websockets.WebSocketServerProtocol):
    connected.add(ws)
    try:
        async for message in ws:
            data = json.loads(message)
            # broadcast to all connected clients
            broadcast = json.dumps({"from": "server", "data": data})
            websockets.broadcast(connected, broadcast)
    finally:
        connected.remove(ws)

async def main():
    async with websockets.serve(handler, "localhost", 8765):
        print("Server running on ws://localhost:8765")
        await asyncio.Future()              # run forever

asyncio.run(main())
```

---

## Consuming an External Feed — Real-World Example

Connecting to a public WebSocket feed and forwarding events to FastAPI clients via the `EventBus` pattern from [[FastAPI - Server-Sent Events (SSE)]].

```python
import asyncio, json, websockets
from bus import event_bus                   # your EventBus instance

BINANCE_WS = "wss://stream.binance.com:9443/ws/btcusdt@trade"

async def binance_feed():
    async with websockets.connect(BINANCE_WS) as ws:
        async for message in ws:
            trade = json.loads(message)
            await event_bus.publish({
                "event": "trade",
                "data":  json.dumps({
                    "symbol": trade["s"],
                    "price":  trade["p"],
                    "qty":    trade["q"],
                }),
            })

# Run alongside FastAPI using lifespan
from contextlib import asynccontextmanager
from fastapi import FastAPI

@asynccontextmanager
async def lifespan(app: FastAPI):
    task = asyncio.create_task(binance_feed())
    yield
    task.cancel()

app = FastAPI(lifespan=lifespan)
```

---

## Quick Reference

|Task|Code|
|---|---|
|Connect|`async with websockets.connect(uri) as ws`|
|Send text|`await ws.send("text")`|
|Send JSON|`await ws.send(json.dumps({...}))`|
|Receive|`await ws.recv()`|
|Listen loop|`async for message in ws`|
|Send with headers|`websockets.connect(uri, additional_headers={...})`|
|Disable ping|`websockets.connect(uri, ping_interval=None)`|
|Broadcast (server)|`websockets.broadcast(connections, message)`|
|Standalone server|`websockets.serve(handler, host, port)`|
|Reconnect|`while True` loop with `asyncio.sleep(delay)`|

---

## Tags

#python #websockets #client #asyncio #realtime #backend #streaming