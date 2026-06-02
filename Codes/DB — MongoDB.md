## What & When

**MongoDB** is a **document database** — JSON-like **BSON** documents in **collections**, flexible schema, horizontal scaling via sharding. Use when data is **nested**, **evolving**, or **read-heavy document-shaped** — not when you need complex multi-row transactions across normalized tables ([[ORM - SQLAlchemy]] often wins there).

Use MongoDB when:

- **Catalogs**, content, user preferences, event payloads
- **Rapid schema change** without migrations every week
- **Embedded documents** (address inside user) fit naturally
- Prototyping alongside [[API - FastAPI]] with Pydantic models

```bash
pip install pymongo
# Motor for async: pip install motor
# Local: docker run -d -p 27017:27017 mongo:7
```

Overview: [[DB]].

---

## MongoDB vs Related Tools

| Need | Use | Notes |
| --- | --- | --- |
| Flexible JSON docs | **MongoDB** | Collections |
| ACID relational | PostgreSQL + [[ORM - SQLAlchemy]] | Joins, reports |
| Cache | [[DB — Redis]] | Not primary store |
| Graph paths | [[DB — Neo4j]] | Relationships first |
| Full-text + logs | [[DB — ELK]] | Elasticsearch |

---

## Document Model

```javascript
// users collection
{
  "_id": ObjectId("..."),
  "email": "ada@example.com",
  "profile": { "name": "Ada", "tags": ["pro"] },
  "created_at": ISODate("2025-01-15")
}
```

`_id` auto-generated if omitted.

---

## pymongo — Sync

```python
from pymongo import MongoClient

client = MongoClient("mongodb://localhost:27017")
db = client["myapp"]
users = db["users"]

users.insert_one({"email": "a@b.com", "profile": {"name": "Ada"}})
doc = users.find_one({"email": "a@b.com"})
users.update_one({"email": "a@b.com"}, {"$set": {"profile.plan": "pro"}})
users.create_index("email", unique=True)
```

Query operators: `$gt`, `$in`, `$elemMatch`, `$regex`.

---

## Motor — Async (FastAPI)

```python
from motor.motor_asyncio import AsyncIOMotorClient

client = AsyncIOMotorClient("mongodb://localhost:27017")
db = client.myapp

async def get_user(email: str):
    return await db.users.find_one({"email": email})
```

Wire via [[API - FastAPI — Dependency Injection & User Management]]:

```python
async def get_db():
    yield client.myapp
```

---

## Pydantic Integration

```python
from pydantic import BaseModel, Field
from bson import ObjectId

class PyObjectId(str):
    @classmethod
    def __get_validators__(cls):
        yield cls.validate
    @classmethod
    def validate(cls, v):
        if not ObjectId.is_valid(v):
            raise ValueError("Invalid ObjectId")
        return str(v)

class UserOut(BaseModel):
    id: PyObjectId = Field(alias="_id")
    email: str
    model_config = {"populate_by_name": True}
```

Share shapes with [[Python — Pydantic]] API models where sensible.

---

## Aggregation Pipeline

```python
pipeline = [
    {"$match": {"profile.plan": "pro"}},
    {"$group": {"_id": "$profile.tags", "count": {"$sum": 1}}},
    {"$sort": {"count": -1}},
]
list(users.aggregate(pipeline))
```

SQL-analog: `$match` → WHERE, `$group` → GROUP BY, `$lookup` → JOIN (limited).

---

## Docker Compose

```yaml
services:
  mongo:
    image: mongo:7
    ports:
      - "27017:27017"
    volumes:
      - mongodata:/data/db
volumes:
  mongodata:
```

Connection: `mongodb://mongo:27017` from other Compose services.

---

## Design Guidelines

| Do | Avoid |
| --- | --- |
| Index query fields | Unindexed regex on huge collections |
| Embed 1:few data | Unbounded arrays that grow forever |
| Use transactions when needed (multi-doc) | Treating Mongo as schemaless junk drawer |
| TTL indexes for ephemeral docs | Manual cleanup cron for expiring events |

---

## Quick Reference

| Task | API |
| --- | --- |
| Insert | `insert_one` / `insert_many` |
| Find | `find`, `find_one` |
| Update | `update_one` with `$set`, `$inc` |
| Delete | `delete_one` |
| Index | `create_index` |
| Async client | `motor.motor_asyncio.AsyncIOMotorClient` |

---

## Related Notes

- [[DB]]
- [[ORM - SQLAlchemy]]
- [[Python — Pydantic]]
- [[API - FastAPI]]
- [[Commands/CLI — Docker & Compose]]

---

## Tags

#database #mongodb #document #nosql #python #pymongo #motor #fastapi
