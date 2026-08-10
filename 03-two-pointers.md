[← back to index](README.md)

# Two Pointers

## When to recognize it
Sorted array (or can be sorted), and the problem asks about pairs/triplets meeting a condition (sum, distance), or you need to process from both ends inward. If a brute force would be O(n²) comparing every pair, and the array is sorted, two pointers usually gets you to O(n).

## Core idea
Two indices move through the array based on a comparison — moving the left pointer up increases some value, moving the right pointer down decreases it. Sorted order gives you a monotonic relationship to exploit, so you never have to backtrack.

## Gotchas
- Empty array / single element — check before entering the loop.
- Forgetting to move a pointer inside a branch → infinite loop.
- Negative numbers before squaring/sorted-merge tricks — anchor at the sign-flip index first.
- Duplicates — decide upfront: skip dupes (3Sum) or allow them (merge).

## Problems

| Problem | Difficulty | Freq | What it asks | Intuition |
|---|---|---|---|---|
| [3Sum](https://leetcode.com/problems/3sum/) | Medium | 126 | Find all unique triplets in the array that sum to zero. | Sort the array, fix one element, then two-pointer search the rest for a pair summing to its negative — skip duplicate values to avoid duplicate triplets. |
| [Trapping Rain Water](https://leetcode.com/problems/trapping-rain-water/) | Hard | 129 | Given an elevation map, compute how much water it can trap after raining. | Two pointers from both ends, tracking the max height seen so far on each side; water at a position is `min(leftMax, rightMax) - height`, move the pointer on the smaller side inward. |
| [Container With Most Water](https://leetcode.com/problems/container-with-most-water/) | Medium | 86 | Given heights at each index, pick two lines that with the x-axis form the container holding the most water. | Start pointers at both ends; always move the pointer at the *shorter* line inward — moving the taller one can never increase the area, since the shorter line still caps it. |
| [Longest Palindromic Substring](https://leetcode.com/problems/longest-palindromic-substring/) | Medium | 93 | Find the longest palindromic substring (expand-from-center is a two-pointer technique). | Expand two pointers outward from every center, stop when characters stop matching, track the longest valid span. |
| [Rotate Array](https://leetcode.com/problems/rotate-array/) | Medium | 51 | Rotate an array to the right by k steps, in place. | Reverse the whole array, then reverse the first `k` elements, then reverse the remaining `n-k` — three reversals achieve the rotation in O(1) extra space. |
| [Valid Palindrome](https://leetcode.com/problems/valid-palindrome/) | Easy | 50 | Determine if a string is a palindrome ignoring non-alphanumeric characters and case. | Two pointers from both ends, skip non-alphanumeric characters, compare lowercased characters as they converge. |
| [Two Sum II (Input Array Is Sorted)](https://leetcode.com/problems/two-sum-ii-input-array-is-sorted/) | Medium | — | Same as Two Sum, but input is sorted — solve with O(1) extra space via two pointers. | Pointer at each end; if the sum is too big move the right pointer left, if too small move the left pointer right, exploiting sorted order instead of hashing. |
| [Sort Colors](https://leetcode.com/problems/sort-colors/) | Medium | 40 | Sort an array of 0s, 1s, 2s in place in one pass (Dutch national flag — three pointers). | Three pointers: `low` (boundary for 0s), `mid` (current), `high` (boundary for 2s) — swap 0s to the front, 2s to the back, leave 1s in the middle as you scan once. |
| [Move Zeroes](https://leetcode.com/problems/move-zeroes/) | Easy | 45 | Move all zeroes in an array to the end while keeping relative order of non-zero elements. | Pointer tracking the next position to write a non-zero value; scan and swap non-zero elements into place as you go, zeros naturally end up at the back. |

## Solutions

### 3Sum
```python
def three_sum(nums):
    nums.sort()
    res = []
    n = len(nums)
    for i in range(n):
        if i > 0 and nums[i] == nums[i - 1]:
            continue  # skip duplicate anchor to avoid duplicate triplets
        if nums[i] > 0:
            break  # sorted array: once the anchor is positive, no triplet can sum to 0

        l, r = i + 1, n - 1
        while l < r:
            total = nums[i] + nums[l] + nums[r]
            if total < 0:
                l += 1
            elif total > 0:
                r -= 1
            else:
                res.append([nums[i], nums[l], nums[r]])
                l += 1
                r -= 1
                while l < r and nums[l] == nums[l - 1]:
                    l += 1  # skip duplicate left values
                while l < r and nums[r] == nums[r + 1]:
                    r -= 1  # skip duplicate right values
    return res
```

### Trapping Rain Water
```python
def trap(height):
    l, r = 0, len(height) - 1
    left_max, right_max = 0, 0
    water = 0
    while l < r:
        if height[l] < height[r]:
            left_max = max(left_max, height[l])
            water += left_max - height[l]  # bounded by the shorter side's max so far
            l += 1
        else:
            right_max = max(right_max, height[r])
            water += right_max - height[r]
            r -= 1
    return water
```

### Container With Most Water
```python
def max_area(height):
    l, r = 0, len(height) - 1
    best = 0
    while l < r:
        best = max(best, (r - l) * min(height[l], height[r]))
        if height[l] < height[r]:
            l += 1  # the shorter line is the bottleneck, move it
        else:
            r -= 1
    return best
```

### Rotate Array
```python
def rotate_array(nums, k):
    n = len(nums)
    k %= n

    def reverse(l, r):
        while l < r:
            nums[l], nums[r] = nums[r], nums[l]
            l += 1
            r -= 1

    reverse(0, n - 1)   # reverse the whole array
    reverse(0, k - 1)    # then reverse the first k elements
    reverse(k, n - 1)    # then reverse the rest
```

### Valid Palindrome
```python
def is_palindrome(s):
    l, r = 0, len(s) - 1
    while l < r:
        while l < r and not s[l].isalnum():
            l += 1
        while l < r and not s[r].isalnum():
            r -= 1
        if s[l].lower() != s[r].lower():
            return False
        l += 1
        r -= 1
    return True
```

### Two Sum II (Input Array Is Sorted)
```python
def two_sum_sorted(numbers, target):
    l, r = 0, len(numbers) - 1
    while l < r:
        total = numbers[l] + numbers[r]
        if total == target:
            return [l + 1, r + 1]  # problem expects 1-indexed result
        elif total < target:
            l += 1
        else:
            r -= 1
    return []
```

### Sort Colors
```python
def sort_colors(nums):
    low, mid, high = 0, 0, len(nums) - 1
    while mid <= high:
        if nums[mid] == 0:
            nums[low], nums[mid] = nums[mid], nums[low]
            low += 1
            mid += 1
        elif nums[mid] == 1:
            mid += 1
        else:  # nums[mid] == 2
            nums[mid], nums[high] = nums[high], nums[mid]
            high -= 1  # don't advance mid — the swapped-in value still needs checking
```

### Move Zeroes
```python
def move_zeroes(nums):
    write = 0  # next position to place a non-zero value
    for read in range(len(nums)):
        if nums[read] != 0:
            nums[write], nums[read] = nums[read], nums[write]
            write += 1
```
