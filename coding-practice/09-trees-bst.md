[← back to index](/coding-practice/README.md)

# Trees & BSTs

## When to recognize it
Problem gives a tree/binary tree/BST structure explicitly, or asks about hierarchical relationships (ancestor, depth, path).

## Core idea
Almost every tree problem is a DFS (pre/in/post-order) or BFS (level order) traversal with some extra bookkeeping. BST adds an ordering invariant (left < node < right) that turns search/insert/delete into O(log n) navigation instead of full traversal.

## Gotchas
- Null checks before recursing left/right — most common crash.
- BST validation: pass down `(low, high)` bounds, not just compare to the immediate parent.
- LCA: recursive return semantics — "found target in this subtree" bubbling up correctly.
- Balanced/height problems: bottom-up (post-order) is almost always cleaner than top-down recomputation.

## Problems

| Problem | Difficulty | Freq | What it asks | Intuition |
|---|---|---|---|---|
| [Binary Tree Right Side View](https://leetcode.com/problems/binary-tree-right-side-view/) | Medium | — | Return the value of the rightmost node visible at each level. | BFS level order, take the last node processed at each level (or DFS visiting right child before left, recording the first node seen at each depth). |
| [Binary Tree Maximum Path Sum](https://leetcode.com/problems/binary-tree-maximum-path-sum/) | Hard | — | Find the maximum sum along any path between two nodes in the tree. | Post-order DFS returning the best single-branch sum through each node (for the parent to use); separately track a global max that considers *both* branches meeting at a node. |
| [Lowest Common Ancestor of a Binary Tree](https://leetcode.com/problems/lowest-common-ancestor-of-a-binary-tree/) | Medium | — | Find the lowest node that has both given nodes as descendants. | Recurse into both children; if a node's left and right subtrees each report finding one of the two targets, that node is the LCA — otherwise bubble up whichever side found something. |
| [Maximum Depth of Binary Tree](https://leetcode.com/problems/maximum-depth-of-binary-tree/) | Easy | — | Find the height (max depth) of a binary tree. | Recursively `1 + max(depth(left), depth(right))`, with `None` returning 0. |
| [Same Tree](https://leetcode.com/problems/same-tree/) | Easy | — | Determine if two binary trees are structurally identical with the same node values. | Recursively compare current node values and both children — any mismatch (including one side being `None` and the other not) returns false. |
| [Binary Tree Level Order Traversal](https://leetcode.com/problems/binary-tree-level-order-traversal/) | Medium | — | Return node values grouped level by level (BFS). | Standard BFS with a queue; process one full level (using the queue's current length as the level size) before moving to the next. |
| [Validate Binary Search Tree](https://leetcode.com/problems/validate-binary-search-tree/) | Medium | — | Determine if a binary tree satisfies the BST ordering property. | Recurse passing down a valid `(low, high)` range for each node — not just comparing to the immediate parent — and narrow the range for children. |
| [Kth Smallest Element in a BST](https://leetcode.com/problems/kth-smallest-element-in-a-bst/) | Medium | — | Find the kth smallest value in a BST (in-order traversal). | In-order traversal visits BST nodes in ascending order — count nodes visited and stop at the kth. |
| [Construct Binary Tree from Preorder and Inorder Traversal](https://leetcode.com/problems/construct-binary-tree-from-preorder-and-inorder-traversal/) | Medium | — | Rebuild the tree given its preorder and inorder traversal arrays. | Preorder's first element is always the root; find that value's position in inorder to split the remaining elements into left-subtree and right-subtree ranges, recurse on each. |
| [Serialize and Deserialize Binary Tree](https://leetcode.com/problems/serialize-and-deserialize-binary-tree/) | Hard | — | Design an algorithm to encode a tree to a string and decode it back. | Preorder DFS, emitting a sentinel (e.g. "null") for missing children; deserialize by consuming tokens in the same preorder sequence, recursively rebuilding left then right. |
| [Diameter of Binary Tree](https://leetcode.com/problems/diameter-of-binary-tree/) | Easy | — | Find the length of the longest path between any two nodes in the tree. | Post-order DFS returning height of each subtree; at each node, update a global max with `leftHeight + rightHeight` (the longest path through that node). |
| [Balanced Binary Tree](https://leetcode.com/problems/balanced-binary-tree/) | Easy | — | Determine if a binary tree is height-balanced. | Post-order DFS returning height, but return an early sentinel (e.g. -1) the moment any subtree is found unbalanced — propagate that failure up instead of recomputing heights top-down. |

## Solutions

```python
class TreeNode:
    def __init__(self, val=0, left=None, right=None):
        self.val = val
        self.left = left
        self.right = right
```

### Binary Tree Right Side View
```python
def right_side_view(root):
    if not root:
        return []
    res = []
    level = [root]
    while level:
        res.append(level[-1].val)  # last node processed in this level = rightmost
        next_level = []
        for node in level:
            if node.left:
                next_level.append(node.left)
            if node.right:
                next_level.append(node.right)
        level = next_level
    return res
```

### Binary Tree Maximum Path Sum
```python
def max_path_sum(root):
    best = float('-inf')

    def dfs(node):
        nonlocal best
        if not node:
            return 0
        left = max(dfs(node.left), 0)   # ignore negative branches
        right = max(dfs(node.right), 0)
        best = max(best, node.val + left + right)  # path bending through this node
        return node.val + max(left, right)  # best single branch to report upward

    dfs(root)
    return best
```

### Lowest Common Ancestor of a Binary Tree
```python
def lowest_common_ancestor(root, p, q):
    if not root or root is p or root is q:
        return root
    left = lowest_common_ancestor(root.left, p, q)
    right = lowest_common_ancestor(root.right, p, q)
    if left and right:
        return root  # p and q found on different sides -> this node is the LCA
    return left or right  # otherwise bubble up whichever side found something
```

### Maximum Depth of Binary Tree
```python
def max_depth(root):
    if not root:
        return 0
    return 1 + max(max_depth(root.left), max_depth(root.right))
```

### Same Tree
```python
def is_same_tree(p, q):
    if not p and not q:
        return True
    if not p or not q or p.val != q.val:
        return False
    return is_same_tree(p.left, q.left) and is_same_tree(p.right, q.right)
```

### Binary Tree Level Order Traversal
```python
def level_order(root):
    if not root:
        return []
    res = []
    level = [root]
    while level:
        values = []
        next_level = []
        for node in level:
            values.append(node.val)
            if node.left:
                next_level.append(node.left)
            if node.right:
                next_level.append(node.right)
        res.append(values)
        level = next_level
    return res
```

### Validate Binary Search Tree
```python
def is_valid_bst(root):
    def valid(node, low, high):
        if not node:
            return True
        if not (low < node.val < high):
            return False
        # pass down narrowed bounds, not just a comparison to the immediate parent
        return valid(node.left, low, node.val) and valid(node.right, node.val, high)

    return valid(root, float('-inf'), float('inf'))
```

### Kth Smallest Element in a BST
```python
def kth_smallest(root, k):
    stack = []
    curr = root
    while stack or curr:
        while curr:
            stack.append(curr)
            curr = curr.left
        curr = stack.pop()   # in-order visits BST nodes in ascending order
        k -= 1
        if k == 0:
            return curr.val
        curr = curr.right
```

### Construct Binary Tree from Preorder and Inorder Traversal
```python
def build_tree(preorder, inorder):
    idx_in_inorder = {val: i for i, val in enumerate(inorder)}
    pre_idx = 0

    def build(left, right):
        nonlocal pre_idx
        if left > right:
            return None
        root_val = preorder[pre_idx]  # preorder's next value is always the current root
        pre_idx += 1
        root = TreeNode(root_val)
        mid = idx_in_inorder[root_val]
        root.left = build(left, mid - 1)    # everything left of root in inorder
        root.right = build(mid + 1, right)  # everything right of root in inorder
        return root

    return build(0, len(inorder) - 1)
```

### Serialize and Deserialize Binary Tree
```python
def serialize(root):
    vals = []

    def dfs(node):
        if not node:
            vals.append('#')
            return
        vals.append(str(node.val))
        dfs(node.left)
        dfs(node.right)

    dfs(root)
    return ','.join(vals)


def deserialize(data):
    vals = iter(data.split(','))

    def build():
        val = next(vals)
        if val == '#':
            return None
        node = TreeNode(int(val))
        node.left = build()   # same preorder sequence used to serialize, so this rebuilds correctly
        node.right = build()
        return node

    return build()
```

### Diameter of Binary Tree
```python
def diameter_of_binary_tree(root):
    best = 0

    def height(node):
        nonlocal best
        if not node:
            return 0
        left = height(node.left)
        right = height(node.right)
        best = max(best, left + right)  # longest path passing through this node
        return 1 + max(left, right)

    height(root)
    return best
```

### Balanced Binary Tree
```python
def is_balanced(root):
    def height(node):
        if not node:
            return 0
        left = height(node.left)
        if left == -1:
            return -1  # already unbalanced somewhere below, propagate immediately
        right = height(node.right)
        if right == -1 or abs(left - right) > 1:
            return -1
        return 1 + max(left, right)

    return height(root) != -1
```
