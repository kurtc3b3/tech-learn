
## What & When

**Pydantic** is a data validation and settings management library built on Python type hints. It parses, validates, and serialises data at runtime — catching type errors before they surface as bugs deep in business logic.

Use Pydantic when:

- Validating external data (API payloads, config files, user input)
- Defining typed data structures with runtime guarantees
- Managing application settings from environment variables
- Serialising/deserialising JSON with type safety
- Replacing `dataclasses` where validation is needed

```bash
pip install pydantic          # v2 — Rust-powered core (pydantic-core)
pip install pydantic-settings # for settings management
```

> [!info] Pydantic v2 All examples use **Pydantic v2** (released 2023). v2 is 5–50× faster than v1 due to a Rust core (`pydantic-core`). The API changed significantly — `validator` → `field_validator`, `Config` class → `model_config`, etc.

---

## BaseModel — Core

```python
from pydantic import BaseModel
from typing import Optional
from datetime import datetime

class User(BaseModel):
    id:         int
    name:       str
    email:      str
    bio:        Optional[str] = None
    created_at: datetime = datetime.utcnow()
    is_active:  bool = True

# Instantiate — validates on creation
user = User(id=1, name="Alice", email="alice@example.com")

# From dict
user = User(**{"id": 1, "name": "Alice", "email": "alice@example.com"})
user = User.model_validate({"id": 1, "name": "Alice", "email": "alice@example.com"})

# From JSON string
user = User.model_validate_json('{"id": 1, "name": "Alice", "email": "alice@example.com"}')

# Access fields
print(user.name)            # "Alice"
print(user.model_fields)    # field definitions
```

---

## Field — Constraints & Metadata

```python
from pydantic import BaseModel, Field
from typing import Optional

class Product(BaseModel):
    name:        str   = Field(...,     min_length=2, max_length=100)
    price:       float = Field(...,     gt=0,         description="Price in GBP")
    discount:    float = Field(0.0,     ge=0.0, le=1.0)
    sku:         str   = Field(...,     pattern=r"^[A-Z]{3}-\d{4}$")
    tags:        list[str] = Field(default_factory=list, max_length=10)
    rating:      Optional[float] = Field(None, ge=0.0, le=5.0)
    title:       str   = Field(...,     alias="productTitle")   # JSON key differs
    internal_id: str   = Field(...,     exclude=True)           # never serialised
```

> [!info] `...` means required — no default value.

|Constraint|Applies to|Meaning|
|---|---|---|
|`gt` / `lt`|numbers|greater / less than (exclusive)|
|`ge` / `le`|numbers|greater / less than or equal|
|`min_length` / `max_length`|str, list|length bounds|
|`pattern`|str|regex match|
|`multiple_of`|numbers|must be a multiple|
|`strict`|any|no type coercion|
|`default_factory`|any|callable for mutable defaults|
|`alias`|any|different key name in input/output|
|`exclude`|any|omit from serialisation|
|`repr`|any|include in `__repr__`|

---

## Validators — Field Level

```python
from pydantic import BaseModel, field_validator

class UserCreate(BaseModel):
    username: str
    email:    str
    password: str

    @field_validator("username")
    @classmethod
    def username_alphanumeric(cls, v: str) -> str:
        if not v.isalnum():
            raise ValueError("Username must be alphanumeric")
        return v.lower()                        # transform while validating

    @field_validator("email")
    @classmethod
    def email_lowercase(cls, v: str) -> str:
        return v.strip().lower()

    @field_validator("password")
    @classmethod
    def password_strength(cls, v: str) -> str:
        if len(v) < 8:
            raise ValueError("Password must be at least 8 characters")
        if not any(c.isupper() for c in v):
            raise ValueError("Password must contain an uppercase letter")
        return v
```

### `mode` Parameter

```python
@field_validator("price", mode="before")    # runs before type coercion
@classmethod
def strip_currency(cls, v):
    if isinstance(v, str):
        return v.replace("£", "").replace(",", "").strip()
    return v

@field_validator("price", mode="after")     # default — runs after coercion
@classmethod
def positive_price(cls, v: float) -> float:
    if v <= 0:
        raise ValueError("Price must be positive")
    return v
```

---

## Validators — Model Level

Cross-field validation — runs after all fields are validated.

```python
from pydantic import BaseModel, model_validator
from datetime import datetime

class BookingRequest(BaseModel):
    check_in:  datetime
    check_out: datetime
    guests:    int
    room_type: str

    @model_validator(mode="after")
    def check_dates(self) -> "BookingRequest":
        if self.check_out <= self.check_in:
            raise ValueError("check_out must be after check_in")
        if (self.check_out - self.check_in).days > 30:
            raise ValueError("Maximum stay is 30 days")
        return self

    @model_validator(mode="before")    # receives raw dict before field validation
    @classmethod
    def normalise_room_type(cls, data: dict) -> dict:
        if "room_type" in data:
            data["room_type"] = data["room_type"].lower().strip()
        return data
```

---

## Computed Fields

Read-only derived properties included in schema and serialisation.

```python
from pydantic import BaseModel, computed_field

class Order(BaseModel):
    items:    list[float]
    tax_rate: float = 0.2

    @computed_field
    @property
    def subtotal(self) -> float:
        return round(sum(self.items), 2)

    @computed_field
    @property
    def tax(self) -> float:
        return round(self.subtotal * self.tax_rate, 2)

    @computed_field
    @property
    def total(self) -> float:
        return round(self.subtotal + self.tax, 2)

order = Order(items=[9.99, 4.99, 14.99])
print(order.total)          # 35.97
print(order.model_dump())   # includes subtotal, tax, total
```

---

## Serialisation

```python
user = User(id=1, name="Alice", email="alice@example.com")

# To dict
user.model_dump()
user.model_dump(exclude={"email"})              # drop field
user.model_dump(include={"id", "name"})        # only these fields
user.model_dump(exclude_none=True)             # drop None values
user.model_dump(exclude_unset=True)            # drop fields not explicitly set
user.model_dump(exclude_defaults=True)         # drop fields at their default
user.model_dump(by_alias=True)                 # use field aliases as keys
user.model_dump(mode="json")                   # JSON-safe types (datetime → str)

# To JSON string
user.model_dump_json()
user.model_dump_json(exclude_none=True, indent=2)

# To JSON schema (OpenAPI compatible)
User.model_json_schema()
```

---

## Nested Models

```python
from pydantic import BaseModel
from typing import Optional

class Address(BaseModel):
    street:   str
    city:     str
    postcode: str
    country:  str = "GB"

class Company(BaseModel):
    name:      str
    address:   Address
    employees: list["Employee"] = []    # forward reference

class Employee(BaseModel):
    name:    str
    role:    str
    company: Optional[Company] = None

Company.model_rebuild()     # resolve forward references
```

---

## Model Inheritance

```python
from pydantic import BaseModel, ConfigDict
from datetime import datetime
from typing import Optional

class ItemBase(BaseModel):
    name:        str
    description: Optional[str] = None
    price:       float

class ItemCreate(ItemBase):
    pass                                # used for POST input

class ItemUpdate(BaseModel):
    name:        Optional[str]  = None  # all optional for PATCH
    description: Optional[str]  = None
    price:       Optional[float]= None

class ItemInDB(ItemBase):
    id:         int
    created_at: datetime
    updated_at: datetime

    model_config = ConfigDict(from_attributes=True)  # ORM mode

class ItemResponse(ItemInDB):
    pass                                # what the API returns
```

---

## `model_config` — ConfigDict

```python
from pydantic import BaseModel, ConfigDict

class StrictItem(BaseModel):
    model_config = ConfigDict(
        # Validation
        strict=False,               # allow type coercion (default)
        extra="forbid",             # reject unknown fields ("allow" / "ignore")
        validate_default=True,      # validate default values too
        validate_assignment=True,   # validate on attribute assignment
        revalidate_instances="always",  # revalidate nested model instances

        # ORM / data sources
        from_attributes=True,       # read from ORM object attributes

        # Strings
        str_strip_whitespace=True,  # auto-strip leading/trailing whitespace
        str_to_lower=True,          # normalise strings to lowercase
        str_min_length=1,           # global min length for all strings

        # Serialisation
        populate_by_name=True,      # accept both alias and field name
        use_enum_values=True,       # store enum value, not enum instance
        ser_json_timedelta="float", # serialise timedelta as seconds float

        # Schema
        title="Item",
        json_schema_extra={"examples": [{"name": "Widget", "price": 9.99}]},
    )
```

---

## RootModel — Top-Level Collections

When the model IS the collection, not a wrapper around one.

```python
from pydantic import RootModel

# Validate a top-level list
class ItemList(RootModel[list[Item]]):
    pass

items = ItemList.model_validate([{"name": "a", "price": 1}, {"name": "b", "price": 2}])
print(items.root)           # [Item(name='a',...), Item(name='b',...)]

# Validate a top-level dict
class ItemMap(RootModel[dict[str, Item]]):
    pass

items = ItemMap.model_validate({"widget": {"name": "Widget", "price": 9.99}})
print(items.root["widget"].price)   # 9.99
```

---

## TypeAdapter — Validate Without a Model

Validate arbitrary types without defining a model class.

```python
from pydantic import TypeAdapter
from typing import list

# Validate a list of ints
adapter = TypeAdapter(list[int])
result  = adapter.validate_python([1, "2", 3.0])    # [1, 2, 3]

# Validate a dict
adapter = TypeAdapter(dict[str, float])
result  = adapter.validate_python({"a": "1.5", "b": 2})    # {"a": 1.5, "b": 2.0}

# Validate from JSON
adapter = TypeAdapter(list[int])
result  = adapter.validate_json("[1, 2, 3]")

# Serialise
adapter.dump_python([1, 2, 3])
adapter.dump_json([1, 2, 3])

# JSON schema
adapter.json_schema()
```

---

## Custom Types

```python
from pydantic import GetCoreSchemaHandler
from pydantic_core import core_schema
from typing import Any

class PositiveInt:
    """Custom type that ensures int > 0."""

    def __init__(self, value: int):
        if value <= 0:
            raise ValueError(f"{value} is not positive")
        self.value = value

    @classmethod
    def __get_pydantic_core_schema__(
        cls, source_type: Any, handler: GetCoreSchemaHandler
    ):
        return core_schema.no_info_plain_validator_function(
            lambda v: cls(int(v)),
            serialization=core_schema.plain_serializer_function_ser_schema(
                lambda x: x.value
            ),
        )


class Item(BaseModel):
    quantity: PositiveInt

item = Item(quantity=5)
print(item.quantity.value)   # 5
```

---

## Annotated Types — Reusable Constraints

```python
from typing import Annotated
from pydantic import Field, BaseModel

# Define reusable constrained types
PositiveFloat = Annotated[float, Field(gt=0)]
ShortStr      = Annotated[str,   Field(min_length=1, max_length=100)]
Email         = Annotated[str,   Field(pattern=r"^[^@]+@[^@]+\.[^@]+$")]
Percentage    = Annotated[float, Field(ge=0.0, le=100.0)]

class Product(BaseModel):
    name:     ShortStr
    price:    PositiveFloat
    discount: Percentage
    contact:  Email
```

---

## Enums

```python
from enum import Enum
from pydantic import BaseModel

class Status(str, Enum):
    active   = "active"
    inactive = "inactive"
    pending  = "pending"

class UserStatus(BaseModel):
    user_id: int
    status:  Status = Status.pending

u = UserStatus(user_id=1, status="active")  # coerced from string
print(u.status)                              # Status.active
print(u.status.value)                        # "active"
print(u.model_dump())                        # {"user_id": 1, "status": "active"}
```

---

## Discriminated Unions

Parse different model variants based on a literal type field.

```python
from typing import Literal, Union
from pydantic import BaseModel, Field

class Cat(BaseModel):
    type:   Literal["cat"]
    indoor: bool
    breed:  str

class Dog(BaseModel):
    type:    Literal["dog"]
    trained: bool
    breed:   str

class Bird(BaseModel):
    type:       Literal["bird"]
    can_fly:    bool
    wing_span:  float

class Pet(BaseModel):
    pet: Union[Cat, Dog, Bird] = Field(discriminator="type")

p = Pet.model_validate({"pet": {"type": "cat", "indoor": True, "breed": "Siamese"}})
print(type(p.pet))   # <class 'Cat'>
```

---

## pydantic-settings — Environment & Config

```python
from pydantic_settings import BaseSettings, SettingsConfigDict
from pydantic import Field
from typing import Optional
from functools import lru_cache

class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",                # load from .env file
        env_file_encoding="utf-8",
        env_nested_delimiter="__",      # DATABASE__URL → database.url
        case_sensitive=False,
        extra="ignore",
    )

    # App
    app_name:    str   = "My Service"
    debug:       bool  = False
    environment: str   = "production"

    # Database
    database_url: str  = Field(..., description="PostgreSQL DSN")
    db_pool_size: int  = 10

    # Redis
    redis_url:   str   = "redis://localhost:6379"

    # Auth
    secret_key:  str   = Field(..., min_length=32)
    jwt_expire:  int   = 3600

    # Optional
    sentry_dsn:  Optional[str] = None


@lru_cache
def get_settings() -> Settings:
    return Settings()


# Usage
settings = get_settings()
print(settings.database_url)
print(settings.debug)
```

```ini
# .env
DATABASE_URL=postgresql+asyncpg://user:pass@localhost/db
SECRET_KEY=supersecretkey1234567890abcdefgh
DEBUG=true
DB_POOL_SIZE=20
```

> [!tip] `@lru_cache` on `get_settings` Ensures `.env` is parsed once, not on every call. Essential when used as a FastAPI dependency. See [[FastAPI - Dependency Injection & User Management]].

---

## Validation Errors

```python
from pydantic import ValidationError

try:
    user = User(id="not-an-int", name="", email="bad-email")
except ValidationError as e:
    print(e.error_count())      # number of errors
    print(e.errors())           # list of error dicts
    print(e.json(indent=2))     # JSON representation

# Error dict structure:
# {
#   "type": "int_parsing",
#   "loc": ("id",),
#   "msg": "Input should be a valid integer",
#   "input": "not-an-int",
#   "url": "https://errors.pydantic.dev/..."
# }
```

---

## Performance Tips

```python
# 1. model_validate is faster than __init__ for dict input
user = User.model_validate(data)           # faster
user = User(**data)                        # slower (unpacks first)

# 2. model_validate_json skips Python dict creation entirely
user = User.model_validate_json(json_str)  # fastest for JSON input

# 3. TypeAdapter for repeated validation of non-model types
from pydantic import TypeAdapter
adapter = TypeAdapter(list[int])           # create once
result  = adapter.validate_python(data)   # reuse adapter

# 4. model_config strict=True avoids coercion overhead when input is trusted
class FastModel(BaseModel):
    model_config = ConfigDict(strict=True)

# 5. exclude unused fields from serialisation
user.model_dump(include={"id", "name"})   # faster than full dump
```

---

## Quick Reference

|Task|Code|
|---|---|
|Define model|`class M(BaseModel): field: type`|
|Required field|`Field(...)` or no default|
|Optional field|`Optional[T] = None`|
|Constrained field|`Field(..., gt=0, min_length=2)`|
|Field alias|`Field(..., alias="jsonKey")`|
|Exclude from output|`Field(..., exclude=True)`|
|Reusable type|`Annotated[type, Field(...)]`|
|Field validator|`@field_validator("field")`|
|Model validator|`@model_validator(mode="after")`|
|Computed field|`@computed_field @property`|
|ORM mode|`ConfigDict(from_attributes=True)`|
|Strict mode|`ConfigDict(extra="forbid")`|
|From dict|`M.model_validate({...})`|
|From JSON|`M.model_validate_json("...")`|
|To dict|`m.model_dump()`|
|To JSON|`m.model_dump_json()`|
|Top-level list|`RootModel[list[T]]`|
|Validate type|`TypeAdapter(list[int]).validate_python(data)`|
|Discriminated union|`Field(discriminator="type")`|
|Settings from env|`class S(BaseSettings)`|
|Validation error|`except ValidationError as e`|

---

## Tags

#python #pydantic #validation #serialisation #settings #types #backend