## What is OpenAPI?

**OpenAPI** (formerly Swagger) is a language-agnostic standard for describing REST APIs. FastAPI generates a fully compliant **OpenAPI 3.x** schema automatically from your route definitions, type hints, and Pydantic models — with zero manual maintenance.

The schema powers:

- **Swagger UI** (`/docs`) — interactive API explorer
- **ReDoc** (`/redoc`) — clean reference documentation
- **Client SDK generation** — typed clients in any language
- **API gateway configuration** — AWS, Kong, Nginx
- **Contract testing** — validate requests/responses against the spec

---

## Accessing the Schema

```
GET /openapi.json   → raw OpenAPI schema (JSON)
GET /docs           → Swagger UI
GET /redoc          → ReDoc UI
```

---

## App-Level Metadata

```python
from fastapi import FastAPI

app = FastAPI(
    title="My Service API",
    summary="Short one-liner about the API",
    description="""
## My Service

Longer markdown description shown at the top of the docs.

- Supports **JWT authentication**
- Full **CRUD** for items and orders
- Real-time updates via **WebSockets**
    """,
    version="2.1.0",
    terms_of_service="https://myapp.com/terms",
    contact={
        "name":  "API Support",
        "email": "support@myapp.com",
        "url":   "https://myapp.com/support",
    },
    license_info={
        "name": "MIT",
        "url":  "https://opensource.org/licenses/MIT",
    },
)
```

---

## Server URLs

Declare multiple target environments — shown in Swagger's server dropdown.

```python
app = FastAPI(
    title="My Service",
    servers=[
        {"url": "https://api.myapp.com",         "description": "Production"},
        {"url": "https://staging.myapp.com",     "description": "Staging"},
        {"url": "http://localhost:8000",          "description": "Local dev"},
    ],
)
```

---

## Tags — Grouping & Ordering Routes

Tags group routes in Swagger UI. Declare them with metadata to control order and add descriptions.

```python
tags_metadata = [
    {
        "name":        "auth",
        "description": "Registration, login, and token management",
    },
    {
        "name":        "users",
        "description": "User profile and account management",
        "externalDocs": {
            "description": "User model reference",
            "url": "https://docs.myapp.com/users",
        },
    },
    {
        "name":        "items",
        "description": "CRUD operations for items",
    },
]

app = FastAPI(openapi_tags=tags_metadata)
```

Tags appear in Swagger in the order they are declared here — regardless of route registration order.

---

## Route-Level Documentation

```python
from fastapi import APIRouter

router = APIRouter()

@router.get(
    "/items/{item_id}",
    summary="Get a single item",                    # short title in Swagger
    description="Retrieve full details for an item by its ID. Returns 404 if not found.",
    response_description="The requested item",      # describes the 200 response
    tags=["items"],
    operation_id="get_item_by_id",                  # unique ID for client generators
    deprecated=False,                               # marks route as deprecated in docs
)
async def get_item(item_id: int): ...
```

> [!tip] Docstrings as descriptions FastAPI uses the function docstring as the route description automatically. Markdown is supported.

```python
@router.get("/items/{item_id}", summary="Get a single item")
async def get_item(item_id: int):
    """
    Retrieve full details for an item by its ID.

    - Returns **404** if the item does not exist.
    - Requires **authentication**.
    """
    ...
```

---

## Documenting Multiple Response Codes

```python
from fastapi import APIRouter
from schemas.item import ItemResponse

router = APIRouter()

@router.get(
    "/items/{item_id}",
    response_model=ItemResponse,
    responses={
        200: {"description": "Item found"},
        404: {"description": "Item not found"},
        401: {"description": "Not authenticated"},
        403: {"description": "Not authorised"},
        422: {"description": "Validation error"},
    },
)
async def get_item(item_id: int): ...
```

You can also include an example response body:

```python
responses={
    200: {
        "description": "Item found",
        "content": {
            "application/json": {
                "example": {
                    "id": 1,
                    "name": "Widget",
                    "price": 9.99,
                }
            }
        },
    },
    404: {"description": "Item not found"},
}
```

---

## Examples in Pydantic Models

Declare request body examples directly on the schema — shown in Swagger's "Try it out" panel.

```python
from pydantic import BaseModel, ConfigDict

class ItemCreate(BaseModel):
    name:  str
    price: float

    model_config = ConfigDict(
        json_schema_extra={
            "examples": [
                {
                    "name":  "Widget",
                    "price": 9.99,
                },
                {
                    "name":  "Gadget",
                    "price": 24.50,
                },
            ]
        }
    )
```

Or on individual fields with `Field(examples=[...])`:

```python
from pydantic import BaseModel, Field

class ItemCreate(BaseModel):
    name:  str   = Field(..., examples=["Widget", "Gadget"])
    price: float = Field(..., examples=[9.99, 24.50], gt=0)
```

---

## Hiding Routes from Docs

```python
# Hide a single route
@app.get("/internal/health", include_in_schema=False)
async def health(): return {"status": "ok"}

# Hide all routes in a router
router = APIRouter(include_in_schema=False)

# Hide routes registered at include time
app.include_router(internal_router, include_in_schema=False)
```

Common candidates to hide: health checks, internal admin endpoints, HTML page routes, webhook receivers.

---

## Deprecating Routes

Mark old routes as deprecated without removing them — shown with a strikethrough in Swagger UI.

```python
@app.get("/v1/items", deprecated=True, summary="List items (deprecated — use /v2/items)")
async def list_items_v1(): ...

@app.get("/v2/items", summary="List items")
async def list_items_v2(): ...
```

---

## Security Schemes in OpenAPI

Declaring a security scheme makes Swagger UI show an **Authorize** button where users can enter their token before trying endpoints.

```python
from fastapi.security import OAuth2PasswordBearer, APIKeyHeader

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/auth/login")
api_key_scheme = APIKeyHeader(name="X-API-Key")

@app.get("/secure")
async def secure_route(token: str = Depends(oauth2_scheme)):
    ...
```

FastAPI automatically adds the security scheme to the OpenAPI schema when you use these security utilities as dependencies.

---

## Customising Swagger UI

```python
from fastapi import FastAPI
from fastapi.openapi.docs import get_swagger_ui_html
from fastapi.staticfiles import StaticFiles

app = FastAPI(docs_url=None, redoc_url=None)   # disable default docs

app.mount("/static", StaticFiles(directory="static"), name="static")

@app.get("/docs", include_in_schema=False)
async def custom_swagger():
    return get_swagger_ui_html(
        openapi_url="/openapi.json",
        title="My API Docs",
        swagger_js_url="/static/swagger-ui-bundle.js",
        swagger_css_url="/static/swagger-ui.css",
        swagger_favicon_url="/static/favicon.png",
    )
```

Useful for: offline/airgapped environments, custom branding, pinning a specific Swagger UI version.

---

## Overriding the OpenAPI Schema

For full control — add custom extensions, merge external schemas, inject security definitions manually.

```python
from fastapi.openapi.utils import get_openapi

def custom_openapi():
    if app.openapi_schema:
        return app.openapi_schema          # cached — only built once

    schema = get_openapi(
        title=app.title,
        version=app.version,
        description=app.description,
        routes=app.routes,
    )

    # Add a custom extension
    schema["info"]["x-internal-id"] = "my-service-v2"

    # Inject a global security requirement
    schema["security"] = [{"BearerAuth": []}]

    app.openapi_schema = schema
    return schema

app.openapi = custom_openapi
```

---

## Using the Schema for Client Generation

The `/openapi.json` endpoint is the contract. Generate typed clients in any language with standard tooling:

```bash
# Install openapi-generator
npm install -g @openapitools/openapi-generator-cli

# Generate a TypeScript/Fetch client
openapi-generator-cli generate \
  -i http://localhost:8000/openapi.json \
  -g typescript-fetch \
  -o ./frontend/src/api

# Generate a Python client
openapi-generator-cli generate \
  -i http://localhost:8000/openapi.json \
  -g python \
  -o ./sdk/python
```

Or use the lightweight `openapi-python-client`:

```bash
pip install openapi-python-client
openapi-python-client generate --url http://localhost:8000/openapi.json
```

---

## Changing Docs & Schema URLs

```python
app = FastAPI(
    docs_url="/api/docs",           # default: /docs
    redoc_url="/api/redoc",         # default: /redoc
    openapi_url="/api/openapi.json" # default: /openapi.json
)

# Disable docs entirely (e.g. in production)
app = FastAPI(docs_url=None, redoc_url=None)
```

> [!tip] Disable docs in production Exposing Swagger UI in production leaks your API surface to attackers. Consider disabling it or putting it behind auth/IP restriction.

---

## Quick Reference

|Feature|How|
|---|---|
|App metadata|`FastAPI(title, description, version, contact, ...)`|
|Server list|`FastAPI(servers=[...])`|
|Tag order & descriptions|`FastAPI(openapi_tags=[...])`|
|Route summary|`@router.get(summary="...")` or first line of docstring|
|Route description|`@router.get(description="...")` or full docstring|
|Multiple responses|`@router.get(responses={404: {...}})`|
|Request body example|`model_config = ConfigDict(json_schema_extra={...})`|
|Hide route|`include_in_schema=False`|
|Deprecate route|`deprecated=True`|
|Custom docs URL|`FastAPI(docs_url="/api/docs")`|
|Disable docs|`FastAPI(docs_url=None, redoc_url=None)`|
|Override schema|`app.openapi = custom_openapi`|

---

## Tags

#fastapi #python #openapi #swagger #redoc #api-design #documentation #backend