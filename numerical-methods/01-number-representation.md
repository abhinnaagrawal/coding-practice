[← back to index](/numerical-methods/README.md)

# Day 1 — Number Representation: IEEE 754 and Why Floats Bite

## Why this matters first

Every numerical method is executed in floating point, so its error budget starts here. Two famous failures: the 1991 Patriot missile rounding bug (0.1 s clock drift accumulated over 100 h → 0.34 s error → missed interception) and the Ariane 5 explosion (float→int overflow). Same root cause: the gap between real arithmetic and what the machine does.

## IEEE 754 double: the anatomy

A `float` (Python/Java `double`) is 64 bits:

```
[1 sign][11 exponent][52 mantissa]   value ≈ ±(1.mantissa)₂ × 2^(exponent − 1023)
```

Key consequences — run these:

```python
import sys, math

# 1. 0.1 has no exact binary representation (like 1/3 in decimal)
print(f"{0.1:.20f}")          # 0.10000000000000000555...
print(0.1 + 0.2 == 0.3)       # False
print(0.1 + 0.2)              # 0.30000000000000004

# 2. Machine epsilon: gap between 1.0 and the next float
print(sys.float_info.epsilon) # 2.220446049250313e-16  (≈ 2^-52)
print(1.0 + 1e-16 == 1.0)     # True — the small number vanished
print(1.0 + 1e-15 == 1.0)     # False — just above the gap

# 3. The gap GROWS with magnitude
print(math.nextafter(1.0, 2.0) - 1.0)          # 2.2e-16
print(math.nextafter(1e16, 2e16) - 1e16)       # 2.0 — gap is 2.0 at 1e16!
print(1e16 + 1 == 1e16)                        # True — 1 is below the gap

# 4. Integers are exact only up to 2^53
print(2**53 == 2**53 + 1)      # False — ints are arbitrary precision (Python)
print(float(2**53) == float(2**53 + 1))  # True — as floats they collide
```

## The mental model

Floats are a **finite, non-uniform grid** on the real line: dense near 0, sparse far out. Every operation rounds the true result to the nearest grid point. Two error types follow:

- **Roundoff error** — per-operation, ≤ half a grid step (ulp), relative size ≈ ε ≈ 1e-16.
- **Catastrophic cancellation** — subtracting near-equal numbers: the common leading bits cancel, leaving mostly accumulated roundoff. Relative error explodes even though each operand was fine.

```python
# Cancellation demo: f(x) = (1 - cos(x)) / x^2, limit as x→0 is 0.5
from math import cos
for x in [1e-2, 1e-4, 1e-6, 1e-8]:
    print(x, (1 - cos(x)) / x**2)
# 0.01 0.49999583334722205     ← good
# 0.0001 0.4999999613448124
# 1e-06 0.5000444502908375
# 1e-08 0.0                    ← 1 - cos(x) rounded to exactly 0!
```

At x=1e-8, `cos(x) ≈ 1 - 5e-17` — below half an ulp of 1.0, so `cos(x)` rounds to exactly `1.0` and the numerator vanishes. The classic fix is **algebraic rewriting** to remove the cancellation: `1 - cos(x) = 2 sin²(x/2)`:

```python
from math import sin
x = 1e-8
print(2 * sin(x/2)**2 / x**2)  # 0.5 — correct again
```

## Addition is not associative

```python
a, b, c = 1e16, 1.0, -1e16
print((a + b) + c)   # 0.0   — b was absorbed into a, then cancelled
print(a + (b + c))   # 1.0   — small terms combined first, then survived
```

Practical rules:
- Sum small terms first (or smallest-magnitude first). Better: use `math.fsum` (exactly-rounded summation) or Kahan compensation (chapter exercise).
- Normalize data before feeding it to algorithms — means near 0 and similar magnitudes keep the grid dense.
- Never compare floats with `==`; use `math.isclose(a, b, rel_tol=1e-9)`.

## Special values

```python
1.0 / 0.0        # ZeroDivisionError in Python! (numpy/IEEE gives inf)
float('inf'), float('-inf'), float('nan')
math.isinf(x), math.isnan(x)
float('nan') == float('nan')   # False — NaN equals nothing, including itself
```

Python raises on `1/0.0` (a language choice); IEEE semantics (`inf`, `nan` propagation) apply in numpy and everywhere downstream (Spark, C extensions). `nan` is sticky: any operation with nan returns nan — that's how a single bad row poisons an entire aggregate.

## Where it's used

- **Finance**: ledgers store integer cents or `Decimal`, never float — a 1e-16 drift on 0.1 becomes real money at scale. Interchange formats (FIX, ISO 20022) specify decimal encodings for this reason.
- **Games/graphics**: float16 in GPU shaders (half precision) — chosen per-effect by tolerance, not habit. The famous *Quake III fast inverse square root* is pure bit-level float manipulation.
- **ML**: mixed-precision training (fp16/bf16 compute, fp32 accumulation) is this chapter applied — bf16 trades mantissa for exponent range because gradients span huge magnitudes.
- **Databases/telemetry**: `SUM(float)` column drift, dedup keys that mismatch after float round-trip, NaN poisoning an aggregate — all day-1 phenomena in production.
- **GPS/timing**: Patriot-style bugs — time stored as tenths of seconds in a 24-bit register; precision loss over 100h of uptime.

## Dry run by hand

**1.** Write `0.1` in binary. 0.1 × 2 = 0.2 → 0; 0.2 × 2 = 0.4 → 0; 0.4 × 2 = 0.8 → 0; 0.8 × 2 = 1.6 → 1; 0.6 × 2 = 1.2 → 1; then 0.2 repeats → `0.000110011001100…`₂, repeating forever. The 52-bit mantissa must cut it off — that cutoff is the error you see in `print(f"{0.1:.20f}")`.

**2.** Cancellation by hand in 3-digit decimal: compute `1 − cos(1e-4)` where cos(1e-4) = 0.999999995 rounds to 1.00 (3 significant digits). Numerator = 0.00. Now the rewrite: `2 sin²(0.5e-4) = 2 × (5e-5)² = 5e-9` — every digit survives because nothing cancelled. Same algebra, different floating-point fate.

**3.** Absorption: in 3-digit decimal, add `1.00e4 + 3.00e-2`. Align exponents: 1.00e4 + 0.000003e4 → the small term shifts right 6 places, its digits fall off the 3-digit register → result 1.00e4. That's `1e16 + 1 == 1e16` at double precision, exactly.

## Gotchas

- `0.1` literals in code, JSON, CSV → binary error at parse time, before any arithmetic.
- Accumulating money as float — use `decimal.Decimal` or integer cents. `0.1 + 0.2` shows up in invoices.
- `min`/`max` with NaN silently propagate or drop depending on argument order — filter NaNs first.
- Rounding modes: Python's `round()` uses banker's rounding (round-half-even): `round(2.5) == 2`, `round(3.5) == 4`. C/Java differ.
- Converting float→int truncates toward zero: `int(-2.7) == -2`.

## Exercises

**1.** Find the smallest float `x` such that `1.0 + x != 1.0`, by bisection in the exponent (start at `2.0**-60`, double until it differs). Compare to `sys.float_info.epsilon`.

**2.** Compute the harmonic sum `H = Σ 1/n` for n = 1..10⁷ two ways: forward and backward (n from 10⁷ down to 1). Compare with `math.fsum`. Which is closer, and why?

**3.** The quadratic formula `x = (-b ± √(b²-4ac)) / 2a` has a cancellation landmine for one root when `b² ≫ 4ac`. For `a=1, b=1e8, c=1`, compute both roots naively, then fix the unstable root using the product-of-roots identity `x₁·x₂ = c/a`. (True roots: ≈ -1e8 and ≈ -1e-8.)

<details>
<summary><b>Solutions</b></summary>

**1.**

```python
x = 2.0**-60
while 1.0 + x == 1.0:
    x *= 2
print(x, sys.float_info.epsilon)   # 2.2...e-16 == epsilon
```

**2.** Backward (smallest terms first) matches `math.fsum` far more closely — each partial sum stays small, so each addend is large relative to it. Forward summation loses the tail terms to absorption.

```python
import math
fwd = sum(1/n for n in range(1, 10**7 + 1))
bwd = sum(1/n for n in range(10**7, 0, -1))
ref = math.fsum(1/n for n in range(1, 10**7 + 1))
print(abs(fwd - ref), abs(bwd - ref))   # fwd error >> bwd error
```

**3.**

```python
a, b, c = 1.0, 1e8, 1.0
d = (b*b - 4*a*c) ** 0.5
x1 = (-b - d) / (2*a)          # -1e8 — fine (same-sign addition)
x2 = (-b + d) / (2*a)          # 0.0 — WRONG: cancellation (−b + d ≈ 0)
x2_fixed = c / (a * x1)        # -1e-8 — correct via x1·x2 = c/a
print(x1, x2, x2_fixed)
```

Rule: compute the root whose numerator *adds* magnitudes directly; recover the other from Vieta's identities.

</details>

## Deep dives (open access)

- **"What Every Computer Scientist Should Know About Floating-Point Arithmetic"** — Goldberg, the canonical paper: https://docs.oracle.com/cd/E19957-01/806-3568/ncg_goldberg.html
- **float.exposed** — interactive bit-level explorer: https://float.exposed
- **Trefethen & Bau, *Numerical Linear Algebra***, Lecture 12–13 (floating point & stability) — the cleanest short treatment; widely mirrored in university course notes.
- Higham, *Accuracy and Stability of Numerical Algorithms*, ch. 1–2 — the reference for error analysis (check library/archive.org).
