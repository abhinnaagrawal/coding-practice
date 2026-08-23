[← back to index](/python-tutorial/README.md)

# Advanced: Decorators, Context Managers & Typing

Not day-one interview material, but senior-level rounds and real codebases assume these. Each builds directly on chapter 4 (first-class functions) and 6 (context managers/generators).

## Decorators — functions that wrap functions

A decorator is just a function that takes a function and returns a (usually wrapped) function. The `@` syntax is sugar:

```python
@timer
def slow():
    ...

# means exactly:  slow = timer(slow)
```

Writing one (memorize this template — it comes up in "design a retry/cache decorator" questions):

```python
import functools, time

def timer(func):
    @functools.wraps(func)              # preserves func's name/docstring — always include
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = func(*args, **kwargs)
        print(f"{func.__name__} took {time.perf_counter() - start:.4f}s")
        return result
    return wrapper
```

Anatomy: `timer` receives the function → `wrapper` receives the call arguments → inner closure captures `func`. `*args, **kwargs` makes it work for any signature.

Decorators with arguments add one more nesting level (a factory that returns the decorator):

```python
def retry(times):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(times):
                try:
                    return func(*args, **kwargs)
                except Exception:
                    if attempt == times - 1:
                        raise
        return wrapper
    return decorator

@retry(3)
def flaky_network_call(): ...
```

You've already used decorators: `@property`, `@staticmethod`, `@lru_cache`, `@dataclass`, `@abstractmethod`. Flask/FastAPI routes (`@app.get("/")`) are the same mechanism. Java analogy: annotations + AOP, but implemented in plain functions you can read.

## Context managers — the `with` protocol

`with open(...) as f` works because file objects implement `__enter__` / `__exit__`:

```python
class Timer:
    def __enter__(self):
        self.start = time.perf_counter()
        return self                    # bound to the `as` variable

    def __exit__(self, exc_type, exc_val, exc_tb):
        print(f"elapsed: {time.perf_counter() - self.start:.4f}s")
        return False                   # False = don't suppress exceptions

with Timer():
    heavy_work()
```

`__exit__` runs **no matter how the block exits** — normal return, exception, even `return` from inside. This is Java's try-with-resources generalized to any object, plus transactions/locks/timers.

The `@contextmanager` decorator writes one as a generator — everything before `yield` is enter, after is exit:

```python
from contextlib import contextmanager

@contextmanager
def timer():
    start = time.perf_counter()
    try:
        yield                          # control passes to the with-block
    finally:
        print(f"elapsed: {time.perf_counter() - start:.4f}s")
```

Use cases you'll meet: locks (`with lock:`), DB transactions, temp-directory fixtures in tests, `contextlib.suppress(FileNotFoundError)`.

## Type hints — optional static typing

Hints are **annotations only** — CPython ignores them at runtime. Tools (`mypy`, Pylance/pyright) check them statically, giving back a slice of Java's compile-time safety:

```python
def two_sum(nums: list[int], target: int) -> list[int]:   # 3.9+ builtin generics
    seen: dict[int, int] = {}
    for i, n in enumerate(nums):
        if target - n in seen:
            return [seen[target - n], i]
        seen[n] = i
    return []
```

Essentials:

```python
from typing import Optional, Union, Callable, Protocol, Any

x: Optional[str] = None        # str | None — Optional[X] means "X or None"
y: int | str = 3               # 3.10+ union syntax
handler: Callable[[int, int], int]   # function taking two ints, returning int
```

```python
class Sized2(Protocol):        # structural interface — duck typing, but checkable
    def __len__(self) -> int: ...
```

Hints shine in: dataclasses (chapter 5 — they're required there), public APIs, large codebases. In interviews they're optional; adding them to function signatures is a small senior-level signal. Do **not** expect runtime enforcement:

```python
def f(x: int) -> int:
    return x
f("not an int")    # runs fine — hints don't validate!
```

(For runtime validation, that's what libraries like `pydantic` exist for.)

## Quick tour: three more "senior" features

**Keyword-only and positional-only parameters** — control how callers may pass args:

```python
def connect(host, /, port, *, timeout, retries=3): ...
# before / : positional only     after * : keyword only
connect("db", 5432, timeout=5)          # ok
connect(host="db", port=5432, timeout=5)  # TypeError: host is positional-only
```

**Walrus operator** `:=` — assign inside an expression (3.8+). The interview-useful form:

```python
if (n := len(items)) > 10:
    print(f"too many: {n}")      # n is usable after the if
while chunk := f.read(8192):     # read until empty — the do-while replacement
    process(chunk)
```

**Structural pattern matching** (`match`, 3.10+) — switch that can destructure:

```python
match command.split():
    case ["quit"]:
        break
    case ["go", direction]:              # binds direction
        move(direction)
    case ["drop", *items]:               # binds a list
        drop(items)
    case _:                              # default
        print("unknown command")
```

## Gotchas

- Forgetting `@functools.wraps` → wrapped functions all report the name `wrapper`; breaks introspection, logging, and some frameworks.
- Decorator executes at **definition time**, not call time — module import runs all decorator factories. Don't do expensive work there.
- `__exit__` returning a truthy value **suppresses** the exception — almost never what you want; return `False`/nothing.
- Type hints are lies the runtime doesn't check: `list[int]` accepts anything. Hints are for humans and static checkers.
- `Optional[X]` does not make the argument optional to *pass* — it means the value may be `None`. Default values make arguments optional: `def f(x: Optional[str] = None)`.
- `match` cases with a bare name (e.g. `case direction:`) match **anything** and bind it — a common footgun; literal patterns must be literals or dotted names.

## Exercises

1. Write a `@count_calls` decorator that tracks how many times a function has been called, exposed as `func.call_count`.
2. Write a `suppress` context manager (don't use `contextlib.suppress`) that swallows a given exception type, using the class-based `__enter__/__exit__` form.
3. Annotate this function with type hints: it takes a list of `(name, score)` tuples and a minimum score, and returns the names at or above the threshold.

<details>
<summary><b>Solutions</b></summary>

**1.**

```python
import functools

def count_calls(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        wrapper.call_count += 1
        return func(*args, **kwargs)
    wrapper.call_count = 0
    return wrapper
```

**2.**

```python
class suppress:
    def __init__(self, *exc_types):
        self.exc_types = exc_types

    def __enter__(self):
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        return exc_type is not None and issubclass(exc_type, self.exc_types)
        # truthy return = suppress matching exceptions
```

**3.**

```python
def qualifying_names(pairs: list[tuple[str, int]], threshold: int) -> list[str]:
    return [name for name, score in pairs if score >= threshold]
```

</details>

---

## Where to go from here

- Practice against the [interview-prep](/coding-practice/README.md) folder — every problem there is now readable.
- The official tutorial's remaining chapters ([standard library tour](https://docs.python.org/3/tutorial/stdlib.html)) are worth a skim.
- For depth: *Fluent Python* (Ramalho) is the canonical "you know Java, now learn real Python" book.
