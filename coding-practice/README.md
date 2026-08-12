# Coding Interview Prep — Senior/Staff FAANG+ Reference

Live site: **https://abhinnaagrawal.github.io/coding-practice/#/coding-practice/**
Top-level site: [Home](/README.md)

## How this section is structured

```
coding-practice/            this section — pattern reference for coding interviews
├── README.md                this file — landing page for the section
└── 01-arrays-hashing.md … 19-design-data-structures.md
                              one file per pattern/category

(repo root also has: index.html, _sidebar.md, .nojekyll — see ../README.md)
```

**Each category file (`0N-name.md`) follows the same four-section shape:**
1. **When to recognize it** — the tell in a problem statement that signals this pattern.
2. **Core idea** — the technique in a paragraph.
3. **Gotchas** — where solutions actually break in practice (off-by-ones, edge cases, wrong invariants).
4. **Problems table** — name, difficulty, company-frequency score (from a live dataset, see below), one-line description, and a one-line intuition — followed by a **Solutions** section with a commented Python implementation for every problem, in the same order as the table.

Problems that recur across multiple categories (e.g. "Trapping Rain Water" fits both Two Pointers and DP) have their full solution written once in whichever category is most canonical for it, and every other occurrence cross-links to that file instead of duplicating code.

## How to add a new problem or category

- **New problem in an existing category**: add a row to that file's problem table, then add a matching `### Problem Name` + \`\`\`python code block under its `## Solutions` section, same order as the table.
- **New category**: copy the four-section shape from any existing file, add it to `_sidebar.md` (repo root) and to the category table below.

For hosting mechanics (docsify, GitHub Pages, how updates deploy), see [Home](/README.md).

Data note: problem lists and frequency numbers below are pulled from the live dataset in `seanprashad/leetcode-patterns` (`src/data/questions.json`), snapshot dated **2026-08-02** — 179 problems, each tagged with real per-company interview-frequency counts (community-reported, not official LeetCode disclosure). Full source: [seanprashad.com/leetcode-patterns](https://seanprashad.com/leetcode-patterns/).

## Why patterns, not raw grinding
~87% of FAANG coding questions map to 10-12 core patterns — recognizing the pattern fast is what's actually being scored at senior level, not whether you've memorized the exact problem. At senior+, the coding round gates the offer; the system-design round sets the level/comp.

## Categories

| # | Category | Problems in dataset | Doc |
|---|---|---|---|
| 1 | Arrays & Hashing | 96 (Array) / 37 (Hash Table) | [01-arrays-hashing.md](01-arrays-hashing.md) |
| 2 | String Manipulation | 39 | [02-string-manipulation.md](02-string-manipulation.md) |
| 3 | Two Pointers | 27 | [03-two-pointers.md](03-two-pointers.md) |
| 4 | Sliding Window | 13 | [04-sliding-window.md](04-sliding-window.md) |
| 5 | Binary Search | 18 | [05-binary-search.md](05-binary-search.md) |
| 6 | Linked Lists | 18 | [06-linked-lists.md](06-linked-lists.md) |
| 7 | Stacks & Queues | 7 (Stack) | [07-stacks-queues.md](07-stacks-queues.md) |
| 8 | Intervals & Greedy | 10 (Greedy) | [08-intervals-greedy.md](08-intervals-greedy.md) |
| 9 | Trees & BSTs | 25 | [09-trees-bst.md](09-trees-bst.md) |
| 10 | Graphs | 33 (DFS) / 24 (BFS) / 7 (Graph Theory) | [10-graphs.md](10-graphs.md) |
| 11 | Topological Sort | 4 | [11-topological-sort.md](11-topological-sort.md) |
| 12 | Union-Find | 4 | [12-union-find.md](12-union-find.md) |
| 13 | Heaps & Priority Queues | 18 | [13-heaps-priority-queues.md](13-heaps-priority-queues.md) |
| 14 | Backtracking | 23 | [14-backtracking.md](14-backtracking.md) |
| 15 | Dynamic Programming | 30 | [15-dynamic-programming.md](15-dynamic-programming.md) |
| 16 | Trie | 10 | [16-trie.md](16-trie.md) |
| 17 | Bit Manipulation | 12 | [17-bit-manipulation.md](17-bit-manipulation.md) |
| 18 | Matrix / Grid | 12 | [18-matrix-grid.md](18-matrix-grid.md) |
| 19 | Design a Data Structure | 9 | [19-design-data-structures.md](19-design-data-structures.md) |

Each doc has three sections: **when to recognize it** (the signal in the problem statement), **core idea** (how the technique works), **gotchas** (where it breaks in practice), and a **problem list** with one-line descriptions so you know what each problem is without re-reading it on LeetCode.

## Top 25 problems overall, by summed company-frequency
The "if you only have a week" shortlist — spans multiple categories:

1. Two Sum (Easy, freq 433)
2. Longest Substring Without Repeating Characters (Medium, 160)
3. Best Time to Buy and Sell Stock (Easy, 130)
4. Trapping Rain Water (Hard, 129)
5. 3Sum (Medium, 126)
6. Add Two Numbers (Medium, 117)
7. Number of Islands (Medium, 107)
8. Group Anagrams (Medium, 105)
9. Merge Intervals (Medium, 104)
10. Longest Palindromic Substring (Medium, 93)
11. Median of Two Sorted Arrays (Hard, 89)
12. Valid Parentheses (Easy, 87)
13. Container With Most Water (Medium, 86)
14. Maximum Subarray (Medium, 69)
15. Longest Consecutive Sequence (Medium, 66)
16. Search in Rotated Sorted Array (Medium, 62)
17. Top K Frequent Elements (Medium, 62)
18. Majority Element (Easy, 60)
19. Climbing Stairs (Easy, 53)
20. Spiral Matrix (Medium, 51)
21. Course Schedule (Medium, 51)
22. Rotate Array (Medium, 51)
23. Rotate Image (Medium, 50)
24. Valid Anagram (Easy, 50)
25. Valid Palindrome (Easy, 50)

## Meta-strategy for the round itself
1. Restate problem + constraints, ask about input size — tells you the required time complexity.
2. Say the pattern out loud before coding ("this smells like sliding window because...") — signals recognition speed, which is what's scored at senior level.
3. Walk a tiny example (2-3 elements) by hand before writing code — catches off-by-ones early.
4. State base cases and loop invariants explicitly before writing the loop.
5. After coding, re-trace on the tiny example, then explicitly call out edge cases (empty input, single element, all-duplicates, negatives) even if not asked.
