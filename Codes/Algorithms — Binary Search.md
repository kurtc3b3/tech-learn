## What & When

**Binary search** finds a target in a **sorted** array by halving the search range each step — O(log n) time, O(1) space.

Use when:

- Array (or answer space) is **sorted** or **monotonic**
- You need an index, boundary, or "first/last position where …"
- Brute-force scan would be O(n) on large input

Hub: [[Algorithms]]. Related: [[Codes/Algorithms — Two Pointers]] on sorted arrays.

```python
# stdlib alternative for insertion point:
from bisect import bisect_left
```

---

## Core Idea

```text
low=0, high=len-1
while low <= high:
    mid = (low + high) // 2
    if arr[mid] == target → found
    if arr[mid] < target  → search right half
    else                  → search left half
```

---

## Implementation

```python
def binary_search(arr: list[int], target: int) -> int:
    """Return index of target, or -1 if absent."""
    low, high = 0, len(arr) - 1
    while low <= high:
        mid = (low + high) // 2
        if arr[mid] == target:
            return mid
        if arr[mid] < target:
            low = mid + 1
        else:
            high = mid - 1
    return -1


# Example
arr = [1, 2, 3, 4, 5, 6, 7, 8]
assert binary_search(arr, 6) == 5
assert binary_search(arr, 9) == -1
```

---

## With `bisect` (stdlib)

```python
from bisect import bisect_left, bisect_right

arr = [1, 2, 2, 2, 5]
bisect_left(arr, 2)   # 1 — first index where 2 can be inserted
bisect_right(arr, 2)  # 4 — after existing 2s
```

Prefer `bisect` in production; implement by hand in interviews to show understanding.

---

## Variants

| Variant | Twist |
| --- | --- |
| First / last occurrence | Continue searching left or right after match |
| Search rotated sorted array | Compare which half is sorted |
| Binary search on answer | Monotonic predicate on integer range (min capacity, max speed) |

---

## Complexity

| | |
| --- | --- |
| **Time** | O(log n) |
| **Space** | O(1) |

---

## Common Mistakes

- Using `mid = (low + high) / 2` on huge indices — use `// 2` (Python 3 int division is fine)
- Infinite loop — ensure `low = mid + 1` or `high = mid - 1` when narrowing
- Forgetting array must be **sorted**

---

## Quick Reference

| Task | Approach |
| --- | --- |
| Find exact value | Loop above or `bisect_left` |
| Insert position | `bisect_left(arr, x)` |
| Count of x | `bisect_right - bisect_left` |

---

## Related Notes

- [[Algorithms]]
- [[Codes/Algorithms — Two Pointers]]
- [[Codes/Algorithms — Prefix Sum]]

---

## Tags

#algorithms #binary-search #python #interview #sorted #log-n
