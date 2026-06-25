## What & When

**Merge intervals** combines **overlapping** ranges after sorting by start — classic scheduling, meeting rooms, and timeline problems.

Use when:

- Input is list of `[start, end]` intervals
- Need union, overlap count, or minimum removals

Hub: [[Algorithms]].

---

## Core Algorithm

1. Sort by start time
2. Merge if current overlaps previous; else append new interval

```python
def merge_intervals(intervals: list[list[int]]) -> list[list[int]]:
    if not intervals:
        return []
    intervals.sort(key=lambda x: x[0])
    merged = [intervals[0][:]]
    for start, end in intervals[1:]:
        last_end = merged[-1][1]
        if start <= last_end:
            merged[-1][1] = max(last_end, end)
        else:
            merged.append([start, end])
    return merged


# Example
intervals = [[1, 3], [2, 6], [8, 10], [15, 18]]
assert merge_intervals(intervals) == [[1, 6], [8, 10], [15, 18]]
```

---

## Insert Interval

```python
def insert_interval(
    intervals: list[list[int]], new: list[int]
) -> list[list[int]]:
    result: list[list[int]] = []
    i = 0
    n = len(intervals)
    start, end = new

    while i < n and intervals[i][1] < start:
        result.append(intervals[i])
        i += 1
    while i < n and intervals[i][0] <= end:
        start = min(start, intervals[i][0])
        end = max(end, intervals[i][1])
        i += 1
    result.append([start, end])
    result.extend(intervals[i:])
    return result
```

---

## Overlap Check

```python
def overlaps(a: list[int], b: list[int]) -> bool:
    return a[0] <= b[1] and b[0] <= a[1]
```

---

## Complexity

| | |
| --- | --- |
| **Time** | O(n log n) for sort + O(n) merge |
| **Space** | O(n) output |

---

## Related Notes

- [[Algorithms]]
- [[Codes/Algorithms — Two Pointers]] — after sort, merge is two-pointer-like

---

## Tags

#algorithms #intervals #merge #python #interview #sorting
