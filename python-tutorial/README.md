# Python in One Day — For C/Java/Bash Engineers

Live site: **https://abhinnaagrawal.github.io/coding-practice/#/python-tutorial/**
Top-level site: [Home](/README.md)

Audience: experienced engineers (C / Java / Bash background) learning Python from scratch, primarily for **coding interviews** and **OOP concepts**. Shareable — no personal context.

Source: structured around the official [Python 3 tutorial](https://docs.python.org/3/tutorial/), compressed and reordered for interview relevance, with every concept framed against what you already know from C/Java.

## How to use this
- Each chapter is standalone — read in order for the full arc, or jump to what you need.
- Every concept has: **what it is → minimal example → C/Java contrast → gotchas**.
- Each chapter ends with 2–3 micro-exercises (solutions collapsed — try before expanding).
- Total pacing: ~8 hours of focused work. The pacing table below splits it into blocks.

## Pacing plan (one day)

| Block | Chapters | Focus | Est. time |
|-------|----------|-------|-----------|
| Morning 1 | 1–2 | Mental model: how Python runs, types, mutability | ~1.75 h |
| Morning 2 | 3–4 | The interview core: data structures, control flow, functions | ~2.5 h |
| Afternoon 1 | 5 | OOP — the deepest chapter, take it slow | ~1.5 h |
| Afternoon 2 | 6–7 | Strings/files/generators + exceptions & interview stdlib | ~1.75 h |
| Evening | 8–9 | Idioms, gotchas, interview playbook + advanced topics | ~1.5 h |

## Chapters

| # | Chapter | What you get |
|---|---------|--------------|
| 1 | [Setup, REPL & Running Code](/python-tutorial/01-setup-and-running-code.md) | Interpreter vs compiled, scripts, venv/pip (≈ Maven), `__main__` |
| 2 | [Types, Variables & Operators](/python-tutorial/02-types-variables-operators.md) | Dynamic typing, mutability model, `==` vs `is`, truthiness |
| 3 | [Core Data Structures](/python-tutorial/03-core-data-structures.md) | list / tuple / dict / set, slicing, comprehensions, complexity table |
| 4 | [Control Flow & Functions](/python-tutorial/04-control-flow-and-functions.md) | Loops, `enumerate`/`zip`, args/kwargs, mutable default args, scope |
| 5 | [OOP in Python](/python-tutorial/05-oop-in-python.md) | Classes vs Java, `self`, dunder methods, inheritance, duck typing, dataclasses |
| 6 | [Strings, Files & Generators](/python-tutorial/06-strings-files-generators.md) | Interview string methods, file I/O, `yield`, `itertools` |
| 7 | [Exceptions, Modules & Stdlib](/python-tutorial/07-exceptions-modules-stdlib.md) | try/except vs Java, imports, `collections` / `heapq` / `bisect` |
| 8 | [Idioms, Gotchas & Interview Playbook](/python-tutorial/08-idioms-gotchas-interview-playbook.md) | Top-20 gotchas, copy semantics, live-coding playbook |
| 9 | [Advanced: Decorators, Context Managers & Typing](/python-tutorial/09-advanced-decorators-context-managers-typing.md) | Nice-to-have depth for senior rounds |

## The five things that will bite you coming from C/Java
If you remember nothing else on day one:

1. **Assignment never copies.** `b = a` makes another name for the *same object*. There are no primitive/value-type copies like in C.
2. **Mutability is per-type, not per-variable.** `list`/`dict`/`set` are mutable; `int`/`str`/`tuple` are not. This drives aliasing bugs, dict-key rules, and the mutable-default-argument trap.
3. **`==` compares values, `is` compares identity** (≈ Java `equals()` vs `==`).
4. **Indentation is syntax.** Braces are gone; a mis-indented block is a *different program*, not just ugly code.
5. **Everything is an object** — functions, classes, even modules. Functions are first-class values you pass around like Java lambdas, but everywhere.
