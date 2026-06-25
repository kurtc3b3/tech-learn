## What & When

**Sliding window** maintains a contiguous **window** over an array/string and slides it — **O(n)** for fixed or variable size subarray/substring problems.

Use when:

- Max/min sum of subarray of **size k**
- Longest substring with at most k distinct characters
- Minimum window substring containing all chars

Hub: [[Algorithms]]. Fixed-size variant complements [[Codes/Algorithms — Kadane's Algorithm]] (variable max sum).

---

## Fixed Size K — Max Sum

```python
def max_sum_subarray_k(arr: list[int], k: int) -> int:
    if len(arr) < k:
        raise ValueError("array shorter than k")
    window = sum(arr[:k])
    best = window
    for i in range(k, len(arr)):
        window += arr[i] - arr[i - k]
        best = max(best, window)
    return best


# Example
arr = [2, 1, 5, 1, 3, 2]
assert max_sum_subarray_k(arr, 3) == 9   # 5 + 1 + 3
```

---

## Variable Size — Longest Substring Without Repeating

```python
def length_of_longest_substring(s: str) -> int:
    last: dict[str, int] = {}
    left = best = 0
    for right, ch in enumerate(s):
        if ch in last and last[ch] >= left:
            left = last[ch] + 1
        last[ch] = right
        best = max(best, right - left + 1)
    return best
```

---

## At Most K Distinct Characters

```python
from collections import defaultdict

def longest_k_distinct(s: str, k: int) -> int:
    freq: dict[str, int] = defaultdict(int)
    left = best = 0
    for right, ch in enumerate(s):
        freq[ch] += 1
        while len(freq) > k:
            freq[s[left]] -= 1
            if freq[s[left]] == 0:
                del freq[s[left]]
            left += 1
        best = max(best, right - left + 1)
    return best
```

---

## Template (variable window)

```python
left = 0
for right in range(len(arr)):
    # expand: include arr[right]
    while window_invalid:
        # shrink from left
        left += 1
    # update answer with window [left, right]
```

---

## Sliding Window vs Others

| Problem | Pattern |
| --- | --- |
| Max sum **any** contiguous subarray | [[Codes/Algorithms — Kadane's Algorithm]] |
| Max sum **exact length k** | **Sliding window** |
| Pair in sorted array | [[Codes/Algorithms — Two Pointers]] |

---

## Complexity

| | |
| --- | --- |
| **Time** | O(n) — each element enters/leaves window once |
| **Space** | O(1) or O(k) for freq map |

---

## Related Notes

- [[Algorithms]]
- [[Codes/Algorithms — Two Pointers]]
- [[Codes/Algorithms — Kadane's Algorithm]]
- [[Codes/Algorithms — Hash Map Patterns]]

---

## Tags

#algorithms #sliding-window #python #interview #subarray #substring
