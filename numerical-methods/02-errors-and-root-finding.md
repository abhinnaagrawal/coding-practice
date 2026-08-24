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

**Linear convergence**: error halves each step $\implies \frac{|e_{k+1}|}{|e_k|} \approx \frac{1}{2}$. Each step buys $\log_{10}(2) \approx 0.301$ decimal digits ($\approx 3.3$ steps per decimal digit).

#### Example by hand: finding $\sqrt{2}$ via $f(x) = x^2 - 2 = 0$ on $[1, 2]$
- Initial: $a_0 = 1$ ($f(1) = -1$), $b_0 = 2$ ($f(2) = +2$). Sign change confirmed.
- **Iteration 1**:
  - Midpoint $m_0 = \frac{1 + 2}{2} = 1.5$.
  - Evaluate $f(1.5) = 1.5^2 - 2 = 2.25 - 2 = +0.25 > 0$.
  - Same sign as $b_0 \implies$ root is in $[1.0, 1.5]$. New interval width $= 0.5$.
- **Iteration 2**:
  - Midpoint $m_1 = \frac{1.0 + 1.5}{2} = 1.25$.
  - Evaluate $f(1.25) = 1.25^2 - 2 = 1.5625 - 2 = -0.4375 < 0$.
  - Same sign as $a_1 \implies$ root is in $[1.25, 1.5]$. New interval width $= 0.25$.
- **Iteration 3**:
  - Midpoint $m_2 = \frac{1.25 + 1.5}{2} = 1.375$.
  - Evaluate $f(1.375) = 1.890625 - 2 = -0.109375 < 0$.
  - Root is in $[1.375, 1.5]$. New interval width $= 0.125$.
- Guaranteed error bound after 3 steps is $\le \frac{2 - 1}{2^3} = 0.125$.

#### Where it's used
- **Collision detection in game engines / physics**: finding the exact moment of contact between moving continuous meshes within a discrete frame $\Delta t$.
- **Safe bracket guards**: outer wrapper for high-speed optimizers (e.g., Brent's method in `scipy.optimize.brentq`) to prevent divergence when Newton steps escape the bounding domain.

---

### Newton's method — quadratic, when it works

Taylor-expand f near the guess: $f(x + \Delta x) \approx f(x) + f'(x)\Delta x = 0 \implies \Delta x = -\frac{f(x)}{f'(x)}$.
Iteration update: $x_{k+1} = x_k - \frac{f(x_k)}{f'(x_k)}$.

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

**Quadratic convergence**: near a simple root, $|e_{k+1}| \approx C \cdot |e_k|^2$. The number of correct decimal digits **doubles** each iteration!

#### Example by hand: finding $\sqrt{5}$ via $f(x) = x^2 - 5 = 0$ ($f'(x) = 2x$)
Update formula: $x_{k+1} = x_k - \frac{x_k^2 - 5}{2x_k} = \frac{1}{2}\left(x_k + \frac{5}{x_k}\right)$.
Start with initial guess $x_0 = 2.0$:
- **Iteration 1**:
  - $x_1 = \frac{1}{2}\left(2.0 + \frac{5}{2.0}\right) = \frac{1}{2}(2.0 + 2.5) = \mathbf{2.25}$
  - Error: $|2.25 - 2.23606798| \approx 1.39 \times 10^{-2}$ ($\sim 2$ correct digits).
- **Iteration 2**:
  - $x_2 = \frac{1}{2}\left(2.25 + \frac{5}{2.25}\right) = \frac{1}{2}\left(\frac{9}{4} + \frac{20}{9}\right) = \frac{161}{72} \approx \mathbf{2.236111\dots}$
  - Error: $|2.236111 - 2.236068| \approx 4.3 \times 10^{-5}$ ($\sim 5$ correct digits — doubled!).
- **Iteration 3**:
  - $x_3 = \frac{1}{2}\left(\frac{161}{72} + \frac{360}{161}\right) = \frac{51841}{23184} \approx \mathbf{2.2360679779}$
  - Error: $\approx 4 \times 10^{-10}$ ($\sim 10$ correct digits — doubled again!).

#### Where it's used
- **Robotics Inverse Kinematics (IK)**: solving for target joint angles given end-effector cartesian coordinates via multidimensional Newton-Raphson on the kinematic Jacobian.
- **Implicit differential equation solvers**: Backward Euler and Radau solvers run Newton's method inside every timestep to solve algebraic constraints.

Failure modes (worth memorizing):
- f'(x) = 0 or tiny → step explodes (division by ~0). Also flat roots: quadratic → linear convergence.
- Cycles/divergence: f(x) = x^(1/3) from any x ≠ 0 diverges; some functions cycle between two guesses forever.
- Needs a good initial guess — global behavior is chaotic (literally: Newton fractals).

---

### Secant method — Newton without the derivative

Replace the analytical derivative $f'(x_k)$ with the finite-difference slope between the last two points:
$$f'(x_k) \approx \frac{f(x_k) - f(x_{k-1})}{x_k - x_{k-1}} \implies x_{k+1} = x_k - f(x_k)\frac{x_k - x_{k-1}}{f(x_k) - f(x_{k-1})}$$

Convergence is **superlinear** (order $\phi = \frac{1+\sqrt{5}}{2} \approx 1.618$) — each step multiplies the number of correct digits by $\approx 1.62$, without requiring derivative calculations.

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

#### Example by hand: root of $f(x) = x^2 - 2 = 0$
Start with $x_0 = 1.0$ ($f(x_0) = -1$) and $x_1 = 2.0$ ($f(x_1) = +2$).
- **Iteration 1**:
  - Slope $s = \frac{f(x_1) - f(x_0)}{x_1 - x_0} = \frac{2 - (-1)}{2 - 1} = 3$.
  - Next point $x_2 = x_1 - \frac{f(x_1)}{s} = 2.0 - \frac{2}{3} = \frac{4}{3} \approx \mathbf{1.3333}$.
  - $f(x_2) = (4/3)^2 - 2 = 16/9 - 18/9 = -2/9 \approx -0.2222$.
- **Iteration 2**:
  - Slope between $x_1=2.0$ ($f=2$) and $x_2=4/3$ ($f=-2/9$):
    $$s = \frac{-2/9 - 2}{4/3 - 2} = \frac{-20/9}{-2/3} = \frac{10}{3} \approx 3.3333$$
  - Next point $x_3 = \frac{4}{3} - \frac{-2/9}{10/3} = \frac{4}{3} + \frac{2}{30} = \frac{42}{30} = \mathbf{1.4000}$.
  - $f(x_3) = 1.4^2 - 2 = 1.96 - 2 = -0.04$ (rapidly approaching $\sqrt{2} \approx 1.4142$).

#### Where it's used
- **Financial mathematics (Implied Volatility / Yield-to-Maturity)**: Black-Scholes formula inversion has no simple closed-form derivative with respect to volatility; secant methods efficiently match market option prices.
- **Complex black-box simulation calibration**: matching physical simulation outputs to sensor readings where the simulator is an opaque executable.

---

### Fixed-point iteration — the unifying view

Rewrite $f(x) = 0$ as $x = g(x)$. The iteration is simply $x_{k+1} = g(x_k)$.
**Banach Fixed-Point Theorem**: the sequence converges to $x^*$ if $|g'(x^*)| < 1$ in a neighborhood of the root. The smaller $|g'(x^*)|$, the faster the convergence. If $g'(x^*) = 0$, convergence is at least quadratic.

#### Example by hand: solving $x = \cos(x)$ (the Dottie number)
- Check derivative: $g(x) = \cos(x) \implies g'(x) = -\sin(x)$.
- Near the root $x^* \approx 0.739$, $|g'(0.739)| = |\sin(0.739)| \approx 0.673 < 1 \implies$ guaranteed convergence!
- Start with $x_0 = 1.0$ (radians):
  - $x_1 = \cos(1.0) \approx \mathbf{0.5403}$
  - $x_2 = \cos(0.5403) \approx \mathbf{0.8576}$
  - $x_3 = \cos(0.8576) \approx \mathbf{0.6543}$
  - $x_4 = \cos(0.6543) \approx \mathbf{0.7935} \to \dots \to 0.739085$
- Spirals inward toward the fixed point with contractive factor $\approx 0.67$ per step.

#### Where it's used
- **Google PageRank**: computing stationary state probabilities $\mathbf{p} = \mathbf{M}\mathbf{p}$ via power iteration is literally linear fixed-point iteration.
- **Reinforcement Learning (Value Iteration)**: solving Bellman equations $V(s) = \max_a \sum P(s'|s,a)[R + \gamma V(s')]$ converges because the discount factor $\gamma < 1$ makes the Bellman backup operator a contraction mapping.

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
