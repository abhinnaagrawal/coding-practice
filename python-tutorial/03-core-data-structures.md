[← back to index](/python-tutorial/README.md)

# Core Data Structures

This is **the** interview chapter. ~90% of Python interview code is list + dict + set + slicing + comprehensions.

## list — the dynamic array (≈ `ArrayList`)

```python
nums = [3, 1, 4, 1, 5]
nums.append(9)        # O(1) amortized — add at end
nums.insert(0, 2)     # O(n) — insert at index
nums.pop()            # O(1) — remove & return last; pop(0) is O(n)!
nums.remove(1)        # O(n) — remove first occurrence by VALUE
del nums[0]           # O(n) — delete by index
len(nums)             # 5
3 in nums             # O(n) membership test
nums.index(4)         # O(n) — first index of value (ValueError if absent)
nums.sort()           # in-place, Timsort O(n log n)
sorted(nums)          # returns NEW sorted list, original untouched
nums.reverse()        # in-place reversal
```

Python lists are not linked lists — indexing is O(1). For O(1) pops from *both* ends, use `collections.deque` (chapter 7).

## Slicing — no direct C/Java equivalent, learn it cold

```python
s = [0, 1, 2, 3, 4, 5]
s[1:4]    # [1, 2, 3]     — [start, stop), stop excluded
s[:3]     # [0, 1, 2]     — from beginning
s[3:]     # [3, 4, 5]     — to end
s[-1]     # 5             — last element (negative = from the end)
s[-2:]    # [4, 5]        — last two
s[::-1]   # [5,4,3,2,1,0] — reversed (step -1)
s[::2]    # [0, 2, 4]     — every other
s[1:4] = [9, 9]           # splice assignment (lengths may differ!)
```

Slices never raise `IndexError` on out-of-range bounds — they clamp. Slicing a list **copies** (shallow). Works identically on `str` and `tuple`.

## dict — the hash map (≈ `HashMap`)

```python
counts = {}
counts["a"] = counts.get("a", 0) + 1   # get with default — THE counting idiom
"a" in counts          # O(1) key membership — use this constantly
counts["a"]            # KeyError if missing!
counts.pop("a", None)  # remove, no error if absent
counts.keys()          # iterable views: keys(), values(), items()
for k, v in counts.items():
    ...
```

- Keys must be hashable (immutable): `str`, `int`, `tuple` ✓; `list` ✗.
- Since Python 3.7, dicts preserve **insertion order** (this is a language guarantee, not an implementation detail).
- Interview upgrades (chapter 7): `collections.defaultdict(int)` kills the `.get(k, 0)` boilerplate; `collections.Counter` counts in one line.

## set — the hash set (≈ `HashSet`)

```python
seen = set()
seen.add(x); seen.discard(y)   # discard = remove without KeyError
x in seen                      # O(1) — the main reason sets exist
a | b    # union          a & b    # intersection
a - b    # difference     a ^ b    # symmetric difference
```

`set(iterable)` dedupes in O(n): `len(set(nums)) != len(nums)` is the canonical duplicate check. Frozenset is the immutable/hashable variant (usable as dict key).

## tuple — immutable sequence

```python
point = (3, 4)
x, y = point            # unpacking — used everywhere
a, b = b, a             # swap via tuple packing — no temp variable!
d = {(0, 0): "origin"}  # tuples are hashable → dict keys (lists are not)
```

Single-element tuple needs a trailing comma: `(1,)` — `(1)` is just `1` in parentheses.

## Comprehensions — the Python superpower

```python
squares = [x * x for x in range(10)]            # list
evens   = {x for x in nums if x % 2 == 0}       # set (with filter)
sq_map  = {x: x * x for x in nums}              # dict
matrix  = [[0] * n for _ in range(m)]           # 2-D init (see gotcha below)
```

Read them as "make a list of `x*x`, for each x, where …". Anything you'd write as "initialize empty list + for + append" should be a comprehension — interviewers notice.

## Sorting with keys

```python
words.sort(key=len)                        # by length
words.sort(key=lambda w: (-len(w), w))     # length desc, then alpha asc — tuple keys!
sorted(pairs, key=lambda p: p[1])          # by second element
```

Python has **no reverse comparators**; negate the key (or use `reverse=True`). Tuple keys compare element-wise — extremely powerful.

## Complexity cheat sheet (quote these in interviews)

| Operation | list | dict/set | deque |
|-----------|------|----------|-------|
| index `[i]` | O(1) | — | O(1) ends |
| append / pop (end) | O(1)* | O(1) add/discard | O(1) both ends |
| insert / pop (front) | O(n) | — | O(1) |
| `in` (membership) | O(n) | O(1) | O(n) |
| sort | O(n log n) | — | — |

\* amortized. dict/set costs are average-case (hash collisions worst-case O(n), practically never).

## Gotchas

- **The 2-D array trap**: `[[0] * n] * m` creates m references to the SAME row — `grid[0][0] = 1` sets every row's first cell. Use the comprehension form `[[0] * n for _ in range(m)]`. Arguably the #1 Python interview bug.
- `list.pop(0)` and `list.remove(x)` are O(n) — reaching for them in a loop is the hidden-quadratic bug. Use `deque.popleft()` or restructure.
- `in` on a list is O(n); if you membership-test inside a loop, convert to a `set` first — this is the Two Sum pattern in one sentence.
- Sorting mixed types (`[1, "a"]`) raises `TypeError`. Sorting `None` with numbers too.
- `s[:]` is a **shallow** copy: `[[1],[2]][:]` copies the outer list but shares the inner ones.
- A tuple containing a list is mutable-by-proxy and unhashable — hashability requires *everything inside* to be immutable.

## Exercises

1. Given `s = "abcdef"`, produce `"fed"` using one slice.
2. Write a one-liner returning the two most common words in a list (you may use `collections.Counter`).
3. Given a list of `(name, score)` tuples, sort by score descending, then name ascending — one line.
4. Initialize a 3×4 zero matrix correctly, then set cell (1,2) to 5 without touching other rows.

<details>
<summary><b>Solutions</b></summary>

1. `s[:3:-1]` → start from the end, step −1, stop *before* index 3 → `"fed"`. (Alternatively `s[5:2:-1]`.)
2. `[w for w, _ in Counter(words).most_common(2)]`
3. `sorted(pairs, key=lambda p: (-p[1], p[0]))`
4. `grid = [[0]*4 for _ in range(3)]; grid[1][2] = 5`

</details>
