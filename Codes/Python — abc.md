## What & When

**abc** (Abstract Base Classes) is Python's standard library module for defining **interfaces with enforcement** — base classes that declare required methods and refuse instantiation until every abstract member is implemented.

Use abc when:

- Defining contracts for plugins, storage backends, scrapers, or notifiers
- Sharing algorithm skeletons while subclasses fill in steps (template method)
- Preventing incomplete implementations at instantiation time
- Building registries of interchangeable components
- Documenting required methods for third-party extensions

```python
from abc import ABC, abstractmethod, abstractclassmethod, abstractstaticmethod
```

No installation needed — `abc` is part of the standard library.

For **structural** typing (duck typing without inheritance), prefer `Protocol` in [[Python — typing]]. ABC and Protocol complement each other — ABC enforces at runtime; Protocol checks with type checkers.

Combine with decorators via [[Python — functools]] (`@wraps`) and lifecycle hooks via [[Python — contextlib]].

---

## abc vs Protocol vs Plain Base Class

| Approach | Enforcement | Inheritance required | Best for |
| --- | --- | --- | --- |
| `abc.ABC` | Runtime — `TypeError` if abstract methods missing | Yes | Plugins, frameworks, shared bases |
| `typing.Protocol` | Static (mypy/pyright); optional `@runtime_checkable` | No | Duck-typed dependencies, DI |
| Plain base class | None — empty methods silently pass | Optional | Documentation only — avoid for contracts |

---

## Basic ABC

```python
from abc import ABC, abstractmethod

class Storage(ABC):
    @abstractmethod
    def save(self, key: str, data: bytes) -> None:
        """Persist data under key."""

    @abstractmethod
    def load(self, key: str) -> bytes:
        """Load data by key."""

class FileStorage(Storage):
    def save(self, key: str, data: bytes) -> None:
        Path(key).write_bytes(data)

    def load(self, key: str) -> bytes:
        return Path(key).read_bytes()

# storage = Storage()              # TypeError: Can't instantiate abstract class
fs = FileStorage()                 # OK
```

Missing even one `@abstractmethod` prevents instantiation:

```python
class BrokenStorage(Storage):
    def save(self, key: str, data: bytes) -> None:
        ...

# BrokenStorage()  # TypeError: Can't instantiate abstract class BrokenStorage
#                  with abstract method load
```

---

## `@abstractmethod` — Required Methods

```python
from abc import ABC, abstractmethod

class Notifier(ABC):
    @abstractmethod
    def send(self, recipient: str, message: str) -> None:
        ...

    def send_bulk(self, recipients: list[str], message: str) -> None:
        """Concrete helper — not abstract."""
        for r in recipients:
            self.send(r, message)
```

Abstract methods can have a body (Python 3.3+) — subclasses must still override, but can call `super()`:

```python
class BaseScraper(ABC):
    @abstractmethod
    def parse(self, html: str) -> dict:
        """Subclasses must implement."""
        raise NotImplementedError
```

---

## Class Methods & Static Methods

```python
from abc import ABC, abstractclassmethod, abstractstaticmethod

class ConfigSource(ABC):
    @abstractclassmethod
    @abstractmethod
    def from_env(cls) -> "ConfigSource":
        ...

    @abstractstaticmethod
    @abstractmethod
    def schema_version() -> str:
        ...
```

> [!tip] Decorator order For `@abstractclassmethod` / `@abstractstaticmethod`, place `@abstractmethod` **innermost** (closest to the function) in Python 3.3+ style, or use the combined decorators from `abc` directly as shown above.

---

## Properties

```python
from abc import ABC, abstractmethod

class Cache(ABC):
    @property
    @abstractmethod
    def backend_name(self) -> str:
        ...

class RedisCache(Cache):
    @property
    def backend_name(self) -> str:
        return "redis"
```

---

## Async Abstract Methods

Works naturally with `async def` — common for HTTP/storage backends.

```python
from abc import ABC, abstractmethod

class HttpFetcher(ABC):
    @abstractmethod
    async def fetch(self, url: str) -> str:
        ...

class HttpxFetcher(HttpFetcher):
    def __init__(self, client):
        self.client = client

    async def fetch(self, url: str) -> str:
        response = await self.client.get(url)
        response.raise_for_status()
        return response.text
```

See [[Python — httpx Package]], [[Python — asyncio]].

---

## Template Method Pattern

ABC defines the workflow; subclasses override hooks.

```python
from abc import ABC, abstractmethod

class BaseScraper(ABC):
    def run(self, url: str) -> dict:
        """Template method — fixed pipeline."""
        html = self.fetch(url)
        data = self.parse(html)
        return self.normalize(data)

    @abstractmethod
    def fetch(self, url: str) -> str:
        ...

    @abstractmethod
    def parse(self, html: str) -> dict:
        ...

    def normalize(self, data: dict) -> dict:
        """Optional override — default pass-through."""
        return data

class BooksScraper(BaseScraper):
    def fetch(self, url: str) -> str:
        return httpx.get(url).text

    def parse(self, html: str) -> dict:
        soup = BeautifulSoup(html, "lxml")
        return {"title": soup.find("h1").get_text(strip=True)}
```

See [[Python — BeautifulSoup4 (bs4)]], [[Python — tenacity]] for retry on `fetch`.

---

## Virtual Subclasses — `register()`

Declare that an unrelated class conforms to an ABC without inheritance.

```python
from abc import ABC, abstractmethod

class Writable(ABC):
    @abstractmethod
    def write(self, data: bytes) -> int:
        ...

@Writable.register
class SocketWrapper:
    def write(self, data: bytes) -> int:
        return self._sock.send(data)

issubclass(SocketWrapper, Writable)    # True
isinstance(SocketWrapper(), Writable)  # True
```

Use sparingly — prefer explicit inheritance for clarity.

---

## `__subclasshook__` — Custom Subclass Checks

```python
from abc import ABC, abstractmethod

class Closeable(ABC):
    @abstractmethod
    def close(self) -> None:
        ...

    @classmethod
    def __subclasshook__(cls, other):
        if cls is Closeable:
            return hasattr(other, "close") and callable(other.close)
        return NotImplemented
```

Related to structural typing — for new code, `Protocol` is often simpler. See [[Python — typing]].

---

## Plugin Registry Pattern

```python
from abc import ABC, abstractmethod

class Exporter(ABC):
    registry: dict[str, type["Exporter"]] = {}

    def __init_subclass__(cls, *, format_name: str, **kwargs):
        super().__init_subclass__(**kwargs)
        cls.registry[format_name] = cls

    @abstractmethod
    def export(self, rows: list[dict]) -> bytes:
        ...

class CsvExporter(Exporter, format_name="csv"):
    def export(self, rows: list[dict]) -> bytes:
        ...

class JsonExporter(Exporter, format_name="json"):
    def export(self, rows: list[dict]) -> bytes:
        ...

def get_exporter(name: str) -> Exporter:
    return Exporter.registry[name]()
```

---

## ABC + Dependency Injection

```python
from abc import ABC, abstractmethod

class UserRepository(ABC):
    @abstractmethod
    async def get_by_id(self, user_id: int) -> User | None:
        ...

class SqlUserRepository(UserRepository):
    def __init__(self, session):
        self.session = session

    async def get_by_id(self, user_id: int) -> User | None:
        return await self.session.get(User, user_id)

class UserService:
    def __init__(self, repo: UserRepository):
        self.repo = repo

    async def get_user(self, user_id: int):
        return await self.repo.get_by_id(user_id)
```

Tests inject a fake repository without subclassing production code:

```python
class FakeUserRepository(UserRepository):
    async def get_by_id(self, user_id: int):
        return User(id=user_id, name="Test")

service = UserService(FakeUserRepository())
```

See [[ORM - CRUD]], [[Unit Testing - pytest]].

---

## ABC vs `Protocol` — When to Use Which

```python
# ABC — explicit contract, runtime check on instantiate
class Storage(ABC):
    @abstractmethod
    def save(self, key: str, data: bytes) -> None: ...

# Protocol — any class with matching methods satisfies the type
from typing import Protocol

class StorageProto(Protocol):
    def save(self, key: str, data: bytes) -> None: ...
    def load(self, key: str) -> bytes: ...
```

| Use ABC when | Use Protocol when |
| --- | --- |
| You own the inheritance tree | Third-party types must satisfy the interface |
| Plugins must register explicitly | Duck typing is enough |
| Runtime failure on incomplete class is desired | Only static type checking needed |
| Template method with shared concrete code | Function accepts minimal surface area |

---

## Testing ABC Implementations

```python
import pytest
from abc import ABC, abstractmethod

class Parser(ABC):
    @abstractmethod
    def parse(self, text: str) -> dict:
        ...

@pytest.fixture(params=[MyParser, AnotherParser])
def parser_impl(request):
    return request()

def test_parser_returns_dict(parser_impl):
    result = parser_impl.parse("input")
    assert isinstance(result, dict)
```

Ensure each concrete subclass implements all abstract members — mypy/ruff help catch gaps before runtime.

---

## Common Mistakes

| Mistake | Fix |
| --- | --- |
| Forgetting `@abstractmethod` on interface method | Subclass instantiates with empty base implementation |
| Abstract class with no abstract methods | Use regular class — ABC adds no value |
| Calling abstract method from `__init__` before subclass is ready | Defer to concrete methods after full init |
| Mixing ABC inheritance with unrelated `register()` | Prefer one registration style |
| Using ABC for every interface | Use `Protocol` for simple duck-typed deps |

---

## Common Backend Patterns

### Storage backend swap (local / S3)

```python
class BlobStore(ABC):
    @abstractmethod
    async def put(self, key: str, body: bytes) -> None: ...

    @abstractmethod
    async def get(self, key: str) -> bytes: ...
```

### Notification channel (email / Slack / webhook)

```python
class Notifier(ABC):
    @abstractmethod
    async def notify(self, subject: str, body: str) -> None: ...
```

### Scraper family with shared `run()` pipeline

See template method example above — pairs with [[Python — markdownify]] for normalize step.

### Service layer over repository ABC

Keeps FastAPI routes thin — [[API - FastAPI — Dependency Injection & User Management]].

---

## Quick Reference

| Task | Code |
| --- | --- |
| Define interface | `class Foo(ABC):` |
| Required method | `@abstractmethod def bar(self): ...` |
| Required async method | `@abstractmethod async def bar(self): ...` |
| Required property | `@property @abstractmethod def x(self): ...` |
| Cannot instantiate | Until all abstract members implemented |
| Virtual subclass | `@Interface.register class Impl: ...` |
| Template method | Concrete `run()` calls abstract hooks |
| Plugin registry | `__init_subclass__` + abstract base |
| Duck typing alternative | `Protocol` — [[Python — typing]] |

---

## Tags

#python #abc #stdlib #abstract-base-class #interfaces #design-patterns #plugins #backend
