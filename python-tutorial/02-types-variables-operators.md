[← back to index](/python-tutorial/README.md)

# Types, Variables & Operators

## The single most important mental shift

In C/Java a variable is a **box** that holds a value of a declared type. In Python a variable is a **name tag stuck on an object**. Assignment moves the tag; it never copies the object.

```python
a = [1, 2, 3]
b = a          # b is another tag on the SAME list — no copy
b.append(4)
print(a)       # [1, 2, 3, 4]  ← "spooky action at a distance" if you think in C
```

Types belong to **objects**, not names. A name can be re-stuck to a different type at any time (dynamic typing):

```python
x = 42        # x tags an int
x = "hello"   # now x tags a str — legal, no error
```

Java parallel: every variable behaves like a non-primitive reference (`Integer`, `String`), and there's no declared type checked at compile time.

## Core built-in types

| Type | Mutable? | Java/C analog | Notes |
|------|----------|---------------|-------|
| `int` | No | `long`… but unbounded | Arbitrary precision — **no overflow, ever** |
| `float` | No | `double` | IEEE 754 double |
| `bool` | No | `boolean` | Subclass of `int`: `True == 1` |
| `str` | No | `String` | Immutable, exactly like Java strings |
| `list` | **Yes** | `ArrayList` | Growable array |
| `tuple` | No | — (record-ish) | Immutable sequence; the hashable "list" |
| `dict` | **Yes** | `HashMap` | Insertion-ordered since 3.7 |
| `set` | **Yes** | `HashSet` | Unordered, unique elements |
| `NoneType` | — | `null` | `None` is a singleton object |

**Why mutability matters** (this drives half the gotchas in the language):
- Only *immutable* (hashable) objects can be dict keys / set members: `str`, `int`, `tuple` yes; `list`, `dict`, `set` no.
- Mutating an object through one name is visible through every other name bound to it (the `b = a` example above).

## Numbers and operators

```python
7 / 2     # 3.5   ← true division ALWAYS returns float (unlike C/Java int division!)
7 // 2    # 3     ← floor division (this is the C/Java int behavior)
-7 // 2   # -4    ← floors toward -inf, NOT truncation toward zero like C/Java!
-7 % 2    # 1     ← result follows the divisor's sign; C gives -1
2 ** 10   # 1024  ← exponent operator, not ^ (which is bitwise XOR, as in C)
```

Other differences from C/Java:
- No `++` / `--`. Use `x += 1`.
- `and`, `or`, `not` are keywords (not `&&`, `||`, `!`). They short-circuit and **return one of the operands**, not a boolean: `0 or "default"` → `"default"`. This enables the idiom `name = supplied or "default"`.
- Chained comparisons: `1 <= x < 10` works and means what a mathematician expects.
- Bitwise ops `& | ^ ~ << >>` work on ints of arbitrary size.

## `==` vs `is`

```python
a = [1, 2]; b = [1, 2]
a == b    # True  — values equal (≈ Java .equals())
a is b    # False — different objects (≈ Java ==)
```

Rules:
- Use `==` for value comparison (99% of the time).
- Use `is` **only** for singletons: `x is None`, `x is True`. Never `is` for strings/ints — CPython caches some small ones, so `is` sometimes "works" and then silently breaks on larger values. Classic interview trap.

## Truthiness

Every object has a truth value. Falsy: `None`, `False`, `0`, `0.0`, `""`, `[]`, `{}`, `set()`, `()`. Everything else is truthy.

```python
if items:        # idiomatic: "if the list is non-empty"
    ...
```

Contrast with Java, where `if (list)` doesn't compile and you write `!list.isEmpty()`. Gotcha: `0` and `""` are falsy, so `if x:` is **not** the same as `if x is not None:` — if `0` is a legitimate value, test explicitly.

## Type conversion

```python
int("42")      # 42        (throws ValueError on "4a2")
float("3.14")  # 3.14
str(42)        # "42"
list("abc")    # ['a', 'b', 'c']
int("ff", 16)  # 255       (base conversion — handy in interviews)
bin(5)         # '0b101'   (also oct(), hex())
ord('a')       # 97        (char → code point)
chr(97)        # 'a'
```

There are no char literals — a "character" is just a `str` of length 1. `ord`/`chr` replace C's char arithmetic; note `'a' + 1` is a **TypeError**, unlike C.

## Gotchas

- `//` floors toward −∞, so `a // b` with mixed signs differs from C/Java. For "C-style truncation" use `int(a / b)` or `math.trunc`.
- `b = a` never copies. For a copy: `b = a.copy()` or `b = a[:]` (shallow — nested objects still shared; see chapter 8).
- `True == 1` and `False == 0` are `True`, and `True + True == 2`. Occasionally useful (`sum(bools)` counts trues), occasionally surprising.
- Floats have the usual IEEE 754 traps: `0.1 + 0.2 != 0.3`. Same as Java, but easier to hit since `/` always makes floats.
- Naming: don't shadow builtins — naming a variable `list`, `str`, `id`, or `max` breaks those functions within the scope.

## Exercises

1. In the REPL: `a = 1000; b = 1000; a is b` — then try with `5`. Explain the difference.
2. What does `10 and [] or "x"` evaluate to? Trace the short-circuit rules, then verify.
3. Write a one-liner that counts how many bits are set in an integer `n` (hint: `bin()` and a string method).

<details>
<summary><b>Solutions</b></summary>

1. `5 is 5` → `True`, `1000 is 1000` → usually `False` in a REPL. CPython caches small ints (−5..256) as singletons; larger ones may be distinct objects. Lesson: `is` on ints is implementation-defined — always use `==`.
2. `10 and []` → `[]` (10 is truthy, so `and` returns the second operand). `[] or "x"` → `"x"`. Result: `"x"`.
3. `bin(n).count('1')`.

</details>
