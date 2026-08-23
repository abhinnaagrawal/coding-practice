[← back to index](/python-tutorial/README.md)

# Control Flow & Functions

## Conditionals and loops

```python
if x < 0:
    sign = -1
elif x == 0:      # note: elif, not else if
    sign = 0
else:
    sign = 1
```

No parentheses needed, colon required, blocks are indentation. Ternary: `sign = -1 if x < 0 else 1` (expression, not statement — like Java's `?:` but reads left-to-right in English order).

```python
for ch in "abc":          # for-each over any iterable — there is NO C-style for(;;)
    ...
while stack:              # truthiness: loops while non-empty
    ...
```

There is no `do-while`; the idiom is `while True:` + `break`.

## The iteration toolkit (this is what replaces index loops)

```python
range(5)          # 0..4 — lazy, not a list; range(start, stop, step)
range(10, 0, -1)  # countdown

for i, val in enumerate(items):        # index + value — kills range(len(x))
for i, val in enumerate(items, 1):     # start index at 1

for a, b in zip(names, scores):        # parallel iteration, stops at shortest
for k, v in d.items():                 # dict iteration
reversed(items)                        # lazy reverse iterator
```

If you write `for i in range(len(items)):` just to access `items[i]`, that's the tell of a C/Java accent — use `enumerate`.

`for...else` quirk: the `else` runs if the loop finished **without `break`**. Useful for "searched and not found":

```python
for n in nums:
    if n == target:
        break
else:
    print("not found")   # runs only when no break happened
```

`match` statement (Python 3.10+) is a structural switch — nice to know, rarely needed in interviews.

## Functions

```python
def gcd(a, b):            # no type declarations (hints optional, chapter 9)
    while b:
        a, b = b, a % b
    return a              # no return statement → returns None implicitly
```

Functions are **first-class objects**: assign them, pass them, store them in dicts.

```python
ops = {"add": lambda a, b: a + b, "mul": lambda a, b: a * b}
ops["add"](2, 3)   # 5
```

`lambda` is a single-expression anonymous function (≈ Java lambda, but no braces, no statements). Great as `key=` arguments.

### Arguments: more flexible than Java

```python
def f(a, b=10, *args, key=None, **kwargs):
    ...
```

- `b=10` — default parameter (call as `f(1)`, `f(1, 2)`, `f(1, b=2)`)
- `*args` — captures extra positional args as a tuple (≈ Java varargs `int...`)
- `**kwargs` — captures extra keyword args as a dict
- Keyword arguments at the call site are first-class style: `requests.get(url, timeout=5)`

### The mutable default argument trap (top-3 interview gotcha)

```python
def add(item, acc=[]):   # WRONG: the list is created ONCE at def time,
    acc.append(item)     #        and SHARED across all calls
    return acc

add(1); add(2)           # [1, 2] — the second call sees the first call's data!
```

Fix — use `None` as sentinel:

```python
def add(item, acc=None):
    if acc is None:
        acc = []
    acc.append(item)
    return acc
```

### Arguments are passed by object reference

Python is **neither** pass-by-value **nor** pass-by-reference in the C sense — it's "pass by assignment": the parameter is a new name bound to the *same object*.

- Mutate the object (`lst.append(x)`) → caller sees it (like Java object refs).
- Rebind the name (`lst = []`) → caller unaffected.
- Ints/strings can't be mutated at all, so they *feel* pass-by-value (like Java primitives).

## Scope: LEGB

Name lookup order: **L**ocal → **E**nclosing → **G**lobal → **B**uiltins.

```python
count = 0
def bump():
    global count    # without this, `count += 1` raises UnboundLocalError
    count += 1
```

Key rule: *reading* a global works without declaration; *assigning* to one requires `global` (or `nonlocal` for enclosing function scopes, used in closures — common in interview solutions with nested `dfs()` helpers).

```python
def traverse(root):
    result = []
    def dfs(node):          # closure: reads `result` from enclosing scope
        if not node:
            return
        result.append(node.val)   # mutation — no nonlocal needed
        dfs(node.left); dfs(node.right)
    dfs(root)
    return result
```

This nested-helper pattern is the standard shape of Python tree/graph interview solutions.

## Gotchas

- Mutable default arguments (above) — memorize it.
- `for` loop variables leak into the enclosing scope (no block scope in Python — unlike Java's `{ }` blocks).
- Late binding in closures: lambdas inside a loop capture the *variable*, not its value. `[lambda: i for i in range(3)]` — all three return 2. Fix: `lambda i=i: i`.
- `return` with no value returns `None`, and `None` is falsy — `if result:` conflates "no result" with "result was 0/empty". Return explicit sentinels or test `is not None`.
- Recursion limit is ~1000 (`sys.setrecursionlimit`). Deep DFS on a skewed tree can blow the stack in Python where the Java equivalent survives — mention it in interviews; iterative-with-explicit-stack is the safe answer.
- Modifying a list while iterating over it skips elements. Iterate over a copy: `for x in items[:]`.

## Exercises

**1.** Write `fizzbuzz(n)` returning a list of strings ("FizzBuzz" for 15, etc.) — with a comprehension if you can.

**2.** Write a function `flatten(nested)` using a nested recursive helper and closure (no `global`), e.g. `flatten([1,[2,[3]],4]) → [1,2,3,4]`.

**3.** Predict, then verify: what does this print?

```python
funcs = [lambda: i for i in range(3)]
print([f() for f in funcs])
```

<details>
<summary><b>Solutions</b></summary>

**1.**

```python
def fizzbuzz(n):
    return ["FizzBuzz" if i % 15 == 0 else "Fizz" if i % 3 == 0
            else "Buzz" if i % 5 == 0 else str(i) for i in range(1, n + 1)]
```

**2.**

```python
def flatten(nested):
    out = []
    def walk(x):
        if isinstance(x, list):
            for e in x:
                walk(e)
        else:
            out.append(x)
    walk(nested)
    return out
```

**3.** `[2, 2, 2]` — all lambdas read the final value of `i`. Fix: `lambda i=i: i`.

</details>
