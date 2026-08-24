[← back to index](/numerical-methods/README.md)

# Day 3 — Interpolation & Approximation

## The two different problems

- **Interpolation**: find a function passing *exactly* through n given points. Assumes data is exact.
- **Approximation / fitting**: find a *simple* function close to many (possibly noisy) points. Assumes data has noise.

Confusing them is the classic mistake: interpolating noisy data amplifies the noise; fitting a low-degree model to exact high-complexity data throws away signal.

## Polynomial interpolation: unique, exact, and dangerous

Through n+1 points there is exactly one polynomial of degree ≤ n. Lagrange form makes it explicit:

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

### The Runge phenomenon — the day's key intuition

High-degree polynomial interpolation on **equally spaced** points oscillates violently near the interval ends, and the oscillation *grows* as you add points. Adding data makes it worse. Root cause: the interpolant is unique — you're not choosing a bad polynomial, the polynomial itself is ill-behaved; equal spacing is an ill-conditioned basis (the Vandermonde matrix has exponential condition number).

Fixes, in order of practicality:
1. **Don't use one high-degree polynomial** — use piecewise low-degree ones (splines, below).
2. If you must interpolate globally, use **Chebyshev nodes** — cluster points toward the ends: xᵢ = cos((2i+1)π/(2n+2)). This makes the problem well-conditioned and kills the oscillation.

```python
import math
n = 16
cheb_xs = [math.cos((2*i+1)*math.pi/(2*n+2)) for i in range(n+1)]
cheb_ys = [1/(1 + 25*x*x) for x in cheb_xs]
print(lagrange(cheb_xs, cheb_ys, 0.9), 1/(1 + 25*0.9**2))  # 0.047 ≈ 0.047 — fixed!
```

## Splines — the workhorse

Piecewise cubic with matching value/slope/curvature at joints ("cubic spline", C² continuous). This is what `scipy.interpolate.interp1d`/`CubicSpline` do and what "smooth curve through points" means in practice.

```python
# natural cubic spline solves a tridiagonal linear system — full code is chapter-4
# material; here the interface intuition:
from scipy.interpolate import CubicSpline      # pip install scipy, or read along
cs = CubicSpline(xs, ys, bc_type='natural')
print(cs(0.9))                                  # sane value, no oscillation
```

Rule of thumb: **interpolation degree ≤ 3, piecewise.** Beyond that you're fighting the Runge phenomenon.

## Least squares — fitting, not interpolating

With noisy data, minimize Σ (model(xᵢ) − yᵢ)². For a linear-in-parameters model y = c₀ + c₁x + c₂x², the minimum is the exact solution of the **normal equations** AᵀAc = Aᵀy where A is the "features" matrix.

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

## Connecting thread: it's all local polynomial models

Interpolation (Lagrange), root finding (Newton fits a line), integration (Simpson fits a parabola, day 5), ODE solvers (RK fits a 4th-order model, day 6) — the whole field is "model the function locally as a polynomial of degree k; the error is the k+1-th Taylor term you dropped." Keep this lens; it makes the next four days click.

## Where it's used

- **Fonts & vector graphics**: TrueType glyphs are quadratic Bézier splines; PostScript/OTF use cubics. "Smooth curve through control points" = day-3 spline, rendered millions of times per screen.
- **Image resizing**: bilinear/bicubic interpolation — bicubic is a local cubic spline in 2-D; the ringing you see on sharp edges is a 1-pixel Runge phenomenon.
- **CAD / animation**: keyframe easing and NURBS surfaces are piecewise-polynomial interpolation with C² continuity constraints.
- **Sensor & telemetry pipelines**: resampling uneven measurements onto a regular grid; least-squares denoising before feature extraction.
- **ML**: regression *is* least squares; feature scaling before fitting is conditioning repair (day 4); overfitting a high-degree polynomial to noisy data is the Runge/least-squares lesson in ML vocabulary.
- **Flight tables / embedded**: precomputed lookup tables + linear/cubic interpolation instead of computing sin/exp on hardware without an FPU.

## Dry run by hand

**1.** Lagrange through 2 points (1, 3) and (3, 7) — build it term by term:
- L₀(x) = (x−3)/(1−3) = −(x−3)/2, and L₀(1) = 1, L₀(3) = 0 — each basis polynomial is 1 at its own node, 0 at the others. That's the whole trick.
- p(x) = 3·L₀(x) + 7·L₁(x) = 3·(−(x−3)/2) + 7·(x−1)/2 = 2x + 1. Check: p(1)=3 ✓, p(3)=7 ✓.

**2.** Feel Runge: interpolate f(x) = 1/(1+25x²) at x = −1, 0, 1 (three points, parabola). f = 1/27, 1, 1/27 → p(x) = 1 − (26/27)x². Evaluate at x = 0.85: p = 0.305, but f(0.85) = 0.052 — the parabola already overshoots 6× at the edge. Adding equally spaced points forces the polynomial to match at more nodes *and oscillate harder between them*.

**3.** Least squares by hand, 3 points: (0, 1), (1, 2), (2, 2), fit a line y = c₀ + c₁x.
- n=3, Σx=3, Σy=5, Σx²=5, Σxy=6
- slope c₁ = (3·6 − 3·5)/(3·5 − 3²) = 3/6 = 0.5; intercept c₀ = (5 − 0.5·3)/3 = 7/6
- Line: y = 7/6 + x/2. Residuals: −1/6, +1/3, −1/6 — sum to zero (least squares always centers residuals). Squaring is why the middle outlier-ish point pulls harder than its size suggests.

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
