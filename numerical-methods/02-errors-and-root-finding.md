[← back to index](/numerical-methods/README.md)

# Day 2 — Errors, Conditioning & Root Finding

## Error vocabulary (used all week)

- **Absolute error** = |approx − true|; **relative error** = |approx − true| / |true|. Relative is what matters for cancellation.
- **Forward error** = how wrong is my answer. **Backward error** = "my answer is the *exact* answer to a slightly different problem — how different?" Backward error analysis is the deep trick of the field: a **stable** algorithm has small backward error; combined with the problem's condition number it bounds forward error:

  $$\text{forward error} \;\lesssim\; \underbrace{\kappa}_{\text{condition}} \times \underbrace{\text{backward error}}_{\text{algorithm}}$$

- **Condition number κ** = amplification factor of the problem itself: how much a relative input perturbation inflates the relative output. κ ≈ 1: benign. κ = 10¹²: no algorithm can save you.

**Example — conditioning of root finding.** A root of f where f'(root) ≈ 0 (a *flat* root) is ill-conditioned: tiny changes to the function move the root a lot. f(x) = (x−1)⁸ has a root at 1, but perturbing the constant term by 1e-8 moves the root to ~0.9 or 1.1 — κ is huge. No root finder can beat this.

## Root finding: find x with f(x) = 0

### Bisection — slow but bulletproof

If f(a) and f(b) have opposite signs (and f is continuous), a root lives in [a, b]. Halve the interval each step:

```python
def bisect(f, a, b, tol=1e-12):
    fa = f(a)
    assert fa * f(b) < 0, "need a sign change"
    while (b - a) / 2 > tol:
        m = (a + b) / 2
        if fa * f(m) <= 0:
            b = m
        else:
            a, fa = m, f(m)
    return (a + b) / 2

print(bisect(lambda x: x**2 - 2, 1, 2))   # 1.414213562373...
```

**Linear convergence**: error halves each step → ~3.3 steps per decimal digit → ~52 iterations for full double precision. Guaranteed. No derivative needed. The price: needs a sign change (misses double roots like x²) and only finds one root in the bracket.

### Newton's method — quadratic, when it works

Taylor-expand f near the guess and solve the *linear* model: x ← x − f(x)/f'(x).

```python
def newton(f, df, x, tol=1e-14, maxit=50):
    for _ in range(maxit):
        fx, dfx = f(x), df(x)
        if dfx == 0:
            raise ValueError("derivative vanished — no unique step")
        x -= fx / dfx
        if abs(fx) < tol:
            return x
    raise ValueError("no convergence")

print(newton(lambda x: x**2 - 2, lambda x: 2*x, 1.0))
```

**Quadratic convergence**: number of correct digits roughly *doubles* each iteration (ε_{n+1} ≈ C·εₙ²). Watch it happen:

```python
x = 1.0
for i in range(6):
    x -= (x*x - 2) / (2*x)
    print(i, x, abs(x - 2**0.5))   # errors: 1e-1, 2e-3, 5e-7, 2e-13, 0
```

Failure modes (worth memorizing):
- f'(x) = 0 or tiny → step explodes (division by ~0). Also flat roots: quadratic → linear convergence.
- Cycles/divergence: f(x) = x^(1/3) from any x ≠ 0 diverges; some functions cycle between two guesses forever.
- Needs a good initial guess — global behavior is chaotic (literally: Newton fractals).

### Secant method — Newton without the derivative

Replace f' with a finite difference of the last two iterates: x ← x − f(x)·(x−x_prev)/(f(x)−f(x_prev)). Convergence is **superlinear** (order ≈ 1.618, the golden ratio) — between bisection and Newton, and no df needed:

```python
def secant(f, x0, x1, tol=1e-14, maxit=50):
    for _ in range(maxit):
        f0, f1 = f(x0), f(x1)
        if f1 == f0:
            raise ValueError("zero slope between iterates")
        x0, x1 = x1, x1 - f1 * (x1 - x0) / (f1 - f0)
        if abs(x1 - x0) < tol:
            return x1
    raise ValueError("no convergence")
```

### Fixed-point iteration — the unifying view

Many iterations are x ← g(x). They converge if |g'(x*)| < 1 near the fixed point, with rate |g'| (smaller = faster; g'=0 → quadratic). Newton's method is exactly the choice g(x) = x − f/f' engineered so g'(x*) = 0. This is the lens that also explains why some numerical schemes converge and others don't.

## Where it's used

- **Physics engines / games**: ray–object intersection ("when does the ray hit the sphere") is root finding; collision event detection between timesteps is a bisection-style bracket search on the timeline.
- **Robotics & graphics**: inverse kinematics solvers are Newton iterations on joint angles — this is why IK fails near singular configurations (Jacobian ≈ singular = f' ≈ 0).
- **Finance**: implied volatility from option prices (invert Black–Scholes — no closed form, always a solver); bond yield-to-maturity from price.
- **ML**: trust-region and line-search optimizers solve 1-D root problems per step; the `brentq`-style hybrid (safe bracket + fast secant steps) is the template for robust 1-D solves everywhere.
- **Systems**: TCP congestion control and PID controllers implicitly solve for an equilibrium — convergence-rate intuition tells you why P-only controllers oscillate.

## Dry run by hand

**1.** Bisect √2 on [1, 2]: signs f(1)=−1, f(2)=+2.
- m=1.5: f=+0.25 → root in [1, 1.5]
- m=1.25: f=−0.4375 → root in [1.25, 1.5]
- m=1.375: f=−0.109 → root in [1.375, 1.5]
- m=1.4375: f=+0.066 → root in [1.375, 1.4375]

Four steps, interval shrunk 2→0.0625. Each step buys exactly one bit — feel why "linear convergence" means ~3.3 steps per decimal digit.

**2.** Newton on the same problem from x₀=1: x ← x − (x²−2)/2x = (x + 2/x)/2 (the ancient Babylonian method!).
- x₁ = (1 + 2)/2 = 1.5, err ≈ 8.6e-2
- x₂ = (1.5 + 1.333)/2 = 1.4167, err ≈ 2.5e-3
- x₃ = 1.4142157, err ≈ 6e-6
- x₄ = 1.4142135624, err ≈ 2e-12

Errors: 1e-1 → 1e-3 → 1e-6 → 1e-12. Count the digits doubling — that *is* quadratic convergence, by hand.

**3.** Conditioning feel: solve x² = 2 exactly, then x² = 2.001. Root moves by ≈ 0.001/(2·1.414) ≈ 3.5e-4 — the √ function at 2 is well-conditioned. Now x⁸ = 1 vs x⁸ = 1.01: root moves from 1 to 1.0012 — wait, try x⁸ = 2 vs x⁸ = 2·(1+1e-3): relative root move = 1e-3/8, tiny. But near a *flat* root (x−1)⁸ = 0 vs (x−1)⁸ = 1e-8: root jumps to 1.1. Same solver, wildly different sensitivity — κ belongs to the problem.

## Gotchas

- Newton converging to *a* root doesn't mean it's *your* root — check f(root) and sanity-check the value.
- Stopping criteria: |f(x)| small ≠ x accurate for flat functions; |step| small ≠ converged if the function is steep. Use both, scaled to the problem.
- Bisection requires a sign change — a tangent root (x²) is invisible to it.
- All solvers degrade gracefully to ~half precision near flat roots; that IS the conditioning, not a bug in your code.
- Library equivalent: `scipy.optimize.brentq` (bisection + secant hybrid — the production standard), `scipy.optimize.newton`.

## Exercises

**1.** Count bisection iterations to reach tol=1e-12 for √2 starting from [1, 2]. Verify it's ≈ 40. Then compute how many Newton iterations reach the same — count correct digits each step.

**2.** Find all three roots of f(x) = x³ − 2x − 5 in a way that's robust: scan a grid for sign changes, bisect each bracket. (One real root near 2.09 — why do the other two never show a sign change?)

**3.** Implement Newton for f(x) = x^(1/3) with df(x) = (1/3)x^(−2/3). Run from x0 = 1 and print the first 8 iterates. Explain the pattern analytically (compute g(x) = x − f/f' and simplify).

<details>
<summary><b>Solutions</b></summary>

**1.** `math.log2((2-1)/1e-12) ≈ 39.9` → 40 iterations. Newton reaches 1e-12 in 4–5 iterations from x0=1: errors go 2e-1 → 6e-3 → 2e-5 → 2e-10 → 0.

**2.**

```python
f = lambda x: x**3 - 2*x - 5
xs = [i/10 for i in range(-30, 31)]
for a, b in zip(xs, xs[1:]):
    if f(a) * f(b) < 0:
        print("root:", bisect(f, a, b))     # only 2.09455...
```

x³ − 2x − 5 has one real root and a complex pair — complex roots never produce a real sign change. (Discriminant or graphing confirms.)

**3.** g(x) = x − x^(1/3) / ((1/3)x^(−2/3)) = x − 3x = **−2x**. Each iterate flips sign and doubles in magnitude: 1, −2, 4, −8, … — divergence by construction. Newton assumes the linear model is locally valid; near an inflection-asymptote like x^(1/3) it overshoots catastrophically.

</details>

## Deep dives

- **Heath, *Scientific Computing: An Introductory Survey*** — ch. 1 (errors/conditioning) and ch. 5 (nonlinear equations); the friendliest rigorous text. Older editions circulate as PDFs in university courses.
- **Trefethen & Bau**, Lectures 12–14 — backward error and stability, the correct mental framework.
- Cleve Moler's *Numerical Computing with MATLAB* (free from MathWorks): https://www.mathworks.com/moler — chapters 1 and 4 cover this exact material with memorable examples; the math is language-agnostic.
- Newton fractals visualization (why initial guesses matter): 3Blue1Brown-adjacent writeups, e.g. https://en.wikipedia.org/wiki/Newton_fractal
