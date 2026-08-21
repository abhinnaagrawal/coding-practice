[← back to index](/python-tutorial/README.md)

# Idioms, Gotchas & the Interview Playbook

The capstone chapter: the gotchas that fail interviews, and how to actually perform in a live Python coding round.

## Copy semantics — shallow vs deep

```python
import copy

a = [[1, 2], [3, 4]]
b = a                 # ALIAS — same object
c = a[:]              # SHALLOW copy — new outer list, shared inner lists
d = copy.copy(a)      # same as above, generic
e = copy.deepcopy(a)  # DEEP copy — fully independent

c[0][0] = 99
print(a[0][0])        # 99 — shallow copy shares the inner lists!
print(e[0][0])        # 1 — deepcopy is independent
```

The rule: `=`, slicing, `.copy()`, `list(x)`, `dict(x)` all share nested structure. Only `deepcopy` is fully recursive. In Java terms: everything is a reference type, and `.clone()` equivalents are shallow by default.

**Sorting and copies in interviews**: `grid2 = [row[:] for row in grid]` for a 2-D copy; or `copy.deepcopy(grid)` — say which you're doing and why.

## The top-20 gotcha checklist

Scan your code against this before saying "done" in an interview:

| # | Gotcha | Wrong | Right |
|---|--------|-------|-------|
| 1 | Mutable default arg | `def f(x, acc=[])` | `def f(x, acc=None)` |
| 2 | `is` for value comparison | `if name is "bob"` | `if name == "bob"` |
| 3 | 2-D matrix aliasing | `[[0]*n]*m` | `[[0]*n for _ in range(m)]` |
| 4 | `list.pop(0)` in a loop | O(n) per pop | `deque.popleft()` O(1) |
| 5 | `in` on list in a loop | O(n²) total | convert to `set` first |
| 6 | String `+=` in a loop | O(n²) | accumulate in list, `"".join()` |
| 7 | Modify list while iterating | skipped elements | iterate `items[:]` or build new |
| 8 | Late-binding closure | `lambda: i` in loop | `lambda i=i: i` |
| 9 | `__eq__` without `__hash__` | unhashable object | define both |
| 10 | Mutable class attribute | `tricks = []` at class level | initialize in `__init__` |
| 11 | Integer division sign | `-7 // 2` → `-4` | `int(-7/2)` for C semantics |
| 12 | `float("inf")` vs maxsize hacks | `sys.maxsize` breaks on negatives | `math.inf` / `-math.inf` |
| 13 | heapq is min-heap only | `heappush(h, x)` for max-heap | `heappush(h, -x)` |
| 14 | Truthiness of 0/empty | `if x:` rejects legit 0 | `if x is not None:` |
| 15 | Dict key must be hashable | `d[[1,2]] = v` TypeError | `d[(1,2)] = v` |
| 16 | Generator single-use | iterate twice → empty | re-create or `list()` it once |
| 17 | Recursion limit | deep DFS blows stack ~1000 | `sys.setrecursionlimit` or iterative |
| 18 | Shadowing builtins | `list = [1,2]` then `list(x)` fails | don't reuse builtin names |
| 19 | `sorted` vs `.sort()` | `x = lst.sort()` → `None` | `.sort()` in place / `sorted()` returns |
| 20 | Comparing floats | `0.1 + 0.2 == 0.3` → False | `math.isclose(a, b)` |

## Pythonic idioms worth adopting (interviewers grade on fluency)

```python
a, b = b, a                          # swap, no temp
x, y, *rest = items                  # starred unpacking: first two + the rest
first, *_, last = items              # ignore the middle
for i, x in enumerate(items)         # never range(len(...))
any(x > 0 for x in nums)             # "exists" — short-circuits
all(x > 0 for x in nums)             # "for all"
min_val = min(nums, default=0)       # safe on possibly-empty iterables
val = d.get(k, default)              # no containsKey dance
result = a if cond else b            # ternary
chunks = [s[i:i+k] for i in range(0, len(s), k)]   # windowing idiom
flat = [x for row in grid for x in row]            # flatten 2-D
```

Read `any`/`all` as ∃ and ∀ — they make validation code one line and self-documenting.

## The live-coding interview playbook

**Before coding (2 minutes):**
1. Clarify input constraints: size? sorted? negative numbers? empty/None allowed?
2. State the brute force and its complexity, then the target: "O(n) time with a hash map."
3. Confirm language-specific assumptions out loud: "dicts are insertion-ordered, and `in` on a dict is O(1)."

**While coding:**
- Name variables like you mean it: `left`, `window_start`, not `i1`, `tmp`. (Interview variable naming is prose, not golf.)
- Use the stdlib loudly: "`deque` gives me O(1) pops from the front — a list would be O(n) here."
- Narrate complexity at each decision; the gotcha table above is your pre-flight checklist.
- Prefer a clear `defaultdict(int)` over clever one-liners under time pressure.

**Common skeletons to have muscle memory for:**

```python
# Two-pointer
l, r = 0, len(arr) - 1
while l < r:
    ...

# Sliding window
left = 0
for right in range(len(s)):
    # expand: add s[right]
    while window_invalid():
        # shrink: remove s[left]
        left += 1

# DFS on grid
def dfs(r, c):
    if not (0 <= r < m and 0 <= c < n) or (r, c) in seen:
        return
    seen.add((r, c))
    for dr, dc in ((0,1),(0,-1),(1,0),(-1,0)):
        dfs(r + dr, c + dc)
```

**Testing:**
- Trace one concrete example by hand before declaring done — out loud.
- Check edge cases in this order: empty input, single element, all duplicates, already sorted / reverse sorted.
- Python-specific re-check: mutating while iterating? `sort()` return value? integer division signs?

## Python vs Java in interview velocity

Same solution, roughly:

| Aspect | Java | Python |
|--------|------|--------|
| Lines of code (typical medium) | 40–60 | 15–25 |
| Boilerplate | types, classes, braces, getters | almost none |
| Data structure setup | `Map<Integer, List<String>> m = new HashMap<>();` | `defaultdict(list)` |
| Risk | compiles but verbose | fast but no compile check |

The trade: Python buys you 2–3× coding speed at the cost of the compiler. The gotcha checklist is how you get that safety back.

## Exercises

1. Spot the three bugs:
```python
def dedupe_adjacent(items, result=[]):
    for x in items:
        if x not in result:
            result.append(x)
    return result
```
2. Fix this max-heap usage and explain the bug:
```python
h = []
for x in [3, 1, 4]:
    heapq.heappush(h, x)
largest = heapq.heappop(h)
```
3. Write a one-line function `has_pair_with_sum(nums, target)` using a set.

<details>
<summary><b>Solutions</b></summary>

1. (a) mutable default `result=[]` shared across calls; (b) `x not in result` is O(n) per lookup → O(n²) overall — use a `set` for membership; (c) subtle: with the default fixed but no set, "dedupe adjacent" doesn't match the code — the code dedupes globally, so the name/spec lies. Interview lesson: code, name, and spec must agree.
2. heapq pops the *minimum* — `largest` gets 1. Fix: push `-x` and negate on pop, or use `heapq.nlargest(1, ...)`.
3. 
```python
def has_pair_with_sum(nums, target):
    seen = set()
    return any((n in seen) or seen.add(target - n) for n in nums)   # cute but obscure
    # clearer version — prefer this in interviews:
    # for n in nums:
    #     if target - n in seen: return True
    #     seen.add(n)
    # return False
```

</details>
