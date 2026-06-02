## What & When

**Flask** is a **WSGI microframework** — one module can define routes, return JSON or HTML, and grow via **blueprints** and extensions. Default choice in this vault for **small server-rendered apps**, internal tools, and learning HTTP without Django's full stack.

Use Flask when:

- You need **HTML pages** with [[Python — Jinja2 Package]] (cheatsheet viewer, admin-lite dashboard)
- The app is **small** — a few routes, no built-in admin/ORM requirement
- You want **full control** over project layout (vs Django conventions)
- You integrate **SQLAlchemy** manually — [[ORM - SQLAlchemy]]
- Sync WSGI + **Gunicorn** is enough (no WebSocket-first design)

For new **JSON APIs**, prefer [[API - FastAPI]]. Overview: [[Web]].

```bash
pip install flask
# Production: pip install gunicorn
# DB: pip install flask-sqlalchemy  # or plain SQLAlchemy
# Env: pip install python-dotenv
```

---

## Flask vs Related Tools

| Need | Use | Notes |
| --- | --- | --- |
| Minimal HTML + routes | **Flask** | `app.route`, Jinja2 |
| Batteries-included monolith | [[Web — Django]] | Admin, ORM, migrations |
| Async API + OpenAPI | [[API - FastAPI]] | ASGI, Pydantic |
| Legacy event-loop server | [[Web — Tornado]] | Prefer FastAPI for new work |
| Templates only | [[Python — Jinja2 Package]] | Shared with Flask/FastAPI |
| Sync HTTP client | [[Python — requests Package]] | Inside view functions |

---

## Minimal Application

```python
# app.py
from flask import Flask, jsonify, render_template

app = Flask(__name__)

@app.get("/")
def index():
    return render_template("index.html", title="Home")

@app.get("/api/health")
def health():
    return jsonify({"status": "ok"})

if __name__ == "__main__":
    app.run(debug=True, port=5000)
```

```bash
flask --app app run --debug
# or: python app.py
```

> [!tip] `FLASK_APP` Set `export FLASK_APP=app:app` (module:variable) for the `flask` CLI.

---

## Routes & Request Data

```python
from flask import Flask, request, abort

app = Flask(__name__)

@app.get("/items")
def list_items():
    page = request.args.get("page", 1, type=int)
    limit = request.args.get("limit", 20, type=int)
    return {"page": page, "limit": limit}

@app.get("/items/<int:item_id>")
def get_item(item_id: int):
    item = find_item(item_id)  # your service layer
    if not item:
        abort(404)
    return item

@app.post("/items")
def create_item():
    data = request.get_json(force=True)
    if not data or "name" not in data:
        abort(400, description="name required")
    return save_item(data), 201
```

| Source | Access |
| --- | --- |
| Query string | `request.args` |
| JSON body | `request.get_json()` |
| Form | `request.form` |
| Path | view function args / `request.view_args` |
| Files | `request.files` |

---

## Blueprints (Modular Apps)

```python
# routes/items.py
from flask import Blueprint, jsonify

items_bp = Blueprint("items", __name__, url_prefix="/items")

@items_bp.get("/")
def list_items():
    return jsonify([])

# app.py
from flask import Flask
from routes.items import items_bp

app = Flask(__name__)
app.register_blueprint(items_bp)
```

Same pattern as [[API - FastAPI — Routers & Modular Applications]] — thin routes, logic in services.

---

## Jinja2 Templates

```python
from flask import Flask, render_template
from pathlib import Path

app = Flask(__name__)
CHEATSHEETS = Path(__file__).parent / "cheatsheets"

@app.get("/")
def index():
    sheets = sorted(p.name for p in CHEATSHEETS.glob("*.html"))
    return render_template("index.html", sheets=sheets)

@app.get("/view/<filename>")
def view(filename: str):
    if ".." in filename or not filename.endswith(".html"):
        abort(404)
    path = CHEATSHEETS / filename
    if not path.is_file():
        abort(404)
    return path.read_text(encoding="utf-8"), 200, {"Content-Type": "text/html; charset=utf-8"}
```

```html
<!-- templates/index.html -->
{% for name in sheets %}
  <li><a href="{{ url_for('view', filename=name) }}">{{ name }}</a></li>
{% endfor %}
```

See [[Python — Jinja2 Package]], [[API - FastAPI — Templates (Jinja2)]] for shared template patterns.

---

## Configuration

```python
import os
from flask import Flask

app = Flask(__name__)
app.config.from_mapping(
    SECRET_KEY=os.environ.get("SECRET_KEY", "dev-only-change-me"),
    DATABASE_URL=os.environ.get("DATABASE_URL"),
    DEBUG=os.environ.get("FLASK_DEBUG", "0") == "1",
)
```

Use [[Python — python-dotenv]] locally; never commit secrets. For typed settings in larger apps, consider [[Python — Pydantic]] in a service layer even when routes stay Flask.

---

## SQLAlchemy Integration

```python
from flask import Flask, g
from sqlalchemy import create_engine, text
from sqlalchemy.orm import sessionmaker

app = Flask(__name__)
engine = create_engine(app.config["DATABASE_URL"])
SessionLocal = sessionmaker(bind=engine)

@app.before_request
def open_session():
    g.db = SessionLocal()

@app.teardown_request
def close_session(exc):
    db = g.pop("db", None)
    if db:
        db.close()

@app.get("/users/count")
def user_count():
    n = g.db.execute(text("SELECT COUNT(*) FROM users")).scalar()
    return {"count": n}
```

Full ORM patterns: [[ORM - Setup]], [[ORM - CRUD]], [[ORM - Queries]]. Django alternative: [[Web — Django]].

---

## Error Handling

```python
from flask import Flask, jsonify

app = Flask(__name__)

@app.errorhandler(404)
def not_found(e):
    return jsonify(error="not found"), 404

@app.errorhandler(500)
def server_error(e):
    return jsonify(error="internal error"), 500
```

---

## Testing

```python
# test_app.py
import pytest
from app import app

@pytest.fixture
def client():
    app.config["TESTING"] = True
    with app.test_client() as c:
        yield c

def test_health(client):
    r = client.get("/api/health")
    assert r.status_code == 200
    assert r.get_json()["status"] == "ok"
```

See [[Unit Testing - pytest]].

---

## Production (Gunicorn)

```bash
gunicorn -w 4 -b 0.0.0.0:8000 "app:app"
```

| Flag | Meaning |
| --- | --- |
| `-w 4` | Worker processes (often `2 × CPU + 1`) |
| `-b` | Bind host:port |
| `--timeout` | Kill slow workers (default 30s) |

Pair with reverse proxy (nginx, Caddy). For async/WebSocket APIs, use [[API - FastAPI]] + Uvicorn instead.

---

## Flask + FastAPI Coexistence

Mount FastAPI as WSGI/ASGI sub-app or run separate services:

- **Same repo, two processes** — Flask for HTML admin, FastAPI for public API (common microservice split)
- **Migrate gradually** — new endpoints on FastAPI; legacy Flask routes until retired

See [[API - FastAPI — Sub-Applications, Mount & CORS]].

---

## Backend Pattern Summary

| Layer | Flask role |
| --- | --- |
| Route | Parse request, call service, return response |
| Service | Business logic (framework-agnostic) |
| Repository | SQLAlchemy / file I/O |
| Template | Jinja2 for HTML only |

Keep routes thin — same rule as [[Python Development]] layered architecture.

---

## Quick Reference

| Task | Snippet |
| --- | --- |
| Create app | `app = Flask(__name__)` |
| GET route | `@app.get("/path")` |
| JSON response | `return jsonify({...})` or `return {...}` (Flask 2.2+) |
| Template | `render_template("x.html", **ctx)` |
| Blueprint | `Blueprint(...); app.register_blueprint(bp)` |
| Run dev | `flask --app app run --debug` |
| Run prod | `gunicorn "app:app"` |
| Test | `app.test_client()` |

---

## Related Notes

- [[Web]]
- [[Web — Django]]
- [[Web — Tornado]]
- [[API - FastAPI]]
- [[Python — Jinja2 Package]]
- [[ORM - SQLAlchemy]]
- [[Python Development]]

---

## Tags

#web #flask #wsgi #jinja2 #python #backend #templates #gunicorn
