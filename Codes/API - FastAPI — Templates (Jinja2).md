## What & When

FastAPI is primarily an API framework, but it supports **server-side HTML rendering** via Jinja2 templates. Use this when:

- Building a simple admin UI or dashboard without a separate frontend
- Rendering emails (HTML)
- Returning HTML pages alongside a REST API
- Prototyping before a proper frontend is built

For SPAs (React, Vue, etc.) serving static files is usually the better approach. See [[FastAPI - Sub-Applications, Mount & CORS]].

---

## Setup

```bash
pip install jinja2 python-multipart
```

```
myapp/
├── main.py
├── routers/
│   └── pages.py
└── templates/
    ├── base.html
    ├── index.html
    └── items/
        ├── list.html
        └── detail.html
```

---

## Basic Usage

```python
from fastapi import FastAPI, Request
from fastapi.templating import Jinja2Templates
from fastapi.responses import HTMLResponse

app = FastAPI()

templates = Jinja2Templates(directory="templates")

@app.get("/", response_class=HTMLResponse)
async def index(request: Request):
    return templates.TemplateResponse(
        request=request,
        name="index.html",
        context={"title": "Home", "items": ["a", "b", "c"]},
    )
```

> [!info] `request` is required in context Jinja2Templates needs the `request` object to build URLs with `url_for`. Pass it via the `request=` parameter (FastAPI 0.108+) or include it in `context={"request": request}` for older versions.

---

## Base Template & Inheritance

```html
<!-- templates/base.html -->
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>{% block title %}My App{% endblock %}</title>
    <link rel="stylesheet" href="{{ url_for('static', path='css/main.css') }}">
</head>
<body>
    <nav>
        <a href="{{ url_for('index') }}">Home</a>
        <a href="{{ url_for('list_items') }}">Items</a>
    </nav>

    <main>
        {% block content %}{% endblock %}
    </main>
</body>
</html>
```

```html
<!-- templates/items/list.html -->
{% extends "base.html" %}

{% block title %}Items{% endblock %}

{% block content %}
  <h1>All Items</h1>
  <ul>
    {% for item in items %}
      <li>
        <a href="{{ url_for('get_item', item_id=item.id) }}">
          {{ item.name }} — £{{ item.price }}
        </a>
      </li>
    {% else %}
      <p>No items found.</p>
    {% endfor %}
  </ul>
{% endblock %}
```

---

## Passing Data to Templates

```python
@app.get("/items", response_class=HTMLResponse)
async def list_items(request: Request, db: AsyncSession = Depends(get_db)):
    items = await db.execute(select(Item))
    return templates.TemplateResponse(
        request=request,
        name="items/list.html",
        context={"items": items.scalars().all()},
    )

@app.get("/items/{item_id}", response_class=HTMLResponse)
async def get_item(item_id: int, request: Request, db: AsyncSession = Depends(get_db)):
    item = await db.get(Item, item_id)
    if not item:
        raise HTTPException(status_code=404, detail="Not found")
    return templates.TemplateResponse(
        request=request,
        name="items/detail.html",
        context={"item": item},
    )
```

---

## Handling Forms

```bash
pip install python-multipart   # required for form parsing
```

```html
<!-- templates/items/create.html -->
{% extends "base.html" %}
{% block content %}
<form method="post" action="/items">
    <input type="text"   name="name"  placeholder="Name"  required>
    <input type="number" name="price" placeholder="Price" step="0.01" required>
    <button type="submit">Create</button>
    {% if error %}
      <p style="color:red">{{ error }}</p>
    {% endif %}
</form>
{% endblock %}
```

```python
from fastapi import Form
from fastapi.responses import RedirectResponse

@app.get("/items/new", response_class=HTMLResponse)
async def new_item_form(request: Request):
    return templates.TemplateResponse(request=request, name="items/create.html")

@app.post("/items")
async def create_item_form(
    request: Request,
    name: str = Form(...),
    price: float = Form(...),
    db: AsyncSession = Depends(get_db),
):
    db.add(Item(name=name, price=price))
    await db.commit()
    return RedirectResponse(url="/items", status_code=303)
```

> [!tip] `303 See Other` after POST Always redirect after a successful form POST to prevent duplicate submissions on browser refresh. This is the **POST/Redirect/GET** pattern.

---

## `url_for` — Generating URLs in Templates

`url_for` builds URLs by route name — safe against path changes.

```html
<!-- Link to a named route -->
<a href="{{ url_for('list_items') }}">Items</a>

<!-- Route with path parameter -->
<a href="{{ url_for('get_item', item_id=item.id) }}">{{ item.name }}</a>

<!-- Static file -->
<link href="{{ url_for('static', path='css/main.css') }}">
<img  src="{{ url_for('static', path='img/logo.png') }}">
```

Route names default to the function name. Set them explicitly with `name=`:

```python
@app.get("/items", name="list_items")
async def list_items_handler(...): ...
```

---

## Custom Error Pages

```python
from fastapi import Request
from fastapi.responses import HTMLResponse
from fastapi.exception_handlers import http_exception_handler
from starlette.exceptions import HTTPException as StarletteHTTPException

@app.exception_handler(StarletteHTTPException)
async def custom_http_exception_handler(request: Request, exc: StarletteHTTPException):
    if exc.status_code == 404:
        return templates.TemplateResponse(
            request=request,
            name="errors/404.html",
            status_code=404,
        )
    return await http_exception_handler(request, exc)
```

---

## Templates in a Router

Keep page routes in a dedicated router, separate from API routes.

```python
# routers/pages.py
from fastapi import APIRouter, Request, Depends
from fastapi.responses import HTMLResponse
from fastapi.templating import Jinja2Templates

router = APIRouter(tags=["pages"], include_in_schema=False)
templates = Jinja2Templates(directory="templates")

@router.get("/", response_class=HTMLResponse)
async def index(request: Request): ...

@router.get("/about", response_class=HTMLResponse)
async def about(request: Request): ...
```

```python
# main.py
app.include_router(pages.router)
```

> [!tip] `include_in_schema=False` Hides the HTML routes from the OpenAPI / Swagger docs — keeps the API docs clean when mixing HTML and JSON endpoints.

---

## Jinja2 Essentials

```jinja2
{# Comment #}

{# Variable output #}
{{ variable }}
{{ item.name | upper }}
{{ price | round(2) }}

{# Conditionals #}
{% if user.is_admin %}
  <a href="/admin">Admin Panel</a>
{% elif user.is_verified %}
  <span>Verified</span>
{% else %}
  <span>Guest</span>
{% endif %}

{# Loops #}
{% for item in items %}
  <li>{{ loop.index }}. {{ item.name }}</li>
{% endfor %}

{# Template inheritance #}
{% extends "base.html" %}
{% block content %} ... {% endblock %}

{# Include a partial #}
{% include "partials/navbar.html" %}
```

---

## Quick Reference

|Feature|How|
|---|---|
|Render template|`templates.TemplateResponse(request, name, context)`|
|Pass data|`context={"key": value}`|
|Generate URL|`{{ url_for('route_name') }}`|
|Static file URL|`{{ url_for('static', path='file.css') }}`|
|Form input|`name: str = Form(...)`|
|POST redirect|`RedirectResponse(url, status_code=303)`|
|Hide from docs|`include_in_schema=False` on router|
|Custom error page|`@app.exception_handler(StarletteHTTPException)`|

---
## Related Notes

- [[Python — Jinja2 Package]]
## Tags

#fastapi #python #templates #jinja2 #html #server-side-rendering #backend