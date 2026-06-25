## What & When

**Prefix sum** preprocesses an array so **range sums** query in O(1) after O(n) build — classic trade extra space for fast queries.

Use when:

- Many queries: sum from index `L` to `R`
- Subarray sum equals K (with hash map)
- 2D prefix sums for matrix region sums (extension)

Hub: [[Algorithms]]. Pairs with [[Codes/Algorithms — Hash Map Patterns]].

---

## Core Idea

```text
prefix[i] = sum(arr[0..i])
sum(L..R) = prefix[R] - prefix[L-1]   (L > 0)
sum(0..R) = prefix[R]
```

---

## Implementation

```python
def build_prefix(arr: list[int]) -> list[int]:
    if not arr:
        return []
    prefix = [0] * len(arr)
    prefix[0] = arr[0]
    for i in range(1, len(arr)):
        prefix[i] = prefix[i - 1] + arr[i]
    return prefix


def range_sum(prefix: list[int], left: int, right: int) -> int:
    """Inclusive indices left..right."""
    if left == 0:
        return prefix[right]
    return prefix[right] - prefix[left - 1]


# Example
arr = [1, 2, 3, 4, 5]
prefix = build_prefix(arr)
assert range_sum(prefix, 1, 3) == 9   # 2+3+4
```

---

## With Padding (common interview style)

```python
def prefix_with_pad(arr: list[int]) -> list[int]:
    """prefix[i] = sum(arr[0:i]) — length n+1."""
    prefix = [0]
    for x in arr:
        prefix.append(prefix[-1] + x)
    return prefix

# sum(L..R) = prefix[R+1] - prefix[L]
```

---

## Subarray Sum Equals K

```python
from collections import defaultdict

def subarray_sum_k(nums: list[int], k: int) -> int:
    count = 0
    running = 0
    freq: dict[int, int] = defaultdict(int)
    freq[0] = 1
    for x in nums:
        running += x
        count += freq[running - k]
        freq[running] += 1
    return count
```

---

## Complexity

| Operation | Time | Space |
| --- | --- | --- |
| Build prefix | O(n) | O(n) |
| Range query | O(1) | — |

---

## Related Notes

- [[Algorithms]]
- [[Codes/Algorithms — Hash Map Patterns]]
- [[Codes/Algorithms — Kadane's Algorithm]]

---

## Tags

#algorithms #prefix-sum #python #interview #subarray #range-query
