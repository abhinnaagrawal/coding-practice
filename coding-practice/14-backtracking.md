[← back to index](/coding-practice/README.md)

# Backtracking

## When to recognize it
"Generate all," "find all combinations/permutations," or constraint satisfaction (place N things so no two conflict). Any time you need to explore a decision tree and abandon (backtrack) branches that can't lead to a valid solution.

## Core idea
Recursively build a partial solution one choice at a time; if a choice leads to a dead end (or you've built a complete valid solution), undo it (backtrack) and try the next option. Pruning — checking constraints before recursing deeper, not after — is what keeps this from being pure brute force.

## Gotchas
- Mutate-then-undo discipline: append/remove from the *same* path list, mirror every state change with its undo.
- Dedup in permutations/combinations with duplicates: sort first, skip `if i > start and nums[i] == nums[i-1]`.
- Pruning early (bounds check before recursing) is what separates a passing solution from TLE.

## Problems

| Problem | Difficulty | Freq | What it asks | Intuition |
|---|---|---|---|---|
| [Letter Combinations of a Phone Number](https://leetcode.com/problems/letter-combinations-of-a-phone-number/) | Medium | 41 | Given a digit string, generate all letter combinations it could represent on a phone keypad. | DFS building up a string one digit at a time, branching over each letter the current digit maps to, backtracking after each complete combination. |
| [Generate Parentheses](https://leetcode.com/problems/generate-parentheses/) | Medium | 41 | Generate all combinations of well-formed parentheses for n pairs. | DFS tracking counts of opens/closes used; only add `(` if opens < n, only add `)` if closes < opens — this guarantees well-formedness by construction. |
| [Subsets](https://leetcode.com/problems/subsets/) | Medium | — | Generate all possible subsets of a given array of unique elements. | DFS where at each element you branch into two paths: include it or don't — the recursion tree naturally enumerates all `2^n` subsets. |
| [Subsets II](https://leetcode.com/problems/subsets-ii/) | Medium | — | Same as Subsets, but the input may contain duplicates — no duplicate subsets in output. | Sort first, then during DFS skip an element if it equals the previous element *and* the previous one was skipped at this same recursion depth (avoids generating the same subset twice). |
| [Permutations](https://leetcode.com/problems/permutations/) | Medium | — | Generate all permutations of a given array. | DFS swapping elements into the current position (or tracking a "used" set), backtracking after exploring each branch to restore state for the next choice. |
| [Combination Sum](https://leetcode.com/problems/combination-sum/) | Medium | — | Find all combinations of numbers (reusable) that sum to a target. | DFS trying each candidate from the current index onward; since numbers are reusable, don't advance past the current index on inclusion; prune when the running sum exceeds the target. |
| [N-Queens](https://leetcode.com/problems/n-queens/) | Hard | — | Place n queens on an n×n board so no two attack each other. | Place queens row by row; track occupied columns and both diagonals with sets, skip any column/diagonal already under attack, backtrack when no valid column remains in a row. |
| [Word Search](https://leetcode.com/problems/word-search/) | Medium | — | Determine if a word exists in a grid, formed by adjacent (up/down/left/right) letters. | DFS from every cell matching the word's first letter, marking cells visited temporarily and un-marking on backtrack, matching the word character by character through adjacent cells. |
| [Palindrome Partitioning](https://leetcode.com/problems/palindrome-partitioning/) | Medium | — | Partition a string so that every substring in the partition is a palindrome; return all such partitions. | DFS trying every prefix of the remaining string; if that prefix is a palindrome, recurse on the rest, backtracking to try longer/shorter prefixes. |

## Solutions

### Letter Combinations of a Phone Number
```python
def letter_combinations(digits):
    if not digits:
        return []
    mapping = {
        '2': 'abc', '3': 'def', '4': 'ghi', '5': 'jkl',
        '6': 'mno', '7': 'pqrs', '8': 'tuv', '9': 'wxyz',
    }
    res = []

    def backtrack(i, path):
        if i == len(digits):
            res.append(''.join(path))
            return
        for c in mapping[digits[i]]:
            path.append(c)
            backtrack(i + 1, path)
            path.pop()  # undo, try the next letter for this digit

    backtrack(0, [])
    return res
```

### Generate Parentheses
See [07-stacks-queues.md](/coding-practice/07-stacks-queues.md) — backtracking tracking open/close counts.

### Subsets
```python
def subsets(nums):
    res = []

    def backtrack(i, path):
        if i == len(nums):
            res.append(path[:])  # copy — path keeps getting mutated after this
            return
        path.append(nums[i])
        backtrack(i + 1, path)  # branch: include nums[i]
        path.pop()
        backtrack(i + 1, path)  # branch: exclude nums[i]

    backtrack(0, [])
    return res
```

### Subsets II
```python
def subsets_with_dup(nums):
    nums.sort()  # duplicates must be adjacent for the skip-check below to work
    res = []

    def backtrack(start, path):
        res.append(path[:])
        for i in range(start, len(nums)):
            if i > start and nums[i] == nums[i - 1]:
                continue  # skip duplicate value at this recursion depth
            path.append(nums[i])
            backtrack(i + 1, path)
            path.pop()

    backtrack(0, [])
    return res
```

### Permutations
```python
def permute(nums):
    res = []

    def backtrack(path, used):
        if len(path) == len(nums):
            res.append(path[:])
            return
        for i, n in enumerate(nums):
            if used[i]:
                continue
            used[i] = True
            path.append(n)
            backtrack(path, used)
            path.pop()
            used[i] = False  # undo, so the next branch can reuse this element

    backtrack([], [False] * len(nums))
    return res
```

### Combination Sum
```python
def combination_sum(candidates, target):
    res = []

    def backtrack(start, path, remaining):
        if remaining == 0:
            res.append(path[:])
            return
        if remaining < 0:
            return  # prune: overshot the target
        for i in range(start, len(candidates)):
            path.append(candidates[i])
            backtrack(i, path, remaining - candidates[i])  # i, not i+1 — numbers are reusable
            path.pop()

    backtrack(0, [], target)
    return res
```

### N-Queens
```python
def solve_n_queens(n):
    res = []
    cols, diag1, diag2 = set(), set(), set()  # diag1: row-col, diag2: row+col
    board = [['.'] * n for _ in range(n)]

    def backtrack(row):
        if row == n:
            res.append([''.join(r) for r in board])
            return
        for col in range(n):
            if col in cols or (row - col) in diag1 or (row + col) in diag2:
                continue  # under attack from an existing queen
            cols.add(col); diag1.add(row - col); diag2.add(row + col)
            board[row][col] = 'Q'
            backtrack(row + 1)
            board[row][col] = '.'  # undo
            cols.remove(col); diag1.remove(row - col); diag2.remove(row + col)

    backtrack(0)
    return res
```

### Word Search
```python
def exist(board, word):
    rows, cols = len(board), len(board[0])

    def dfs(r, c, i):
        if i == len(word):
            return True
        if r < 0 or r >= rows or c < 0 or c >= cols or board[r][c] != word[i]:
            return False

        temp, board[r][c] = board[r][c], '#'  # mark visited temporarily
        found = (dfs(r + 1, c, i + 1) or dfs(r - 1, c, i + 1) or
                 dfs(r, c + 1, i + 1) or dfs(r, c - 1, i + 1))
        board[r][c] = temp  # backtrack — restore for other search paths

        return found

    for r in range(rows):
        for c in range(cols):
            if dfs(r, c, 0):
                return True
    return False
```

### Palindrome Partitioning
```python
def partition(s):
    res = []

    def is_pal(sub):
        return sub == sub[::-1]

    def backtrack(start, path):
        if start == len(s):
            res.append(path[:])
            return
        for end in range(start + 1, len(s) + 1):
            prefix = s[start:end]
            if is_pal(prefix):
                path.append(prefix)
                backtrack(end, path)
                path.pop()

    backtrack(0, [])
    return res
```
