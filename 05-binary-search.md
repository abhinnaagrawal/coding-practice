[← back to index](README.md)

# Binary Search

## When to recognize it
Sorted (or partially sorted, e.g. rotated) array and you need O(log n). Also: "binary search on the answer" — the search space is a *range of possible answers* (minimum capacity, minimum speed), not array indices, but the feasibility check is monotonic (true up to some point, then false, or vice versa).

## Core idea
Repeatedly halve the search space by checking the midpoint against a condition and discarding the half that can't contain the answer. Requires the search space to have a monotonic property — once you cross the boundary, the answer to "is this feasible" flips and never flips back.

## Gotchas
- `mid = lo + (hi - lo) // 2` — avoids overflow habit from other languages (moot in Python, but keep it as habit).
- Loop invariant: decide `lo <= hi` vs `lo < hi` once, stay consistent — mixing causes infinite loops or missed elements.
- "Binary search on the answer": the tell is a monotonic predicate over a range, not indices into an array.
- Rotated array: figure out which half is sorted before deciding which side to discard.

## Problems

| Problem | Difficulty | Freq | What it asks | Intuition |
|---|---|---|---|---|
| [Median of Two Sorted Arrays](https://leetcode.com/problems/median-of-two-sorted-arrays/) | Hard | 89 | Find the median of two sorted arrays combined, in better than O(m+n) — target O(log(min(m,n))). | Binary search for a partition point in the smaller array such that elements to the left of both partitions (combined) are all ≤ elements to the right — adjust the partition based on boundary comparisons. |
| [Search in Rotated Sorted Array](https://leetcode.com/problems/search-in-rotated-sorted-array/) | Medium | 62 | Find the index of a target in a rotated sorted array in O(log n). | At each step, compare `mid` to `lo`/`hi` to determine which half is the "normal" sorted half, then check if target falls in that half's range to decide which side to keep. |
| [Find Minimum in Rotated Sorted Array](https://leetcode.com/problems/find-minimum-in-rotated-sorted-array/) | Medium | — | Find the minimum element in a rotated sorted array. | Compare `nums[mid]` to `nums[hi]` — if greater, the minimum is to the right of mid; if smaller, minimum is at or left of mid. Narrow until `lo == hi`. |
| [Koko Eating Bananas](https://leetcode.com/problems/koko-eating-bananas/) | Medium | — | Find the minimum eating speed to finish all banana piles within h hours (binary search on the answer). | Binary search over possible eating speeds (1 to max pile); feasibility check is `sum(ceil(pile / speed) for pile in piles) <= h` — monotonic, so binary search applies directly. |
| [Find Peak Element](https://leetcode.com/problems/find-peak-element/) | Medium | — | Find an index where the element is greater than both neighbors. | Compare `nums[mid]` to `nums[mid+1]`: if increasing, a peak must exist to the right; if decreasing, a peak exists at or before mid — binary search toward the "uphill" side. |
| [Search a 2D Matrix](https://leetcode.com/problems/search-a-2d-matrix/) | Medium | — | Search for a target in a matrix sorted row-wise and column-wise. | Treat the matrix as a flattened sorted array and binary search with `row = idx // cols`, `col = idx % cols`; or start from the top-right corner and move left/down based on comparison. |
| [Time Based Key-Value Store](https://leetcode.com/problems/time-based-key-value-store/) | Medium | — | Design a structure supporting `set(key, value, timestamp)` and `get(key, timestamp)` returning the value at or before that time. | Store a list of `(timestamp, value)` per key, appended in increasing timestamp order (guaranteed by input); binary search that list for the largest timestamp ≤ the query. |

## Solutions

### Median of Two Sorted Arrays
```python
def find_median_sorted_arrays(nums1, nums2):
    if len(nums1) > len(nums2):
        nums1, nums2 = nums2, nums1  # always binary search on the smaller array

    m, n = len(nums1), len(nums2)
    lo, hi = 0, m
    while lo <= hi:
        i = (lo + hi) // 2         # partition index in nums1
        j = (m + n + 1) // 2 - i   # partition index in nums2, keeps left halves balanced

        left1 = nums1[i - 1] if i > 0 else float('-inf')
        right1 = nums1[i] if i < m else float('inf')
        left2 = nums2[j - 1] if j > 0 else float('-inf')
        right2 = nums2[j] if j < n else float('inf')

        if left1 <= right2 and left2 <= right1:  # valid partition found
            if (m + n) % 2 == 0:
                return (max(left1, left2) + min(right1, right2)) / 2
            return max(left1, left2)
        elif left1 > right2:
            hi = i - 1   # partition in nums1 is too far right
        else:
            lo = i + 1
```

### Search in Rotated Sorted Array
```python
def search_rotated(nums, target):
    lo, hi = 0, len(nums) - 1
    while lo <= hi:
        mid = (lo + hi) // 2
        if nums[mid] == target:
            return mid
        if nums[lo] <= nums[mid]:  # left half [lo..mid] is sorted
            if nums[lo] <= target < nums[mid]:
                hi = mid - 1
            else:
                lo = mid + 1
        else:  # right half [mid..hi] is sorted
            if nums[mid] < target <= nums[hi]:
                lo = mid + 1
            else:
                hi = mid - 1
    return -1
```

### Find Minimum in Rotated Sorted Array
```python
def find_min_rotated(nums):
    lo, hi = 0, len(nums) - 1
    while lo < hi:
        mid = (lo + hi) // 2
        if nums[mid] > nums[hi]:
            lo = mid + 1  # minimum lies to the right of mid
        else:
            hi = mid       # minimum is at or to the left of mid
    return nums[lo]
```

### Koko Eating Bananas
```python
import math

def min_eating_speed(piles, h):
    def hours_needed(speed):
        return sum(math.ceil(p / speed) for p in piles)

    lo, hi = 1, max(piles)
    while lo < hi:
        mid = (lo + hi) // 2
        if hours_needed(mid) <= h:  # feasible — try an even slower speed
            hi = mid
        else:
            lo = mid + 1
    return lo
```

### Find Peak Element
```python
def find_peak_element(nums):
    lo, hi = 0, len(nums) - 1
    while lo < hi:
        mid = (lo + hi) // 2
        if nums[mid] < nums[mid + 1]:
            lo = mid + 1  # still ascending, peak is further right
        else:
            hi = mid       # descending here, peak is at or before mid
    return lo
```

### Search a 2D Matrix
```python
def search_matrix(matrix, target):
    if not matrix or not matrix[0]:
        return False
    rows, cols = len(matrix), len(matrix[0])
    lo, hi = 0, rows * cols - 1
    while lo <= hi:
        mid = (lo + hi) // 2
        val = matrix[mid // cols][mid % cols]  # treat the matrix as a flattened sorted array
        if val == target:
            return True
        elif val < target:
            lo = mid + 1
        else:
            hi = mid - 1
    return False
```

### Time Based Key-Value Store
```python
import bisect

class TimeMap:
    def __init__(self):
        self.store = {}  # key -> list of (timestamp, value), timestamps increasing

    def set(self, key, value, timestamp):
        self.store.setdefault(key, []).append((timestamp, value))

    def get(self, key, timestamp):
        entries = self.store.get(key, [])
        # find the rightmost entry with timestamp <= query timestamp
        i = bisect.bisect_right(entries, (timestamp, chr(0x10FFFF)))
        return entries[i - 1][1] if i > 0 else ""
```
