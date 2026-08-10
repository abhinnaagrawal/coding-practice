[← back to index](README.md)

# Dynamic Programming

## When to recognize it
"Count the number of ways," "find the min/max," and the problem has overlapping subproblems — the brute-force recursive solution would recompute the same smaller cases repeatedly. If you can describe the answer for size `n` in terms of the answer for smaller sizes, it's DP.

## Core idea
Define a state (`dp[i]` or `dp[i][j]`) that captures "the answer to the subproblem ending/starting at this point." Write the recurrence — how does `dp[i]` relate to earlier states? Solve base cases first, then build up (bottom-up/iterative) or recurse with memoization (top-down) — same math, different control flow.

## Gotchas
- Define the state explicitly before coding: "what does `dp[i]` mean in one sentence" — if you can't say it, you'll get the transition wrong.
- Base cases first, always — most DP bugs are wrong/missing base cases, not wrong transitions.
- 2D DP: watch the off-by-one between "index into array" and "index into dp table" (often offset by 1 for an empty-prefix row/col).
- Space optimization (rolling array) — only attempt after the correct 2D version works; premature optimization causes silent bugs.
- Top-down (memoized recursion) vs bottom-up (iterative table) — pick whichever you can state the recurrence for fastest under pressure; interviewers rarely penalize the choice.

## Problems

| Problem | Difficulty | Freq | What it asks | Intuition |
|---|---|---|---|---|
| [Best Time to Buy and Sell Stock](https://leetcode.com/problems/best-time-to-buy-and-sell-stock/) | Easy | 130 | Given daily prices, find max profit from one buy then one sell. | Track the minimum price seen so far; at each day, `profit = price - minSoFar`, keep the running max. |
| [Trapping Rain Water](https://leetcode.com/problems/trapping-rain-water/) | Hard | 129 | Given an elevation map, compute how much water it can trap. | Precompute `leftMax[i]` and `rightMax[i]` arrays (max height to the left/right of each index); water at `i` is `min(leftMax[i], rightMax[i]) - height[i]`. |
| [Longest Palindromic Substring](https://leetcode.com/problems/longest-palindromic-substring/) | Medium | 93 | Find the longest palindromic substring. | `dp[i][j] = true` if `s[i] == s[j]` and `dp[i+1][j-1]` is true — but expand-from-center is simpler and equally efficient in practice. |
| [Maximum Subarray](https://leetcode.com/problems/maximum-subarray/) | Medium | 69 | Find the contiguous subarray with the largest sum (Kadane's algorithm). | Keep a running sum; whenever it drops below the current element alone, reset it — track the max running sum seen throughout. |
| [Climbing Stairs](https://leetcode.com/problems/climbing-stairs/) | Easy | 53 | Count the distinct ways to climb n stairs, taking 1 or 2 steps at a time. | `dp[i] = dp[i-1] + dp[i-2]` — the number of ways to reach step i is the sum of ways to reach the two steps before it (Fibonacci in disguise). |
| [House Robber](https://leetcode.com/problems/house-robber/) | Medium | 44 | Maximize money robbed from a line of houses without robbing two adjacent ones. | `dp[i] = max(dp[i-1], dp[i-2] + nums[i])` — at each house, either skip it (take the best up to the previous house) or rob it (best up to two houses back, plus this house). |
| [House Robber II](https://leetcode.com/problems/house-robber-ii/) | Medium | — | Same as House Robber, but houses are arranged in a circle. | Run the linear House Robber DP twice — once excluding the first house, once excluding the last — and take the max, since the circular constraint only matters for that one wraparound pair. |
| [Coin Change](https://leetcode.com/problems/coin-change/) | Medium | 43 | Find the minimum number of coins to make a target amount (or -1 if impossible). | `dp[amount] = min(dp[amount - coin] + 1)` over all coins, with `dp[0] = 0` as the base case — build up from 0 to the target amount. |
| [Longest Increasing Subsequence](https://leetcode.com/problems/longest-increasing-subsequence/) | Medium | — | Find the length of the longest strictly increasing subsequence. | `dp[i] = 1 + max(dp[j])` for all `j < i` with `nums[j] < nums[i]` — O(n²); or maintain a "patience sorting" array with binary search for O(n log n). |
| [Word Break](https://leetcode.com/problems/word-break/) | Medium | — | Determine if a string can be segmented into a sequence of dictionary words. | `dp[i] = true` if there's some `j < i` where `dp[j]` is true and `s[j:i]` is a dictionary word — build up whether each prefix of the string is segmentable. |
| [Longest Common Subsequence](https://leetcode.com/problems/longest-common-subsequence/) | Medium | — | Find the length of the longest subsequence common to two strings. | `dp[i][j] = dp[i-1][j-1] + 1` if characters match, else `max(dp[i-1][j], dp[i][j-1])` — classic 2D grid DP comparing prefixes of both strings. |
| [Edit Distance](https://leetcode.com/problems/edit-distance/) | Hard | — | Find the minimum number of insert/delete/replace operations to convert one string into another. | `dp[i][j] = dp[i-1][j-1]` if characters match, else `1 + min(insert, delete, replace)` where each corresponds to a neighboring dp cell — same grid shape as LCS. |
| [Unique Paths](https://leetcode.com/problems/unique-paths/) | Medium | — | Count the number of paths from top-left to bottom-right of a grid, moving only right or down. | `dp[i][j] = dp[i-1][j] + dp[i][j-1]` — the number of ways to reach a cell is the sum of ways to reach the cell above and the cell to the left. |
| [Partition Equal Subset Sum](https://leetcode.com/problems/partition-equal-subset-sum/) | Medium | — | Determine if an array can be split into two subsets with equal sum. | Classic subset-sum DP: target is `total / 2` (must be an integer); `dp[s] = true` if some subset sums to `s`, built up by deciding include/exclude for each number. |
| [Target Sum](https://leetcode.com/problems/target-sum/) | Medium | — | Count the number of ways to assign +/- signs to array elements to reach a target sum. | Reduces to subset-sum: if `P` is the positive-signed subset and `N` the negative-signed subset, `P - N = target` and `P + N = total`, so `P = (target + total) / 2` — count subsets summing to `P`. |
| [Best Time to Buy and Sell Stock with Cooldown](https://leetcode.com/problems/best-time-to-buy-and-sell-stock-with-cooldown/) | Medium | — | Maximize profit with unlimited transactions, but a mandatory cooldown day after selling. | State machine DP with three states per day — holding a stock, just sold (cooldown), or free to buy — transition between them based on buy/sell/rest decisions. |

## Solutions

### Best Time to Buy and Sell Stock
See [04-sliding-window.md](04-sliding-window.md) — track running min price, max profit = `price - minSoFar`.

### Trapping Rain Water
See [03-two-pointers.md](03-two-pointers.md) — two-pointer version; the DP version precomputes `leftMax`/`rightMax` arrays instead.

### Longest Palindromic Substring
See [02-string-manipulation.md](02-string-manipulation.md) — expand-from-center (simpler and equally efficient to the 2D `dp[i][j]` formulation).

### Maximum Subarray
```python
def max_sub_array(nums):
    best = curr = nums[0]
    for n in nums[1:]:
        curr = max(n, curr + n)  # extend the running subarray, or restart at n
        best = max(best, curr)
    return best
```

### Climbing Stairs
```python
def climb_stairs(n):
    if n <= 2:
        return n
    a, b = 1, 2  # ways to reach step 1, step 2
    for _ in range(3, n + 1):
        a, b = b, a + b  # dp[i] = dp[i-1] + dp[i-2]
    return b
```

### House Robber
```python
def rob(nums):
    prev, curr = 0, 0  # best up to two houses back, best up to one house back
    for n in nums:
        prev, curr = curr, max(curr, prev + n)  # skip this house, or rob it
    return curr
```

### House Robber II
```python
def rob_circular(nums):
    if len(nums) == 1:
        return nums[0]

    def rob_line(houses):
        prev, curr = 0, 0
        for n in houses:
            prev, curr = curr, max(curr, prev + n)
        return curr

    # circular constraint only matters for the first/last pair — run linear version twice
    return max(rob_line(nums[1:]), rob_line(nums[:-1]))
```

### Coin Change
```python
def coin_change(coins, amount):
    dp = [0] + [float('inf')] * amount
    for a in range(1, amount + 1):
        for c in coins:
            if c <= a:
                dp[a] = min(dp[a], dp[a - c] + 1)
    return dp[amount] if dp[amount] != float('inf') else -1
```

### Longest Increasing Subsequence
```python
import bisect

def length_of_lis(nums):
    tails = []  # tails[i] = smallest possible tail value of an increasing subsequence of length i+1
    for n in nums:
        pos = bisect.bisect_left(tails, n)
        if pos == len(tails):
            tails.append(n)   # n extends the longest subsequence so far
        else:
            tails[pos] = n    # n gives a smaller tail for a subsequence of this length
    return len(tails)
```

### Word Break
```python
def word_break(s, word_dict):
    words = set(word_dict)
    n = len(s)
    dp = [False] * (n + 1)
    dp[0] = True  # empty prefix is trivially segmentable

    for i in range(1, n + 1):
        for j in range(i):
            if dp[j] and s[j:i] in words:
                dp[i] = True
                break

    return dp[n]
```

### Longest Common Subsequence
```python
def longest_common_subsequence(text1, text2):
    m, n = len(text1), len(text2)
    dp = [[0] * (n + 1) for _ in range(m + 1)]

    for i in range(1, m + 1):
        for j in range(1, n + 1):
            if text1[i - 1] == text2[j - 1]:
                dp[i][j] = dp[i - 1][j - 1] + 1
            else:
                dp[i][j] = max(dp[i - 1][j], dp[i][j - 1])

    return dp[m][n]
```

### Edit Distance
```python
def min_distance(word1, word2):
    m, n = len(word1), len(word2)
    dp = [[0] * (n + 1) for _ in range(m + 1)]

    for i in range(m + 1):
        dp[i][0] = i  # delete all of word1's prefix
    for j in range(n + 1):
        dp[0][j] = j  # insert all of word2's prefix

    for i in range(1, m + 1):
        for j in range(1, n + 1):
            if word1[i - 1] == word2[j - 1]:
                dp[i][j] = dp[i - 1][j - 1]
            else:
                dp[i][j] = 1 + min(
                    dp[i - 1][j],      # delete from word1
                    dp[i][j - 1],      # insert into word1
                    dp[i - 1][j - 1],  # replace
                )

    return dp[m][n]
```

### Unique Paths
```python
def unique_paths(m, n):
    dp = [1] * n  # first row: only one way to reach each cell (move right repeatedly)
    for _ in range(1, m):
        for j in range(1, n):
            dp[j] += dp[j - 1]  # dp[j] currently holds the value from the row above
    return dp[-1]
```

### Partition Equal Subset Sum
```python
def can_partition(nums):
    total = sum(nums)
    if total % 2:
        return False
    target = total // 2

    dp = {0}  # set of sums reachable using some subset of numbers seen so far
    for n in nums:
        dp |= {n + s for s in dp}
        if target in dp:
            return True

    return target in dp
```

### Target Sum
```python
def find_target_sum_ways(nums, target):
    total = sum(nums)
    if (total + target) % 2 or abs(target) > total:
        return 0
    p = (total + target) // 2  # size of the positive-signed subset (derived algebraically)

    dp = [0] * (p + 1)
    dp[0] = 1
    for n in nums:
        for s in range(p, n - 1, -1):  # iterate backward: classic 0/1 knapsack, avoid reuse
            dp[s] += dp[s - n]

    return dp[p]
```

### Best Time to Buy and Sell Stock with Cooldown
```python
def max_profit_cooldown(prices):
    sold, held, rest = float('-inf'), float('-inf'), 0
    for price in prices:
        prev_sold = sold
        sold = held + price              # sell today (must have been holding)
        held = max(held, rest - price)   # keep holding, or buy today (must have been resting)
        rest = max(rest, prev_sold)      # keep resting, or just finished cooldown after a sell
    return max(sold, rest)
```
