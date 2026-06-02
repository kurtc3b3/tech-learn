## What & When

**itertools** is Python's standard library module for **iterator algebra** — building efficient, memory-friendly pipelines over sequences. Instead of creating intermediate lists, you chain lazy iterators: filter, slice, group, combine, and batch data in one pass.

Use itertools when:

- Processing large datasets without loading everything into memory
- Building ETL/scraper pipelines (batch → transform → write)
- Grouping, windowing, or flattening iterables
- Generating combinations/permutations for tests or cartesian products
- Paginating or slicing lazy streams (`islice`)
- Chaining multiple iterables into one stream

```python
from itertools import (
    chain, islice, groupby, batched,
    accumulate, pairwise, zip_longest,
    product, combinations, permutations,
    cycle, repeat, count,
    starmap, tee, compress,
    dropwhile, takewhile, filterfalse,
)
```

No installation needed — `itertools` is part of the standard library.

For folding to a single value, see `functools.reduce` in [[Python — functools]]. For async streams, use [[Python — asyncio]] — itertools is sync-only.

---

## itertools vs Alternatives

| Need | Prefer | Notes |
| --- | --- | --- |
| Lazy pipeline over a list | itertools | No intermediate copies |
| Simple one-liner transform | list comprehension | Often clearer for small data |
| Parallel map over iterable | `map()` / asyncio | itertools stays single-threaded |
| Batch DB inserts | `batched` / `islice` | Common backend pattern |
| Group records by key | `groupby` | **Requires sorted input** |
| Cache iterator for reuse | `tee` or `list()` | Iterators are single-pass by default |

---

## Core Idea — Iterators Are Single-Pass

```python
data = iter([1, 2, 3])

list(data)   # [1, 2, 3]
list(data)   # [] — exhausted; create a new iterator or use tee/list
```

Most itertools functions return iterators. Consume once, or materialize with `list()` when you need multiple passes.

---

## `chain` — Flatten Iterables

Join iterables end-to-end without building a nested list first.

```python
from itertools import chain

pages = [
    [{"id": 1}, {"id": 2}],
    [{"id": 3}],
]

all_items = list(chain.from_iterable(pages))
# [{"id": 1}, {"id": 2}, {"id": 3}]

# chain(a, b, c) — flat args, not nested
list(chain([1, 2], [3, 4], [5]))        # [1, 2, 3, 4, 5]
```

```python
# Flatten dict values
by_category = {"a": [1, 2], "b": [3]}
list(chain.from_iterable(by_category.values()))    # [1, 2, 3]
```

---

## `islice` — Slice a Lazy Stream

Like list slicing, but on any iterator — works with infinite streams.

```python
from itertools import islice, count

# First 5 items
list(islice(range(100), 5))             # [0, 1, 2, 3, 4]

# Skip 10, take 5
list(islice(range(100), 10, 15))       # [10, 11, 12, 13, 14]

# Every 2nd item (step)
list(islice(range(10), 0, None, 2))    # [0, 2, 4, 6, 8]

# From infinite counter
list(islice(count(10, 2), 4))          # [10, 12, 14, 16]
```

```python
# Pagination pattern
def paginate(iterable, page_size: int):
    it = iter(iterable)
    while batch := list(islice(it, page_size)):
        yield batch

list(paginate(range(7), page_size=3))
# [[0, 1, 2], [3, 4, 5], [6]]
```

---

## `batched` — Fixed-Size Chunks (Python 3.12+)

Clean replacement for manual `islice` batching.

```python
from itertools import batched

list(batched("ABCDEFG", 3))
# [('A', 'B', 'C'), ('D', 'E', 'F'), ('G',)]

list(batched(range(10), 4))
# [(0, 1, 2, 3), (4, 5, 6, 7), (8, 9)]
```

```python
# Pre-3.12 equivalent
def batched(iterable, n):
    it = iter(iterable)
    while chunk := tuple(islice(it, n)):
        yield chunk
```

```python
# Bulk DB insert — process 500 rows at a time
for chunk in batched(rows, 500):
    session.execute(insert(User), [dict(r) for r in chunk])
    session.commit()
```

---

## `groupby` — Group Consecutive Equal Keys

**Critical:** `groupby` only groups **adjacent** items with the same key. Sort (or pre-group) first.

```python
from itertools import groupby
from operator import itemgetter

rows = [
    {"dept": "eng", "name": "Alice"},
    {"dept": "eng", "name": "Bob"},
    {"dept": "sales", "name": "Carol"},
    {"dept": "sales", "name": "Dave"},
]

# WRONG — unsorted input splits groups incorrectly
# Always sort by the group key first
rows.sort(key=itemgetter("dept"))

for dept, group in groupby(rows, key=itemgetter("dept")):
    names = [r["name"] for r in group]
    print(dept, names)
# eng ['Alice', 'Bob']
# sales ['Carol', 'Dave']
```

```python
# Group lines in a log file by level (after sorting)
from itertools import groupby

lines = sorted(log_lines, key=lambda l: l.level)
for level, group in groupby(lines, key=lambda l: l.level):
    print(level, sum(1 for _ in group))
```

For arbitrary grouping without sorting, use `dict` + loop or `collections.defaultdict`.

---

## `accumulate` — Running Reduction

Yield running totals, running max, string concatenation, etc.

```python
from itertools import accumulate
import operator

list(accumulate([1, 2, 3, 4, 5]))                    # [1, 3, 6, 10, 15]
list(accumulate([1, 2, 3, 4, 5], operator.mul))     # [1, 2, 6, 24, 120]
list(accumulate([3, 1, 4, 1, 5], max))              # [3, 3, 4, 4, 5]

# Running join
list(accumulate(["a", "b", "c"], operator.concat))  # ['a', 'ab', 'abc']
```

---

## `pairwise` — Sliding Window of Pairs (Python 3.10+)

```python
from itertools import pairwise

list(pairwise([1, 2, 3, 4]))
# [(1, 2), (2, 3), (3, 4)]

# Detect consecutive duplicates or gaps
for a, b in pairwise(timestamps):
    if (b - a).total_seconds() > 3600:
        flag_gap(a, b)
```

```python
# Pre-3.10 equivalent
def pairwise(iterable):
    a, b = tee(iterable)
    next(b, None)
    return zip(a, b)
```

---

## Windowed Iteration — `tee` + `islice`

General sliding window of size `n`:

```python
from itertools import tee, islice

def windowed(iterable, n: int):
    iterators = tee(iterable, n)
    for i, it in enumerate(iterators):
        for _ in range(i):
            next(it, None)
    return zip(*iterators)

list(windowed([1, 2, 3, 4, 5], 3))
# [(1, 2, 3), (2, 3, 4), (3, 4, 5)]
```

> [!tip] `tee` stores consumed values — use sparingly on large streams Each branch duplicates memory for cached items. For huge data, prefer index-based slicing on sequences or manual loops.

---

## `zip_longest` — Parallel Iteration with Padding

```python
from itertools import zip_longest

names  = ["Alice", "Bob", "Carol"]
scores = [95, 87]

list(zip_longest(names, scores, fillvalue=0))
# [('Alice', 95), ('Bob', 87), ('Carol', 0)]

# Pad to longest — useful for merging columns
list(zip_longest([1, 2], [3, 4, 5], [6]))
# [(1, 3, 6), (2, 4, None), (None, 5, None)]
```

---

## `starmap` — Unpack Tuple Arguments

Like `map`, but each item is unpacked as `*args`.

```python
from itertools import starmap

pairs = [(2, 3), (4, 5), (10, 1)]
list(starmap(pow, pairs))       # [8, 1024, 10]

list(starmap(lambda a, b: a + b, [(1, 2), (3, 4)]))   # [3, 7]
```

---

## `product` — Cartesian Product

```python
from itertools import product

list(product([1, 2], ["a", "b"]))
# [(1, 'a'), (1, 'b'), (2, 'a'), (2, 'b')]

list(product("AB", repeat=2))
# [('A', 'A'), ('A', 'B'), ('B', 'A'), ('B', 'B')]

# Generate test matrix — all param combos
params = product(
    ["sqlite", "postgres"],
    [True, False],          # debug
    [1, 10, 100],           # pool size
)
for db, debug, pool in params:
    run_integration_test(db=db, debug=debug, pool=pool)
```

---

## `combinations` & `permutations`

```python
from itertools import combinations, combinations_with_replacement, permutations

list(combinations("ABC", 2))
# [('A', 'B'), ('A', 'C'), ('B', 'C')]

list(combinations_with_replacement("AB", 2))
# [('A', 'A'), ('A', 'B'), ('B', 'B')]

list(permutations("AB", 2))
# [('A', 'B'), ('B', 'A')]
```

Useful for test case generation and combinatorial APIs — not for large `n` (explodes quickly).

---

## `compress` — Filter by Boolean Selector

```python
from itertools import compress

data = ["A", "B", "C", "D", "E"]
select = [1, 0, 1, 0, 1]        # truthy = keep

list(compress(data, select))     # ['A', 'C', 'E']
list(compress(data, (True, False, True, False, True)))
```

---

## `dropwhile` & `takewhile`

Take or skip elements while a predicate holds — **only from the start** of the iterator.

```python
from itertools import dropwhile, takewhile

list(dropwhile(lambda x: x < 5, [1, 3, 6, 2, 8]))
# [6, 2, 8]  — drops 1,3 then stops dropping (2 stays even though < 5)

list(takewhile(lambda x: x < 5, [1, 3, 4, 6, 2]))
# [1, 3, 4]  — stops at first failure
```

For filtering all elements, use built-in `filter()`.

---

## `filterfalse` — Inverse of `filter`

```python
from itertools import filterfalse

list(filterfalse(lambda x: x % 2 == 0, range(10)))
# [1, 3, 5, 7, 9]
```

---

## Infinite Iterators — `count`, `cycle`, `repeat`

```python
from itertools import count, cycle, repeat, islice

# count(start=0, step=1) — infinite arithmetic sequence
list(islice(count(10, 2), 5))     # [10, 12, 14, 16, 18]

# cycle — repeat sequence forever
list(islice(cycle("AB"), 7))       # ['A', 'B', 'A', 'B', 'A', 'B', 'A']

# repeat object n times (or forever if n omitted)
list(repeat(7, 3))                 # [7, 7, 7]
```

```python
# Enumerate with custom start using zip
for i, row in zip(count(1), rows):
    process(row_id=i, row=row)
```

---

## Pipeline Patterns (Backend)

### Scrape → flatten → batch → persist

```python
from itertools import chain, batched

def scrape_all(urls: list[str]):
    return (fetch(url) for url in urls)          # generator per URL

pages = scrape_all(url_list)
items = chain.from_iterable(pages)                # flatten page results

for chunk in batched(items, 100):
    db.bulk_insert(list(chunk))
```

See [[Python — httpx Package]], [[Python — BeautifulSoup4 (bs4)]].

### Read large file in chunks

```python
from itertools import islice

def read_in_chunks(file_obj, chunk_size: int = 8192):
    while chunk := file_obj.read(chunk_size):
        yield chunk

# Line-based batching
def line_batches(lines, n: int):
    it = iter(lines)
    while batch := list(islice(it, n)):
        yield batch
```

See [[Python — pathlib]].

### Rate-limited iteration with `islice` + manual throttle

```python
from itertools import islice
import time

def throttled(iterable, rate_per_sec: float):
    interval = 1.0 / rate_per_sec
    for item in iterable:
        yield item
        time.sleep(interval)

urls = ["https://example.com/1", "..."]
for url in islice(throttled(urls, 2), 50):   # max 50 URLs, 2/sec
    fetch(url)
```

Combine with [[Python — tenacity]] for retries on individual items.

### Unique consecutive dedup (adjacent only)

```python
from itertools import groupby

def consecutive_unique(iterable, key=None):
    return (k for k, _ in groupby(iterable, key=key))

list(consecutive_unique([1, 1, 2, 2, 2, 1, 3]))
# [1, 2, 1, 3]
```

For full dedup, use `set()` or `dict.fromkeys()`.

---

## itertools + functools

```python
from itertools import groupby
from functools import reduce
import operator

# Longest streak of same value (sorted input required)
data = sorted([1, 1, 2, 2, 2, 3])
streaks = [(k, len(list(g))) for k, g in groupby(data)]
longest = reduce(lambda a, b: a if a[1] >= b[1] else b, streaks)
```

See [[Python — functools]].

---

## Common Mistakes

| Mistake | Fix |
| --- | --- |
| `groupby` without sorting | `data.sort(key=...)` or `sorted(data, key=...)` first |
| Reusing exhausted iterator | `list(it)` or `tee()` before second pass |
| Huge `product` / `permutations` | Cap `n`; estimate size before materializing |
| `dropwhile` to filter all items | Use `filter()` instead |
| `tee` on very large streams | Memory grows — batch or use indexes |

---

## Quick Reference

| Task | Code |
| --- | --- |
| Flatten nested lists | `chain.from_iterable(nested)` |
| First N items | `islice(it, n)` |
| Skip/take slice | `islice(it, start, stop)` |
| Batch/chunk | `batched(it, n)` (3.12+) |
| Group adjacent keys | `groupby(sorted_data, key=fn)` |
| Running total | `accumulate(nums)` |
| Sliding pairs | `pairwise(it)` |
| Sliding window n | `windowed(it, n)` via `tee` |
| Zip uneven lengths | `zip_longest(a, b, fillvalue=x)` |
| Cartesian product | `product(a, b)` |
| Combinations | `combinations(items, r)` |
| Unpack arg tuples | `starmap(func, pairs)` |
| Filter by mask | `compress(data, selectors)` |
| Infinite counter | `count(start, step)` |
| Repeat value | `repeat(x, n)` |

---

## Tags

#python #itertools #stdlib #iterators #pipelines #batching #backend
