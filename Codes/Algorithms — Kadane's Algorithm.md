## What & When

**Kadane's algorithm** finds the **maximum sum of a contiguous subarray** in one pass — O(n) time, O(1) space.

Use when:

- "Maximum subarray sum" (may include negatives)
- Tracking running best vs starting fresh at current index

Not for: max **product** subarray (different logic) or non-contiguous subsets.

Hub: [[Algorithms]]. Compare [[Codes/Algorithms — Prefix Sum]] for range queries.

---

## Core Idea

At each index, either **extend** the current subarray or **start new** at this element:

```text
curr_sum = max(x, curr_sum + x)
max_sum  = max(max_sum, curr_sum)
```

---

## Implementation

```python
def max_subarray(nums: list[int]) -> int:
    max_sum = nums[0]
    curr_sum = nums[0]
    for x in nums[1:]:
        curr_sum = max(x, curr_sum + x)
        max_sum = max(max_sum, curr_sum)
    return max_sum


# Example
arr = [-2, 1, -3, 4, -1, 2, 1, -5, 4]
assert max_subarray(arr) == 6   # [4, -1, 2, 1]
```

---

## Return Indices (optional)

```python
def max_subarray_range(nums: list[int]) -> tuple[int, int, int]:
    """Return (max_sum, start_index, end_index)."""
    best_sum = curr_sum = nums[0]
    start = end = temp_start = 0

    for i in range(1, len(nums)):
        x = nums[i]
        if curr_sum + x < x:
            curr_sum = x
            temp_start = i
        else:
            curr_sum += x
        if curr_sum > best_sum:
            best_sum = curr_sum
            start, end = temp_start, i
    return best_sum, start, end
```

---

## Complexity

| | |
| --- | --- |
| **Time** | O(n) |
| **Space** | O(1) |

---

## Related Notes

- [[Algorithms]]
- [[Codes/Algorithms — Prefix Sum]]
- [[Codes/Algorithms — Sliding Window]] — fixed-size max sum variant

---

## Tags

#algorithms #kadane #dynamic-programming #python #interview #subarray
