
## What & When

**enum** is Python's built-in module for creating enumerated constants. Enums give names to a fixed set of values, making code more readable, self-documenting, and less error-prone than bare strings or integers.

Use enums when:

- A variable can only take one of a fixed set of values
- Replacing magic strings or numbers with named constants
- Defining status codes, categories, directions, or states
- Working with databases, APIs, or config where values are constrained
- Pattern matching on a closed set of cases

```python
from enum import Enum, IntEnum, StrEnum, Flag, auto
```

No installation needed — `enum` is part of the standard library.

---

## Basic Enum

```python
from enum import Enum

class Direction(Enum):
    NORTH = "north"
    SOUTH = "south"
    EAST  = "east"
    WEST  = "west"

# Access
d = Direction.NORTH
d.name          # "NORTH"   — the member name
d.value         # "north"   — the assigned value

# From value
Direction("north")          # Direction.NORTH
Direction["NORTH"]          # Direction.NORTH — by name

# Comparison
Direction.NORTH == Direction.NORTH  # True
Direction.NORTH == Direction.SOUTH  # False
Direction.NORTH is Direction.NORTH  # True — enums are singletons

# Iteration
list(Direction)             # [Direction.NORTH, Direction.SOUTH, ...]

# Membership
Direction.NORTH in Direction    # True
"north" in Direction            # False — compares members, not values
```

---

## `auto()` — Automatic Values

```python
from enum import Enum, auto

class Status(Enum):
    PENDING   = auto()    # 1
    ACTIVE    = auto()    # 2
    SUSPENDED = auto()    # 3
    DELETED   = auto()    # 4

print(Status.PENDING.value)     # 1
print(Status.ACTIVE.value)      # 2
```

`auto()` assigns incrementing integers by default. Customise with `_generate_next_value_`:

```python
from enum import Enum, auto

class Status(Enum):
    @staticmethod
    def _generate_next_value_(name, start, count, last_values):
        return name.lower()     # use the member name as value

    PENDING   = auto()          # "pending"
    ACTIVE    = auto()          # "active"
    SUSPENDED = auto()          # "suspended"
```

---

## `IntEnum` — Integer Enum

Members are `int` subclasses — interoperable with integers directly.

```python
from enum import IntEnum

class HTTPStatus(IntEnum):
    OK                  = 200
    CREATED             = 201
    NO_CONTENT          = 204
    BAD_REQUEST         = 400
    UNAUTHORIZED        = 401
    FORBIDDEN           = 403
    NOT_FOUND           = 404
    UNPROCESSABLE       = 422
    INTERNAL_ERROR      = 500

# Behaves like int
HTTPStatus.OK == 200            # True
HTTPStatus.OK + 1               # 201
HTTPStatus.NOT_FOUND > 400      # True
int(HTTPStatus.OK)              # 200

# Useful for HTTP response codes
def check_status(code: int):
    if code == HTTPStatus.NOT_FOUND:
        return "Not found"
```

---

## `StrEnum` — String Enum (Python 3.11+)

Members are `str` subclasses — interoperable with strings directly.

```python
from enum import StrEnum

class Role(StrEnum):
    ADMIN   = "admin"
    USER    = "user"
    GUEST   = "guest"

# Behaves like str
Role.ADMIN == "admin"           # True
f"Role: {Role.ADMIN}"          # "Role: admin"
Role.ADMIN.upper()             # "ADMIN"
"admin" == Role.ADMIN          # True

# JSON serialisation just works
import json
json.dumps({"role": Role.ADMIN})    # '{"role": "admin"}'
```

> [!tip] `StrEnum` vs `str, Enum` Pre-3.11 pattern: `class Role(str, Enum)` — same behaviour, more verbose. Use `StrEnum` in 3.11+, `str, Enum` for older compatibility.

---

## `str, Enum` — Pre-3.11 String Enum

```python
from enum import Enum

class Status(str, Enum):
    ACTIVE    = "active"
    INACTIVE  = "inactive"
    PENDING   = "pending"

# Interoperable with str
Status.ACTIVE == "active"       # True
f"Status: {Status.ACTIVE}"     # "Status: active"
```

---

## `Flag` — Bitmask Enum

For combining multiple options — like Unix permissions or feature flags.

```python
from enum import Flag, auto

class Permission(Flag):
    READ    = auto()    # 1
    WRITE   = auto()    # 2
    EXECUTE = auto()    # 4

    # Convenience aliases
    READ_WRITE = READ | WRITE

# Combine
user_perms = Permission.READ | Permission.WRITE
print(user_perms)           # Permission.READ|WRITE

# Test membership
Permission.READ in user_perms       # True
Permission.EXECUTE in user_perms    # False

# Iterate combined members
list(user_perms)    # [Permission.READ, Permission.WRITE]

# All permissions
Permission.READ | Permission.WRITE | Permission.EXECUTE
```

---

## `IntFlag` — Integer Bitmask

```python
from enum import IntFlag, auto

class FileMode(IntFlag):
    READ    = 4
    WRITE   = 2
    EXECUTE = 1

mode = FileMode.READ | FileMode.WRITE
int(mode)                   # 6
mode & FileMode.READ        # FileMode.READ (truthy)
mode & FileMode.EXECUTE     # FileMode(0) (falsy)
```

---

## Methods & Properties on Enums

```python
from enum import Enum

class Planet(Enum):
    MERCURY = (3.303e+23, 2.4397e6)
    VENUS   = (4.869e+24, 6.0518e6)
    EARTH   = (5.976e+24, 6.37814e6)
    MARS    = (6.421e+23, 3.3972e6)

    def __init__(self, mass: float, radius: float):
        self.mass   = mass
        self.radius = radius

    @property
    def surface_gravity(self) -> float:
        G = 6.67430e-11
        return G * self.mass / (self.radius ** 2)

    def weight_on(self, earth_weight: float) -> float:
        return earth_weight * self.surface_gravity / Planet.EARTH.surface_gravity

print(Planet.MARS.surface_gravity)          # 3.71
print(Planet.MARS.weight_on(75))            # ~28.1 kg
```

---

## `_missing_` — Handle Unknown Values

```python
from enum import Enum

class Color(Enum):
    RED   = "red"
    GREEN = "green"
    BLUE  = "blue"

    @classmethod
    def _missing_(cls, value):
        # Return a default instead of raising ValueError
        return cls.RED

Color("purple")     # Color.RED — instead of ValueError
```

---

## `unique` — Enforce Unique Values

```python
from enum import Enum, unique

@unique
class Direction(Enum):
    NORTH = "N"
    SOUTH = "S"
    EAST  = "E"
    WEST  = "W"
    # BACKWARD = "N"   # would raise ValueError — duplicate value

# Without @unique, duplicate values create aliases
class Color(Enum):
    RED    = 1
    CRIMSON = 1     # alias for RED — not a separate member
    GREEN  = 2

Color(1)            # Color.RED — CRIMSON is invisible
list(Color)         # [Color.RED, Color.GREEN] — aliases excluded
```

---

## Aliases

```python
from enum import Enum

class Status(Enum):
    ACTIVE   = 1
    ENABLED  = 1    # alias — same value as ACTIVE
    INACTIVE = 2
    DISABLED = 2    # alias — same value as INACTIVE

Status(1)               # Status.ACTIVE — canonical member
Status.ENABLED          # Status.ACTIVE — alias resolves to canonical
Status.ENABLED is Status.ACTIVE  # True

# List only canonical members (no aliases)
list(Status)            # [Status.ACTIVE, Status.INACTIVE]

# List including aliases
list(Status.__members__.values())  # [ACTIVE, ENABLED, INACTIVE, DISABLED]
```

---

## Iteration & Introspection

```python
from enum import Enum

class Status(Enum):
    PENDING  = "pending"
    ACTIVE   = "active"
    INACTIVE = "inactive"

# Iterate members
for member in Status:
    print(member.name, member.value)

# Get all names
[s.name for s in Status]           # ["PENDING", "ACTIVE", "INACTIVE"]

# Get all values
[s.value for s in Status]          # ["pending", "active", "inactive"]

# Check membership
"active" in [s.value for s in Status]   # True
Status.ACTIVE in Status                  # True

# Number of members
len(Status)                         # 3

# Access __members__ dict (includes aliases)
Status.__members__                  # {"PENDING": ..., "ACTIVE": ..., ...}
```

---

## Enum with Pydantic

```python
from enum import Enum, StrEnum
from pydantic import BaseModel

class Status(StrEnum):
    ACTIVE   = "active"
    INACTIVE = "inactive"
    PENDING  = "pending"

class OrderStatus(str, Enum):       # pre-3.11 compatible
    PLACED    = "placed"
    PAID      = "paid"
    SHIPPED   = "shipped"
    DELIVERED = "delivered"

class Order(BaseModel):
    id:     int
    status: OrderStatus = OrderStatus.PLACED

# Pydantic accepts the string value
order = Order(id=1, status="paid")
print(order.status)             # OrderStatus.PAID
print(order.status.value)       # "paid"

# Serialised as the value
order.model_dump()              # {"id": 1, "status": "paid"}
order.model_dump_json()         # '{"id": 1, "status": "paid"}'
```

---

## Enum with FastAPI

FastAPI uses enum values in path parameters, query params, and request bodies — and documents them as choices in Swagger UI automatically.

```python
from enum import Enum
from fastapi import FastAPI

class ModelName(str, Enum):
    alexnet = "alexnet"
    resnet  = "resnet"
    lenet   = "lenet"

app = FastAPI()

@app.get("/models/{model_name}")
async def get_model(model_name: ModelName):
    if model_name == ModelName.alexnet:
        return {"model": model_name, "message": "Deep Learning FTW!"}
    return {"model": model_name.value}

# Swagger UI shows a dropdown with: alexnet, resnet, lenet
```

---

## Enum with SQLAlchemy

```python
from enum import Enum as PyEnum
import sqlalchemy as sa
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column

class UserStatus(PyEnum):
    active   = "active"
    inactive = "inactive"
    banned   = "banned"

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "users"

    id:     Mapped[int]        = mapped_column(primary_key=True)
    status: Mapped[UserStatus] = mapped_column(
        sa.Enum(UserStatus),
        default=UserStatus.active,
    )
```

---

## Pattern Matching with Enums (3.10+)

```python
from enum import Enum

class Status(Enum):
    PENDING  = "pending"
    ACTIVE   = "active"
    INACTIVE = "inactive"
    DELETED  = "deleted"

def describe(status: Status) -> str:
    match status:
        case Status.PENDING:
            return "Awaiting activation"
        case Status.ACTIVE:
            return "Up and running"
        case Status.INACTIVE:
            return "Temporarily disabled"
        case Status.DELETED:
            return "Permanently removed"
        case _:
            return "Unknown status"
```

---

## Common Enum Patterns

### State Machine

```python
from enum import Enum

class OrderState(Enum):
    DRAFT     = "draft"
    PLACED    = "placed"
    PAID      = "paid"
    SHIPPED   = "shipped"
    DELIVERED = "delivered"
    CANCELLED = "cancelled"

    def transitions(self) -> set["OrderState"]:
        return {
            OrderState.DRAFT:     {OrderState.PLACED, OrderState.CANCELLED},
            OrderState.PLACED:    {OrderState.PAID, OrderState.CANCELLED},
            OrderState.PAID:      {OrderState.SHIPPED, OrderState.CANCELLED},
            OrderState.SHIPPED:   {OrderState.DELIVERED},
            OrderState.DELIVERED: set(),
            OrderState.CANCELLED: set(),
        }[self]

    def can_transition_to(self, next_state: "OrderState") -> bool:
        return next_state in self.transitions()

state = OrderState.PLACED
state.can_transition_to(OrderState.PAID)        # True
state.can_transition_to(OrderState.DELIVERED)   # False
```

### Choices for CLI / Config

```python
from enum import Enum
import argparse

class Environment(str, Enum):
    DEV     = "dev"
    STAGING = "staging"
    PROD    = "prod"

parser = argparse.ArgumentParser()
parser.add_argument(
    "--env",
    type=Environment,
    choices=list(Environment),
    default=Environment.DEV,
)
args = parser.parse_args()
print(args.env)     # Environment.DEV
```

---

## Quick Reference

|Task|Code|
|---|---|
|Define|`class Status(Enum): ACTIVE = "active"`|
|Auto value|`ACTIVE = auto()`|
|Access member|`Status.ACTIVE`|
|Access name|`Status.ACTIVE.name` → `"ACTIVE"`|
|Access value|`Status.ACTIVE.value` → `"active"`|
|From value|`Status("active")`|
|From name|`Status["ACTIVE"]`|
|Int enum|`class Status(IntEnum)`|
|Str enum|`class Status(StrEnum)` (3.11+)|
|Str enum old|`class Status(str, Enum)`|
|Bitmask|`class Perm(Flag): READ = auto()`|
|Combine flags|`Permission.READ \| Permission.WRITE`|
|Test flag|`Permission.READ in user_perms`|
|Enforce unique|`@unique`|
|Handle unknown|`_missing_` classmethod|
|Iterate|`for m in Status`|
|All names|`[m.name for m in Status]`|
|All values|`[m.value for m in Status]`|
|Pattern match|`match status: case Status.ACTIVE:`|

---

## Tags

#python #enum #constants #types #stdlib #backend #fastapi