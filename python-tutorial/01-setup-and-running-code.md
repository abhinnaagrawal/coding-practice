[← back to index](/python-tutorial/README.md)

# Setup, REPL & Running Code

## The mental model: interpreter vs compiler

| | C / Java | Python |
|---|----------|--------|
| Compile step | Explicit (`gcc`, `javac`) | None you invoke — CPython compiles to **bytecode** automatically at load time (cached in `__pycache__/`) |
| Runs on | Native machine code / JVM | CPython interpreter executes the bytecode |
| Type errors | Caught at compile time | Caught at **runtime** — the line must actually execute |
| REPL | None (or `jshell` in Java 9+) | First-class: type `python3` and experiment |
| Entry point | `main()` / `public static void main` | Whatever is at the top level of the script, top to bottom |

Key consequence for interviews: **no compiler safety net**. A typo in a branch that never runs won't be caught until it runs. Read your code once through before saying "done".

## Running code

```bash
python3 script.py          # run a file
python3                    # open the REPL (exit with exit() or Ctrl-D)
python3 -c "print(2**10)"  # one-liner from the shell
python3 -m venv .venv      # run a *module* as a script (more below)
```

In the REPL, the result of every expression is echoed — this is the fastest way to learn the language. Use it while reading chapters 2–4.

## Scripts and the `__main__` guard

A `.py` file is just top-level statements executed in order. But files can also be **imported** by other files — and import *runs the top-level code too*. The idiom that separates "script behavior" from "import behavior":

```python
def main():
    print("running as a script")

if __name__ == "__main__":
    main()
```

`__name__` is a magic variable: it equals `"__main__"` when the file is run directly, and the module name when imported. This is the exact analog of Java's `public static void main` — but opt-in. Interviewers like seeing it; it signals you've written real Python.

Shebang for direct execution (familiar from Bash):

```python
#!/usr/bin/env python3
```
plus `chmod +x script.py`, then `./script.py`.

## Packages: pip & venv (the Maven/Gradle analog)

- **pip** installs third-party packages from PyPI (≈ Maven Central): `pip install requests`
- **venv** creates an isolated per-project environment (≈ per-project classpath, but stronger — it isolates the interpreter itself):

```bash
python3 -m venv .venv        # create
source .venv/bin/activate    # activate (bash/zsh)
pip install requests         # installs into .venv only
deactivate                   # leave
```

Always use a venv for real projects; for interview prep the system Python plus the standard library is enough. **The standard library is huge and interview-relevant** — chapter 7 covers the parts that matter (`collections`, `heapq`, `bisect`).

## Gotchas

- `python` vs `python3`: on macOS, `python` may be missing or Python 2. Always use `python3`.
- Python 2 is dead (EOL 2020). If a snippet uses `print x` without parentheses, it's Python 2 — ignore it.
- Indentation is syntax (4 spaces, never tabs). Mis-indentation changes program meaning, not just style — unlike C/Java where braces define blocks regardless of whitespace.
- There is no `++`/`--`. Use `x += 1`.
- Semicolons are legal but unidiomatic; one statement per line.

## Exercises

1. Open the REPL. Compute `2 ** 100`, then `len(str(2 ** 100))`. Note that this didn't overflow — hold that thought for chapter 2.
2. Write a file `hello.py` with a `main()` and the `__main__` guard. Run it two ways: `python3 hello.py` and from the REPL with `import hello` (then `hello.main()`). Observe that the guard block did *not* run on import.
3. Create a venv, activate it, and `pip install` any small package. Run `pip list` before and after to see the isolation.

<details>
<summary><b>Solutions</b></summary>

1. `2 ** 100` → `1267650600228229401496703205376`; `len(str(...))` → `31`. Python ints have arbitrary precision — no overflow.
2. On `import hello`, Python executes the file top-to-bottom (defining `main`) but `__name__` is `"hello"`, so the guard block is skipped. You then call `hello.main()` manually.
3. `python3 -m venv .venv && source .venv/bin/activate && pip install cowsay && pip list` — `cowsay` appears only while the venv is active.

</details>
