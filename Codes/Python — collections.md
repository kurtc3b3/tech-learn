## What & When

**collections** is Python's standard library module of **specialized container types** — alternatives to plain `dict`, `list`, and `tuple` when you need counting, default values, fixed-size queues, ordered keys, or lightweight named records.

Use collections when:

- Counting occurrences or finding top-N items (`Counter`)
- Grouping items without checking key existence (`defaultdict`)
- FIFO/LIFO queues or sliding windows (`deque`)
- Lightweight immutable records with field names (`namedtuple`)
- Layered configuration lookup (`ChainMap`)
- Dict insertion order with `move_to_end` (`OrderedDict`)

```python
from collections import (
    Counter, defaultdict, deque, OrderedDict,
    namedtuple, ChainMap,
    UserDict, UserList, UserString,
)
```

No installation needed — `collections` is part of the standard library.

For typed immutable records with validation, prefer [[Python — dataclasses]] or [[Python — Pydantic]]. For lazy grouping of sorted streams, see `groupby` in [[Python — itertools]].

---

## collections vs Built-ins

| Need | Use | Instead of |
| --- | --- | --- |
| Count frequencies | `Counter` | Manual `dict` + increment loop |
| Group by key | `defaultdict(list)` | `if key not in d: d[key] = []` |
| FIFO / bounded buffer | `deque` | `list.pop(0)` (slow) |
| Named fields, no methods | `namedtuple` | plain `tuple` or verbose class |
| Merge config layers | `ChainMap` | manual dict unpacking chain |
| Full validation / JSON | [[Python — Pydantic]] | `namedtuple` / `Counter` |

---

## `Counter` — Count Hashable Items

A dict subclass for counting — missing keys return `0`.

```python
from collections import Counter

words = ["apple", "banana", "apple", "cherry", "banana", "apple"]
counts = Counter(words)
# Counter({'apple': 3, 'banana': 2, 'cherry': 1})

counts["apple"]         # 3
counts["missing"]       # 0 — no KeyError

# Most common
counts.most_common(2)   # [('apple', 3), ('banana', 2)]
counts.most_common()    # all, descending frequency

# Update counts
counts.update(["apple", "date"])
counts["apple"] += 1

# Set-like operations on counters
c1 = Counter(a=3, b=1)
c2 = Counter(a=1, b=2, c=1)
c1 + c2                 # Counter({'a': 4, 'b': 3, 'c': 1})
c1 - c2                 # Counter({'a': 2}) — keeps only positive
c1 & c2                 # intersection — min counts
c1 | c2                 # union — max counts
```

```python
# Log analysis — HTTP status codes
statuses = [200, 200, 404, 500, 200, 404]
Counter(statuses).most_common()
# [(200, 3), (404, 2), (500, 1)]

# Word frequency in scraped text
from collections import Counter
import re

text = soup.get_text()
words = re.findall(r"\w+", text.lower())
top = Counter(words).most_common(20)
```

```python
# Counter as multiset / bag
inventory = Counter({"apple": 5, "banana": 3})
order = Counter({"apple": 2, "banana": 1})
remaining = inventory - order          # Counter({'apple': 3, 'banana': 2})
```

---

## `defaultdict` — Dict with Factory Defaults

Calls a factory function when a missing key is accessed — ideal for grouping and nested structures.

```python
from collections import defaultdict

# Group records by key
rows = [
    {"dept": "eng", "name": "Alice"},
    {"dept": "sales", "name": "Bob"},
    {"dept": "eng", "name": "Carol"},
]

by_dept = defaultdict(list)
for row in rows:
    by_dept[row["dept"]].append(row["name"])

dict(by_dept)
# {'eng': ['Alice', 'Carol'], 'sales': ['Bob']}
```

```python
# Nested defaultdict — tree / graph adjacency
def nested_dict():
    return defaultdict(nested_dict)

tree = nested_dict()
tree["users"]["alice"]["settings"]["theme"] = "dark"
```

```python
# Count without Counter
counts = defaultdict(int)
for item in items:
    counts[item] += 1
```

```python
# Common factories
defaultdict(list)       # append groups
defaultdict(set)        # unique groups
defaultdict(int)        # counters
defaultdict(float)      # sums
defaultdict(dict)       # two-level maps
```

> [!tip] `defaultdict` vs `itertools.groupby` Use `defaultdict(list)` when input is **not sorted** and you need all groups in one pass. Use [[Python — itertools]] `groupby` only on **sorted** adjacent runs.

---

## `deque` — Double-Ended Queue

O(1) append/pop from both ends — thread-safe appends/pops in CPython.

```python
from collections import deque

d = deque([1, 2, 3])
d.append(4)             # right: [1, 2, 3, 4]
d.appendleft(0)         # left:  [0, 1, 2, 3, 4]
d.pop()                 # 4
d.popleft()             # 0

d[0], d[-1]             # random access — O(n) at extremes still ok for small n
len(d)
"d" in deque("abcd")    # membership
```

```python
# Bounded history — last N requests (rate limiting, logging)
recent = deque(maxlen=100)
recent.append({"path": "/api/users", "ms": 42})
# auto-drops oldest when len > 100

# FIFO work queue
queue = deque()
queue.append(task)
task = queue.popleft()
```

```python
# Rotate — round-robin
workers = deque(["w1", "w2", "w3"])
workers.rotate(-1)      # w1 moves to end → w2, w3, w1
next_worker = workers[0]
```

```python
# BFS skeleton
def bfs(start, neighbors):
    seen = {start}
    q = deque([start])
    while q:
        node = q.popleft()
        for nxt in neighbors(node):
            if nxt not in seen:
                seen.add(nxt)
                q.append(nxt)
```

**Avoid** `list.pop(0)` in hot loops — it is O(n). Use `deque.popleft()`.

---

## `namedtuple` — Lightweight Named Records

Creates a tuple subclass with named fields — immutable and memory-efficient.

```python
from collections import namedtuple

Point = namedtuple("Point", ["x", "y"])
p = Point(10, 20)

p.x, p.y              # 10, 20
p[0], p[1]            # 10, 20 — still a tuple
p._asdict()           # {'x': 10, 'y': 20}
p._replace(x=15)      # Point(x=15, y=20) — new instance

# NamedTuple with defaults (Python 3.7+ factory pattern)
from typing import NamedTuple

class Point(NamedTuple):
    x: float
    y: float
    z: float = 0.0
```

```python
# Parse structured rows from CSV / DB
UserRow = namedtuple("UserRow", "id name email")

for row in cursor:
    user = UserRow(*row)
    process(user.id, user.name)
```

```python
# _make / _fields
Point._make([1, 2])           # Point(x=1, y=2)
Point._fields                 # ('x', 'y')
```

Use [[Python — dataclasses]] when you need mutability, defaults with factories, or `__post_init__`. Use `NamedTuple` when immutability and tuple unpacking matter.

---

## `OrderedDict` — Ordered Mapping

Since Python 3.7+, regular `dict` preserves insertion order. `OrderedDict` still adds:

- `move_to_end(key, last=True)` — reorder LRU-style
- Stable equality that considers order
- `popitem(last=False)` — FIFO pop

```python
from collections import OrderedDict

cache = OrderedDict()
cache["a"] = 1
cache["b"] = 2
cache["c"] = 3

cache.move_to_end("a")          # a moves to end — MRU
cache.popitem(last=False)       # pop FIFO — ('a', 1) if a was moved... depends on order

# Simple LRU cap
def lru_set(od: OrderedDict, key, value, maxsize: int):
    od[key] = value
    od.move_to_end(key)
    while len(od) > maxsize:
        od.popitem(last=False)
```

For production LRU caching, prefer `@lru_cache` from [[Python — functools]].

---

## `ChainMap` — Layered Lookup

Search multiple dicts/mappings in order — first match wins. Useful for scoped configuration.

```python
from collections import ChainMap

defaults = {"timeout": 30, "retries": 3, "debug": False}
env_overrides = {"debug": True}
cli_overrides = {"timeout": 5}

settings = ChainMap(cli_overrides, env_overrides, defaults)

settings["timeout"]     # 5  — from CLI
settings["retries"]     # 3  — from defaults
settings["debug"]       # True — from env

# New scope — child sees parent but writes local
child = settings.new_child({"timeout": 1})
child["timeout"]        # 1
child.maps              # list of dicts in search order
```

```python
# Template context — local vars shadow globals
globals_ctx = {"site_name": "Example", "year": 2026}
locals_ctx = {"title": "Home"}
ctx = ChainMap(locals_ctx, globals_ctx)
# render with ctx["title"], ctx["site_name"]
```

See [[Python — Jinja2 Package]] for template rendering.

---

## `UserDict`, `UserList`, `UserString`

Wrapper base classes for subclassing built-ins — delegate to inner `data` attribute. Prefer when you need custom dict/list behavior without fragile inheritance from `dict` itself.

```python
from collections import UserDict

class CaseInsensitiveDict(UserDict):
    def __getitem__(self, key):
        return self.data[key.lower()]

    def __setitem__(self, key, value):
        self.data[key.lower()] = value

d = CaseInsensitiveDict()
d["Host"] = "example.com"
d["host"]     # "example.com"
```

---

## Common Backend Patterns

### Group API results by category

```python
from collections import defaultdict

def group_by(items: list[dict], key: str) -> dict[str, list[dict]]:
    groups = defaultdict(list)
    for item in items:
        groups[item[key]].append(item)
    return dict(groups)
```

### Top-N error messages from logs

```python
from collections import Counter

def top_errors(log_lines: list[str], n: int = 10) -> list[tuple[str, int]]:
    return Counter(log_lines).most_common(n)
```

### Rate limiter with sliding window

```python
from collections import deque
import time

class SlidingWindowRateLimiter:
    def __init__(self, max_calls: int, window_sec: float):
        self.max_calls = max_calls
        self.window_sec = window_sec
        self.timestamps: deque[float] = deque()

    def allow(self) -> bool:
        now = time.monotonic()
        while self.timestamps and self.timestamps[0] <= now - self.window_sec:
            self.timestamps.popleft()
        if len(self.timestamps) >= self.max_calls:
            return False
        self.timestamps.append(now)
        return True
```

### Layered app settings

```python
from collections import ChainMap

def load_settings():
    file_cfg = load_yaml("config.yaml")
    env_cfg = os.environ  # or parsed env dict
    return ChainMap(env_cfg, file_cfg, DEFAULTS)
```

See [[Python — python-dotenv]], [[Python — Pydantic]].

### WebSocket connections by room

```python
from collections import defaultdict

connections: defaultdict[str, set] = defaultdict(set)

def connect(room: str, ws):
    connections[room].add(ws)

def disconnect(room: str, ws):
    connections[room].discard(ws)
```

See [[API - FastAPI — WebSockets]].

---

## Choosing the Right Container

| Scenario | Container |
| --- | --- |
| Count tag/status frequencies | `Counter` |
| Group unsorted records by key | `defaultdict(list)` |
| Job queue, BFS, recent N items | `deque` |
| Immutable row from query | `namedtuple` / `NamedTuple` |
| Config: CLI > env > defaults | `ChainMap` |
| LRU key reorder | `OrderedDict.move_to_end` |
| Custom dict behavior | `UserDict` subclass |

---

## Quick Reference

| Task | Code |
| --- | --- |
| Count items | `Counter(iterable)` |
| Top N frequent | `Counter(...).most_common(n)` |
| Group into lists | `defaultdict(list)` + append |
| Nested auto-vivify | `defaultdict(nested_factory)` |
| FIFO queue | `deque` + `append` / `popleft` |
| Last N items | `deque(maxlen=N)` |
| Named tuple row | `namedtuple("Row", "id name")` |
| Replace field | `row._replace(name="Bob")` |
| Layered config | `ChainMap(override, defaults)` |
| Reorder dict key | `OrderedDict.move_to_end(k)` |

---

## Tags

#python #collections #stdlib #data-structures #counter #defaultdict #deque #backend
