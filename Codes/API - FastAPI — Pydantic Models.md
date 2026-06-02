 
Pydantic is FastAPI's validation backbone. Every request body, response model, and config object is a Pydantic model. FastAPI uses them to validate incoming data, serialise outgoing data, and generate the OpenAPI schema — all from the same class definition.

---

## Basic Model

```python
from pydantic import BaseModel
from typing import Optional

class Item(BaseModel):
    name: str
    description: Optional[str] = None
    price: float
    in_stock: bool = True
```

---

## Field — Constraints & Metadata

`Field()` adds validation rules and enriches the OpenAPI schema (shown in Swagger).

```python
from pydantic import BaseModel, Field

class Item(BaseModel):
    name: str        = Field(..., min_length=2, max_length=100, description="Product name")
    price: float     = Field(..., gt=0, description="Must be greater than zero")
    discount: float  = Field(default=0.0, ge=0.0, le=1.0, description="0.0 – 1.0")
    sku: str         = Field(..., pattern=r"^[A-Z]{3}-\d{4}$", description="e.g. ABC-1234")
    tags: list[str]  = Field(default_factory=list)
```

> [!info] `...` means the field is **required** (no default).

|Constraint|Meaning|
|---|---|
|`gt` / `lt`|Greater / less than (exclusive)|
|`ge` / `le`|Greater / less than or equal|
|`min_length` / `max_length`|String or list length bounds|
|`pattern`|Regex match|
|`default_factory`|Callable for mutable defaults|

---

## Nested Models

Models can be composed — FastAPI validates and documents the full nested structure.

```python
class Address(BaseModel):
    street: str
    city: str
    postcode: str

class Supplier(BaseModel):
    name: str
    address: Address        # nested model
    contacts: list[str]     # list of primitives
    items: list[Item]       # list of nested models
```

---

## Model Inheritance — Base / Create / Response Pattern

This is the most common FastAPI pattern for keeping models DRY while controlling exactly what fields are accepted on input vs. exposed on output.

```python
# Shared fields
class ItemBase(BaseModel):
    name: str
    description: Optional[str] = None
    price: float = Field(..., gt=0)

# What the client sends on POST (no id — server assigns it)
class ItemCreate(ItemBase):
    pass

# What the client sends on PATCH (all optional)
class ItemUpdate(BaseModel):
    name: Optional[str]        = None
    description: Optional[str] = None
    price: Optional[float]     = Field(default=None, gt=0)

# What the API returns (includes server-assigned fields)
class ItemResponse(ItemBase):
    id: int
    created_at: datetime

    model_config = ConfigDict(from_attributes=True)  # enables ORM mode
```

```python
from datetime import datetime
from pydantic import ConfigDict

@app.post("/items", response_model=ItemResponse, status_code=201)
async def create_item(payload: ItemCreate):
    # persist to DB, get back ORM object...
    return orm_object  # Pydantic maps ORM attributes automatically
```

> [!tip] ORM Mode (`from_attributes=True`) Without this, Pydantic only reads from dicts. With it, it can read from ORM objects (SQLAlchemy, Tortoise, etc.) via attribute access.

---

## Field Validators

Custom validation logic that runs after type coercion.

```python
from pydantic import field_validator

class UserCreate(BaseModel):
    username: str
    email: str
    password: str

    @field_validator("username")
    @classmethod
    def username_no_spaces(cls, v: str) -> str:
        if " " in v:
            raise ValueError("Username must not contain spaces")
        return v.lower()  # normalise while validating

    @field_validator("email")
    @classmethod
    def email_must_be_corporate(cls, v: str) -> str:
        if not v.endswith("@company.com"):
            raise ValueError("Only company emails are allowed")
        return v
```

---

## Model Validators — Cross-field Logic

When validation depends on multiple fields at once.

```python
from pydantic import model_validator

class DateRange(BaseModel):
    start: datetime
    end: datetime

    @model_validator(mode="after")
    def end_must_be_after_start(self) -> "DateRange":
        if self.end <= self.start:
            raise ValueError("end must be after start")
        return self
```

---

## Computed Fields

Derived, read-only properties included in serialisation and OpenAPI schema.

```python
from pydantic import computed_field

class Product(BaseModel):
    price: float
    tax_rate: float = 0.2

    @computed_field
    @property
    def price_with_tax(self) -> float:
        return round(self.price * (1 + self.tax_rate), 2)
```

---

## Serialisation Control

```python
item = Item(name="Widget", price=9.99, in_stock=True)

# To dict
item.model_dump()
item.model_dump(exclude={"in_stock"})          # drop a field
item.model_dump(include={"name", "price"})     # only these fields
item.model_dump(exclude_none=True)             # drop None values
item.model_dump(exclude_unset=True)            # drop fields not explicitly set

# To JSON string
item.model_dump_json()
item.model_dump_json(exclude_none=True)
```

> [!tip] `exclude_unset=True` is essential for PATCH endpoints. See [[FastAPI - REST Principles & HTTP Methods]].

---

## Response Model Filtering in FastAPI

`response_model` strips any fields not declared on the output model — even if the underlying object has them. Use this to prevent data leakage.

```python
class UserCreate(BaseModel):
    email: str
    password: str               # accepted on input

class UserResponse(BaseModel):
    id: int
    email: str
                                # password is NOT here → never returned

@app.post("/users", response_model=UserResponse)
async def create_user(user: UserCreate):
    return {"id": 1, "email": user.email, "password": user.password}
    # FastAPI silently strips 'password' from the response
```

---

## `model_config` Options

```python
from pydantic import ConfigDict

class Item(BaseModel):
    model_config = ConfigDict(
        from_attributes=True,       # read from ORM objects
        str_strip_whitespace=True,  # auto-strip leading/trailing spaces
        str_to_lower=True,          # normalise strings to lowercase
        populate_by_name=True,      # allow field name OR alias
        extra="forbid",             # reject unknown fields (strict mode)
    )
```

> [!warning] `extra="forbid"` Useful for strict APIs — any unexpected field in the request body causes a 422 validation error immediately.

---

## Aliases — Different External & Internal Names

```python
from pydantic import Field, AliasPath

class Order(BaseModel):
    order_id: int    = Field(alias="orderId")         # camelCase in JSON
    user_email: str  = Field(alias="userEmail")

# Accept both alias and field name
model_config = ConfigDict(populate_by_name=True)
```

In FastAPI, set `response_model_by_alias=True` on the route to serialise with aliases:

```python
@app.get("/orders/{id}", response_model=Order, response_model_by_alias=True)
async def get_order(id: int): ...
```

---

## Union Types & Discriminated Unions

```python
from typing import Literal, Union
from pydantic import BaseModel

class Cat(BaseModel):
    type: Literal["cat"]
    indoor: bool

class Dog(BaseModel):
    type: Literal["dog"]
    breed: str

class Pet(BaseModel):
    pet: Union[Cat, Dog] = Field(discriminator="type")
```

FastAPI validates and documents the correct variant based on the `type` field.

---

## Quick Reference

|Feature|Tool|
|---|---|
|Required field|`Field(...)` or no default|
|Optional field|`Optional[T] = None`|
|Constraints|`Field(gt=, min_length=, pattern=)`|
|Custom validation|`@field_validator`|
|Cross-field validation|`@model_validator(mode="after")`|
|Derived property|`@computed_field`|
|ORM compatibility|`model_config = ConfigDict(from_attributes=True)`|
|Strict mode|`model_config = ConfigDict(extra="forbid")`|
|Serialise|`model_dump()` / `model_dump_json()`|

---

## Related Notes

- [[ORM - SQLAlchemy]]
- [[ORM - Async]]
- [[API - FastAPI — Dependency Injection & User Management]]
- [[API - FastAPI — Lifespan]]
- [[Python — Pydantic]]

## Tags

#fastapi #pydantic #python #validation #serialisation #backend