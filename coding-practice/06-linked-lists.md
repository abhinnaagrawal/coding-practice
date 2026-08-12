[← back to index](README.md)

# Linked Lists

## When to recognize it
Problem explicitly gives a linked list, or you need O(1) insert/delete at arbitrary positions without shifting elements (unlike arrays).

## Core idea
Manipulate `next` pointers directly. Two recurring sub-patterns: fast/slow pointers (cycle detection, finding the middle) and dummy/sentinel nodes (removing special-casing of the head during insert/delete).

## Gotchas
- Always sanity check `head is None` and single-node list.
- Dummy/sentinel node removes special-casing the head during insert/delete.
- Fast/slow pointer cycle detection: after finding the cycle, a second pointer from head at the same speed finds the cycle *start* — not the same as merely detecting a cycle exists.
- Reversal: track `prev`, `curr`, `next_temp` — losing `next` before rewiring is the #1 bug.

## Problems

| Problem | Difficulty | Freq | What it asks | Intuition |
|---|---|---|---|---|
| [Add Two Numbers](https://leetcode.com/problems/add-two-numbers/) | Medium | 117 | Two numbers represented as linked lists (digits reversed) — return their sum as a linked list. | Simulate elementary-school addition: walk both lists together, add digits + carry, build a new node per resulting digit, carry propagates to the next pair. |
| [Reverse Linked List](https://leetcode.com/problems/reverse-linked-list/) | Easy | 39 | Reverse a singly linked list. | Iterate with `prev` and `curr`; at each node, save `curr.next`, point `curr.next` back to `prev`, then advance both pointers. |
| [Merge Two Sorted Lists](https://leetcode.com/problems/merge-two-sorted-lists/) | Easy | 47 | Merge two sorted linked lists into one sorted list. | Dummy head node; at each step, attach whichever of the two current nodes has the smaller value, advance that list's pointer. |
| [Merge k Sorted Lists](https://leetcode.com/problems/merge-k-sorted-lists/) | Hard | 40 | Merge k sorted linked lists into one sorted list. | Min-heap holding the current head of each list; repeatedly pop the smallest, append it to the result, push its `next` node back into the heap. |
| [Linked List Cycle](https://leetcode.com/problems/linked-list-cycle/) | Easy | — | Determine if a linked list has a cycle. | Fast pointer moves 2 steps, slow moves 1 step per iteration — if there's a cycle, fast eventually laps slow and they meet; if not, fast hits `None`. |
| [Reorder List](https://leetcode.com/problems/reorder-list/) | Medium | — | Reorder list L0→L1→…→Ln into L0→Ln→L1→Ln-1→… | Find the middle (fast/slow pointers), reverse the second half, then merge the two halves by alternating nodes from each. |
| [Remove Nth Node From End of List](https://leetcode.com/problems/remove-nth-node-from-end-of-list/) | Medium | — | Remove the nth node from the end of the list in one pass. | Dummy head + two pointers: advance the fast pointer `n` steps first, then move both pointers together until fast hits the end — slow now sits just before the node to remove. |
| [Palindrome Linked List](https://leetcode.com/problems/palindrome-linked-list/) | Easy | — | Determine if a linked list reads the same forwards and backwards. | Find the middle, reverse the second half in place, then walk both halves comparing values node by node. |
| [LRU Cache](https://leetcode.com/problems/lru-cache/) | Medium | — | Design a cache with O(1) get/put that evicts the least-recently-used item when full (hashmap + doubly linked list). | Hashmap gives O(1) key lookup to a node; a doubly linked list keeps recency order — move a node to the front on every access, evict from the tail when over capacity. |

## Solutions

```python
class ListNode:
    def __init__(self, val=0, next=None):
        self.val = val
        self.next = next
```

### Add Two Numbers
```python
def add_two_numbers(l1, l2):
    dummy = ListNode()
    curr = dummy
    carry = 0
    while l1 or l2 or carry:
        v1 = l1.val if l1 else 0
        v2 = l2.val if l2 else 0
        total = v1 + v2 + carry
        carry, digit = divmod(total, 10)
        curr.next = ListNode(digit)
        curr = curr.next
        l1 = l1.next if l1 else None
        l2 = l2.next if l2 else None
    return dummy.next
```

### Reverse Linked List
```python
def reverse_list(head):
    prev = None
    curr = head
    while curr:
        next_temp = curr.next  # save before we overwrite curr.next
        curr.next = prev
        prev = curr
        curr = next_temp
    return prev
```

### Merge Two Sorted Lists
```python
def merge_two_lists(l1, l2):
    dummy = ListNode()
    curr = dummy
    while l1 and l2:
        if l1.val <= l2.val:
            curr.next, l1 = l1, l1.next
        else:
            curr.next, l2 = l2, l2.next
        curr = curr.next
    curr.next = l1 or l2  # attach whichever list still has leftover nodes
    return dummy.next
```

### Merge k Sorted Lists
```python
import heapq

def merge_k_lists(lists):
    heap = []
    for i, node in enumerate(lists):
        if node:
            heapq.heappush(heap, (node.val, i, node))  # i breaks ties, avoids comparing nodes directly

    dummy = ListNode()
    curr = dummy
    while heap:
        val, i, node = heapq.heappop(heap)
        curr.next = node
        curr = curr.next
        if node.next:
            heapq.heappush(heap, (node.next.val, i, node.next))
    return dummy.next
```

### Linked List Cycle
```python
def has_cycle(head):
    slow = fast = head
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next
        if slow is fast:  # fast lapped slow -> cycle exists
            return True
    return False
```

### Reorder List
```python
def reorder_list(head):
    if not head:
        return

    # 1. find the middle with fast/slow pointers
    slow, fast = head, head
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next

    # 2. reverse the second half
    second = slow.next
    slow.next = None
    prev = None
    while second:
        nxt = second.next
        second.next = prev
        prev = second
        second = nxt
    second = prev  # head of the reversed second half

    # 3. merge the two halves, alternating nodes
    first = head
    while second:
        tmp1, tmp2 = first.next, second.next
        first.next = second
        second.next = tmp1
        first, second = tmp1, tmp2
```

### Remove Nth Node From End of List
```python
def remove_nth_from_end(head, n):
    dummy = ListNode(0, head)
    slow = fast = dummy
    for _ in range(n):
        fast = fast.next  # advance fast n steps ahead
    while fast.next:
        slow = slow.next
        fast = fast.next
    slow.next = slow.next.next  # slow is now just before the node to remove
    return dummy.next
```

### Palindrome Linked List
```python
def is_palindrome_list(head):
    vals = []
    while head:
        vals.append(head.val)
        head = head.next
    return vals == vals[::-1]  # O(n) space; O(1) space version reverses the second half in place
```

### LRU Cache
```python
from collections import OrderedDict

class LRUCache:
    def __init__(self, capacity):
        self.capacity = capacity
        self.cache = OrderedDict()  # preserves insertion/access order

    def get(self, key):
        if key not in self.cache:
            return -1
        self.cache.move_to_end(key)  # mark as most recently used
        return self.cache[key]

    def put(self, key, value):
        if key in self.cache:
            self.cache.move_to_end(key)
        self.cache[key] = value
        if len(self.cache) > self.capacity:
            self.cache.popitem(last=False)  # evict the least recently used (front of order)
```
