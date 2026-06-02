
## What & When

**dataclasses** is a Python standard library module (3.7+) that auto-generates boilerplate methods (`__init__`, `__repr__`, `__eq__`, etc.) for classes that primarily hold data. It sits between plain classes and full Pydantic models — more structured than dicts, lighter than Pydantic.

Use dataclasses when:

- Representing structured data without runtime validation overhead
- Writing domain models, DTOs, or value objects
- You want type hints without a third-party dependency
- Working with Python's `typing` system in a lightweight way
- Replacing verbose `__init__` boilerplate

Do NOT use dataclasses when:

- You need runtime **validation** → use Pydantic
- You need JSON serialisation control → use Pydantic or `attrs`
- You need immutability enforcement with hashability → use `frozen=True` or `NamedTuple`

```python
from dataclasses import dataclass, field
```

No installation needed — dataclasses is part of the standard library.

---

## dataclasses vs Alternatives


|                    | dataclasses          | Pydantic   | NamedTuple | attrs    |
| ------------------ | -------------------- | ---------- | ---------- | -------- |
| Stdlib             | ✅                    | ❌          | ✅          | ❌        |
| Runtime validation | ❌                    | ✅          | ❌          | Optional |
| JSON serialisation | Manual               | ✅          | Manual     | Optional |
| Immutability       | `frozen=True`        | ❌          | ✅          | Optional |
| Inheritance        | ✅                    | ✅          | Limited    | ✅        |
| Performance        | Fast                 | v2 fast    | Fastest    | Fast     |
| `__slots__`        | `slots=True` (3.10+) | Via config | ✅          | ✅        |

---

## Basic Dataclass

```python
from dataclasses import dataclass
from typing import Optional
from datetime import datetime

@dataclass
class User:
    id:         int
    name:       str
    email:      str
    bio:        Optional[str] = None
    is_active:  bool = True
    created_at: datetime = field(default_factory=datetime.utcnow)

# Instantiate
user = User(id=1, name="Alice", email="alice@example.com")

# Auto-generated __repr__
print(user)
# User(id=1, name='Alice', email='alice@example.com', bio=None, is_active=True, ...)

# Auto-generated __eq__
user1 = User(id=1, name="Alice", email="alice@example.com")
user2 = User(id=1, name="Alice", email="alice@example.com")
print(user1 == user2)   # True

# Access fields
print(user.name)        # "Alice"
```

---

## `field()` — Per-Field Configuration

```python
from dataclasses import dataclass, field
from typing import ClassVar

@dataclass
class Product:
    name:        str
    price:       float

    # Mutable default — must use field(default_factory=...)
    tags:        list[str]  = field(default_factory=list)
    metadata:    dict       = field(default_factory=dict)

    # Excluded from __init__ — set manually or in __post_init__
    slug:        str        = field(init=False)

    # Excluded from __repr__
    _password:   str        = field(default="", repr=False)

    # Excluded from __eq__ and __hash__
    cache:       dict       = field(default_factory=dict, compare=False)

    # Excluded from everything (internal use)
    _logger:     object     = field(default=None, init=False, repr=False, compare=False)

    # Class variable — not a dataclass field
    count: ClassVar[int] = 0

    def __post_init__(self):
        self.slug = self.name.lower().replace(" ", "-")
        Product.count += 1
```

|`field()` parameter|Description|
|---|---|
|`default`|Default value|
|`default_factory`|Callable returning default (for mutables)|
|`init`|Include in `__init__`|
|`repr`|Include in `__repr__`|
|`compare`|Include in `__eq__` and `__lt__` etc.|
|`hash`|Include in `__hash__`|
|`metadata`|Arbitrary metadata dict (read-only)|
|`kw_only`|Force keyword-only argument (3.10+)|

---

## `__post_init__` — Post-Initialisation Logic

Runs after `__init__` — use for derived fields, validation, or setup.

```python
from dataclasses import dataclass, field
from typing import Optional

@dataclass
class Rectangle:
    width:    float
    height:   float
    area:     float = field(init=False)
    label:    Optional[str] = None

    def __post_init__(self):
        if self.width <= 0 or self.height <= 0:
            raise ValueError("Width and height must be positive")
        self.area = self.width * self.height
        if self.label is None:
            self.label = f"{self.width}x{self.height}"

r = Rectangle(width=3.0, height=4.0)
print(r.area)    # 12.0
print(r.label)   # "3.0x4.0"
```

---

## Decorator Parameters

```python
from dataclasses import dataclass

@dataclass(
    init=True,          # generate __init__          (default: True)
    repr=True,          # generate __repr__           (default: True)
    eq=True,            # generate __eq__             (default: True)
    order=False,        # generate __lt__, __le__ etc (default: False)
    unsafe_hash=False,  # generate __hash__           (default: False)
    frozen=False,       # make immutable              (default: False)
    match_args=True,    # set __match_args__          (default: True, 3.10+)
    kw_only=False,      # all fields keyword-only     (default: False, 3.10+)
    slots=False,        # use __slots__               (default: False, 3.10+)
    weakref_slot=False, # add weakref slot            (default: False, 3.10+)
)
class MyClass:
    ...
```

---

## Frozen Dataclasses — Immutability

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class Point:
    x: float
    y: float

p = Point(1.0, 2.0)
# p.x = 3.0   # raises FrozenInstanceError

# Frozen + eq=True → hashable (usable in sets and as dict keys)
points = {Point(0, 0), Point(1, 1), Point(0, 0)}
print(len(points))  # 2

d = {Point(0, 0): "origin"}
```

---

## Ordering

```python
from dataclasses import dataclass

@dataclass(order=True)
class Version:
    major: int
    minor: int
    patch: int

v1 = Version(1, 2, 3)
v2 = Version(1, 3, 0)

print(v1 < v2)          # True
print(sorted([v2, v1])) # [Version(1,2,3), Version(1,3,0)]
```

> [!tip] Control sort key with `field(compare=False)` Fields with `compare=False` are excluded from ordering comparisons. Use this to sort by a subset of fields only.

---

## `__slots__` — Memory Efficient (Python 3.10+)

```python
from dataclasses import dataclass

@dataclass(slots=True)
class Point:
    x: float
    y: float

# Uses __slots__ — faster attribute access, less memory per instance
# Cannot add arbitrary attributes after creation
p = Point(1.0, 2.0)
# p.z = 3.0   # AttributeError
```

For Python < 3.10, define `__slots__` manually:

```python
from dataclasses import dataclass

@dataclass
class Point:
    __slots__ = ("x", "y")
    x: float
    y: float
```

---

## Inheritance

```python
from dataclasses import dataclass
from datetime import datetime
from typing import Optional

@dataclass
class Base:
    id:         int
    created_at: datetime = field(default_factory=datetime.utcnow)
    updated_at: datetime = field(default_factory=datetime.utcnow)

@dataclass
class Item(Base):
    name:  str = ""
    price: float = 0.0

@dataclass
class DiscountedItem(Item):
    discount: float = 0.0

    @property
    def final_price(self) -> float:
        return round(self.price * (1 - self.discount), 2)

item = DiscountedItem(id=1, name="Widget", price=9.99, discount=0.1)
print(item.final_price)   # 8.99
```

> [!warning] Fields with defaults must come after fields without defaults In dataclass inheritance, if a parent has fields with defaults, all child fields must also have defaults — otherwise Python raises a `TypeError`.

```python
@dataclass
class Parent:
    name: str
    value: int = 0        # has default

@dataclass
class Child(Parent):
    extra: str = ""       # ✅ must have default too
    # extra: str          # ❌ TypeError — non-default after default
```

---

## `dataclasses` Module Functions

```python
from dataclasses import (
    dataclass, field,
    fields,         # get field metadata
    asdict,         # convert to dict (recursive)
    astuple,        # convert to tuple (recursive)
    replace,        # create copy with field overrides
    is_dataclass,   # check if object/class is a dataclass
    make_dataclass, # create dataclass dynamically
)

@dataclass
class Point:
    x: float
    y: float

p = Point(1.0, 2.0)

# fields() — inspect field metadata
for f in fields(p):
    print(f.name, f.type, f.default)

# asdict() — deep conversion to dict
from dataclasses import asdict
d = asdict(p)               # {"x": 1.0, "y": 2.0}

# astuple() — deep conversion to tuple
t = astuple(p)              # (1.0, 2.0)

# replace() — immutable-style update (like Pydantic's model_copy)
p2 = replace(p, x=5.0)     # Point(x=5.0, y=2.0) — p unchanged

# is_dataclass()
is_dataclass(p)             # True
is_dataclass(Point)         # True
is_dataclass({"x": 1})      # False

# make_dataclass() — dynamic creation
DynPoint = make_dataclass("DynPoint", ["x", "y"])
DynPoint = make_dataclass("DynPoint", [("x", float), ("y", float, 0.0)])
```

---

## Serialisation

Dataclasses don't have built-in JSON serialisation — use `asdict` + `json`.

```python
import json
from dataclasses import dataclass, asdict, field
from datetime import datetime

@dataclass
class Event:
    name:       str
    timestamp:  datetime = field(default_factory=datetime.utcnow)
    tags:       list[str] = field(default_factory=list)

event = Event(name="user.login", tags=["auth"])

# To dict
d = asdict(event)

# To JSON — handle non-serialisable types manually
def serialise(obj):
    if isinstance(obj, datetime):
        return obj.isoformat()
    raise TypeError(f"Not serialisable: {type(obj)}")

json_str = json.dumps(asdict(event), default=serialise, indent=2)

# From JSON — manual reconstruction
data = json.loads(json_str)
event = Event(
    name=data["name"],
    timestamp=datetime.fromisoformat(data["timestamp"]),
    tags=data["tags"],
)
```

> [!tip] For complex serialisation, use Pydantic instead If you find yourself writing a lot of custom serialisation logic, it's a signal to switch to [[Python - Pydantic]].

---

## `InitVar` — Init-Only Fields

Fields used in `__post_init__` but not stored on the instance.

```python
from dataclasses import dataclass, InitVar, field

@dataclass
class HashedPassword:
    username:  str
    password:  InitVar[str]     # accepted in __init__, not stored
    password_hash: str = field(init=False)

    def __post_init__(self, password: str):
        import hashlib
        self.password_hash = hashlib.sha256(password.encode()).hexdigest()

u = HashedPassword(username="alice", password="secret123")
print(u.password_hash)  # sha256 hash
# u.password            # AttributeError — not stored
```

---

## `KW_ONLY` Sentinel — Keyword-Only Fields (3.10+)

```python
from dataclasses import dataclass, field, KW_ONLY

@dataclass
class Config:
    host:  str
    port:  int
    _:     KW_ONLY              # all fields after this are keyword-only
    debug: bool = False
    workers: int = 4

# host and port can be positional or keyword
# debug and workers MUST be keyword
c = Config("localhost", 8000, debug=True, workers=8)
```

---

## Pattern Matching (3.10+)

Dataclasses set `__match_args__` automatically — works with structural pattern matching.

```python
from dataclasses import dataclass

@dataclass
class Point:
    x: float
    y: float

@dataclass
class Circle:
    center: Point
    radius: float

shape = Circle(center=Point(0, 0), radius=5.0)

match shape:
    case Circle(center=Point(x=0, y=0), radius=r):
        print(f"Circle at origin with radius {r}")
    case Circle(center=Point(x=x, y=y), radius=r):
        print(f"Circle at ({x}, {y}) with radius {r}")
```

---

## Dataclasses with FastAPI

FastAPI supports dataclasses as request/response models — though Pydantic models are more feature-rich.

```python
from dataclasses import dataclass, field
from typing import Optional
from fastapi import FastAPI

@dataclass
class ItemCreate:
    name:  str
    price: float
    tags:  list[str] = field(default_factory=list)

@dataclass
class ItemResponse:
    id:    int
    name:  str
    price: float
    tags:  list[str] = field(default_factory=list)

app = FastAPI()

@app.post("/items", response_model=ItemResponse)
async def create_item(item: ItemCreate) -> ItemResponse:
    return ItemResponse(id=1, name=item.name, price=item.price, tags=item.tags)
```

> [!warning] Limited compared to Pydantic in FastAPI Dataclasses in FastAPI lack validators, `Field` constraints, ORM mode, and rich serialisation control. Use Pydantic models for API-facing schemas and dataclasses for internal domain objects.

---

## Common Patterns

### Value Object

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class Money:
    amount: float
    currency: str = "GBP"

    def __add__(self, other: "Money") -> "Money":
        if self.currency != other.currency:
            raise ValueError("Cannot add different currencies")
        return Money(self.amount + other.amount, self.currency)

    def __str__(self) -> str:
        return f"{self.currency} {self.amount:.2f}"

price = Money(9.99)
tax   = Money(2.00)
total = price + tax
print(total)    # GBP 11.99
```

### DTO — Data Transfer Object

```python
from dataclasses import dataclass
from datetime import datetime

@dataclass
class OrderSummaryDTO:
    order_id:    int
    customer:    str
    total:       float
    placed_at:   datetime
    item_count:  int

    @classmethod
    def from_orm(cls, order) -> "OrderSummaryDTO":
        return cls(
            order_id   = order.id,
            customer   = order.customer.name,
            total      = sum(i.price for i in order.items),
            placed_at  = order.created_at,
            item_count = len(order.items),
        )
```

### Configuration Object

```python
from dataclasses import dataclass, field

@dataclass
class DatabaseConfig:
    host:      str = "localhost"
    port:      int = 5432
    name:      str = "mydb"
    user:      str = "postgres"
    password:  str = field(default="", repr=False)   # hide from repr
    pool_size: int = 10

    @property
    def url(self) -> str:
        return f"postgresql://{self.user}:{self.password}@{self.host}:{self.port}/{self.name}"
```

---

## Quick Reference

|Task|Code|
|---|---|
|Define|`@dataclass class Foo: x: int`|
|Required field|`x: int` (no default)|
|Optional field|`x: int = 0`|
|Mutable default|`x: list = field(default_factory=list)`|
|Post-init logic|`def __post_init__(self): ...`|
|Init-only field|`x: InitVar[str]`|
|Derived field|`x: int = field(init=False)`|
|Hide from repr|`field(repr=False)`|
|Exclude from eq|`field(compare=False)`|
|Immutable|`@dataclass(frozen=True)`|
|Orderable|`@dataclass(order=True)`|
|Memory efficient|`@dataclass(slots=True)` (3.10+)|
|Keyword-only|`@dataclass(kw_only=True)` (3.10+)|
|To dict|`asdict(obj)`|
|To tuple|`astuple(obj)`|
|Copy with changes|`replace(obj, field=value)`|
|Inspect fields|`fields(obj)`|
|Check type|`is_dataclass(obj)`|
|Dynamic creation|`make_dataclass("Name", ["x", "y"])`|

---

## Tags

#python #dataclasses #stdlib #types #domain-model #dto #backend