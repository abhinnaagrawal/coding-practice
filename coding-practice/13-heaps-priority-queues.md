[← back to index](/coding-practice/README.md)

# Heaps & Priority Queues

## When to recognize it
"Top K," "kth largest/smallest," "median of a stream," "merge k sorted things," or any scheduling problem where you always need the current min/max, not a full sort.

## Core idea
A heap keeps the min (or max) accessible in O(1), with O(log n) insert/remove — cheaper than a full sort (O(n log n)) when you only ever need the extreme element repeatedly, or a bounded top-K. Two-heap pattern (one min-heap, one max-heap) tracks a running median by keeping both halves balanced.

## Gotchas
- Python `heapq` is min-heap only — negate values for max-heap behavior.
- Top-K: maintain a heap of size K, pop when exceeding it — don't push everything then sort (defeats the point).
- Two-heap pattern (median finder): keep sizes balanced within 1 after every insert.

## Problems

| Problem | Difficulty | Freq | What it asks | Intuition |
|---|---|---|---|---|
| [Top K Frequent Elements](https://leetcode.com/problems/top-k-frequent-elements/) | Medium | 62 | Return the k most frequently occurring elements. | Count frequencies, then maintain a min-heap of size k (pop when it exceeds k) — or bucket-sort by frequency for O(n). |
| [Meeting Rooms II](https://leetcode.com/problems/meeting-rooms-ii/) | Medium | 49 | Find the minimum number of conference rooms needed for a set of meeting intervals. | Sort meetings by start time; use a min-heap of currently occupied rooms' end times — if the earliest-ending room frees up before the next meeting starts, reuse it, otherwise add a new room. |
| [Sliding Window Maximum](https://leetcode.com/problems/sliding-window-maximum/) | Hard | 41 | Return the max element for each sliding window of size k. | A monotonic deque of indices (decreasing values) is the standard approach; a heap with lazy deletion of out-of-window entries also works but is less clean. |
| [Merge k Sorted Lists](https://leetcode.com/problems/merge-k-sorted-lists/) | Hard | 40 | Merge k sorted linked lists into one sorted list (heap of k current-head pointers). | Min-heap holding the current head node of each list; repeatedly pop the smallest, append it, push its `next` back into the heap. |
| [Kth Largest Element in an Array](https://leetcode.com/problems/kth-largest-element-in-an-array/) | Medium | 39 | Find the kth largest element in an unsorted array. | Min-heap of size k (the top is always the kth largest so far), or quickselect partitioning for average O(n) without a full sort. |
| [Find Median from Data Stream](https://leetcode.com/problems/find-median-from-data-stream/) | Hard | — | Design a structure that supports adding numbers and finding the running median efficiently (two heaps). | Max-heap for the lower half of numbers, min-heap for the upper half; keep their sizes balanced within 1 after every insert — median is derived from the top(s) of the heaps. |
| [Task Scheduler](https://leetcode.com/problems/task-scheduler/) | Medium | — | Given tasks and a cooldown period between identical tasks, find the minimum time to execute all of them. | Max-heap by remaining frequency; greedily schedule the most frequent available task each cycle, using a cooldown queue to track when a task becomes eligible again. |
| [K Closest Points to Origin](https://leetcode.com/problems/k-closest-points-to-origin/) | Medium | — | Find the k points closest to the origin. | Max-heap of size k by distance (evict the farthest when exceeding k), or quickselect on distance for average O(n). |

## Solutions

### Top K Frequent Elements
See [01-arrays-hashing.md](/coding-practice/01-arrays-hashing.md) — `heapq.nlargest` over a frequency Counter.

### Meeting Rooms II
See [08-intervals-greedy.md](/coding-practice/08-intervals-greedy.md) — min-heap of end times.

### Sliding Window Maximum
See [04-sliding-window.md](/coding-practice/04-sliding-window.md) — monotonic deque (the standard approach; a heap with lazy deletion also works but is less clean).

### Merge k Sorted Lists
See [06-linked-lists.md](/coding-practice/06-linked-lists.md) — min-heap of each list's current head.

### Kth Largest Element in an Array
```python
import heapq

def find_kth_largest(nums, k):
    heap = nums[:k]
    heapq.heapify(heap)  # min-heap of the k largest values seen so far
    for n in nums[k:]:
        if n > heap[0]:
            heapq.heapreplace(heap, n)  # pop the smallest, push the new larger value
    return heap[0]  # the smallest of the k largest = the kth largest overall
```

### Find Median from Data Stream
```python
import heapq

class MedianFinder:
    def __init__(self):
        self.small = []  # max-heap (store negated values), holds the lower half
        self.large = []  # min-heap, holds the upper half

    def addNum(self, num):
        heapq.heappush(self.small, -num)
        heapq.heappush(self.large, -heapq.heappop(self.small))  # keep small's max <= large's min
        if len(self.large) > len(self.small):
            heapq.heappush(self.small, -heapq.heappop(self.large))  # rebalance sizes

    def findMedian(self):
        if len(self.small) > len(self.large):
            return -self.small[0]
        return (-self.small[0] + self.large[0]) / 2
```

### Task Scheduler
```python
from collections import Counter, deque
import heapq

def least_interval(tasks, n):
    counts = Counter(tasks)
    heap = [-c for c in counts.values()]
    heapq.heapify(heap)  # max-heap of remaining frequencies (negated)

    time = 0
    cooldown_queue = deque()  # (time_when_available_again, remaining_count)
    while heap or cooldown_queue:
        time += 1
        if heap:
            count = 1 + heapq.heappop(heap)  # schedule the most frequent available task
            if count:
                cooldown_queue.append((time + n, count))
        if cooldown_queue and cooldown_queue[0][0] == time:
            heapq.heappush(heap, cooldown_queue.popleft()[1])

    return time
```

### K Closest Points to Origin
```python
import heapq

def k_closest(points, k):
    heap = []
    for x, y in points:
        dist = -(x * x + y * y)  # negate so heapq (min-heap) behaves like a max-heap
        heapq.heappush(heap, (dist, x, y))
        if len(heap) > k:
            heapq.heappop(heap)  # evict the current farthest point
    return [[x, y] for _, x, y in heap]
```
