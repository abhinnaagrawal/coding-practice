[← back to index](/python-tutorial/README.md)

# Strings, Files & Generators

## Strings — the interview workhorse

Strings are immutable (like Java) — every "modification" creates a new string. All slicing from chapter 3 applies: `s[::-1]` reverses, `s[:i] + s[i+1:]` deletes a char.

### Methods you'll use constantly in interviews

```python
s.split()              # split on any whitespace runs: "a  b\tc" → ['a','b','c']
s.split(",")           # split on delimiter
s.split(",", 1)        # maxsplit — split at most once
",".join(parts)        # THE way to build strings (≈ String.join)
s.strip()              # trim whitespace (lstrip/rstrip for one side)
s.startswith("pre")    # also endswith; both accept a TUPLE of options
s.find("x")            # index or -1  (index() raises instead — find is safer)
s.replace("a", "b")    # ALL occurrences (unlike Java replaceFirst)
s.lower(), s.upper()
s.isdigit(), s.isalpha(), s.isalnum()   # char-class checks
"abc" * 3              # 'abcabcabc' — repetition operator
```

### f-strings — interpolation (≈ Java 15 text blocks / `String.format`)

```python
name, score = "ada", 97.456
f"{name} scored {score:.1f}"       # 'ada scored 97.5'  — format spec after :
f"{2+2=}"                          # '2+2=4' — debug printing (3.8+)
f"{x:05d}"  f"{x:>10}"  f"{x:,}"   # zero-pad, right-align, thousands separator
```

Prefer f-strings everywhere; `"...".format()` and `%` formatting exist in legacy code.

### Character math

```python
ord('a')          # 97
chr(97)           # 'a'
ord(ch) - ord('a')   # 0..25 — the letter-index trick for counting/grouping
```

Remember: `'a' + 1` is a TypeError (unlike C). And since strings are immutable, **building a string with `+=` in a loop is O(n²)** — accumulate in a list and `"".join(parts)` at the end (the `StringBuilder` pattern).

## Files — context managers do the closing

```python
with open("data.txt") as f:        # auto-closes even on exception (≈ try-with-resources)
    for line in f:                 # iterate lines lazily — memory-safe for huge files
        process(line.strip())

content = f.read()                 # whole file as one string (small files only)
lines = f.readlines()              # list of lines, with '\n' attached

with open("out.txt", "w") as f:    # 'w' truncates; 'a' appends; 'r' default
    f.write("hello\n")
```

The `with` statement is a **context manager** — it guarantees cleanup. Rule: never call bare `open()` without `with` in interview code; it's the Python equivalent of leaking a file handle. (Chapter 9 shows how to write your own.)

## Iterators — the protocol underneath `for`

Everything `for` can consume is an **iterable**. The protocol (Java's `Iterable`/`Iterator`, exposed):

```python
it = iter([1, 2, 3])     # get iterator
next(it)                 # 1 — advances; raises StopIteration when exhausted
```

`for x in xs` is sugar for `iter()` + repeated `next()` until `StopIteration`.

## Generators — `yield` changes everything

A function containing `yield` doesn't run when called — it returns a **generator**, which computes values lazily, one at a time, suspending and resuming between them:

```python
def countdown(n):
    while n > 0:
        yield n          # emit value, freeze here
        n -= 1           # resume here on next()

list(countdown(3))       # [3, 2, 1]
```

Why interviews care:
- **O(1) memory** for large/infinite sequences: `def naturals(): ...` can yield forever.
- **Clean graph/tree code**: `yield from dfs(child)` flattens recursion beautifully.

```python
def inorder(node):                  # yields values in-order, no result list needed
    if node:
        yield from inorder(node.left)
        yield node.val
        yield from inorder(node.right)
```

Generator expressions are the lazy version of comprehensions:

```python
sum(x*x for x in range(10**7))    # generator — O(1) memory
[x*x for x in range(10**7)]       # list — materializes 10M items
```

## itertools — the standard library's iteration toolbox

```python
from itertools import permutations, combinations, accumulate, groupby, product, chain

permutations("abc", 2)     # ('a','b'), ('a','c'), ('b','a'), ... — ordered arrangements
combinations("abc", 2)     # ('a','b'), ('a','c'), ('b','c') — unordered subsets
product("AB", "12")        # cartesian product — nested loops in one call
accumulate([1,2,3,4])      # 1, 3, 6, 10 — running prefix sums!
chain(list1, list2)        # concatenate lazily
groupby("aaabbbcca")       # groups of equal CONSECUTIVE items: a×3, b×3, c×2, a×1
```

`accumulate` turns prefix-sum problems into one-liners; `combinations`/`permutations` cover brute-force enumeration; `product` replaces nested loops. Knowing these exists is a differentiator.

## Gotchas

- String concatenation in a loop is O(n²) — use join.
- `s.split()` (no arg) is usually what you want; `s.split(" ")` treats every single space as a separator and produces empty strings between consecutive spaces.
- Iterators/generators are **single-use and consumed**: after `list(gen)`, `gen` is empty. Iterating a generator twice silently yields nothing the second time.
- `sorted(s)` on a string returns a **list of chars**; `"".join(sorted(s))` gives the sorted string.
- Files iterate including the trailing `\n` — that's why `.strip()` appears in every file loop.
- `f.read()` on a multi-GB file loads it all into memory; iterate lines instead.
- `next(it)` raises `StopIteration` on exhaustion; `next(it, default)` returns a default instead — useful in interview code.

## Exercises

1. Write a one-liner that checks whether two strings are anagrams (no `Counter` — use sorting).
2. Write a generator `fib()` that yields Fibonacci numbers forever, then print the first 10.
3. Given `"aabbccccdddeee"`, use `groupby` to produce `["a2", "b2", "c4", "d3", "e3"]` (run-length encoding) in one comprehension.

<details>
<summary><b>Solutions</b></summary>

1. `sorted(s1) == sorted(s2)`
2. 
```python
def fib():
    a, b = 0, 1
    while True:
        yield a
        a, b = b, a + b

from itertools import islice
print(list(islice(fib(), 10)))   # islice takes the first n from an infinite generator
```

3. 
```python
from itertools import groupby
[f"{ch}{len(list(g))}" for ch, g in groupby("aabbccccdddeee")]
```
(the inner `list(g)` is needed because the group iterator is lazy)

</details>
