
- **`exclude_unset=True`** on the PATCH handler is highlighted with an Obsidian callout block — it's genuinely the most common mistake people make when implementing PATCH; without it, unset fields overwrite existing data with defaults.
- **HEAD & OPTIONS** are included since they're part of the HTTP spec and FastAPI supports them natively, even if they're less commonly hand-written.
- The two cheat sheet tables at the bottom (status codes + method properties) are good quick-reference material in a graph context.
- The in-memory `db` dict keeps the snippets self-contained — easy to swap for a real DB session later.

## Setup

```python
from fastapi import FastAPI, HTTPException, status
from pydantic import BaseModel
from typing import Optional

app = FastAPI()
```

---

## Data Models

```python
class Item(BaseModel):
    name: str
    description: Optional[str] = None
    price: float
    in_stock: bool = True

class ItemUpdate(BaseModel):
    name: Optional[str] = None
    description: Optional[str] = None
    price: Optional[float] = None
    in_stock: Optional[bool] = None

# In-memory store for demo purposes
db: dict[int, Item] = {}
```

---

## GET — Retrieve a Resource

> Read-only. Must not modify server state. Safe & idempotent.

```python
# GET all items
@app.get("/items", response_model=list[Item])
async def get_items():
    return list(db.values())


# GET a single item by ID
@app.get("/items/{item_id}", response_model=Item)
async def get_item(item_id: int):
    if item_id not in db:
        raise HTTPException(status_code=404, detail="Item not found")
    return db[item_id]
```

---

## POST — Create a Resource

> Creates a new resource. Not idempotent — calling it twice creates two resources.

```python
@app.post("/items", response_model=Item, status_code=status.HTTP_201_CREATED)
async def create_item(item: Item):
    item_id = len(db) + 1
    db[item_id] = item
    return item
```

---

## PUT — Full Replacement

> Replaces the entire resource. Idempotent — same call, same result. All fields must be provided; missing fields revert to defaults.

```python
@app.put("/items/{item_id}", response_model=Item)
async def replace_item(item_id: int, item: Item):
    if item_id not in db:
        raise HTTPException(status_code=404, detail="Item not found")
    db[item_id] = item
    return item
```

---

## PATCH — Partial Update

> Updates only the fields provided. Non-provided fields remain unchanged.

```python
@app.patch("/items/{item_id}", response_model=Item)
async def update_item(item_id: int, updates: ItemUpdate):
    if item_id not in db:
        raise HTTPException(status_code=404, detail="Item not found")

    stored = db[item_id].model_dump()
    patch_data = updates.model_dump(exclude_unset=True)  # only changed fields
    updated = {**stored, **patch_data}

    db[item_id] = Item(**updated)
    return db[item_id]
```

> [!tip] `exclude_unset=True` This is the key to correct PATCH behaviour. Pydantic only includes fields that were explicitly sent in the request — not defaults.

---

## DELETE — Remove a Resource

> Removes the resource. Idempotent — deleting a non-existent resource should not error (or return 404 consistently).

```python
@app.delete("/items/{item_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_item(item_id: int):
    if item_id not in db:
        raise HTTPException(status_code=404, detail="Item not found")
    del db[item_id]
```

---

## HEAD — Metadata Only

> Same as GET but returns headers only — no body. Useful for checking existence or cache validity.

```python
from fastapi import Response

@app.head("/items/{item_id}")
async def head_item(item_id: int, response: Response):
    if item_id not in db:
        raise HTTPException(status_code=404, detail="Item not found")
    response.headers["X-Item-Name"] = db[item_id].name
    # No body returned
```

---

## OPTIONS — Allowed Methods

> Describes what HTTP methods are available for a given endpoint. Browsers use this for CORS preflight checks.

```python
@app.options("/items")
async def options_items(response: Response):
    response.headers["Allow"] = "GET, POST, OPTIONS"
    return {}
```

---

## HTTP Status Code Reference

|Code|Meaning|Typical Use|
|---|---|---|
|`200 OK`|Success|Default for GET, PUT, PATCH|
|`201 Created`|Resource created|POST|
|`204 No Content`|Success, no body|DELETE|
|`400 Bad Request`|Invalid input|Validation failures|
|`401 Unauthorized`|Not authenticated|Missing/invalid token|
|`403 Forbidden`|Not authorised|Valid token, wrong permissions|
|`404 Not Found`|Resource missing|GET/PUT/PATCH/DELETE on unknown ID|
|`409 Conflict`|State conflict|Duplicate creation|
|`422 Unprocessable Entity`|Schema validation failed|FastAPI default for bad payloads|
|`500 Internal Server Error`|Unexpected failure|Unhandled exceptions|

---

## REST Method Cheat Sheet

|Method|Idempotent|Safe|Body|Common Status|
|---|---|---|---|---|
|GET|✅|✅|❌|200|
|POST|❌|❌|✅|201|
|PUT|✅|❌|✅|200|
|PATCH|✅*|❌|✅|200|
|DELETE|✅|❌|❌|204|
|HEAD|✅|✅|❌|200|
|OPTIONS|✅|✅|❌|200|

> *PATCH is idempotent if the operation is absolute (set price to 10), not relative (add 5 to price).

---

## Running the App

```bash
uvicorn main:app --reload
```

Then visit:

- **Swagger UI** → http://127.0.0.1:8000/docs
- **ReDoc** → http://127.0.0.1:8000/redoc

---

## Tags

#fastapi #python #rest #http #backend #api-design 
