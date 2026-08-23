[← back to index](/python-tutorial/README.md)

# OOP in Python

The deepest chapter. Python OOP is Java OOP with the ceremony removed and the machinery exposed — everything Java hides (methods as objects, dispatch, attribute lookup) is visible and customizable.

## Classes: the 80% case

```python
class Stack:
    """A simple stack."""            # docstring — becomes Stack.__doc__

    def __init__(self):              # constructor (initializer, technically)
        self._items = []             # instance attributes created in __init__

    def push(self, item):
        self._items.append(item)

    def pop(self):
        return self._items.pop()

    def __len__(self):               # dunder: enables len(stack)
        return len(self._items)

    def __repr__(self):              # dunder: enables print(stack) / debugging
        return f"Stack({self._items})"
```

Mapping from Java:

| Java | Python |
|------|--------|
| `new Stack()` | `Stack()` — calls `__init__` |
| implicit `this` | **explicit `self`** — first parameter of every instance method |
| constructor | `__init__` (never returns anything) |
| `private` | no enforcement — `_name` = "internal by convention", `__name` = name-mangled (see below) |
| `toString()` | `__repr__` (unambiguous, for devs) and `__str__` (pretty, for users) |
| `equals()` / `hashCode()` | `__eq__` / `__hash__` |
| interface | duck typing, or `abc.ABC` when you need enforcement |

`self` is just a convention for the first parameter — Python passes the instance explicitly. `s.push(x)` is sugar for `Stack.push(s, x)`. This is why forgetting `self` in the signature gives the famous "takes N arguments but N+1 given" error.

## Instance vs class attributes

```python
class Dog:
    species = "canis"       # class attribute — shared by ALL instances (≈ Java static field)

    def __init__(self, name):
        self.name = name    # instance attribute — per object
```

**Trap**: a *mutable* class attribute is shared mutable state across all instances — the Java `static List` bug, but easier to write by accident:

```python
class Dog:
    tricks = []             # WRONG — shared by every Dog!
    def __init__(self, name):
        self.name = name
        self.tricks = []    # RIGHT — per instance
```

## No real access modifiers

- `self._cache` — leading underscore: convention meaning "internal, don't touch". IDEs and linters respect it; the language doesn't enforce it.
- `self.__cache` — double underscore triggers **name mangling**: the attribute is renamed to `_ClassName__cache`, making accidental override in subclasses hard. Not security — just hygiene.
- Python's philosophy: "we're all consenting adults." Document intent; don't build walls.

## Dunder methods = operator overloading and protocol hooks

"Dunder" = double underscore. Defining them plugs your class into Python syntax:

```python
class Vector:
    def __init__(self, x, y):
        self.x, self.y = x, y

    def __repr__(self):           return f"Vector({self.x}, {self.y})"
    def __eq__(self, other):      return (self.x, self.y) == (other.x, other.y)
    def __hash__(self):           return hash((self.x, self.y))   # needed for dict/set!
    def __add__(self, other):     return Vector(self.x + other.x, self.y + other.y)
    def __abs__(self):            return (self.x**2 + self.y**2) ** 0.5
    def __len__(self):            return 2
```

Now `v1 + v2`, `abs(v)`, `len(v)`, `v in some_set`, `{v: "value"}` all work.

**Interview-critical rule**: `__eq__` without `__hash__` makes the object unhashable (Python sets `__hash__` to None). Java has the same contract (`equals`/`hashCode` together) — in Python it *enforces* it by removing `__hash__`. If you define equality, define the hash from the same immutable fields.

Other dunders worth knowing: `__lt__` (sorting/`heapq`), `__getitem__` (enables `obj[i]` and iteration), `__contains__` (enables `in`), `__call__` (makes instances callable like functions), `__iter__`/`__next__` (iterator protocol, chapter 6).

## Inheritance

```python
class Animal:
    def speak(self):
        return "..."

class Dog(Animal):
    def speak(self):                # override
        return "woof"

    def describe(self):
        return f"{super().speak()} but really {self.speak()}"   # super() ≈ Java super
```

- No `@Override` annotation (use `@override` from `typing` in 3.12+ if you want checking).
- **Multiple inheritance is supported**: `class C(A, B)`. Method resolution order (MRO) is left-to-right depth-first (C3 linearization); `C.__mro__` shows it. Java avoids this with interfaces; Python embraces it — but keep hierarchies shallow in interview code.
- `isinstance(d, Animal)` and `issubclass(Dog, Animal)` work as expected. Prefer `isinstance` over `type(x) is ...` (respects inheritance).

## Duck typing & protocols

Python doesn't need interfaces for polymorphism: "if it walks like a duck and quacks like a duck…"

```python
def total_length(things):       # works for lists, strings, Stacks — anything with __len__
    return sum(len(t) for t in things)
```

Type is never checked; the operation is attempted and fails at runtime if unsupported. When you *do* want Java-style enforced interfaces:

```python
from abc import ABC, abstractmethod

class Shape(ABC):
    @abstractmethod
    def area(self): ...

class Circle(Shape):
    def __init__(self, r): self.r = r
    def area(self): return 3.14159 * self.r ** 2

# Shape() raises TypeError — cannot instantiate abstract class
```

## @property, @staticmethod, @classmethod

```python
class Temperature:
    def __init__(self, celsius=0):
        self._celsius = celsius

    @property                       # getter — access as t.celsius, no parens
    def celsius(self):
        return self._celsius

    @celsius.setter                 # setter with validation
    def celsius(self, value):
        if value < -273.15:
            raise ValueError("below absolute zero")
        self._celsius = value

    @staticmethod                   # no self — utility namespaced to the class (≈ Java static)
    def from_fahrenheit(f):
        return Temperature((f - 32) * 5 / 9)

    @classmethod                    # receives the CLASS — alternative constructors, inheritance-aware
    def freezing(cls):
        return cls(0)
```

Why `@property` matters: in Java you write getters/setters up front "just in case." In Python, start with plain public attributes (`self.x`), and upgrade to `@property` later **without changing call sites** — `obj.x` keeps working. Never write trivial getters/setters in Python; it's a Java accent.

## dataclasses — boilerplate-free classes

```python
from dataclasses import dataclass

@dataclass
class Point:
    x: int
    y: int = 0            # default value
```

One decorator generates `__init__`, `__repr__`, and `__eq__`. Options: `@dataclass(frozen=True)` (immutable + hashable — great dict keys), `order=True` (adds comparison methods). This is the modern answer to "Java record / Lombok @Data" and shows up constantly in interview code for clean state objects.

## Gotchas

- Forgetting `self` in method signatures → "takes 0 positional arguments but 1 was given".
- Mutable class attributes shared across instances (the `tricks = []` trap).
- `__eq__` without `__hash__` → unhashable object.
- Overriding `__init__` in a subclass without calling `super().__init__()` when the parent initializes needed state.
- `isinstance(x, (int, str))` takes a **tuple** of types, not multiple args.
- Naming a method `__foo` (double leading underscore, no trailing) mangles it — subclasses can't easily override it. You almost never want this; use single `_foo`.
- Classes are objects too — `Point` can be passed to functions, stored in dicts (`{"point": Point}`), instantiated dynamically.

## Exercises

1. Implement a `MinStack` class with `push`, `pop`, `top`, `get_min` — all O(1). (Classic interview problem; use a parallel stack of running minimums.)
2. Create a `Card` dataclass (`rank: str, suit: str`), frozen, and put instances in a `set`. Then sort a list of cards by rank using a rank-order dict and `sorted(key=...)`.
3. Implement `__len__` and `__getitem__` on a `Deck` class so `len(deck)` and `deck[0]` work, and verify that `random.choice(deck)` works *without* you writing anything extra. Why does it?

<details>
<summary><b>Solutions</b></summary>

**1.**

```python
class MinStack:
    def __init__(self):
        self._stack = []
        self._mins = []

    def push(self, x):
        self._stack.append(x)
        self._mins.append(min(x, self._mins[-1] if self._mins else x))

    def pop(self):
        self._mins.pop()
        return self._stack.pop()

    def top(self):
        return self._stack[-1]

    def get_min(self):
        return self._mins[-1]
```

**2.**

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class Card:
    rank: str
    suit: str

RANKS = {r: i for i, r in enumerate("23456789TJQKA")}
hand = {Card("A", "S"), Card("A", "S"), Card("T", "H")}   # set dedupes: 2 elements
sorted(hand, key=lambda c: RANKS[c.rank])
```

**3.**

```python
class Deck:
    def __init__(self, cards):
        self._cards = list(cards)
    def __len__(self):
        return len(self._cards)
    def __getitem__(self, i):
        return self._cards[i]
```

`random.choice` works because it only needs `len()` and integer indexing — duck typing via the sequence protocol. No inheritance, no interface.

</details>
