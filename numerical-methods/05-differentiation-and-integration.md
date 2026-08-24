[← back to index](/numerical-methods/README.md)

# Day 5 — Numerical Differentiation & Integration

## Differentiation: the step-size dilemma in its purest form

The derivative is a limit; the computer takes a finite difference:

$$f'(x) \approx \frac{f(x+h) - f(x)}{h} \quad \text{(forward diff, error } O(h)\text{)}$$

$$f'(x) \approx \frac{f(x+h) - f(x-h)}{2h} \quad \text{(centered diff, error } O(h^2)\text{)}$$

Smaller h shrinks the *truncation* error (Taylor term you dropped) but grows *roundoff* error: f(x+h)−f(x−h) is catastrophic cancellation (day 1) with relative error ~ε/h. Total error: C₁h² + C₂ε/h — a U-curve with a minimum.

```python
def d_centered(f, x, h):
    return (f(x + h) - f(x - h)) / (2 * h)

import math
for k in range(2, 17, 2):
    h = 10.0 ** -k
    err = abs(d_centered(math.sin, 1.0, h) - math.cos(1.0))
    print(f"h=1e-{k:2d}  err={err:.3e}")
# h=1e-02  err=9.001e-05      ← truncation dominates, err ~ h²
# h=1e-06  err=2.675e-12
# h=1e-10  err=1.669e-06      ← roundoff dominates — error GROWS again
# h=1e-16  err=5.402e-01      ← f(x+h)==f(x-h): garbage
```

Optimal h ≈ √ε ≈ 1e-8 for forward, ≈ ε^(1/3) ≈ 6e-6 for centered. **There is no "h → 0" on a computer.**

### Example by hand: differentiating $f(x) = x^3$ at $x = 2$ ($f'(2) = 12$)
Let step size $h = 0.1$:
1. **Forward difference**:
   $$D_{\text{fwd}} = \frac{f(2.1) - f(2.0)}{0.1} = \frac{2.1^3 - 2.0^3}{0.1} = \frac{9.261 - 8.000}{0.1} = \frac{1.261}{0.1} = \mathbf{12.61}$$
   Error $= |12.61 - 12.0| = 0.61 \approx O(h)$.
2. **Centered difference**:
   $$D_{\text{ctr}} = \frac{f(2.1) - f(1.9)}{2(0.1)} = \frac{9.261 - 6.859}{0.2} = \frac{2.402}{0.2} = \mathbf{12.01}$$
   Error $= |12.01 - 12.0| = 0.01 = h^2 \implies O(h^2)$.
   Notice: centered difference gave $60\times$ less error from the exact same step size $h=0.1$ because the leading $h$ Taylor terms cancelled symmetrically.

### Where it's used
- **Neural Network Gradient Checking**: validating custom CUDA backprop implementations by comparing analytical gradients against centered finite differences with $h = 10^{-5}$.
- **Numerical Jacobians in nonlinear optimization**: computing local slope matrices when explicit formulas for equations are unavailable.

---

### Richardson extrapolation — free accuracy

If error = C·h² + O(h⁴) (centered diff), compute at h and h/2 and cancel the leading term:
$$D_{\text{extrap}} = \frac{4 D(h/2) - D(h)}{3} \implies \text{Error } O(h^4)$$

```python
def d_richardson(f, x, h):
    return (4*d_centered(f, x, h/2) - d_centered(f, x, h)) / 3

print(abs(d_richardson(math.sin, 1.0, 1e-3) - math.cos(1.0)))   # ~1e-13
```

### Example by hand: Richardson extrapolation
Using the centered differences from our cubic $f(x) = x^3$ at $x=2$:
- At $h = 0.2$: $D(0.2) = \frac{2.2^3 - 1.8^3}{0.4} = \frac{10.648 - 5.832}{0.4} = \frac{4.816}{0.4} = 12.04$.
- At $h/2 = 0.1$: $D(0.1) = 12.01$.
- Combine them:
  $$D_{\text{extrap}} = \frac{4(12.01) - 12.04}{3} = \frac{48.04 - 12.04}{3} = \frac{36.00}{3} = \mathbf{12.0000\dots}$$
  The $O(h^2)$ error was cancelled completely, recovering the exact derivative!

### Where it's used
- **Romberg Integration**: systematically accelerating trapezoid sums to $O(h^4), O(h^6), O(h^8)$ quadrature rules.
- **Adaptive ODE step-size error estimators**: computing local truncation error without additional function evaluations.

---

## Integration: area without antiderivatives

Approximate f by a low-degree polynomial on each subinterval and integrate *that*:

| Rule | Local model | Formula on $[a, b]$ | Error (n panels) |
|------|-------------|---------------------|------------------|
| Trapezoid | Line | $\frac{h}{2}[f(a) + f(b)]$ | $O(h^2) = O(1/n^2)$ |
| Simpson | Parabola | $\frac{h}{3}[f(a) + 4f(\frac{a+b}{2}) + f(b)]$ | $O(h^4) = O(1/n^4)$ |
| Gauss-Legendre (k pts) | Optimal polynomial | $\sum w_i f(x_i)$ | Spectral / Exponential |

```python
def trapezoid(f, a, b, n):
    h = (b - a) / n
    return h * (0.5*f(a) + sum(f(a + i*h) for i in range(1, n)) + 0.5*f(b))

def simpson(f, a, b, n):              # n must be even
    h = (b - a) / n
    s = f(a) + f(b)
    s += 4 * sum(f(a + i*h) for i in range(1, n, 2))
    s += 2 * sum(f(a + i*h) for i in range(2, n, 2))
    return s * h / 3

import math
true = 2.0                             # ∫₀^π sin = 2
for n in (10, 100, 1000):
    print(n,
          abs(trapezoid(math.sin, 0, math.pi, n) - true),
          abs(simpson(math.sin, 0, math.pi, n) - true))
# n=10    trap 1.7e-2   simp 8.2e-5
# n=100   trap 1.6e-4   simp 8.2e-9    ← ×100 panels → trap /100, simp /10000
# n=1000  trap 1.6e-6   simp 8.1e-13
```

### Example by hand: integrating $\int_0^2 x^3 dx$ (exact value = $\left[\frac{x^4}{4}\right]_0^2 = 4.0$)
Use $n=2$ panels over $[0, 2] \implies h = 1.0$. Nodes are $x_0 = 0, x_1 = 1, x_2 = 2$.
Function values: $f(0) = 0, f(1) = 1, f(2) = 8$.
1. **Trapezoid Rule**:
   $$T = 1.0 \times \left(\frac{1}{2}(0) + 1 + \frac{1}{2}(8)\right) = 1.0 \times (0 + 1 + 4) = \mathbf{5.0} \quad (\text{Error} = 1.0)$$
2. **Simpson's Rule**:
   $$S = \frac{1.0}{3} \times (f(0) + 4 f(1) + f(2)) = \frac{1}{3} \times (0 + 4(1) + 8) = \frac{12}{3} = \mathbf{4.0} \quad (\text{Error} = \mathbf{0.0}!)$$
- Why is Simpson exact for a cubic $x^3$? Symmetry cancels the 3rd-order error term; Simpson integrates all polynomials up to degree 3 with zero truncation error!

### Where it's used
- **Machine Learning Evaluation (ROC-AUC / PR-AUC)**: calculating the Area Under the Curve for classification model metrics via trapezoidal integration over discrete threshold points.
- **Digital Signal Processing (PID Controllers)**: the integral term $I(t) = K_i \int_0^t e(\tau)d\tau$ accumulates error using trapezoidal integration each sampling period.

---

### Adaptive quadrature — spend points where the function is hard

Fixed n wastes effort on flat regions and undersamples sharp ones. Adaptive: integrate with two rules on each interval; if they disagree beyond tolerance, split the interval. That's what `scipy.integrate.quad` does:

```python
def adapt(f, a, b, tol):
    whole = simpson(f, a, b, 2)
    left  = simpson(f, a, (a+b)/2, 2)
    right = simpson(f, (a+b)/2, b, 2)
    if abs(left + right - whole) < 15 * tol:      # Richardson-flavored error estimate
        return left + right + (left + right - whole) / 15
    return adapt(f, a, (a+b)/2, tol/2) + adapt(f, (a+b)/2, b, tol/2)

f = lambda x: 1 / (1 + 100*(x - 0.3)**2)          # sharp spike at 0.3
print(adapt(f, 0, 1, 1e-8))                        # concentrates panels near the spike
```

### Where it's used
- **Boundary element methods & fluid dynamics**: calculating aerodynamic drag where pressure gradients spike along sharp wing edges but stay smooth elsewhere.
- **Black-Scholes option barrier pricing**: integrating probability densities with discontinuous pay-off triggers.

---

### Monte Carlo — when dimension defeats grids

Grid methods cost n^d points in d dimensions (curse of dimensionality: a 10-point grid in 20 dimensions requires $10^{20}$ points).
Monte Carlo samples uniformly at random: $I \approx \text{Volume} \times \frac{1}{N}\sum_{i=1}^N f(x_i)$.
Error is $O\left(\frac{\sigma}{\sqrt{N}}\right)$ **independent of dimension $d$**!

```python
import random, math
random.seed(1)
N = 1_000_000
hits = sum(1 for _ in range(N) if random.random()**2 + random.random()**2 <= 1)
print(4 * hits / N)   # ≈ π (area of quarter unit circle × 4)
```

### Example by hand: Monte Carlo estimating $\int_0^1 x^2 dx$ (exact = $1/3 \approx 0.3333$)
Take 4 random numbers: $x_1 = 0.2, x_2 = 0.4, x_3 = 0.6, x_4 = 0.8$.
- Evaluate $f(x) = x^2$:
  - $f(0.2) = 0.04$
  - $f(0.4) = 0.16$
  - $f(0.6) = 0.36$
  - $f(0.8) = 0.64$
- Sample mean: $\hat{I} = \frac{0.04 + 0.16 + 0.36 + 0.64}{4} = \frac{1.20}{4} = \mathbf{0.3000}$.
- Error $= |0.30 - 0.3333| = 0.0333$.
- To get 1 more digit of accuracy ($\div 10$ error), we need $100\times$ more samples ($N = 400$).

### Where it's used
- **Photorealistic 3D Rendering (Path Tracing)**: rendering scenes with global illumination in Pixar/Blender traces millions of light rays per pixel via Monte Carlo integration over hemisphere angles.
- **Quantitative Finance**: pricing multi-asset exotic basket options where payoff depends on 50+ correlated stock prices simultaneously.

## Gotchas

- Never differentiate noisy data with finite differences — it amplifies noise by 1/h. Smooth first (fit, then differentiate the fit).
- Simpson requires even n; off-by-one silently degrades it to trapezoid-like accuracy.
- Sharp features (spikes, near-discontinuities) defeat fixed-grid rules — adaptive or split the domain at the feature.
- Infinite intervals: transform (∫₀^∞ → ∫₀¹ via t = 1/(1+x)) or truncate carefully; don't just "use a big b."
- Monte Carlo results need error bars (σ/√N) — a bare number without an estimate of its own error is numerology.

## Exercises

**1.** Reproduce the U-curve: for f(x) = eˣ at x=1, compute forward-difference error for h = 10⁻ᵏ, k = 1..16, and find the k of minimum error. Verify it sits near √ε.

**2.** Compute ∫₀¹ √(1−x²) dx (= π/4) with trapezoid at n = 10, 100, 1000. The convergence is *slower* than O(h²) — why? (Look at the integrand's derivative at x=1.)

**3.** Estimate ∫₀¹ 4/(1+x²) dx (= π) by Monte Carlo with N = 10⁴, 10⁶. Print the estimate ± σ/√N each time.

<details>
<summary><b>Solutions</b></summary>

**1.** Minimum near k = 8 (h ≈ 1e-8 = √ε); error ~h before, ~ε/h after.

**2.** √(1−x²) has an infinite derivative at x=1 — the Taylor-based error theory assumes bounded derivatives. Singular derivatives degrade the order (empirically ~O(h^1.5)). Splitting the interval or a substitution (x = sin θ) restores it.

**3.**

```python
import random, math
for N in (10**4, 10**6):
    vals = [4/(1 + random.random()**2) for _ in range(N)]
    mean = sum(vals)/N
    var = sum((v-mean)**2 for v in vals)/(N-1)
    print(N, mean, "±", math.sqrt(var/N))   # error bar shrinks by ×10 per ×100 samples
```

</details>

## Deep dives

- Heath, *Scientific Computing*, ch. 8 — differentiation & quadrature, the right depth for this chapter.
- **Nick Trefethen's 6-lecture "Numerical Methods" series** (Oxford, free on YouTube) — quadrature lectures are superb.
- AutoDiff: *"Automatic Differentiation in Machine Learning: a Survey"* (Baydin et al., free on arXiv: https://arxiv.org/abs/1502.05767) — why modern ML doesn't finite-difference.
- Monte Carlo intuition: 3Blue1Brown-adjacent — "But what is the Central Limit Theorem?" for why error bars shrink as 1/√N.
- QUADPACK (the Fortran behind `scipy.integrate.quad`) — original paper freely available; the adaptive strategy above is its essence.
