## What & When

**Hash maps** (Python `dict`, `collections.defaultdict`, `Counter`) give **O(1)** average lookup, insert, and count — the default tool for frequency, dedup, and "complement" problems.

Use when:

- Count occurrences or track last-seen index
- Two-sum / subarray-sum-k with running total
- Group anagrams, first unique character
- Cache results (memoization toward DP)

Hub: [[Algorithms]]. See [[Codes/Algorithms — Prefix Sum]] for subarray-sum-k pattern.

```python
from collections import Counter, defaultdict
```

---

## Frequency Count

```python
from collections import defaultdict

def count_freq(arr: list[int]) -> dict[int, int]:
    freq: dict[int, int] = defaultdict(int)
    for x in arr:
        freq[x] += 1
    return dict(freq)


# Example
arr = [1, 2, 2, 3, 1, 4, 2]
assert count_freq(arr) == {1: 2, 2: 3, 3: 1, 4: 1}
```

```python
from collections import Counter

Counter([1, 2, 2, 3, 1, 4, 2])
# Counter({2: 3, 1: 2, 3: 1, 4: 1})
```

---

## Two Sum

```python
def two_sum(nums: list[int], target: int) -> tuple[int, int] | None:
    seen: dict[int, int] = {}   # value → index
    for i, x in enumerate(nums):
        need = target - x
        if need in seen:
            return seen[need], i
        seen[x] = i
    return None


assert two_sum([2, 7, 11, 15], 9) == (0, 1)
```

---

## First Non-Repeating

```python
def first_unique(s: str) -> str:
    freq = Counter(s)
    for ch in s:
        if freq[ch] == 1:
            return ch
    return ""
```

---

## Group Anagrams

```python
from collections import defaultdict

def group_anagrams(words: list[str]) -> list[list[str]]:
    groups: dict[tuple[str, ...], list[str]] = defaultdict(list)
    for w in words:
        key = tuple(sorted(w))
        groups[key].append(w)
    return list(groups.values())
```

---

## Complexity

| Operation | Average | Worst (hash collisions) |
| --- | --- | --- |
| get / set | O(1) | O(n) |
| Full pass count | O(n) space O(unique) | |

---

## Hash Map vs Other Patterns

| Signal | Use |
| --- | --- |
| Sorted array, pair sum | [[Codes/Algorithms — Two Pointers]] |
| Unsorted, need complement | **Hash map** |
| Range sums many times | [[Codes/Algorithms — Prefix Sum]] |

---

## Quick Reference

| Task | Pattern |
| --- | --- |
| Frequency | `Counter` or `defaultdict(int)` |
| Index lookup | `dict[value] = index` |
| Complement | `target - x in seen` |
| Memo DP | `cache[key] = result` |

---

## Related Notes

- [[Algorithms]]
- [[Codes/Algorithms — Prefix Sum]]
- [[Codes/Algorithms — Two Pointers]]
- [[Python Development]]

---

## Tags

#algorithms #hash-map #dictionary #python #interview #two-sum #frequency
