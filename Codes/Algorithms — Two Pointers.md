## What & When

**Two pointers** uses two indices into an array (often **sorted**) to solve problems in **O(n)** with **O(1)** extra space — pairs, palindromes, in-place removal.

Use when:

- Sorted array — find pair with given sum
- Opposite ends moving inward
- Same-direction slow/fast for in-place dedup (sorted)

Hub: [[Algorithms]]. Sorted lookup alternative: [[Codes/Algorithms — Binary Search]].

---

## Pair with Target Sum (sorted)

```python
def two_sum_sorted(arr: list[int], target: int) -> tuple[int, int] | None:
    left, right = 0, len(arr) - 1
    while left < right:
        total = arr[left] + arr[right]
        if total == target:
            return left, right
        if total < target:
            left += 1
        else:
            right -= 1
    return None


# Example
arr = [1, 2, 3, 4, 5]
assert two_sum_sorted(arr, 5) == (0, 3)   # 1 + 4
```

---

## Remove Duplicates (sorted, in-place)

```python
def remove_duplicates(nums: list[int]) -> int:
    if not nums:
        return 0
    write = 1
    for read in range(1, len(nums)):
        if nums[read] != nums[read - 1]:
            nums[write] = nums[read]
            write += 1
    return write
```

---

## Valid Palindrome (ignore non-alnum)

```python
def is_palindrome(s: str) -> bool:
    left, right = 0, len(s) - 1
    while left < right:
        while left < right and not s[left].isalnum():
            left += 1
        while left < right and not s[right].isalnum():
            right -= 1
        if s[left].lower() != s[right].lower():
            return False
        left += 1
        right -= 1
    return True
```

---

## Container With Most Water (max area)

```python
def max_area(height: list[int]) -> int:
    left, right = 0, len(height) - 1
    best = 0
    while left < right:
        h = min(height[left], height[right])
        best = max(best, h * (right - left))
        if height[left] < height[right]:
            left += 1
        else:
            right -= 1
    return best
```

---

## Two Pointers vs Hash Map

| Input | Prefer |
| --- | --- |
| Sorted array, pair sum | **Two pointers** O(n), O(1) |
| Unsorted, need indices | [[Codes/Algorithms — Hash Map Patterns]] |
| Three-sum | Sort + fix one + two pointers |

---

## Complexity

Typically **O(n)** time, **O(1)** space.

---

## Related Notes

- [[Algorithms]]
- [[Codes/Algorithms — Binary Search]]
- [[Codes/Algorithms — Hash Map Patterns]]
- [[Codes/Algorithms — Sliding Window]]

---

## Tags

#algorithms #two-pointers #python #interview #sorted-array #palindrome
