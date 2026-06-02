## What & When

**typing** is Python's built-in module for type hints. Type hints are annotations that describe the expected types of variables, function parameters, and return values. They are not enforced at runtime by default — but are used by static type checkers (mypy, pyright), IDEs, and libraries like Pydantic and FastAPI that do enforce them at runtime.

Use typing when:

- Writing code that benefits from IDE autocompletion and refactoring
- Catching type errors before runtime with mypy or pyright
- Defining contracts for function signatures
- Working with Pydantic, FastAPI, or dataclasses
- Documenting complex data structures

```python
from typing import Optional, Union, Any, Callable, ...
```

No installation needed — `typing` is part of the standard library.

> [!info] Modern Python (3.10+) Many `typing` constructs have built-in equivalents since 3.9/3.10. `list[int]` instead of `List[int]`, `X | Y` instead of `Union[X, Y]`, `X | None` instead of `Optional[X]`. Both styles are covered here.

---

## Basic Annotations

```python
# Variables
name:    str   = "Alice"
age:     int   = 30
price:   float = 9.99
active:  bool  = True
data:    bytes = b"\x00"

# Functions
def greet(name: str) -> str:
    return f"Hello, {name}!"

def add(a: int, b: int) -> int:
    return a + b

def log(message: str) -> None:      # returns nothing
    print(message)

def identity(x):                     # untyped — avoid this
    return x
```

---

## Collections

```python
# Python 3.9+ — use built-in types directly
def process(items: list[int]) -> list[str]:
    return [str(i) for i in items]

def lookup(data: dict[str, int]) -> None: ...
def unique(items: set[str]) -> None: ...
def pair(t: tuple[int, str]) -> None: ...
def coords(t: tuple[float, ...]) -> None: ...   # variable-length tuple

# Python 3.8 and below — import from typing
from typing import List, Dict, Set, Tuple

def process(items: List[int]) -> List[str]: ...
def lookup(data: Dict[str, int]) -> None: ...
```

---

## Optional & Union

```python
from typing import Optional, Union

# Optional[X] = X | None
def find_user(user_id: int) -> Optional[str]:   # returns str or None
    ...

# Python 3.10+ shorthand
def find_user(user_id: int) -> str | None: ...

# Union — one of several types
def parse(value: Union[int, str, float]) -> str:
    return str(value)

# Python 3.10+ shorthand
def parse(value: int | str | float) -> str: ...

# Optional with default
def greet(name: Optional[str] = None) -> str:
    return f"Hello, {name or 'World'}!"
```

---

## `Any` — Opt Out of Type Checking

```python
from typing import Any

def accept_anything(value: Any) -> Any:
    return value

# Any is compatible with every type — use sparingly
data: Any = get_external_data()
```

> [!warning] Avoid `Any` where possible `Any` disables type checking for that value. Use it only at system boundaries (external APIs, legacy code) and add proper types as soon as you can.

---

## `Callable` — Function Types

```python
from typing import Callable

# Callable[[arg_types], return_type]
def apply(fn: Callable[[int], str], value: int) -> str:
    return fn(value)

# Multiple arguments
def transform(fn: Callable[[int, int], bool]) -> None: ...

# No arguments
def run(fn: Callable[[], None]) -> None:
    fn()

# Any arguments (less strict)
def call(fn: Callable[..., int]) -> int:
    return fn()

# Type alias for clarity
Handler = Callable[[str, int], bool]

def register(handler: Handler) -> None: ...
```

---

## `TypeVar` — Generic Functions

```python
from typing import TypeVar

T = TypeVar("T")

def first(items: list[T]) -> T:
    return items[0]

first([1, 2, 3])        # return type inferred as int
first(["a", "b"])       # return type inferred as str

# Constrained TypeVar
Number = TypeVar("Number", int, float)

def double(x: Number) -> Number:
    return x * 2

# Bounded TypeVar — T must be a subclass of Base
from typing import TypeVar

class Animal:
    def speak(self) -> str: ...

A = TypeVar("A", bound=Animal)

def make_speak(animal: A) -> str:
    return animal.speak()
```

---

## `Generic` — Generic Classes

```python
from typing import Generic, TypeVar

T = TypeVar("T")
K = TypeVar("K")
V = TypeVar("V")

class Stack(Generic[T]):
    def __init__(self) -> None:
        self._items: list[T] = []

    def push(self, item: T) -> None:
        self._items.append(item)

    def pop(self) -> T:
        return self._items.pop()

    def peek(self) -> T:
        return self._items[-1]

    def is_empty(self) -> bool:
        return not self._items

stack: Stack[int] = Stack()
stack.push(1)
stack.push(2)
value: int = stack.pop()    # type checker knows this is int

# Generic with multiple type vars
class Pair(Generic[K, V]):
    def __init__(self, key: K, value: V) -> None:
        self.key   = key
        self.value = value

pair: Pair[str, int] = Pair("age", 30)
```

---

## `Protocol` — Structural Subtyping (Duck Typing)

Define an interface without inheritance — any class that has the right methods satisfies the protocol.

```python
from typing import Protocol, runtime_checkable

class Drawable(Protocol):
    def draw(self) -> None: ...
    def resize(self, factor: float) -> None: ...

class Circle:
    def draw(self) -> None:
        print("Drawing circle")

    def resize(self, factor: float) -> None:
        self.radius *= factor

class Square:
    def draw(self) -> None:
        print("Drawing square")

    def resize(self, factor: float) -> None:
        self.side *= factor

def render(shape: Drawable) -> None:
    shape.draw()

render(Circle())    # ✅ — Circle satisfies Drawable
render(Square())    # ✅ — Square satisfies Drawable

# Runtime checkable
@runtime_checkable
class Sized(Protocol):
    def __len__(self) -> int: ...

isinstance([], Sized)       # True
isinstance("hello", Sized)  # True
isinstance(42, Sized)       # False
```

---

## `TypedDict` — Typed Dictionaries

```python
from typing import TypedDict, Required, NotRequired

class UserDict(TypedDict):
    id:    int
    name:  str
    email: str

# With optional fields (Python 3.11+)
class UserDict(TypedDict, total=False):
    id:    int         # optional
    name:  str         # optional

# Mixed required and optional
class UserDict(TypedDict):
    id:    Required[int]        # always required
    name:  Required[str]        # always required
    bio:   NotRequired[str]     # optional

# Usage
user: UserDict = {"id": 1, "name": "Alice", "email": "alice@example.com"}
```

---

## `NamedTuple` — Typed Named Tuples

```python
from typing import NamedTuple

class Point(NamedTuple):
    x: float
    y: float
    z: float = 0.0          # default value

p = Point(1.0, 2.0)
p.x                         # 1.0
p[0]                        # 1.0  — tuple access still works
x, y, z = p                # unpacking

# Immutable — like a frozen dataclass
# p.x = 5.0  → AttributeError
```

---

## `Literal` — Specific Values

```python
from typing import Literal

def set_direction(direction: Literal["north", "south", "east", "west"]) -> None: ...

def http_method(method: Literal["GET", "POST", "PUT", "PATCH", "DELETE"]) -> None: ...

# Combine with Union
def align(value: Literal["left", "center", "right"]) -> None: ...

Mode = Literal["r", "w", "rb", "wb", "a"]

def open_file(path: str, mode: Mode = "r") -> None: ...
```

---

## `Final` — Constants

```python
from typing import Final

MAX_SIZE:  Final = 100
API_URL:   Final[str] = "https://api.example.com"

# Prevents reassignment (caught by type checkers)
MAX_SIZE = 200      # mypy error: Cannot assign to final name "MAX_SIZE"

# Final class — prevents subclassing
from typing import final

@final
class Singleton:
    pass

# class Child(Singleton): ...  # mypy error
```

---

## `ClassVar` — Class-Level Variables

```python
from typing import ClassVar
from dataclasses import dataclass

@dataclass
class Config:
    VERSION: ClassVar[str] = "1.0.0"    # class variable, not instance field
    name:    str = "default"

Config.VERSION                  # "1.0.0"
Config(name="test").VERSION     # "1.0.0"
# dataclass __init__ ignores ClassVar fields
```

---

## `Annotated` — Metadata on Types

Attach arbitrary metadata to a type hint — used by Pydantic, FastAPI, and other libraries to add validation or documentation.

```python
from typing import Annotated
from pydantic import Field

# Reusable constrained types
PositiveInt   = Annotated[int,   Field(gt=0)]
NonEmptyStr   = Annotated[str,   Field(min_length=1)]
Percentage    = Annotated[float, Field(ge=0.0, le=100.0)]
Email         = Annotated[str,   Field(pattern=r"^[^@]+@[^@]+\.[^@]+$")]

from pydantic import BaseModel

class Product(BaseModel):
    name:     NonEmptyStr
    price:    PositiveInt
    discount: Percentage

# FastAPI uses Annotated for dependency injection
from fastapi import Depends, Query

def pagination(
    skip:  Annotated[int, Query(ge=0)]       = 0,
    limit: Annotated[int, Query(ge=1, le=100)] = 20,
) -> dict:
    return {"skip": skip, "limit": limit}
```

---

## `overload` — Multiple Signatures

```python
from typing import overload

@overload
def process(value: int) -> int: ...
@overload
def process(value: str) -> str: ...
@overload
def process(value: list) -> list: ...

def process(value):             # actual implementation — untyped
    if isinstance(value, int):
        return value * 2
    if isinstance(value, str):
        return value.upper()
    return [x for x in value]

result: int = process(5)        # type checker knows return is int
result: str = process("hi")    # type checker knows return is str
```

---

## `cast` — Type Narrowing

```python
from typing import cast

# Tell the type checker "trust me, this is X"
value: object = get_something()
name: str = cast(str, value)    # no runtime effect — just a type hint

# Prefer isinstance narrowing when possible
if isinstance(value, str):
    name = value                # type narrowed automatically
```

---

## `TYPE_CHECKING` — Avoid Circular Imports

```python
from __future__ import annotations  # make all annotations strings (lazy eval)
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from myapp.models import User   # only imported for type checking, not runtime

def get_user(user_id: int) -> "User":   # quoted if not using __future__
    ...
```

---

## `ParamSpec` — Generic Callables (3.10+)

Preserve the parameter types of a callable through decorators.

```python
from typing import ParamSpec, TypeVar, Callable
import functools

P = ParamSpec("P")
T = TypeVar("T")

def logged(fn: Callable[P, T]) -> Callable[P, T]:
    @functools.wraps(fn)
    def wrapper(*args: P.args, **kwargs: P.kwargs) -> T:
        print(f"Calling {fn.__name__}")
        result = fn(*args, **kwargs)
        print(f"Done {fn.__name__}")
        return result
    return wrapper

@logged
def add(a: int, b: int) -> int:
    return a + b

add(1, 2)   # type checker knows add still takes (int, int) -> int
```

---

## `Self` — Return Type of Current Class (3.11+)

```python
from typing import Self

class Builder:
    def set_name(self, name: str) -> Self:
        self.name = name
        return self

    def set_age(self, age: int) -> Self:
        self.age = age
        return self

class ExtendedBuilder(Builder):
    def set_role(self, role: str) -> Self:
        self.role = role
        return self

# Works correctly with subclasses
builder = ExtendedBuilder().set_name("Alice").set_age(30).set_role("admin")
# type checker knows builder is ExtendedBuilder, not Builder
```

---

## `Unpack` & `TypeVarTuple` — Variadic Generics (3.11+)

```python
from typing import TypeVarTuple, Unpack

Ts = TypeVarTuple("Ts")

def zip_equal(*iterables: Unpack[tuple[*Ts]]) -> zip[tuple[*Ts]]:
    return zip(*iterables)
```

---

## Type Aliases

```python
from typing import TypeAlias

# Simple alias
Vector:  TypeAlias = list[float]
Matrix:  TypeAlias = list[list[float]]
JSON:    TypeAlias = dict[str, "JSON"] | list["JSON"] | str | int | float | bool | None

# Python 3.12+ — type statement
type Vector = list[float]
type Matrix = list[list[float]]

# Usage
def dot_product(a: Vector, b: Vector) -> float:
    return sum(x * y for x, y in zip(a, b))

def add_matrices(a: Matrix, b: Matrix) -> Matrix:
    return [[a[i][j] + b[i][j] for j in range(len(a[0]))] for i in range(len(a))]
```

---

## `NewType` — Distinct Types

```python
from typing import NewType

UserId  = NewType("UserId",  int)
OrderId = NewType("OrderId", int)

def get_user(user_id: UserId) -> None: ...
def get_order(order_id: OrderId) -> None: ...

uid = UserId(42)
oid = OrderId(42)

get_user(uid)       # ✅
get_user(oid)       # ❌ mypy error — wrong type even though both are int
get_user(42)        # ❌ mypy error
```

---

## Quick Reference

|Type|Usage|
|---|---|
|`list[int]`|List of ints|
|`dict[str, int]`|Dict with str keys and int values|
|`tuple[int, str]`|Fixed tuple|
|`tuple[int, ...]`|Variable-length tuple of ints|
|`set[str]`|Set of strings|
|`Optional[X]` / `X \| None`|X or None|
|`Union[X, Y]` / `X \| Y`|X or Y|
|`Any`|Any type|
|`Callable[[X], Y]`|Function taking X, returning Y|
|`TypeVar("T")`|Generic type variable|
|`Generic[T]`|Generic class|
|`Protocol`|Structural interface|
|`TypedDict`|Typed dict|
|`NamedTuple`|Typed named tuple|
|`Literal["a","b"]`|One of specific values|
|`Final`|Constant|
|`ClassVar[T]`|Class-level variable|
|`Annotated[T, ...]`|Type + metadata|
|`overload`|Multiple function signatures|
|`cast(T, value)`|Type narrowing hint|
|`NewType("X", T)`|Distinct subtype|
|`TypeAlias`|Named type alias|
|`ParamSpec`|Preserve callable params|
|`Self`|Current class return type|

---

## Tags

#python #typing #type-hints #mypy #pyright #generics #stdlib #backend