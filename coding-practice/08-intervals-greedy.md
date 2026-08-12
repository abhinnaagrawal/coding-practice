[← back to index](README.md)

# Intervals & Greedy

## When to recognize it
Problem gives ranges/intervals (meetings, schedules) and asks about overlap, merging, or minimum resources. Greedy applies when a locally optimal choice at each step provably leads to a globally optimal solution — usually after sorting.

## Core idea
Sort intervals (by start, or sometimes by end) so overlap becomes a simple adjacency check instead of an all-pairs comparison. Greedy: make the choice that looks best right now, and trust a swap-argument proof that no other choice could do better later.

## Gotchas
- Sort first — by start, but sometimes by end (activity selection) — greedy correctness depends on which.
- Merge condition: `current.start <= previous.end` (overlap) vs `<` — off-by-one changes whether touching intervals merge.
- Greedy "prove it's optimal" — if you can't justify the swap argument, it's probably DP, not greedy.

## Problems

| Problem | Difficulty | Freq | What it asks | Intuition |
|---|---|---|---|---|
| [Merge Intervals](https://leetcode.com/problems/merge-intervals/) | Medium | 104 | Given a collection of intervals, merge all overlapping ones. | Sort by start time; walk through, and if the current interval's start ≤ the last merged interval's end, extend it — otherwise start a new merged interval. |
| [Meeting Rooms II](https://leetcode.com/problems/meeting-rooms-ii/) | Medium | 49 | Given meeting time intervals, find the minimum number of conference rooms required. | Sort start times and end times separately, sweep through with two pointers counting concurrent meetings (or use a min-heap of end times, popping when a new meeting starts after the earliest end). |
| [Insert Interval](https://leetcode.com/problems/insert-interval/) | Medium | — | Insert a new interval into a sorted list of non-overlapping intervals, merging as needed. | Walk the sorted list: copy intervals entirely before the new one, merge all intervals overlapping the new one into it, then copy the rest unchanged. |
| [Non-overlapping Intervals](https://leetcode.com/problems/non-overlapping-intervals/) | Medium | — | Find the minimum number of intervals to remove so the rest don't overlap. | Sort by end time; greedily keep an interval if it starts after the last kept interval's end — this maximizes how many intervals you can keep, minimizing removals. |
| [Jump Game](https://leetcode.com/problems/jump-game/) | Medium | 40 | Given max-jump-length at each index, determine if you can reach the last index. | Track the farthest index reachable so far while scanning left to right; if the current index ever exceeds that farthest-reachable value, it's unreachable — fail immediately. |
| [Gas Station](https://leetcode.com/problems/gas-station/) | Medium | — | Given gas/cost at each station in a circuit, find the starting station index that lets you complete the full circuit. | Track a running tank total; whenever it goes negative, the current start can't work, so reset the candidate start to the next station and reset the tank — the final surviving start is the answer. |
| [Container With Most Water](https://leetcode.com/problems/container-with-most-water/) | Medium | 86 | Pick two lines that with the x-axis form the container holding the most water (greedy pointer movement). | Start pointers at both ends; always move the pointer at the shorter line inward, since it's the bottleneck and moving the taller one can't help. |

## Solutions

### Merge Intervals
```python
def merge_intervals(intervals):
    intervals.sort(key=lambda x: x[0])
    merged = [intervals[0]]
    for start, end in intervals[1:]:
        if start <= merged[-1][1]:  # overlaps with (or touches) the last merged interval
            merged[-1][1] = max(merged[-1][1], end)
        else:
            merged.append([start, end])
    return merged
```

### Meeting Rooms II
```python
import heapq

def min_meeting_rooms(intervals):
    if not intervals:
        return 0
    intervals.sort(key=lambda x: x[0])
    heap = []  # end times of rooms currently in use
    for start, end in intervals:
        if heap and heap[0] <= start:  # earliest-ending room is free by now
            heapq.heappop(heap)
        heapq.heappush(heap, end)
    return len(heap)  # number of rooms simultaneously in use at the peak
```

### Insert Interval
```python
def insert_interval(intervals, new_interval):
    res = []
    i, n = 0, len(intervals)

    while i < n and intervals[i][1] < new_interval[0]:
        res.append(intervals[i])  # entirely before the new interval
        i += 1

    while i < n and intervals[i][0] <= new_interval[1]:
        new_interval[0] = min(new_interval[0], intervals[i][0])  # absorb overlapping interval
        new_interval[1] = max(new_interval[1], intervals[i][1])
        i += 1
    res.append(new_interval)

    while i < n:
        res.append(intervals[i])  # entirely after the new interval
        i += 1

    return res
```

### Non-overlapping Intervals
```python
def erase_overlap_intervals(intervals):
    intervals.sort(key=lambda x: x[1])  # sort by end time
    count = 0
    prev_end = float('-inf')
    for start, end in intervals:
        if start >= prev_end:
            prev_end = end  # keep this interval, it doesn't overlap
        else:
            count += 1  # must remove this one to resolve the overlap
    return count
```

### Jump Game
```python
def can_jump(nums):
    farthest = 0
    for i, n in enumerate(nums):
        if i > farthest:  # this index is unreachable from anywhere before it
            return False
        farthest = max(farthest, i + n)
    return True
```

### Gas Station
```python
def can_complete_circuit(gas, cost):
    total_tank = 0
    curr_tank = 0
    start = 0
    for i in range(len(gas)):
        diff = gas[i] - cost[i]
        total_tank += diff
        curr_tank += diff
        if curr_tank < 0:  # can't reach the next station from the current start
            start = i + 1  # any station between old start and i also fails, skip them all
            curr_tank = 0
    return start if total_tank >= 0 else -1
```

### Container With Most Water
```python
def max_area(height):
    l, r = 0, len(height) - 1
    best = 0
    while l < r:
        best = max(best, (r - l) * min(height[l], height[r]))
        if height[l] < height[r]:
            l += 1
        else:
            r -= 1
    return best
```
