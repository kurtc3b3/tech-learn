## What & When

**Fast & slow pointers** (Floyd's cycle detection) move at different speeds — slow +1 step, fast +2 steps. Detects **cycles in linked lists**, finds middle node, sometimes happy number problems.

Use when:

- Linked list may have a cycle
- Find middle without counting length
- O(1) space instead of hash set of visited nodes

Hub: [[Algorithms]]. Graph cycles: [[Codes/Algorithms — BFS & DFS]].

---

## List Node Definition

```python
class ListNode:
    def __init__(self, val: int = 0, next: "ListNode | None" = None):
        self.val = val
        self.next = next
```

---

## Has Cycle

```python
def has_cycle(head: ListNode | None) -> bool:
    slow = fast = head
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next
        if slow is fast:
            return True
    return False
```

If fast reaches `None`, no cycle.

---

## Find Cycle Start (optional)

```python
def detect_cycle_start(head: ListNode | None) -> ListNode | None:
    slow = fast = head
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next
        if slow is fast:
            break
    else:
        return None
    slow = head
    while slow is not fast:
        slow = slow.next
        fast = fast.next
    return slow
```

---

## Middle of List

```python
def middle_node(head: ListNode | None) -> ListNode | None:
    slow = fast = head
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next
    return slow
```

When fast exits, slow is at middle (lower middle for even length).

---

## Happy Number (fast/slow on sequence)

```python
def next_happy(n: int) -> int:
    total = 0
    while n:
        n, rem = divmod(n, 10)
        total += rem * rem
    return total


def is_happy(n: int) -> bool:
    slow = fast = n
    while True:
        slow = next_happy(slow)
        fast = next_happy(next_happy(fast))
        if slow == 1:
            return True
        if slow == fast:
            return False
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
- [[Codes/Algorithms — BFS & DFS]]
- [[Codes/Algorithms — Two Pointers]]

---

## Tags

#algorithms #linked-list #fast-slow-pointers #floyd #python #interview #cycle
