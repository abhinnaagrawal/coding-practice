[← back to index](/numerical-methods/README.md)

# Day 3 — Interpolation & Approximation

## The two different problems

- **Interpolation**: find a function passing *exactly* through n given points. Assumes data is exact.
- **Approximation / fitting**: find a *simple* function close to many (possibly noisy) points. Assumes data has noise.

Confusing them is the classic mistake: interpolating noisy data amplifies the noise; fitting a low-degree model to exact high-complexity data throws away signal.

## Polynomial interpolation: unique, exact, and dangerous

Through n+1 points there is exactly one polynomial of degree ≤ n. Lagrange form makes it explicit:
$$P(x) = \sum_{i=0}^n y_i L_i(x), \quad \text{where } L_i(x) = \prod_{j \ne i} \frac{x - x_j}{x_i - x_j}$$

```python
def lagrange(xs, ys, x):
    total = 0.0
    n = len(xs)
    for i in range(n):
        term = ys[i]
        for j in range(n):
            if i != j:
                term *= (x - xs[j]) / (xs[i] - xs[j])
        total += term
    return total

# interpolate f(x) = 1/(1+25x²) — the Runge function — on 7 evenly spaced points
xs = [-1 + 2*i/6 for i in range(7)]
ys = [1/(1 + 25*x*x) for x in xs]
print(lagrange(xs, ys, 0.9), 1/(1 + 25*0.9**2))   # -0.23... vs 0.047 — WAY off!
```

### Example by hand: fitting a parabola through 3 points
Given $(x_0, y_0) = (0, 1)$, $(x_1, y_1) = (1, 3)$, $(x_2, y_2) = (2, 7)$:
1. **Basis $L_0(x)$** (must equal 1 at $x_0=0$, and 0 at $x=1, 2$):
   $$L_0(x) = \frac{(x - 1)(x - 2)}{(0 - 1)(0 - 2)} = \frac{x^2 - 3x + 2}{2}$$
2. **Basis $L_1(x)$** (must equal 1 at $x_1=1$, and 0 at $x=0, 2$):
   $$L_1(x) = \frac{(x - 0)(x - 2)}{(1 - 0)(1 - 2)} = \frac{x(x - 2)}{-1} = -x^2 + 2x$$
3. **Basis $L_2(x)$** (must equal 1 at $x_2=2$, and 0 at $x=0, 1$):
   $$L_2(x) = \frac{(x - 0)(x - 1)}{(2 - 0)(2 - 1)} = \frac{x(x - 1)}{2} = \frac{x^2 - x}{2}$$
4. **Assemble polynomial $P(x) = 1\cdot L_0(x) + 3\cdot L_1(x) + 7\cdot L_2(x)$**:
   $$P(x) = \left(\frac{1}{2} - 3 + \frac{7}{2}\right)x^2 + \left(-\frac{3}{2} + 6 - \frac{7}{2}\right)x + 1 = \mathbf{x^2 + x + 1}$$
   Check: $P(0) = 1$, $P(1) = 1+1+1=3$, $P(2) = 4+2+1=7$. Exact match!

### Where it's used
- **Shamir's Secret Sharing (Cryptography)**: a secret is encoded as the constant term $P(0)$ of an unknown degree-$k$ polynomial; any $k+1$ parties combine their shares via Lagrange interpolation to reconstruct the secret.
- **Deriving numerical quadrature and finite difference weights**: Simpson's rule and higher-order stencils are derived by integrating Lagrange basis polynomials analytically.

---

### The Runge phenomenon & Chebyshev nodes — the day's key intuition

High-degree polynomial interpolation on **equally spaced** points oscillates violently near the interval ends, and the oscillation *grows* as you add points. Adding data makes it worse. Root cause: the interpolant is unique — you're not choosing a bad polynomial, the polynomial itself is ill-behaved; equal spacing is an ill-conditioned basis (the Vandermonde matrix has exponential condition number).

Fixes, in order of practicality:
1. **Don't use one high-degree polynomial** — use piecewise low-degree ones (splines, below).
2. If you must interpolate globally, use **Chebyshev nodes** — cluster points toward the ends:
   $$x_k = \cos\left(\frac{2k+1}{2n+2}\pi\right), \quad k = 0, \dots, n$$
   This clusters nodes near $\pm 1$ and guarantees near-optimal, non-oscillating minimax polynomial approximation.

```python
import math
n = 16
cheb_xs = [math.cos((2*i+1)*math.pi/(2*n+2)) for i in range(n+1)]
cheb_ys = [1/(1 + 25*x*x) for x in cheb_xs]
print(lagrange(cheb_xs, cheb_ys, 0.9), 1/(1 + 25*0.9**2))  # 0.047 ≈ 0.047 — fixed!
```

#### Example by hand: computing 3 Chebyshev nodes on $[-1, 1]$ ($n=2$)
Formula: $x_k = \cos\left(\frac{2k+1}{6}\pi\right)$ for $k=0, 1, 2$:
- $k=0 \implies x_0 = \cos(\pi/6) = \frac{\sqrt{3}}{2} \approx +\mathbf{0.866}$
- $k=1 \implies x_1 = \cos(3\pi/6) = \cos(\pi/2) = \mathbf{0.0}$
- $k=2 \implies x_2 = \cos(5\pi/6) = -\frac{\sqrt{3}}{2} \approx -\mathbf{0.866}$
- Notice: the points are not evenly spaced ($0.866 \to 0 \to -0.866$). They are bunched closer to the boundaries $\pm 1$, suppressing the boundary divergence term $\prod (x - x_j)$ exactly where Runge oscillation spikes.

#### Where it's used
- **Libm standard math library functions**: implementations of $\sin(x), \exp(x), \text{erf}(x)$ in standard C/C++ libraries evaluate Chebyshev-derived polynomial approximations (Remez algorithm) across sub-intervals.
- **Spectral PDE solvers**: fluid dynamics simulations on geometries (like airflow over airplane wings) use Chebyshev grids to avoid numerical boundary instability.

---

## Splines — the workhorse

Piecewise cubic with matching value/slope/curvature at joints ("cubic spline", $C^2$ continuous).
On each subinterval $[x_i, x_{i+1}]$, define a cubic $S_i(x) = a_i + b_i(x-x_i) + c_i(x-x_i)^2 + d_i(x-x_i)^3$.
At every internal joint $x_i$:
- $S_{i-1}(x_i) = S_i(x_i) = y_i$ (continuous value)
- $S_{i-1}'(x_i) = S_i'(x_i)$ (continuous slope)
- $S_{i-1}''(x_i) = S_i''(x_i)$ (continuous curvature)

Solving for all coefficients reduces to a simple $O(n)$ tridiagonal linear system.

```python
# natural cubic spline solves a tridiagonal linear system — full code is chapter-4
# material; here the interface intuition:
from scipy.interpolate import CubicSpline      # pip install scipy, or read along
cs = CubicSpline(xs, ys, bc_type='natural')
print(cs(0.9))                                  # sane value, no oscillation
```

Rule of thumb: **interpolation degree ≤ 3, piecewise.** Beyond that you're fighting the Runge phenomenon.

#### Example by hand: cubic spline continuity constraints
Suppose we have 3 points: $(0, 0), (1, 1), (2, 0)$ and two cubic segments $S_0(x)$ on $[0,1]$ and $S_1(x)$ on $[1,2]$.
- Segment 0: $S_0(x) = a_0 + b_0 x + c_0 x^2 + d_0 x^3$
- Segment 1: $S_1(x) = a_1 + b_1(x-1) + c_1(x-1)^2 + d_1(x-1)^3$
- Value conditions:
  - $S_0(0) = 0 \implies a_0 = 0$
  - $S_0(1) = 1 \implies b_0 + c_0 + d_0 = 1$
  - $S_1(1) = 1 \implies a_1 = 1$
  - $S_1(2) = 0 \implies 1 + b_1 + c_1 + d_1 = 0$
- Derivatives matching at joint $x=1$:
  - $S_0'(1) = S_1'(1) \implies b_0 + 2c_0 + 3d_0 = b_1$
  - $S_0''(1) = S_1''(1) \implies 2c_0 + 6d_0 = 2c_1$
- Natural boundary conditions set zero curvature at endpoints: $S_0''(0) = 0 \implies c_0 = 0$; $S_1''(2) = 0 \implies 2c_1 + 6d_1 = 0$.
- Solving this small system yields smooth, non-oscillating curves.

#### Where it's used
- **Computer Typography (TrueType / PostScript fonts)**: vector font outlines use quadratic/cubic Bézier splines to scale smoothly from 6pt text to 4K displays.
- **Robotics path planning**: generating smooth trajectories for industrial robot arms where continuous first and second derivatives ($C^2$) prevent motor torque shocks.
- **Computer animation & easing curves**: keyframe animation in CSS, After Effects, and Blender.

---

## Least squares — fitting, not interpolating

With noisy data, minimize $\sum_{i=1}^n (y_i - (c_0 + c_1 x_i))^2$.
In matrix form: minimize $\|A\mathbf{c} - \mathbf{y}\|_2^2$, where $A = \begin{pmatrix} 1 & x_1 \\ \vdots & \vdots \\ 1 & x_n \end{pmatrix}$.
Setting gradient to zero gives the **Normal Equations**:
$$A^T A \mathbf{c} = A^T \mathbf{y}$$

```python
import random
random.seed(7)
xs = [i/10 for i in range(51)]
ys = [2 + 3*x + random.gauss(0, 0.5) for x in xs]   # true line + noise

def least_squares_line(xs, ys):
    n = len(xs)
    sx, sy = sum(xs), sum(ys)
    sxx = sum(x*x for x in xs)
    sxy = sum(x*y for x, y in zip(xs, ys))
    # solve the 2x2 normal equations by Cramer's rule
    det = n*sxx - sx*sx
    c1 = (n*sxy - sx*sy) / det        # slope
    c0 = (sy - c1*sx) / n             # intercept
    return c0, c1

print(least_squares_line(xs, ys))     # ≈ (2.0, 3.0) — recovers the true line through noise
```

Why squared error? It makes the problem *linear* (normal equations), it penalizes outliers quadratically (a feature or a bug, depending on the data), and under Gaussian noise it's the maximum-likelihood estimator (Gauss-Markov theorem: best linear unbiased).

#### Example by hand: fitting a line to 3 noisy points
Data points: $(0, 1)$, $(1, 2)$, $(2, 4)$.
1. Matrix formulation:
   $$A = \begin{pmatrix} 1 & 0 \\ 1 & 1 \\ 1 & 2 \end{pmatrix}, \quad \mathbf{y} = \begin{pmatrix} 1 \\ 2 \\ 4 \end{pmatrix}$$
2. Compute $A^T A$:
   $$A^T A = \begin{pmatrix} 1 & 1 & 1 \\ 0 & 1 & 2 \end{pmatrix} \begin{pmatrix} 1 & 0 \\ 1 & 1 \\ 1 & 2 \end{pmatrix} = \begin{pmatrix} 3 & 3 \\ 3 & 5 \end{pmatrix}$$
3. Compute $A^T \mathbf{y}$:
   $$A^T \mathbf{y} = \begin{pmatrix} 1 & 1 & 1 \\ 0 & 1 & 2 \end{pmatrix} \begin{pmatrix} 1 \\ 2 \\ 4 \end{pmatrix} = \begin{pmatrix} 1+2+4 \\ 0+2+8 \end{pmatrix} = \begin{pmatrix} 7 \\ 10 \end{pmatrix}$$
4. Solve $\begin{pmatrix} 3 & 3 \\ 3 & 5 \end{pmatrix} \begin{pmatrix} c_0 \\ c_1 \end{pmatrix} = \begin{pmatrix} 7 \\ 10 \end{pmatrix}$:
   - Multiply row 1 by 1: $3c_0 + 3c_1 = 7 \implies c_0 + c_1 = 7/3$.
   - Row 2: $3c_0 + 5c_1 = 10 \implies 3(7/3 - c_1) + 5c_1 = 10 \implies 7 + 2c_1 = 10 \implies c_1 = \mathbf{1.5}$.
   - $c_0 = 7/3 - 1.5 = 7/3 - 3/2 = \frac{14 - 9}{6} = \frac{5}{6} \approx \mathbf{0.833}$.
- Best fit line: $\hat{y} = 0.833 + 1.5x$.

#### Where it's used
- **Machine Learning (Linear / Ridge Regression)**: fitting linear weights to feature matrices to predict continuous target variables.
- **Sensor drift calibration**: calibrating IMU accelerometers and temperature sensors by fitting linear conversion factors against ground-truth readings.
- **Signal processing & econometrics**: trend line extraction in noisy stock prices and time-series telemetry.

---

## Connecting thread: it's all local polynomial models

Interpolation (Lagrange), root finding (Newton fits a line), integration (Simpson fits a parabola, day 5), ODE solvers (RK fits a 4th-order model, day 6) — the whole field is "model the function locally as a polynomial of degree k; the error is the k+1-th Taylor term you dropped." Keep this lens; it makes the next four days click.

## Gotchas

- Extrapolation (evaluating outside the data range) with polynomials diverges fast — never trust it.
- Interpolating with degree ≥ ~10 on equally spaced points: expect garbage at the ends.
- Least squares is sensitive to outliers; one bad row can tilt the whole fit (robust alternatives: RANSAC, Huber loss).
- Normal equations square the condition number — for ill-conditioned fits use QR/SVD-based solvers (`numpy.linalg.lstsq` does this; chapter 4).
- Spline boundary conditions matter: "natural" (zero curvature at ends) is a modeling assumption that distorts the fit near boundaries.

## Exercises

**1.** Interpolate exp(x) at 5 equally spaced points in [−1, 1] with `lagrange`, and evaluate at 100 points to find the max error. Repeat with 9 and 13 points. Does error shrink everywhere, or grow at the ends?

**2.** Repeat exercise 1 with Chebyshev nodes at n = 13. Compare max error to the equally-spaced case.

**3.** Fit a line by least squares to `ys = [3*x + random.gauss(0, 3) for ...]` (heavy noise), then add one corrupted point `(0.1, 100)`. Observe the slope change. This is why robust regression exists.

<details>
<summary><b>Solutions</b></summary>

**1.** For exp (analytic, smooth), error shrinks at first but end-oscillation appears as degree grows — Runge is mild here compared to 1/(1+25x²) because exp has no nearby complex singularities, but the pattern is the same. Run it:

```python
import math
for n in (4, 8, 12):
    xs = [-1 + 2*i/n for i in range(n+1)]
    ys = [math.exp(x) for x in xs]
    err = max(abs(lagrange(xs, ys, -1 + 2*k/200) - math.exp(-1 + 2*k/200))
              for k in range(201))
    print(n, err)        # watch the last-decimal behavior as n grows
```

**2.** Same code with `cheb_xs` — max error drops by orders of magnitude and is uniform across the interval (near-minimax property).

**3.** Slope tilts noticeably (often 3.0 → 4+). One point with leverage (extreme x) and large residual dominates squared error. `scipy.stats.theilslopes` or RANSAC resist this.

</details>

## Deep dives

- **Trefethen, *Approximation Theory and Approximation Practice*** — the modern reference on why Chebyshev points are the right default; free chapter samples at https://www.chebfun.org/ATAP/ (MATLAB-flavored but the theory transfers).
- Heath, *Scientific Computing*, ch. 7 — interpolation with full error analysis.
- Runge phenomenon animated: https://en.wikipedia.org/wiki/Runge%27s_phenomenon
- 3Blue1Brown has no dedicated interpolation video, but his *"But what is a Fourier series"* is the right next mental model for global (non-polynomial) approximation: https://www.youtube.com/watch?v=r6sGWTCMz2k
