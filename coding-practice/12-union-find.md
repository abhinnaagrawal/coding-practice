[← back to index](/coding-practice/README.md)

# Union-Find (Disjoint Set)

## When to recognize it
"Are these two nodes connected," "count groups," "will adding this edge create a cycle" — especially when edges/unions arrive incrementally rather than as a static graph you can BFS/DFS once.

## Core idea
Maintain a forest where each set has a representative ("root"). `find(x)` walks up to the root; `union(a, b)` links one root under the other. Two optimizations make this near O(1) per operation: path compression (flatten the tree during `find`) and union by rank/size (always attach the smaller tree under the bigger one).

## Gotchas
- Path compression + union by rank/size — without both, degrades to O(n) per op, defeats the purpose.
- "Same component" query — compare `find(a) == find(b)`, never compare parent pointers directly.

## Problems

| Problem | Difficulty | Freq | What it asks | Intuition |
|---|---|---|---|---|
| [Number of Provinces](https://leetcode.com/problems/number-of-provinces/) | Medium | — | Given a connectivity matrix of cities, count the number of provinces (connected groups). | Union every pair of directly connected cities, then count the number of distinct roots remaining. |
| [Accounts Merge](https://leetcode.com/problems/accounts-merge/) | Medium | — | Merge accounts that belong to the same person based on shared emails. | For every email shared between two accounts, union those account indices; afterward, group all accounts by their root and merge their email sets. |
| [Redundant Connection](https://leetcode.com/problems/redundant-connection/) | Medium | — | Given a tree with one extra edge added, find the edge that can be removed to restore a valid tree. | Process edges in input order, union-find each one; the first edge whose two endpoints already share a root (i.e. union would create a cycle) is the redundant one. |
| [Graph Valid Tree](https://leetcode.com/problems/graph-valid-tree/) | Medium | — | Given n nodes and a list of edges, determine if they form a valid tree. | A valid tree needs exactly n-1 edges and no cycles — union-find each edge; if any union finds the endpoints already connected, it's a cycle, not a tree. |
| [Longest Consecutive Sequence](https://leetcode.com/problems/longest-consecutive-sequence/) | Medium | 66 | Find the length of the longest run of consecutive integers (union-find is an alternative to the hash-set approach). | Union each number with `number + 1` if that value is present in the set; track component sizes, the answer is the largest component size. |

## Solutions

```python
class UnionFind:
    def __init__(self, n):
        self.parent = list(range(n))
        self.rank = [0] * n

    def find(self, x):
        if self.parent[x] != x:
            self.parent[x] = self.find(self.parent[x])  # path compression
        return self.parent[x]

    def union(self, a, b):
        ra, rb = self.find(a), self.find(b)
        if ra == rb:
            return False  # already connected -> union would create a cycle
        if self.rank[ra] < self.rank[rb]:
            ra, rb = rb, ra
        self.parent[rb] = ra  # attach the smaller-rank tree under the bigger one
        if self.rank[ra] == self.rank[rb]:
            self.rank[ra] += 1
        return True
```

### Number of Provinces
```python
def find_circle_num(is_connected):
    n = len(is_connected)
    uf = UnionFind(n)
    for i in range(n):
        for j in range(i + 1, n):
            if is_connected[i][j] == 1:
                uf.union(i, j)
    return len({uf.find(i) for i in range(n)})  # count distinct roots
```

### Accounts Merge
```python
from collections import defaultdict

def accounts_merge(accounts):
    uf = UnionFind(len(accounts))
    email_to_idx = {}
    for i, account in enumerate(accounts):
        for email in account[1:]:
            if email in email_to_idx:
                uf.union(i, email_to_idx[email])  # same email seen before -> same person
            else:
                email_to_idx[email] = i

    groups = defaultdict(set)
    for email, idx in email_to_idx.items():
        groups[uf.find(idx)].add(email)

    return [[accounts[root][0]] + sorted(emails) for root, emails in groups.items()]
```

### Redundant Connection
```python
def find_redundant_connection(edges):
    n = len(edges)
    uf = UnionFind(n + 1)
    for a, b in edges:
        if not uf.union(a, b):  # union fails -> a, b already connected -> this edge is redundant
            return [a, b]
    return []
```

### Graph Valid Tree
```python
def valid_tree(n, edges):
    if len(edges) != n - 1:  # a tree on n nodes has exactly n-1 edges
        return False
    uf = UnionFind(n)
    for a, b in edges:
        if not uf.union(a, b):
            return False  # cycle detected
    return True
```

### Longest Consecutive Sequence (Union-Find version)
```python
def longest_consecutive_uf(nums):
    if not nums:
        return 0
    uf = UnionFind(len(nums))
    index = {n: i for i, n in enumerate(nums)}
    size = [1] * len(nums)

    def union(a, b):
        ra, rb = uf.find(a), uf.find(b)
        if ra != rb:
            uf.parent[ra] = rb
            size[rb] += size[ra]  # track component size to answer "longest run" directly

    for n in nums:
        if n + 1 in index:
            union(index[n], index[n + 1])

    return max(size)
```
