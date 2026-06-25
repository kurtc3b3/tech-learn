## What & When

**BFS** (breadth-first search) explores a graph **level by level** with a **queue** — finds **shortest path** in unweighted graphs. **DFS** goes **deep first** via stack or recursion — good for cycles, connected components, and backtracking.

Use BFS when:

- Shortest path (unweighted), minimum steps
- Level-order tree traversal

Use DFS when:

- Detect cycles, count islands, exhaustive paths
- Topological sort (with indegree variant)

Hub: [[Algorithms]]. Production graphs: [[ML — NetworkX]].

```python
from collections import deque
```

---

## Graph Representation

```python
# Adjacency list
graph: dict[int, list[int]] = {
    0: [1, 2],
    1: [0, 3],
    2: [0],
    3: [1],
}
```

---

## BFS — Order of Visit

```python
from collections import deque

def bfs(graph: dict[int, list[int]], start: int) -> list[int]:
    visited = {start}
    q: deque[int] = deque([start])
    order: list[int] = []

    while q:
        node = q.popleft()
        order.append(node)
        for nei in graph[node]:
            if nei not in visited:
                visited.add(nei)
                q.append(nei)
    return order
```

---

## BFS — Shortest Path (unweighted)

```python
def bfs_shortest_path(
    graph: dict[int, list[int]], start: int, goal: int
) -> list[int] | None:
    if start == goal:
        return [start]
    visited = {start}
    q: deque[tuple[int, list[int]]] = deque([(start, [start])])

    while q:
        node, path = q.popleft()
        for nei in graph[node]:
            if nei in visited:
                continue
            new_path = path + [nei]
            if nei == goal:
                return new_path
            visited.add(nei)
            q.append((nei, new_path))
    return None
```

For large graphs, store `parent[node]` instead of copying paths.

---

## DFS — Recursive

```python
def dfs(
    graph: dict[int, list[int]],
    node: int,
    visited: set[int],
    order: list[int],
) -> list[int]:
    visited.add(node)
    order.append(node)
    for nei in graph[node]:
        if nei not in visited:
            dfs(graph, nei, visited, order)
    return order


# Usage
dfs(graph, start=0, visited=set(), order=[])
```

---

## DFS — Iterative (stack)

```python
def dfs_iterative(graph: dict[int, list[int]], start: int) -> list[int]:
    visited: set[int] = set()
    stack = [start]
    order: list[int] = []

    while stack:
        node = stack.pop()
        if node in visited:
            continue
        visited.add(node)
        order.append(node)
        for nei in reversed(graph[node]):   # optional: match recursive order
            if nei not in visited:
                stack.append(nei)
    return order
```

---

## Grid as Graph (islands pattern)

```python
def count_islands(grid: list[list[str]]) -> int:
    if not grid:
        return 0
    rows, cols = len(grid), len(grid[0])
    count = 0

    def dfs(r: int, c: int) -> None:
        if r < 0 or c < 0 or r >= rows or c >= cols or grid[r][c] != "1":
            return
        grid[r][c] = "0"
        dfs(r + 1, c)
        dfs(r - 1, c)
        dfs(r, c + 1)
        dfs(r, c - 1)

    for r in range(rows):
        for c in range(cols):
            if grid[r][c] == "1":
                dfs(r, c)
                count += 1
    return count
```

---

## BFS vs DFS

| | BFS | DFS |
| --- | --- | --- |
| Structure | Queue | Stack / recursion |
| Shortest path (unweighted) | ✅ | ❌ |
| Memory | Often higher (frontier) | O(depth) stack |
| Cycles / components | Possible | Natural fit |

---

## Complexity

| | Time | Space |
| --- | --- | --- |
| BFS / DFS | O(V + E) | O(V) |

---

## Related Notes

- [[Algorithms]]
- [[ML — NetworkX]]
- [[Codes/Algorithms — Fast & Slow Pointers]]

---

## Tags

#algorithms #bfs #dfs #graph #python #interview #queue #shortest-path
