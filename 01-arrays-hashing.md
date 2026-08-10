[← back to index](README.md)

# Arrays & Hashing

## When to recognize it
Problem mentions "find pair/triplet," "count occurrences," "duplicates," or needs O(1) lookup instead of O(n) scan. If you'd otherwise write a nested loop to compare every pair, a hash map usually collapses it to one pass.

## Core idea
Trade space for time: a hash map/set gives O(1) average lookup, so instead of scanning the array again to check "does X exist," you build the map once (O(n)) and query it in the same pass or a second pass. Prefix sums extend this to range-sum queries — precompute cumulative sums so any subarray sum is a subtraction, not a re-scan.

## Gotchas
- Hashing floats/mutable objects — unreliable keys; stick to hashable immutable types.
- Counting problems: decide up front whether you need frequency (`dict`) or just presence (`set`) — cheaper structure, cheaper code.
- Two Sum-style: check "does complement exist" *before* inserting current element, or you'll match an element with itself.
- Prefix sum off-by-one: `prefix[i]` = sum of first `i` elements means `prefix[0] = 0` — subarray sum `(l, r)` is `prefix[r+1] - prefix[l]`.

## Problems

| Problem | Difficulty | Freq | What it asks | Intuition |
|---|---|---|---|---|
| [Two Sum](https://leetcode.com/problems/two-sum/) | Easy | 433 | Given an array and a target, return indices of the two numbers that add up to target. | Hash map of value→index while scanning; for each element check if `target - element` already exists in the map before inserting current element. |
| [Group Anagrams](https://leetcode.com/problems/group-anagrams/) | Medium | 105 | Given a list of strings, group the ones that are anagrams of each other. | Anagrams share the same sorted string (or same 26-char frequency count) — use that as the hashmap key, group strings under it. |
| [Longest Consecutive Sequence](https://leetcode.com/problems/longest-consecutive-sequence/) | Medium | 66 | Given an unsorted array, find the length of the longest run of consecutive integers (no sorting allowed for full credit — O(n)). | Put all numbers in a set. Only start counting a sequence from a number `n` where `n-1` is *not* in the set (that's a sequence start) — walk `n+1, n+2, ...` while present. |
| [Top K Frequent Elements](https://leetcode.com/problems/top-k-frequent-elements/) | Medium | 62 | Return the k most frequently occurring elements in an array. | Count frequencies with a hashmap, then either bucket-sort by frequency (O(n)) or use a min-heap of size k (O(n log k)). |
| [Majority Element](https://leetcode.com/problems/majority-element/) | Easy | 60 | Find the element that appears more than n/2 times (Boyer-Moore voting is the O(1)-space trick). | Boyer-Moore: keep a candidate and a count; increment count on match, decrement on mismatch, swap candidate when count hits 0 — the majority element always survives. |
| [Contains Duplicate](https://leetcode.com/problems/contains-duplicate/) | Easy | — | Determine if any value appears at least twice in the array. | Add each element to a set as you scan; if it's already there, return true immediately. |
| [Product of Array Except Self](https://leetcode.com/problems/product-of-array-except-self/) | Medium | — | Return an array where each element is the product of all others, without using division. | Two passes: prefix product (product of everything to the left) going forward, suffix product (product of everything to the right) going backward, multiply the two for each index. |
| [Valid Anagram](https://leetcode.com/problems/valid-anagram/) | Easy | 50 | Determine if two strings are anagrams of each other. | Count character frequencies of both strings and compare — equal counts means anagram. |
| [First Missing Positive](https://leetcode.com/problems/first-missing-positive/) | Hard | 42 | Find the smallest missing positive integer in O(n) time, O(1) extra space (in-place index-marking trick). | Use the array itself as a hash: swap each value `v` (if in range `1..n`) to index `v-1`. After placing everything, scan for the first index whose value doesn't match `index+1`. |

## Solutions

### Two Sum
```python
def two_sum(nums, target):
    seen = {}  # value -> index seen so far
    for i, n in enumerate(nums):
        complement = target - n
        if complement in seen:
            return [seen[complement], i]
        seen[n] = i  # insert after checking, so we never match an element with itself
    return []
```

### Group Anagrams
```python
def group_anagrams(strs):
    groups = {}
    for s in strs:
        key = tuple(sorted(s))  # anagrams share the same sorted character sequence
        groups.setdefault(key, []).append(s)
    return list(groups.values())
```

### Longest Consecutive Sequence
```python
def longest_consecutive(nums):
    num_set = set(nums)
    longest = 0
    for n in num_set:
        if n - 1 not in num_set:  # only start counting from the beginning of a run
            length = 1
            while n + length in num_set:
                length += 1
            longest = max(longest, length)
    return longest
```

### Top K Frequent Elements
```python
import heapq
from collections import Counter

def top_k_frequent(nums, k):
    counts = Counter(nums)
    return heapq.nlargest(k, counts.keys(), key=counts.get)
```

### Majority Element
```python
def majority_element(nums):
    candidate, count = None, 0
    for n in nums:
        if count == 0:
            candidate = n  # start a new candidate when the running count hits 0
        count += 1 if n == candidate else -1
    return candidate
```

### Contains Duplicate
```python
def contains_duplicate(nums):
    seen = set()
    for n in nums:
        if n in seen:
            return True
        seen.add(n)
    return False
```

### Product of Array Except Self
```python
def product_except_self(nums):
    n = len(nums)
    res = [1] * n

    prefix = 1
    for i in range(n):
        res[i] = prefix       # product of everything to the left of i
        prefix *= nums[i]

    suffix = 1
    for i in range(n - 1, -1, -1):
        res[i] *= suffix      # multiply in the product of everything to the right of i
        suffix *= nums[i]

    return res
```

### Valid Anagram
```python
from collections import Counter

def is_anagram(s, t):
    return Counter(s) == Counter(t)  # same character frequencies means anagram
```

### First Missing Positive
```python
def first_missing_positive(nums):
    n = len(nums)
    # place each value v (1..n) at index v-1, using swaps, in place
    for i in range(n):
        while 1 <= nums[i] <= n and nums[nums[i] - 1] != nums[i]:
            correct_idx = nums[i] - 1
            nums[i], nums[correct_idx] = nums[correct_idx], nums[i]

    for i in range(n):
        if nums[i] != i + 1:
            return i + 1  # first index that doesn't hold its "correct" value

    return n + 1  # all of 1..n present, so the answer is n+1
```
