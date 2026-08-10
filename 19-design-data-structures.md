[← back to index](README.md)

# Design a Data Structure

## When to recognize it
"Design a class that supports operations X, Y, Z with these complexity requirements" — heavy at senior level, since it tests composing multiple structures under a constraint, not one algorithm.

## Core idea
Pick the data structure(s) whose native operation costs match the stated requirement. O(1) get/put with eviction → hashmap + doubly linked list (LRU). O(1) insert/remove/getRandom → array + hashmap of value→index, swap-with-last for O(1) removal. The design is usually a composition of 2 simple structures, not one exotic one.

## Gotchas
- Clarify operation complexity requirements *before* coding — it determines the structure choice, not the other way around.
- State invariants out loud — the interviewer is grading whether your structure choice matches the constraint, not just correctness.
- Don't over-engineer: if O(n) is acceptable for a rare operation, don't add complexity to make it O(1) too.

## Problems

| Problem | Difficulty | Freq | What it asks | Intuition |
|---|---|---|---|---|
| [LRU Cache](https://leetcode.com/problems/lru-cache/) | Medium | — | Design a cache with O(1) get/put that evicts the least-recently-used item when full. | Hashmap of key→node for O(1) lookup, combined with a doubly linked list ordered by recency — move a node to the front on every access, evict from the tail when over capacity. |
| [LFU Cache](https://leetcode.com/problems/lfu-cache/) | Hard | — | Same as LRU, but evicts the least-frequently-used item (tie-broken by least-recently-used). | Hashmap of key→node, plus a hashmap of frequency→doubly linked list of nodes at that frequency (ordered by recency within the frequency); track the current minimum frequency to know which list to evict from. |
| [Design Twitter](https://leetcode.com/problems/design-twitter/) | Medium | — | Design a simplified Twitter: post tweet, follow/unfollow, get the 10 most recent tweets in a user's news feed. | Hashmap of user→list of `(timestamp, tweetId)`; building a feed is a k-way merge (min-heap) of the current user's own tweets plus all followees' tweets, taking the 10 most recent. |
| [Insert Delete GetRandom O(1)](https://leetcode.com/problems/insert-delete-getrandom-o1/) | Medium | — | Design a structure supporting insert, remove, and getRandom, all in average O(1). | Array for O(1) random access, plus a hashmap of value→index; to delete in O(1), swap the target with the last element in the array (updating the hashmap), then pop the last element. |
| [Min Stack](https://leetcode.com/problems/min-stack/) | Medium | — | Design a stack supporting push/pop/top/getMin, all in O(1). | Store each element alongside the minimum-so-far at the time it was pushed (or maintain a parallel stack of running minimums) — the current min is always readable in O(1). |
| [Design Browser History](https://leetcode.com/problems/design-browser-history/) | Medium | — | Design a browser history supporting visit, back, and forward. | A list with a current-position pointer works well: `visit` truncates everything after the current position and appends; `back`/`forward` just move the pointer within bounds. |
| [Time Based Key-Value Store](https://leetcode.com/problems/time-based-key-value-store/) | Medium | — | Design a structure supporting `set(key, value, timestamp)` and `get(key, timestamp)` returning the value at or before that time. | Hashmap of key→list of `(timestamp, value)` appended in increasing timestamp order; `get` binary searches that list for the largest timestamp ≤ the query. |
| [Find Median from Data Stream](https://leetcode.com/problems/find-median-from-data-stream/) | Hard | — | Design a structure supporting adding numbers and finding the running median efficiently. | Two heaps — max-heap for the lower half, min-heap for the upper half — kept balanced in size within 1 after every insert; the median comes directly from the top(s). |
| [Serialize and Deserialize Binary Tree](https://leetcode.com/problems/serialize-and-deserialize-binary-tree/) | Hard | — | Design an algorithm to encode a tree to a string and decode it back to the original tree. | Preorder DFS emitting a sentinel for null children during serialization; deserialization consumes tokens in the same preorder sequence, recursively rebuilding left then right. |

## Solutions

### LRU Cache
See [06-linked-lists.md](06-linked-lists.md) — `OrderedDict` (or hashmap + doubly linked list) with move-to-end/evict-from-front.

### LFU Cache
```python
from collections import defaultdict, OrderedDict

class LFUCache:
    def __init__(self, capacity):
        self.capacity = capacity
        self.min_freq = 0
        self.key_to_val = {}
        self.key_to_freq = {}
        self.freq_to_keys = defaultdict(OrderedDict)  # freq -> {key: None}, ordered by recency

    def _bump_freq(self, key):
        freq = self.key_to_freq[key]
        del self.freq_to_keys[freq][key]
        if not self.freq_to_keys[freq] and freq == self.min_freq:
            self.min_freq += 1  # that frequency bucket is now empty
        self.key_to_freq[key] = freq + 1
        self.freq_to_keys[freq + 1][key] = None

    def get(self, key):
        if key not in self.key_to_val:
            return -1
        self._bump_freq(key)
        return self.key_to_val[key]

    def put(self, key, value):
        if self.capacity == 0:
            return
        if key in self.key_to_val:
            self.key_to_val[key] = value
            self._bump_freq(key)
            return
        if len(self.key_to_val) >= self.capacity:
            # evict the least-recently-used key within the minimum frequency bucket
            evict_key, _ = self.freq_to_keys[self.min_freq].popitem(last=False)
            del self.key_to_val[evict_key]
            del self.key_to_freq[evict_key]
        self.key_to_val[key] = value
        self.key_to_freq[key] = 1
        self.freq_to_keys[1][key] = None
        self.min_freq = 1
```

### Design Twitter
```python
from collections import defaultdict
import heapq

class Twitter:
    def __init__(self):
        self.time = 0
        self.tweets = defaultdict(list)  # user -> list of (time, tweetId)
        self.following = defaultdict(set)

    def postTweet(self, userId, tweetId):
        self.tweets[userId].append((self.time, tweetId))
        self.time -= 1  # decreasing "time" so a plain min-heap pops the most recent first

    def getNewsFeed(self, userId):
        heap = []
        users = self.following[userId] | {userId}
        for u in users:
            if self.tweets[u]:
                idx = len(self.tweets[u]) - 1
                t, tid = self.tweets[u][idx]
                heap.append((t, tid, u, idx - 1))
        heapq.heapify(heap)  # k-way merge of each followed user's tweet list

        res = []
        while heap and len(res) < 10:
            t, tid, u, idx = heapq.heappop(heap)
            res.append(tid)
            if idx >= 0:
                nt, ntid = self.tweets[u][idx]
                heapq.heappush(heap, (nt, ntid, u, idx - 1))
        return res

    def follow(self, followerId, followeeId):
        if followerId != followeeId:
            self.following[followerId].add(followeeId)

    def unfollow(self, followerId, followeeId):
        self.following[followerId].discard(followeeId)
```

### Insert Delete GetRandom O(1)
```python
import random

class RandomizedSet:
    def __init__(self):
        self.arr = []
        self.idx = {}  # value -> index in arr

    def insert(self, val):
        if val in self.idx:
            return False
        self.idx[val] = len(self.arr)
        self.arr.append(val)
        return True

    def remove(self, val):
        if val not in self.idx:
            return False
        i = self.idx[val]
        last = self.arr[-1]
        self.arr[i] = last   # move the last element into the removed slot for O(1) delete
        self.idx[last] = i
        self.arr.pop()
        del self.idx[val]
        return True

    def getRandom(self):
        return random.choice(self.arr)
```

### Min Stack
See [07-stacks-queues.md](07-stacks-queues.md) — store `(value, min_so_far)` pairs.

### Design Browser History
```python
class BrowserHistory:
    def __init__(self, homepage):
        self.history = [homepage]
        self.curr = 0

    def visit(self, url):
        self.history = self.history[:self.curr + 1]  # drop any forward history
        self.history.append(url)
        self.curr += 1

    def back(self, steps):
        self.curr = max(0, self.curr - steps)
        return self.history[self.curr]

    def forward(self, steps):
        self.curr = min(len(self.history) - 1, self.curr + steps)
        return self.history[self.curr]
```

### Time Based Key-Value Store
See [05-binary-search.md](05-binary-search.md) — list of `(timestamp, value)` per key, binary searched.

### Find Median from Data Stream
See [13-heaps-priority-queues.md](13-heaps-priority-queues.md) — two heaps kept balanced in size.

### Serialize and Deserialize Binary Tree
See [09-trees-bst.md](09-trees-bst.md) — preorder DFS with a null sentinel.
