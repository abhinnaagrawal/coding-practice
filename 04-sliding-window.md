[← back to index](README.md)

# Sliding Window

## When to recognize it
"Longest/shortest/max/min substring or subarray satisfying condition X." Fixed-size window ("every subarray of size k") or variable-size window (grows until invalid, then shrinks) — both are sliding window.

## Core idea
Maintain a window `[left, right]` over the array/string. Expand `right` to include more elements; when the window violates the condition, shrink from `left` until valid again. Because both pointers only move forward, total work is O(n), not O(n²) from re-scanning each window from scratch.

## Gotchas
- Window shrink condition must run in a `while`, not `if` — one shrink often isn't enough.
- Off-by-one on window size: `right - left + 1` for inclusive bounds.
- Character frequency map: remember to delete the key when count hits 0 (affects "distinct count" checks).
- Fixed-size window: subtract the element leaving, add the element entering — don't recompute the sum from scratch each slide.

## Problems

| Problem | Difficulty | Freq | What it asks | Intuition |
|---|---|---|---|---|
| [Longest Substring Without Repeating Characters](https://leetcode.com/problems/longest-substring-without-repeating-characters/) | Medium | 160 | Find the length of the longest substring with no repeated characters. | Expand `right`, track last-seen index of each character; when a repeat appears inside the window, jump `left` past its previous occurrence. |
| [Sliding Window Maximum](https://leetcode.com/problems/sliding-window-maximum/) | Hard | 41 | For each window of size k sliding across the array, return the max element (monotonic deque). | Maintain a deque of indices in decreasing value order; pop smaller elements from the back before adding a new one, pop from the front when it falls outside the window — the front is always the current max. |
| [Minimum Window Substring](https://leetcode.com/problems/minimum-window-substring/) | Hard | — | Find the smallest substring of s containing every character of t (with counts). | Expand `right` until the window contains all required characters (track counts and a "satisfied" counter), then shrink `left` as far as possible while still valid, recording the smallest valid window seen. |
| [Longest Repeating Character Replacement](https://leetcode.com/problems/longest-repeating-character-replacement/) | Medium | — | Find the longest substring achievable by replacing at most k characters, that consists of one repeated letter. | Window is valid while `(window size - count of most frequent character in window) <= k` — expand right, shrink left only when that condition breaks. |
| [Fruit Into Baskets](https://leetcode.com/problems/fruit-into-baskets/) | Medium | 41 | Find the longest subarray containing at most 2 distinct values (disguised sliding window). | Maintain a frequency map of values in the window; when distinct count exceeds 2, shrink from the left (decrementing/removing counts) until back to 2 types. |
| [Best Time to Buy and Sell Stock](https://leetcode.com/problems/best-time-to-buy-and-sell-stock/) | Easy | 130 | Given daily prices, find max profit from one buy then one sell (single-pass min-tracking, window-adjacent). | Track the minimum price seen so far while scanning; at each day, profit = `price - minSoFar`, keep the max profit across all days. |

## Solutions

### Longest Substring Without Repeating Characters
```python
def length_of_longest_substring(s):
    seen = {}  # char -> last index seen
    left = 0
    longest = 0
    for right, ch in enumerate(s):
        if ch in seen and seen[ch] >= left:
            left = seen[ch] + 1
        seen[ch] = right
        longest = max(longest, right - left + 1)
    return longest
```

### Sliding Window Maximum
```python
from collections import deque

def max_sliding_window(nums, k):
    dq = deque()  # stores indices; values at those indices are decreasing
    res = []
    for i, n in enumerate(nums):
        while dq and nums[dq[-1]] < n:
            dq.pop()  # remove indices whose value can never be the max again
        dq.append(i)
        if dq[0] <= i - k:
            dq.popleft()  # front index has fallen outside the window
        if i >= k - 1:
            res.append(nums[dq[0]])  # front of deque is always the current window's max
    return res
```

### Minimum Window Substring
```python
from collections import Counter

def min_window(s, t):
    if not t:
        return ""
    need = Counter(t)
    missing = len(t)  # total characters still needed (with multiplicity)
    left = start = end = 0

    for right, ch in enumerate(s, 1):
        if need[ch] > 0:
            missing -= 1
        need[ch] -= 1

        if missing == 0:  # window currently contains all of t
            while need[s[left]] < 0:
                need[s[left]] += 1  # shrink away characters we have extra of
                left += 1
            if end == 0 or right - left < end - start:
                start, end = left, right
            need[s[left]] += 1  # release leftmost char to try shrinking further next time
            missing += 1
            left += 1

    return s[start:end]
```

### Longest Repeating Character Replacement
```python
def character_replacement(s, k):
    counts = {}
    left = 0
    max_count = 0
    longest = 0
    for right, ch in enumerate(s):
        counts[ch] = counts.get(ch, 0) + 1
        max_count = max(max_count, counts[ch])
        window_len = right - left + 1
        if window_len - max_count > k:  # more replacements needed than allowed
            counts[s[left]] -= 1
            left += 1
        longest = max(longest, right - left + 1)
    return longest
```

### Fruit Into Baskets
```python
def total_fruit(fruits):
    counts = {}
    left = 0
    longest = 0
    for right, f in enumerate(fruits):
        counts[f] = counts.get(f, 0) + 1
        while len(counts) > 2:  # more than 2 distinct fruit types in window
            counts[fruits[left]] -= 1
            if counts[fruits[left]] == 0:
                del counts[fruits[left]]
            left += 1
        longest = max(longest, right - left + 1)
    return longest
```

### Best Time to Buy and Sell Stock
```python
def max_profit(prices):
    min_price = float('inf')
    best = 0
    for p in prices:
        min_price = min(min_price, p)
        best = max(best, p - min_price)
    return best
```
