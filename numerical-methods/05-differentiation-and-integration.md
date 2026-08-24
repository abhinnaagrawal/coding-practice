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

### Richardson extrapolation — free accuracy

If error = C·h² + O(h⁴) (centered diff), compute at h and h/2 and cancel the leading term: D ≈ (4·D(h/2) − D(h))/3 → error O(h⁴):

```python
def d_richardson(f, x, h):
    return (4*d_centered(f, x, h/2) - d_centered(f, x, h)) / 3

print(abs(d_richardson(math.sin, 1.0, 1e-3) - math.cos(1.0)))   # ~1e-13
```

Same trick upgrades trapezoid → Romberg integration below. The pattern "combine estimates at two scales to cancel the leading error term" is everywhere in numerical methods.

**Modern alternatives** (know they exist): automatic differentiation (exact derivatives of code via chain rule — what PyTorch/JAX do; not finite differences at all) and complex-step differentiation (f(x+ih)/imag — no cancellation, machine precision).

## Integration: area without antiderivatives

Approximate f by a low-degree polynomial on each subinterval and integrate *that*:

| Rule | Local model | Error (n panels) |
|------|-------------|------------------|
| Trapezoid | line | O(h²) = O(1/n²) |
| Simpson | parabola | O(h⁴) = O(1/n⁴) |
| Gauss-Legendre (k pts) | optimal polynomial | spectral — needs ~few points per wavelength |

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

The table's error orders are visible: doubling n divides trapezoid error by 4, Simpson by 16.

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

### Monte Carlo — when dimension defeats grids

Grid methods cost n^d points in d dimensions (curse of dimensionality). Monte Carlo: sample uniformly, average f × volume; error ~ σ/√N regardless of dimension.

```python
import random, math
random.seed(1)
N = 1_000_000
hits = sum(1 for _ in range(N) if random.random()**2 + random.random()**2 <= 1)
print(4 * hits / N)   # ≈ π (area of quarter unit circle × 4)
```

Slow (1/√N: 100× more samples for one more digit) but dimension-proof — that's why rendering, finance, and Bayesian inference run on it. Variance reduction (importance/stratified sampling) is how the constant factor gets tamed.

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
