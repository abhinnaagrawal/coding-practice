[← back to index](/numerical-methods/README.md)

# Day 7 — Numerical Optimization

Optimization = root-finding's sibling: minimize f(x) instead of solving f(x) = 0. (At a minimum, ∇f = 0 — so optimization *is* root-finding on the gradient.) This chapter reframes the ML optimizers you already know through the week's numerical lens.

## Gradient descent — Euler's method on the gradient flow

$$\theta_{k+1} = \theta_k - \eta \, \nabla f(\theta_k)$$

This is exactly forward Euler applied to the continuous trajectory $\theta'(t) = -\nabla f(\theta(t))$, with learning rate $\eta$ as the step size $h$.

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

For quadratics $f(x) = \frac{1}{2} x^T A x$, the gradient is $\nabla f = A x$. The update is:
$$x_{k+1} = x_k - \eta A x_k = (I - \eta A) x_k$$
Convergence requires $|1 - \eta \lambda_i| < 1 \implies \mathbf{\eta < \frac{2}{\lambda_{\max}(A)}}$.

### Example by hand: optimizing $f(x) = 2x^2$ ($f'(x) = 4x$) from $x_0 = 1$
Update: $x_{k+1} = x_k - \eta(4x_k) = (1 - 4\eta)x_k$.
- **Case 1: $\eta = 0.2$** ($1 - 4(0.2) = 0.2$):
  - $x_1 = 0.2(1) = \mathbf{0.2}$
  - $x_2 = 0.2(0.2) = \mathbf{0.04}$
  - $x_3 = 0.2(0.04) = \mathbf{0.008}$ (monotonically geometric decay).
- **Case 2: $\eta = 0.4$** ($1 - 4(0.4) = -0.6$):
  - $x_1 = -0.6(1) = \mathbf{-0.6}$
  - $x_2 = -0.6(-0.6) = \mathbf{+0.36}$
  - $x_3 = -0.6(0.36) = \mathbf{-0.216}$ (converges while oscillating across the minimum).
- **Case 3: $\eta = 0.5$** ($1 - 4(0.5) = -1.0$):
  - $x_1 = -1.0, x_2 = +1.0, x_3 = -1.0$ (stuck in eternal limit cycle!).
- **Case 4: $\eta = 0.55$** ($1 - 4(0.55) = -1.2$):
  - $x_1 = -1.2, x_2 = +1.44, x_3 = -1.728 \to \infty$ (explosive divergence!).

### Where it's used
- **Deep Learning Model Training (SGD)**: updating millions of weights across neural network layers per mini-batch.
- **Logistic Regression & Matrix Factorization**: estimating recommender system embeddings.

---

## Conditioning: why gradient descent crawls on valleys

A long thin valley (ill-conditioned Hessian: $\lambda_{\max} \gg \lambda_{\min}$) forces $\eta \le \frac{2}{\lambda_{\max}}$.
Along the flat direction, progress per step is governed by $(1 - \eta \lambda_{\min}) \approx 1 - \frac{2}{\kappa}$.
**Convergence rate $\approx 1 - \frac{2}{\kappa}$** — for $\kappa = 1000$, error shrinks by only $0.2\%$ per step!

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

---

## The second-order view: Newton & Quasi-Newton (BFGS)

Newton's optimization method approximates $f(\theta)$ with a local quadratic Taylor model:
$$\theta_{k+1} = \theta_k - H^{-1} \nabla f(\theta_k), \quad \text{where } H_{ij} = \frac{\partial^2 f}{\partial \theta_i \partial \theta_j}$$
Because the step multiplies by $H^{-1}$, curvature is neutralized across all axes ($\kappa \to 1$). Near the minimum, convergence is **quadratic**!

### Example by hand: Newton's method on $f(x) = x^4 - 2x^2$ ($f'(x) = 4x^3 - 4x, \quad f''(x) = 12x^2 - 4$)
Start from $x_0 = 1.5$ (seeking the local minimum at $x^* = 1.0$):
- **Iteration 1**:
  - Gradient $f'(1.5) = 4(3.375) - 4(1.5) = 13.5 - 6.0 = 7.5$.
  - Hessian $f''(1.5) = 12(2.25) - 4 = 27 - 4 = 23.0$.
  - Newton step $\Delta x = -\frac{7.5}{23.0} \approx -0.3261$.
  - $x_1 = 1.5 - 0.3261 = \mathbf{1.1739}$.
- **Iteration 2**:
  - $f'(1.1739) \approx 1.7725$, $f''(1.1739) \approx 12.536$.
  - $x_2 = 1.1739 - \frac{1.7725}{12.536} \approx \mathbf{1.0325}$.
- **Iteration 3**:
  - $x_3 \approx \mathbf{1.0015} \to x_4 \approx \mathbf{1.00000003}$ (rapid quadratic convergence!).

### Where it's used
- **Quasi-Newton (L-BFGS)**: the default engine in `scipy.optimize.minimize` for scientific calibration, climate models, and non-deep ML (e.g. CRFs).
- **Model Predictive Control (MPC) in autonomous driving**: solving nonlinear trajectory optimization problems in milliseconds via real-time Sequential Quadratic Programming (SQP).

---

## Momentum & Adaptive optimizers

- **Polyak Momentum (Heavy Ball)**:
  $$v_{k+1} = \mu v_k - \eta \nabla f(\theta_k), \quad \theta_{k+1} = \theta_k + v_{k+1}$$
  Dampens high-frequency oscillations across steep valley walls and accelerates along the shallow floor.
- **Adam (Adaptive Moment Estimation)**: divides step by running root-mean-square of recent gradients:
  $$\theta \leftarrow \theta - \frac{\eta}{\sqrt{v_t} + \epsilon} m_t$$
  Acts as a diagonal approximation to the inverse Hessian $H^{-1}$, rescaling ill-conditioned coordinates independently.

### Example by hand: one step of Momentum on valley $f(x, y) = x^2 + 10y^2$
Let $\eta = 0.05, \mu = 0.5$. Start at $(x_0, y_0) = (1, 1)$ with initial velocity $(v_x, v_y) = (0, 0)$.
- Gradients: $\nabla f = (2x, 20y) \implies \nabla f(1, 1) = (2, 20)$.
- Update velocity:
  - $v_x = 0.5(0) - 0.05(2) = \mathbf{-0.10}$
  - $v_y = 0.5(0) - 0.05(20) = \mathbf{-1.00}$
- Update position:
  - $x_1 = 1.0 + (-0.10) = \mathbf{0.90}$
  - $y_1 = 1.0 + (-1.00) = \mathbf{0.00}$
- Next step gradients at $(0.90, 0.00)$: $\nabla f = (1.8, 0.0)$.
  - $v_x = 0.5(-0.10) - 0.05(1.8) = -0.05 - 0.09 = \mathbf{-0.14}$ (accumulated speed along $x$!).
  - $v_y = 0.5(-1.00) - 0.05(0) = \mathbf{-0.50}$.
- Momentum doubled down on moving down the flat valley floor while damping vertical bounce.

---

## Regularized least squares — fixing conditioning by design

When $A^T A$ is near-singular (ill-conditioned), **Ridge Regression (Tikhonov Regularization)** solves:
$$\min \|A\mathbf{c} - \mathbf{y}\|_2^2 + \lambda \|\mathbf{c}\|_2^2 \implies (A^T A + \lambda I)\mathbf{c} = A^T \mathbf{y}$$
Adding $\lambda I$ shifts all eigenvalues: $\lambda_i(A^T A + \lambda I) = \lambda_i(A^T A) + \lambda$.
Condition number drops from $\frac{\lambda_{\max}}{\lambda_{\min}} \to \frac{\lambda_{\max} + \lambda}{\lambda_{\min} + \lambda}$.

### Example by hand: regularizing an ill-conditioned $2\times 2$ normal equation
Let $A^T A = \begin{pmatrix} 100 & 0 \\ 0 & 0.01 \end{pmatrix}$.
- Condition number $\kappa = 100 / 0.01 = \mathbf{10,000}$ (terribly ill-conditioned).
- Add regularization $\lambda = 1.0 \implies A^T A + 1.0 I = \begin{pmatrix} 101 & 0 \\ 0 & 1.01 \end{pmatrix}$.
- New condition number: $\kappa_{\text{ridge}} = \frac{101}{1.01} = \mathbf{100}$ ($100\times$ improvement in numerical stability!).

### Where it's used
- **Econometrics & Multicollinear regression**: preventing regression coefficients from exploding when two features (e.g., house square footage and number of rooms) are highly correlated.
- **Medical Imaging (CT / MRI reconstruction)**: solving underdetermined inverse Radon transforms where physical sensor measurements are fewer than voxel count.

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
