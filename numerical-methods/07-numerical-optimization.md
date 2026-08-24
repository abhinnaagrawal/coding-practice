[← back to index](/numerical-methods/README.md)

# Day 7 — Numerical Optimization

Optimization = root-finding's sibling: minimize f(x) instead of solving f(x) = 0. (At a minimum, ∇f = 0 — so optimization *is* root-finding on the gradient.) This chapter reframes the ML optimizers you already know through the week's numerical lens.

## Gradient descent — Euler's method on the gradient flow

$$\theta \leftarrow \theta - \eta \, \nabla f(\theta)$$

This is exactly forward Euler applied to the ODE θ' = −∇f(θ), with learning rate η as the step size h. Every day-6 lesson transfers: too-large η → instability/divergence; stiff (ill-conditioned) f → tiny η forced on you.

```python
import math, random
random.seed(0)

def gd(grad, theta0, eta, n):
    theta = theta0
    for _ in range(n):
        theta = theta - eta * grad(theta)
    return theta

# f(x) = x²  (grad = 2x), min at 0
for eta in (0.1, 0.9, 1.0, 1.01):
    print(eta, gd(lambda x: 2*x, 5.0, eta, 30))
# 0.1   → 0.011     converging, |1-2η|=0.8 per step
# 0.9   → 0.0..     fast, factor 0.8→ wait |1-1.8|=0.8 oscillating converge
# 1.0   → 5.0       STUCK: factor |1-2| = 1, oscillates forever
# 1.01  → huge      DIVERGES: factor 1.02 > 1 — the Euler stability boundary!
```

For quadratics f = ½xᵀAx the stability boundary is η < 2/λ_max(A) — the exact same |1 + hλ| ≤ 1 condition as Euler. Learning-rate schedules, warmups, and divergence spikes in ML training are this inequality wearing a trenchcoat.

## Conditioning: why gradient descent crawls on valleys

A long thin valley (ill-conditioned Hessian: λ_max/λ_min big) forces η ≤ 2/λ_max, but progress along the shallow direction then moves at rate ~(1 − η·λ_min) ≈ 1 − λ_min/λ_max = 1 − 1/κ. **Convergence rate ≈ condition number of the Hessian.** Zig-zagging across the valley is the visible symptom.

```python
# f(x,y) = x² + 100y² — valley along x; κ = 100
def valley_grad(t):
    x, y = t
    return (2*x, 200*y)
def gd2(grad, t0, eta, n):
    x, y = t0
    for _ in range(n):
        gx, gy = grad((x, y))
        x, y = x - eta*gx, y - eta*gy
    return x, y
print(gd2(valley_grad, (1.0, 1.0), 0.009, 200))   # y died fast, x still ~0.15 — crawling
```

## The fixes, as numerical methods

- **Newton in n-D**: θ ← θ − H⁻¹∇f (H = Hessian). Second-order — rescales each direction by its curvature, κ disappears, quadratic convergence near the optimum. Cost: Hessian O(n²) storage, O(n³) solve — impossible for 10⁹ params.
- **Quasi-Newton (BFGS, L-BFGS)**: build H⁻¹ approximately from gradient history — day-3's secant idea in n-D. The standard for medium-size problems (`scipy.optimize.minimize(method='L-BFGS-B')`).
- **Momentum**: heavy ball — θ accumulates velocity, dampens the zig (cross-valley), amplifies the zag-free direction. Polyak/Nesterov are second-order ODE discretizations of damped descent.
- **Adam**: per-coordinate adaptive learning rate ≈ diagonal approximation of curvature (a diagonal quasi-Newton, roughly). Works because the diagonal captures most of the conditioning in practice.

## Linear least squares — the optimization you can solve exactly

min ‖Ax − b‖₂: convex, closed-form via the gradient = 0 → normal equations AᵀAx = Aᵀb — but solve it with QR/SVD (`lstsq`), never by inverting AᵀA (day 4: κ gets squared). Every ML "closed-form" trick (ridge regression = least squares + λI regularization, which also *improves conditioning*: κ((AᵀA + λI)) < κ(AᵀA)) is this problem in costume.

```python
import numpy as np
# ridge: adds λI — watch the condition number drop
A = np.random.default_rng(0).normal(size=(50, 3))
A[:, 2] = A[:, 0] + 1e-8 * np.random.default_rng(1).normal(size=50)   # near-collinear
AtA = A.T @ A
print(np.linalg.cond(AtA))                 # huge
print(np.linalg.cond(AtA + 1e-4*np.eye(3)))  # ≈ 1e4·smaller — regularization as conditioning
```

## The week's threads converge

- Step size dilemma (days 1, 5, 6) → learning-rate tuning.
- Conditioning (days 2, 4) → why valleys are slow, why preconditioning/normalizing features helps.
- Local polynomial models (day 3) → gradient = linear model, Newton = quadratic model, trust regions = "model valid only within radius r."
- Root finding (day 2) → Newton for optimization is Newton for ∇f = 0.

## Where it's used

- **ML training**: SGD/momentum/Adam are this chapter; learning-rate finders, warmup, and loss-spike divergence are the Euler stability boundary in production.
- **Logistics & ops research**: linear programming (simplex/interior point) routes trucks and schedules crews — interior point methods are Newton + barrier functions.
- **Finance**: portfolio optimization (Markowitz = quadratic program); calibration of pricing models to market data = nonlinear least squares.
- **Engineering design**: shape/topology optimization — every airfoil and antenna is a constrained optimization with PDE constraints (adjoint methods = cheap gradients).
- **Robotics**: trajectory optimization and MPC solve a constrained QP or NLP every control cycle (milliseconds), warm-started from the previous solution — Newton with a great initial guess, exactly as day 2 prescribed.
- **Statistics**: maximum likelihood estimation = optimization; logistic regression's IRLS algorithm is literally Newton's method on the log-likelihood.

## Dry run by hand

**1.** GD on f(x) = x² from x₀ = 1, η = 0.25: update x ← x − 0.25·2x = 0.5x. Steps: 1 → 0.5 → 0.25 → 0.125 → … Geometric: error halves per step. Now η = 0.75: x ← x − 1.5x = −0.5x: 1 → −0.5 → 0.25 → −0.125 — converges while *oscillating across the minimum*. η = 1: x ← −x — bounces forever. η = 1.1: explodes. The entire learning-rate intuition in one line of arithmetic: |1 − 2η| vs 1.

**2.** Newton by hand on f(x) = x² − 2 as optimization (minimize, not root-find — same thing since f' = 0 at the min of ½(x²−2)²... simpler: minimize f(x) = x², Newton: x ← x − f'/f'' = x − 2x/2 = **0** — one step, from anywhere). Quadratic problems are solved exactly in one Newton step; everything else is measured against that ideal. That's why second-order methods are the gold standard and why κ (non-quadratic-ness, direction-dependent curvature) is the enemy.

**3.** The valley zigzag by feel: f(x,y) = x² + 100y² from (1, 1) with η = 0.01:
- grad = (2, 200) → step (−0.02, −2) → lands at (0.98, −1): crossed the valley and overshot y by a mile.
- next grad = (1.96, −200) → (0.96, +1) — crossed back. x creeps 0.02/iteration while y ping-pongs ±1.
- 50 steps later: x ≈ 0.37, still far from 0 — while y has visited ±1 dozens of times. You have *seen* κ = 100 throttle convergence. Momentum's fix, felt: the y-gradients alternate sign and cancel in the velocity; the x-gradients reinforce. Directional averaging, nothing magic.

## Gotchas

- Loss decreasing ≠ converged to a minimum — saddle points have ∇f ≈ 0 too (Hessian indefinite). In high-D, saddles vastly outnumber local minima.
- Gradient checking: verify hand-written gradients against centered finite differences at h ≈ 1e-5 (day 5!) before trusting an optimizer on them.
- Normalizing input features *is* preconditioning — unscaled features = ill-conditioned Hessian = slow zigzag.
- Line search vs fixed η: production optimizers pick η per step (Armijo/backtracking) — a 1-D root-find on the sufficient-decrease condition.
- Non-convex: initialization decides which basin you land in. Reproducibility requires seeding; results vary across seeds — report distributions, not single runs.

## Exercises

**1.** On f(x) = x⁴ (flat minimum, Hessian → 0), run gradient descent with η = 0.1 from x=1 for 1000 steps and measure |x|. Why is convergence sublinear here (compare with the quadratic case's geometric rate)?

**2.** Implement Newton's method for f(x) = x⁴ using f'' = 12x²: x ← x − f'/f''. Show each step just multiplies the error by 2/3 — second-order method, first-order behavior. What's the cause? (Hint: what's f''(0)?)

**3.** For the valley f = x² + 100y², run gradient descent with momentum: v ← μv − η∇f; θ ← θ + v, with η=0.009, μ=0.9. Compare iterations-to-1e-4 against plain GD.

<details>
<summary><b>Solutions</b></summary>

**1.** Error decays like n^(−1/2)-ish (each step multiplies by (1 − 6ηx²) with x shrinking) — polynomial, not geometric. The Hessian vanishes at the optimum: the problem's local conditioning worsens as you approach it. Flat minima are intrinsically slow for first-order methods.

**2.** x ← x − 4x³/12x² = x − x/3 = (2/3)x. Exactly linear convergence. Newton's quadratic rate requires a non-singular Hessian at the optimum; here f''(0) = 0, so the assumption fails — degenerate minimum.

**3.** Momentum converges in roughly 1/(1−√μ)-style fewer steps along the shallow direction (typically ~3–5× fewer iterations here) and damps the cross-valley oscillation. Print both trajectories to see the zigzag vanish.

</details>

## Deep dives

- **Nocedal & Wright, *Numerical Optimization*** — the standard text; ch. 1–3 (line search, trust region) and 6 (quasi-Newton). Springer preview + widely used course notes.
- **Boyd & Vandenberghe, *Convex Optimization*** — free at https://web.stanford.edu/~boyd/cvxbook/ — ch. 9 (unconstrained minimization) connects conditioning to convergence rates rigorously.
- 3Blue1Brown — gradient descent video in the neural-network series for visuals: https://www.youtube.com/watch?v=IHZwWFHWa-w
- *"Why Momentum Really Works"* — Distill.pub (https://distill.pub/2017/momentum/) — the best single read on momentum + conditioning, interactive.
- Sebastian Ruder's optimization overview: https://ruder.io/optimizing-gradient-descent/ — the ML-side map (SGD → Adam).

---

## Where to go from here

- The sibling skill: **numerical PDEs** (finite differences on grids) — Heath ch. 11 as the entry point.
- Practitioner's reference on your shelf: *Numerical Recipes* (opinionated, practical) alongside Trefethen & Bau (rigorous).
- Practice: reimplement `scipy.optimize.minimize`'s BFGS path on Rosenbrock's banana function — the field's favorite torture test: f(x,y) = (1−x)² + 100(y−x²)².
