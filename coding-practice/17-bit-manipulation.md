[← back to index](/coding-practice/README.md)

# Bit Manipulation

## When to recognize it
"Without using extra space," "find the single/odd element," subset enumeration, or the problem explicitly deals with binary representation.

## Core idea
Bitwise ops let you encode set membership, toggle state, or do arithmetic without extra data structures. XOR cancels identical pairs — that's the trick behind most "find the unique element" problems. Bitmasks represent subsets as integers, useful when subset count (`2^n`) is small enough for DP over masks.

## Gotchas
- XOR cancels pairs — the "find the single/odd-one-out" family all reduce to this.
- `x & (x-1)` clears the lowest set bit; `x & -x` isolates the lowest set bit — both come up in subset/count problems.
- Bitmask DP: state space is `2^n` — only viable when `n` is small (~20 or less); recognize the size hint in the problem.
- Python ints are unbounded — `~x` behaves as `-x - 1`, not a fixed-width bit flip like in C/Java.

## Problems

| Problem | Difficulty | Freq | What it asks | Intuition |
|---|---|---|---|---|
| [Single Number](https://leetcode.com/problems/single-number/) | Easy | — | Find the element that appears once while every other element appears exactly twice (XOR all elements). | XOR every element together — identical pairs cancel to 0 (`x ^ x = 0`), leaving only the element that appeared once. |
| [Number of 1 Bits](https://leetcode.com/problems/number-of-1-bits/) | Easy | — | Count the number of set bits in an integer. | Repeatedly apply `n = n & (n-1)` (clears the lowest set bit) and count iterations until n becomes 0. |
| [Counting Bits](https://leetcode.com/problems/counting-bits/) | Easy | — | For every number from 0 to n, count its number of set bits. | `dp[i] = dp[i >> 1] + (i & 1)` — the bit count of `i` is the bit count of `i` with its last bit dropped, plus that last bit. |
| [Reverse Bits](https://leetcode.com/problems/reverse-bits/) | Easy | — | Reverse the bits of a 32-bit unsigned integer. | Loop 32 times: shift the result left by 1, OR in the lowest bit of `n`, then shift `n` right by 1 — building the reversed number bit by bit. |
| [Missing Number](https://leetcode.com/problems/missing-number/) | Easy | — | Given an array containing n distinct numbers from 0 to n, find the one missing (XOR trick works, sum trick also works). | XOR every index `0..n` with every array value — all present numbers cancel out, leaving the missing one (or simpler: `n*(n+1)/2 - sum(nums)`). |
| [Sum of Two Integers](https://leetcode.com/problems/sum-of-two-integers/) | Medium | — | Add two integers without using the `+` or `-` operators (bit manipulation with carry). | `a ^ b` gives the sum without carrying, `(a & b) << 1` gives the carry — repeat, feeding the carry back in, until the carry becomes 0. |
| [Find the Duplicate Number](https://leetcode.com/problems/find-the-duplicate-number/) | Medium | — | Array of n+1 integers in range 1..n — find the one duplicate without extra space (also solvable via Floyd's cycle detection). | Treat the array as a function `f(i) = nums[i]` mapping indices to values — a duplicate value means two indices point to the same place, creating a cycle findable with Floyd's fast/slow pointer technique. |
| [Subsets](https://leetcode.com/problems/subsets/) | Medium | — | Generate all subsets of an array — bitmask over `2^n` is an alternative to backtracking. | Iterate every integer from `0` to `2^n - 1`; bit `i` of the integer decides whether `nums[i]` is included in that subset. |

## Solutions

### Single Number
```python
def single_number(nums):
    result = 0
    for n in nums:
        result ^= n  # x ^ x = 0, so every pair cancels, leaving the unique element
    return result
```

### Number of 1 Bits
```python
def hamming_weight(n):
    count = 0
    while n:
        n &= n - 1  # clears the lowest set bit each iteration
        count += 1
    return count
```

### Counting Bits
```python
def count_bits(n):
    dp = [0] * (n + 1)
    for i in range(1, n + 1):
        dp[i] = dp[i >> 1] + (i & 1)  # bit count of i = bit count of (i without last bit) + last bit
    return dp
```

### Reverse Bits
```python
def reverse_bits(n):
    result = 0
    for _ in range(32):
        result = (result << 1) | (n & 1)  # shift result left, append n's lowest bit
        n >>= 1
    return result
```

### Missing Number
```python
def missing_number(nums):
    n = len(nums)
    return n * (n + 1) // 2 - sum(nums)  # expected sum of 0..n minus the actual sum
```

### Sum of Two Integers
```python
def get_sum(a, b):
    mask = 0xFFFFFFFF  # simulate 32-bit overflow, since Python ints are unbounded
    while b & mask:
        carry = (a & b) << 1  # bits where both a and b are 1 produce a carry
        a = a ^ b              # add without carrying
        b = carry
    return a & mask if b > mask else a  # a might need sign-reinterpretation
```

### Find the Duplicate Number
```python
def find_duplicate(nums):
    # treat nums as a function f(i) = nums[i]; a duplicate value creates a cycle
    slow = fast = 0
    while True:
        slow = nums[slow]
        fast = nums[nums[fast]]
        if slow == fast:
            break

    slow2 = 0
    while slow != slow2:
        slow = nums[slow]
        slow2 = nums[slow2]
    return slow  # the cycle's entry point is the duplicate value
```

### Subsets (bitmask version)
```python
def subsets_bitmask(nums):
    n = len(nums)
    res = []
    for mask in range(1 << n):  # every integer from 0 to 2^n - 1
        subset = [nums[i] for i in range(n) if mask & (1 << i)]  # bit i decides inclusion
        res.append(subset)
    return res
```
