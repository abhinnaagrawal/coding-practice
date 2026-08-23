[← back to index](/python-tutorial/README.md)

# Exceptions, Modules & the Interview Stdlib

## Exceptions — Java's model, minus checked exceptions

```python
try:
    value = int(user_input)
except ValueError as e:                    # catch specific type; `as e` binds the exception
    print(f"bad input: {e}")
except (KeyError, IndexError):             # catch several
    print("lookup failed")
except Exception:                          # catch-all — avoid bare `except:` (catches Ctrl-C too)
    print("unexpected")
else:
    print("runs only if NO exception")     # no Java equivalent
finally:
    print("always runs")                   # cleanup — same as Java
```

Key differences from Java:
- **No checked exceptions.** Nothing forces you to declare or catch anything (`throws` doesn't exist).
- Raising: `raise ValueError("msg")` (≈ `throw new`).
- Custom exceptions are one line: `class CircuitOpenError(Exception): pass`
- Tracebacks read bottom-up — the last line names the exception, work upward to your code.

## EAFP vs LBYL — a genuine style fork

Java/C culture: **LBYL** — "look before you leap" (`if (map.containsKey(k)) ...`).  
Python culture: **EAFP** — "easier to ask forgiveness than permission":

```python
try:
    return d[key]
except KeyError:
    return default
```

Both are acceptable in interviews; know the names — "I'd use EAFP here" is a signal fluency point. Use EAFP when the failure is rare, LBYL when the check is cheap and failure is common. (And in this specific case, `d.get(key, default)` beats both.)

## Modules & imports

Every `.py` file is a module; a directory with an `__init__.py` is a package.

```python
import heapq                 # use as heapq.heappush(...) — namespace preserved, preferred
from heapq import heappush   # use as heappush(...) — fine for heavy-use names
import collections as c      # alias
from math import *           # NEVER — pollutes namespace, hides shadowing
```

- Imports execute the module's top-level code once, then cache it (`sys.modules`).
- Circular imports (A imports B, B imports A) fail — restructure or import inside the function.
- Interviewers won't test package mechanics, but clean imports (`import heapq` at top, not inside functions) are part of "writes real Python."

## The interview standard library — learn these cold

### collections

```python
from collections import Counter, defaultdict, deque

Counter("aabbc")                        # Counter({'a': 2, 'b': 2, 'c': 1}) — one-line frequency map
Counter("aabbc").most_common(1)         # [('a', 2)]

d = defaultdict(int)                    # missing key → 0 — kills the .get(k,0) dance
d["x"] += 1                             # just works
g = defaultdict(list)                   # missing key → [] — for grouping
g[key].append(val)

dq = deque([1, 2, 3])
dq.append(4); dq.appendleft(0)          # O(1) both ends — THE queue for BFS
dq.popleft()                            # O(1) — list.pop(0) is O(n), this is why deque exists
```

**BFS template** — the deque's canonical interview use:

```python
from collections import deque

def bfs(start):
    seen = {start}
    q = deque([start])
    while q:
        node = q.popleft()
        for nxt in neighbors(node):
            if nxt not in seen:
                seen.add(nxt)
                q.append(nxt)
```

### heapq — the priority queue (min-heap only!)

```python
import heapq

heapq.heapify(nums)           # O(n) in-place — nums[0] is now the min
heapq.heappush(heap, x)       # O(log n)
heapq.heappop(heap)           # O(log n) — pops the MINIMUM
heapq.nlargest(k, nums)       # top-k without full sort
heapq.nsmallest(k, nums)
```

- **No max-heap.** The idiom is pushing negated values: `heappush(h, -x)`. (Or store tuples with negated priority.)
- Tuples compare element-wise: `heappush(h, (dist, node))` — priority first, payload second. If priorities tie, the payloads must be comparable — add a tiebreaker counter if not: `(dist, counter(), node)`.
- `heap[0]` peeks the min in O(1) without popping.

### bisect — binary search on sorted lists

```python
from bisect import bisect_left, bisect_right, insort

bisect_left(sorted_list, x)   # leftmost index where x could be inserted
bisect_right(sorted_list, x)  # rightmost — lo == hi means x absent
insort(sorted_list, x)        # insert keeping sorted order
```

`bisect_left(a, x) != len(a) and a[bisect_left(a, x)] == x` is your "is x in sorted list" check. When an interview says "the array is sorted," reach for `bisect` or write the binary search by hand (they often want the hand-written one — see the interview-prep folder).

### math, functools, sys

```python
import math
math.inf          # float infinity — initialize "best so far" trackers (NOT sys.maxsize games)
math.gcd(a, b)    # since 3.5 — no need to hand-roll
math.sqrt, math.ceil, math.floor, math.comb

from functools import lru_cache

@lru_cache(maxsize=None)     # memoization decorator — turns naive recursion into DP, one line
def fib(n):
    return n if n < 2 else fib(n-1) + fib(n-2)
```

`@lru_cache` on a recursive function is the fastest legit way to add memoization in an interview. (Modern alternative: `@functools.cache`.)

## Gotchas

- Bare `except:` swallows `KeyboardInterrupt` and `SystemExit` — always catch at least `Exception`, preferably a specific type.
- Catching too broadly hides bugs: `except Exception: pass` is how silent data corruption begins.
- Don't name your file `heapq.py`, `math.py`, etc. — it shadows the stdlib module and your own imports break with baffling errors.
- `heapq` is min-heap only — forgetting to negate is a classic silent bug (results look "roughly right").
- `@lru_cache` arguments must be hashable; decorating a method caches per `self` too (memory leak consideration in long-running code, rarely interview-relevant).
- `defaultdict` *creates* keys on read: `if k in d` before access if you don't want the key materialized.
- Integer division for "infinity": use `float('inf')` / `math.inf`; `-math.inf` for the opposite. Comparisons work as expected (`x < math.inf` always true for finite x).

## Exercises

1. Implement "top K frequent elements" using `Counter` and `heapq` in two lines.
2. Write a function `merge_k_sorted(lists)` that merges k sorted lists using a heap with a tiebreaker counter. Why is the counter needed?
3. Rewrite naive recursive `climb_stairs(n)` (f(n) = f(n-1) + f(n-2)) with `@lru_cache` and state its time complexity before and after.

<details>
<summary><b>Solutions</b></summary>

**1.**

```python
def top_k(nums, k):
    return [x for x, _ in Counter(nums).most_common(k)]
    # heapq version: heapq.nlargest(k, Counter(nums), key=Counter(nums).get)
```

**2.**

```python
import heapq
from itertools import count

def merge_k_sorted(lists):
    heap, result, tie = [], [], count()
    for lst in lists:
        for x in lst:                     # (value, tiebreak) — no third element needed here
            heapq.heappush(heap, (x, next(tie)))
    while heap:
        result.append(heapq.heappop(heap)[0])
    return result
```

The counter prevents Python from comparing payloads when priorities tie. Here values are ints so it's defensive, but if payload were a list/node object, ties without a tiebreaker raise `TypeError`. (For real k-way merge, push per-list iterators: `(val, tie, iterator)` and push the next element on each pop.)

**3.**

```python
from functools import lru_cache

@lru_cache(maxsize=None)
def climb_stairs(n):
    return n if n <= 2 else climb_stairs(n - 1) + climb_stairs(n - 2)
```

Before: O(2ⁿ) time. After: O(n) time, O(n) space — each distinct n computed once.

</details>
