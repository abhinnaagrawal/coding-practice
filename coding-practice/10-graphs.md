[← back to index](README.md)

# Graphs

## When to recognize it
Problem describes nodes and connections/relationships — grid adjacency counts too (each cell is a node, adjacent cells are edges). Look for: "connected," "reachable," "shortest path," "islands," "dependencies."

## Core idea
BFS explores level by level (shortest path when edges are unweighted); DFS explores depth-first (simpler when you just need existence or need to explore full components). Weighted shortest-path needs Dijkstra, not plain BFS.

## Gotchas
- Directed vs undirected changes cycle detection logic entirely — clarify before coding.
- Disconnected components — outer loop over *all* nodes, not just from one start.
- Visited set: mark visited at *enqueue* time for BFS, not at *dequeue* — avoids duplicate enqueues.
- BFS = shortest path only when edges are unweighted/equal weight; weighted → Dijkstra; only need existence → DFS suffices, skip BFS overhead.

## Problems

| Problem | Difficulty | Freq | What it asks | Intuition |
|---|---|---|---|---|
| [Number of Islands](https://leetcode.com/problems/number-of-islands/) | Medium | 107 | Given a 2D grid of land/water cells, count the number of islands (connected land components). | Scan every cell; on an unvisited land cell, flood-fill (DFS or BFS) marking the whole connected component visited, and count that as one island. |
| [Course Schedule](https://leetcode.com/problems/course-schedule/) | Medium | 51 | Given course prerequisites, determine if it's possible to finish all courses (cycle detection in a directed graph). | Build a directed graph from prerequisites, then detect a cycle — either DFS with a "currently visiting" marker (back-edge = cycle) or Kahn's BFS (if you can't process all nodes via in-degree-0 removal, there's a cycle). |
| [Course Schedule II](https://leetcode.com/problems/course-schedule-ii/) | Medium | 46 | Same as above, but return a valid course order (topological sort). | Same cycle-detection setup, but record the order nodes are finished (DFS post-order reversed, or Kahn's BFS processing order directly). |
| [Clone Graph](https://leetcode.com/problems/clone-graph/) | Medium | — | Given a reference node in a connected graph, return a deep copy of the entire graph. | DFS or BFS from the given node, using a hashmap of original-node→cloned-node to avoid re-cloning nodes already visited and to handle cycles. |
| [Pacific Atlantic Water Flow](https://leetcode.com/problems/pacific-atlantic-water-flow/) | Medium | — | Find all grid cells from which water can flow to both the Pacific and Atlantic borders. | Reverse the problem: multi-source BFS/DFS *inward* from each ocean's border cells (water can flow uphill in reverse), then intersect the two reachable sets. |
| [Number of Connected Components in an Undirected Graph](https://leetcode.com/problems/number-of-connected-components-in-an-undirected-graph/) | Medium | — | Count the number of connected components in an undirected graph. | DFS/BFS from every unvisited node, incrementing a counter each time you start a new traversal (or use Union-Find, counting distinct roots). |
| [Network Delay Time](https://leetcode.com/problems/network-delay-time/) | Medium | — | Find the minimum time for a signal to reach all nodes from a source (Dijkstra). | Dijkstra's algorithm: min-heap of `(distance, node)`, always expand the closest unvisited node next, relax neighboring edges — answer is the max distance across all reached nodes. |
| [Word Ladder](https://leetcode.com/problems/word-ladder/) | Hard | — | Find the minimum number of single-letter transformations from a start word to an end word, using only dictionary words. | BFS where each node is a word and edges connect words differing by exactly one letter — BFS guarantees the shortest transformation sequence. |
| [Rotting Oranges](https://leetcode.com/problems/rotting-oranges/) | Medium | — | Find the minimum minutes until no fresh orange remains, given rot spreads to adjacent cells each minute (multi-source BFS). | Multi-source BFS starting simultaneously from every initially-rotten cell; process level by level (each level = one minute), track the number of minutes until the queue empties. |

## Solutions

### Number of Islands
```python
def num_islands(grid):
    if not grid:
        return 0
    rows, cols = len(grid), len(grid[0])

    def dfs(r, c):
        if r < 0 or r >= rows or c < 0 or c >= cols or grid[r][c] != '1':
            return
        grid[r][c] = '0'  # sink this cell so we never revisit it
        dfs(r + 1, c); dfs(r - 1, c); dfs(r, c + 1); dfs(r, c - 1)

    islands = 0
    for r in range(rows):
        for c in range(cols):
            if grid[r][c] == '1':
                islands += 1
                dfs(r, c)  # flood-fill the whole connected component
    return islands
```

### Course Schedule
```python
from collections import deque, defaultdict

def can_finish(num_courses, prerequisites):
    graph = defaultdict(list)
    indegree = [0] * num_courses
    for course, prereq in prerequisites:
        graph[prereq].append(course)
        indegree[course] += 1

    queue = deque(c for c in range(num_courses) if indegree[c] == 0)
    visited = 0
    while queue:
        node = queue.popleft()
        visited += 1
        for nxt in graph[node]:
            indegree[nxt] -= 1
            if indegree[nxt] == 0:
                queue.append(nxt)

    return visited == num_courses  # if some nodes never hit indegree 0, there's a cycle
```

### Course Schedule II
```python
def find_order(num_courses, prerequisites):
    graph = defaultdict(list)
    indegree = [0] * num_courses
    for course, prereq in prerequisites:
        graph[prereq].append(course)
        indegree[course] += 1

    queue = deque(c for c in range(num_courses) if indegree[c] == 0)
    order = []
    while queue:
        node = queue.popleft()
        order.append(node)
        for nxt in graph[node]:
            indegree[nxt] -= 1
            if indegree[nxt] == 0:
                queue.append(nxt)

    return order if len(order) == num_courses else []
```

### Clone Graph
```python
class GraphNode:
    def __init__(self, val=0, neighbors=None):
        self.val = val
        self.neighbors = neighbors or []


def clone_graph(node):
    if not node:
        return None
    visited = {}  # original node -> cloned node

    def dfs(n):
        if n in visited:
            return visited[n]
        copy = GraphNode(n.val)
        visited[n] = copy  # register before recursing, to handle cycles
        for neighbor in n.neighbors:
            copy.neighbors.append(dfs(neighbor))
        return copy

    return dfs(node)
```

### Pacific Atlantic Water Flow
```python
def pacific_atlantic(heights):
    if not heights:
        return []
    rows, cols = len(heights), len(heights[0])
    pacific, atlantic = set(), set()

    def dfs(r, c, visited, prev_height):
        if (r < 0 or r >= rows or c < 0 or c >= cols or
                (r, c) in visited or heights[r][c] < prev_height):
            return
        visited.add((r, c))
        for dr, dc in ((1, 0), (-1, 0), (0, 1), (0, -1)):
            dfs(r + dr, c + dc, visited, heights[r][c])

    for c in range(cols):
        dfs(0, c, pacific, heights[0][c])           # top border -> Pacific
        dfs(rows - 1, c, atlantic, heights[rows - 1][c])  # bottom border -> Atlantic
    for r in range(rows):
        dfs(r, 0, pacific, heights[r][0])            # left border -> Pacific
        dfs(r, cols - 1, atlantic, heights[r][cols - 1])  # right border -> Atlantic

    return list(pacific & atlantic)
```

### Number of Connected Components in an Undirected Graph
```python
def count_components(n, edges):
    graph = defaultdict(list)
    for a, b in edges:
        graph[a].append(b)
        graph[b].append(a)

    visited = set()

    def dfs(node):
        visited.add(node)
        for nxt in graph[node]:
            if nxt not in visited:
                dfs(nxt)

    count = 0
    for node in range(n):
        if node not in visited:
            count += 1
            dfs(node)  # explore the entire component
    return count
```

### Network Delay Time
```python
import heapq

def network_delay_time(times, n, k):
    graph = defaultdict(list)
    for u, v, w in times:
        graph[u].append((v, w))

    dist = {}
    heap = [(0, k)]
    while heap:
        d, node = heapq.heappop(heap)
        if node in dist:
            continue  # already finalized with a shorter distance
        dist[node] = d
        for nxt, w in graph[node]:
            if nxt not in dist:
                heapq.heappush(heap, (d + w, nxt))

    return max(dist.values()) if len(dist) == n else -1
```

### Word Ladder
```python
def ladder_length(begin_word, end_word, word_list):
    words = set(word_list)
    if end_word not in words:
        return 0

    queue = deque([(begin_word, 1)])
    words.discard(begin_word)
    while queue:
        word, steps = queue.popleft()
        if word == end_word:
            return steps
        for i in range(len(word)):
            for c in 'abcdefghijklmnopqrstuvwxyz':
                candidate = word[:i] + c + word[i + 1:]
                if candidate in words:
                    words.remove(candidate)  # mark visited by removing from the pool
                    queue.append((candidate, steps + 1))
    return 0
```

### Rotting Oranges
```python
def oranges_rotting(grid):
    rows, cols = len(grid), len(grid[0])
    queue = deque()
    fresh = 0
    for r in range(rows):
        for c in range(cols):
            if grid[r][c] == 2:
                queue.append((r, c, 0))  # all initially-rotten cells are BFS sources
            elif grid[r][c] == 1:
                fresh += 1

    minutes = 0
    while queue:
        r, c, minute = queue.popleft()
        minutes = max(minutes, minute)
        for dr, dc in ((1, 0), (-1, 0), (0, 1), (0, -1)):
            nr, nc = r + dr, c + dc
            if 0 <= nr < rows and 0 <= nc < cols and grid[nr][nc] == 1:
                grid[nr][nc] = 2
                fresh -= 1
                queue.append((nr, nc, minute + 1))

    return minutes if fresh == 0 else -1
```
