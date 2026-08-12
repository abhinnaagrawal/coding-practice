[← back to index](/coding-practice/README.md)

# Topological Sort

## When to recognize it
"Dependencies," "prerequisites," "must happen before" — any directed graph where you need a linear ordering respecting all edges.

## Core idea
Kahn's algorithm (BFS-based): repeatedly remove nodes with in-degree 0, decrementing their neighbors' in-degree. If you can't process all nodes this way, the graph has a cycle — no valid ordering exists. DFS-based topo sort (post-order reversal) is the alternative.

## Gotchas
- Cycle = no valid topo order — Kahn's algorithm naturally detects this if the final output size < node count.
- Multiple valid orders often exist — don't assume a unique "correct" answer when testing.

## Problems

| Problem | Difficulty | Freq | What it asks | Intuition |
|---|---|---|---|---|
| [Course Schedule](https://leetcode.com/problems/course-schedule/) | Medium | 51 | Determine if all courses can be finished given prerequisite pairs (cycle detection). | Kahn's BFS: repeatedly remove courses with no remaining prerequisites, decrementing dependents' in-degree; if you can't remove all courses this way, a cycle exists. |
| [Course Schedule II](https://leetcode.com/problems/course-schedule-ii/) | Medium | 46 | Return a valid order to take all courses given prerequisites. | Same Kahn's BFS, but record the order in which courses are removed — that's a valid topological order. |
| [Alien Dictionary](https://leetcode.com/problems/alien-dictionary/) | Hard | — | Given a sorted list of words in an unknown alien language, derive a valid ordering of its alphabet. | Compare each pair of adjacent words to find the first differing character — that gives a directed edge (earlier letter → later letter); topologically sort the resulting letter graph. |
| [Minimum Height Trees](https://leetcode.com/problems/minimum-height-trees/) | Medium | — | Find the root(s) of a tree that minimize its height (repeatedly trim leaves — topological-sort-flavored). | Repeatedly strip leaf nodes (degree 1) layer by layer, like Kahn's algorithm from the outside in — the last remaining node(s) are the centers that minimize height. |

## Solutions

### Course Schedule / Course Schedule II
See [10-graphs.md](/coding-practice/10-graphs.md) — both are Kahn's-algorithm (BFS topological sort) applications.

### Alien Dictionary
```python
def alien_order(words):
    graph = {c: set() for word in words for c in word}

    for first, second in zip(words, words[1:]):
        for c1, c2 in zip(first, second):
            if c1 != c2:
                graph[c1].add(c2)  # c1 must come before c2 in the alphabet
                break
        else:
            if len(second) < len(first):
                return ""  # e.g. "abc" before "ab" is impossible in a valid ordering

    visited = {}  # char -> False (currently visiting) / True (finished)
    order = []

    def dfs(c):
        if c in visited:
            return visited[c]  # True = safely finished, False = we're mid-cycle
        visited[c] = False
        for nxt in graph[c]:
            if not dfs(nxt):
                return False  # cycle detected
        visited[c] = True
        order.append(c)
        return True

    for c in graph:
        if not dfs(c):
            return ""

    return ''.join(reversed(order))  # post-order DFS gives reverse topological order
```

### Minimum Height Trees
```python
from collections import defaultdict, deque

def find_min_height_trees(n, edges):
    if n <= 2:
        return list(range(n))

    graph = defaultdict(set)
    for a, b in edges:
        graph[a].add(b)
        graph[b].add(a)

    leaves = deque(node for node in graph if len(graph[node]) == 1)
    remaining = n
    while remaining > 2:
        leaf_count = len(leaves)
        remaining -= leaf_count
        for _ in range(leaf_count):
            leaf = leaves.popleft()
            for neighbor in graph[leaf]:
                graph[neighbor].discard(leaf)
                if len(graph[neighbor]) == 1:
                    leaves.append(neighbor)  # newly exposed leaf

    return list(leaves)  # the 1 or 2 nodes left are the centers
```
