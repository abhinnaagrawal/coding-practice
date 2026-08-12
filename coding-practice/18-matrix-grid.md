[← back to index](README.md)

# Matrix / Grid

## When to recognize it
Input is explicitly a 2D grid/matrix, and the problem asks about traversal, transformation (rotate, in-place edits), or connectivity between cells (which overlaps with Graphs — see [10-graphs.md](10-graphs.md)).

## Core idea
Most grid problems reduce to careful index bookkeeping — rows/cols, boundaries, and (for traversal) treating each cell as a graph node with up to 4 (or 8) neighbors. In-place transformations (rotate, zero-out) usually have a clever swap or marker trick to avoid O(n²) extra space.

## Gotchas
- Rotate in place: work in layers/rings, or transpose + reverse rows — know one method cold.
- Set Matrix Zeroes: use the first row/column themselves as markers instead of allocating a separate visited structure, if O(1) extra space is required.
- Spiral traversal: track four shrinking boundaries (top, bottom, left, right) rather than trying to compute direction from a formula.
- Grid BFS/DFS: same visited-set and boundary-check gotchas as general Graphs apply here.

## Problems

| Problem | Difficulty | Freq | What it asks | Intuition |
|---|---|---|---|---|
| [Spiral Matrix](https://leetcode.com/problems/spiral-matrix/) | Medium | 51 | Return all elements of a matrix in spiral order. | Track four shrinking boundaries (top, bottom, left, right); traverse right along the top, down along the right, left along the bottom, up along the left, shrinking the corresponding boundary after each side. |
| [Rotate Image](https://leetcode.com/problems/rotate-image/) | Medium | 50 | Rotate an n×n matrix 90 degrees clockwise, in place. | Transpose the matrix (swap `matrix[i][j]` with `matrix[j][i]`), then reverse each row — equivalent to a 90° clockwise rotation. |
| [Set Matrix Zeroes](https://leetcode.com/problems/set-matrix-zeroes/) | Medium | — | If an element is 0, set its entire row and column to 0, in place. | Use the first row and first column of the matrix itself as marker space (recording which rows/cols need zeroing), avoiding a separate O(m+n) structure. |
| [Word Search](https://leetcode.com/problems/word-search/) | Medium | — | Determine if a word exists in the grid, formed from adjacent letters. | DFS from every cell matching the word's first character, marking visited cells temporarily and un-marking on backtrack. |
| [Number of Islands](https://leetcode.com/problems/number-of-islands/) | Medium | 107 | Count connected land components in a grid of land/water cells. | Flood-fill (DFS or BFS) from every unvisited land cell, marking the whole connected component visited — each flood-fill triggered is one island. |
| [Search a 2D Matrix](https://leetcode.com/problems/search-a-2d-matrix/) | Medium | — | Search for a target in a matrix sorted row-wise and column-wise. | Treat the matrix as a flattened sorted array and binary search using `row = idx // cols`, `col = idx % cols`. |

## Solutions

### Spiral Matrix
```python
def spiral_order(matrix):
    res = []
    top, bottom = 0, len(matrix) - 1
    left, right = 0, len(matrix[0]) - 1

    while top <= bottom and left <= right:
        for c in range(left, right + 1):
            res.append(matrix[top][c])
        top += 1

        for r in range(top, bottom + 1):
            res.append(matrix[r][right])
        right -= 1

        if top <= bottom:
            for c in range(right, left - 1, -1):
                res.append(matrix[bottom][c])
            bottom -= 1

        if left <= right:
            for r in range(bottom, top - 1, -1):
                res.append(matrix[r][left])
            left += 1

    return res
```

### Rotate Image
```python
def rotate_image(matrix):
    n = len(matrix)
    for i in range(n):
        for j in range(i + 1, n):
            matrix[i][j], matrix[j][i] = matrix[j][i], matrix[i][j]  # transpose in place

    for row in matrix:
        row.reverse()  # transpose + reverse rows = 90-degree clockwise rotation
```

### Set Matrix Zeroes
```python
def set_zeroes(matrix):
    rows, cols = len(matrix), len(matrix[0])
    first_row_zero = any(matrix[0][c] == 0 for c in range(cols))
    first_col_zero = any(matrix[r][0] == 0 for r in range(rows))

    # use the first row/column as marker space instead of a separate structure
    for r in range(1, rows):
        for c in range(1, cols):
            if matrix[r][c] == 0:
                matrix[r][0] = 0
                matrix[0][c] = 0

    for r in range(1, rows):
        for c in range(1, cols):
            if matrix[r][0] == 0 or matrix[0][c] == 0:
                matrix[r][c] = 0

    if first_row_zero:
        for c in range(cols):
            matrix[0][c] = 0
    if first_col_zero:
        for r in range(rows):
            matrix[r][0] = 0
```

### Word Search
See [14-backtracking.md](14-backtracking.md) — DFS with temporary visited-marking and backtracking.

### Number of Islands
See [10-graphs.md](10-graphs.md) — flood-fill DFS/BFS from every unvisited land cell.

### Search a 2D Matrix
See [05-binary-search.md](05-binary-search.md) — binary search over the matrix treated as a flattened sorted array.
