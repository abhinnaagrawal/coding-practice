[← back to index](/numerical-methods/README.md)

# Day 6 — ODEs: Simulating Dynamics

Problem: y'(t) = f(t, y), y(t₀) = y₀ — given the slope field and a starting point, trace the trajectory. This is physics sims, control systems, epidemic models, and gradient flows.

## Euler — the prototype (and its fatal honesty)

Step along the current slope:
$$y_{k+1} = y_k + h \cdot f(t_k, y_k)$$

```python
def euler(f, t0, y0, h, n):
    t, y = t0, y0
    for _ in range(n):
        y = y + h * f(t, y)
        t += h
    return y

# y' = y, y(0)=1 → exact answer e^t; simulate to t=1
for h in (0.1, 0.01, 0.001):
    approx = euler(lambda t, y: y, 0, 1.0, h, round(1/h))
    print(h, approx, abs(approx - 2.718281828459045))
# 0.1   2.5937  err 0.12     ← err ~ h (first-order): ×10 steps → ÷10 error
# 0.01  2.7048  err 0.013
# 0.001 2.7169  err 0.0013
```

Global error $O(h)$: linear convergence — to halve the error, double the work.

### Example by hand: solving $y' = 2y, \quad y(0) = 1$ with $h = 0.1$
Exact analytical solution is $y(t) = e^{2t}$.
- **Step 1 ($t_0 = 0 \to t_1 = 0.1$)**:
  - Slope $f(0, 1) = 2(1) = 2.0$.
  - $y_1 = y_0 + h \cdot f(t_0, y_0) = 1.0 + 0.1(2.0) = \mathbf{1.2000}$.
  - Exact: $e^{0.2} \approx 1.2214$ (error $= 0.0214$).
- **Step 2 ($t_1 = 0.1 \to t_2 = 0.2$)**:
  - Slope $f(0.1, 1.2) = 2(1.2) = 2.4$.
  - $y_2 = y_1 + h \cdot f(t_1, y_1) = 1.2 + 0.1(2.4) = \mathbf{1.4400}$.
  - Exact: $e^{0.4} \approx 1.4918$ (error $= 0.0518$).
- Notice the step underestimate: because the true curve curves upward, evaluating slope only at the beginning of the step consistently underestimates growth.

### Where it's used
- **Game physics / basic character controller kinematics**: position integration $\mathbf{p} \leftarrow \mathbf{p} + \mathbf{v}\Delta t$ where speed and predictability matter more than high-precision trajectory fidelity.
- **Microcontroller embedded simulations**: small sensors running simple real-time filter integrations where compute is severely constrained.

---

## The Runge-Kutta ladder

Sample the slope field *several times per step* and blend — a higher-order local model.
The classic **RK4** (4 slope samples per step, error $O(h^4)$):
1. $k_1 = f(t_n, y_n)$ (slope at start)
2. $k_2 = f(t_n + \frac{h}{2}, y_n + \frac{h}{2}k_1)$ (trial slope at midpoint)
3. $k_3 = f(t_n + \frac{h}{2}, y_n + \frac{h}{2}k_2)$ (refined slope at midpoint)
4. $k_4 = f(t_n + h, y_n + h k_3)$ (trial slope at end)
$$y_{n+1} = y_n + \frac{h}{6}(k_1 + 2k_2 + 2k_3 + k_4)$$

```python
def rk4(f, t0, y0, h, n):
    t, y = t0, y0
    for _ in range(n):
        k1 = f(t, y)
        k2 = f(t + h/2, y + h*k1/2)
        k3 = f(t + h/2, y + h*k2/2)
        k4 = f(t + h,   y + h*k3)
        y += h * (k1 + 2*k2 + 2*k3 + k4) / 6
        t += h
    return y

for h in (0.1, 0.01):
    approx = rk4(lambda t, y: y, 0, 1.0, h, round(1/h))
    print(h, abs(approx - 2.718281828459045))
# 0.1   err 1.6e-6     ← at h=0.1 RK4 already beats Euler at h=0.001
# 0.01  err 1.3e-10    ← ×10 steps → ÷10000 error (4th order visible)
```

### Example by hand: one step of RK4 on $y' = y, \quad y(0) = 1$ with $h = 0.2$
- $k_1 = f(0, 1) = \mathbf{1.0}$
- $k_2 = f(0 + 0.1, 1 + 0.1(1.0)) = f(0.1, 1.1) = \mathbf{1.1}$
- $k_3 = f(0 + 0.1, 1 + 0.1(1.1)) = f(0.1, 1.11) = \mathbf{1.11}$
- $k_4 = f(0 + 0.2, 1 + 0.2(1.11)) = f(0.2, 1.222) = \mathbf{1.222}$
- Weighted blend:
  $$y_1 = 1.0 + \frac{0.2}{6}(1.0 + 2(1.1) + 2(1.11) + 1.222) = 1.0 + \frac{0.2}{6}(6.642) = 1.0 + 0.2214 = \mathbf{1.221400}$$
- Compare to exact solution: $e^{0.2} = \mathbf{1.221402758\dots}$
- Error after a large step of $h=0.2$ is only $2.7 \times 10^{-6}$!

### Where it's used
- **Flight simulators & vehicle dynamics**: high-fidelity 6-DOF aircraft simulation where aerodynamics require smooth, highly accurate trajectory tracking.
- **Generative AI (Continuous Normalizing Flows & Diffusion Sampling)**: diffusion models integrate a reverse-time probability flow ODE where high-order solvers reduce the number of model evaluations needed to generate an image.

---

## Stability & Stiff equations

Accuracy isn't the killer; **stability** is.
Test equation: $y' = \lambda y$ (decay, $\lambda < 0$).
- Forward Euler gives: $y_{n+1} = (1 + h\lambda) y_n$.
- Blows up if $|1 + h\lambda| > 1 \implies h > \frac{2}{|\lambda|}$!

```python
import math
for h in (0.01, 0.05):
    approx = euler(lambda t, y: -50*y, 0, 1.0, h, round(1/h))
    print(h, approx)     # exact: e^-50 ≈ 2e-22 ≈ 0
# 0.01  3.4e-17         stable (1 + hλ = 0.5, shrinks)
# 0.05  -2.9e+10        EXPLODED (1 + hλ = -1.5, |·|>1 → growing oscillation)
```

The true solution is ≈ 0 and Euler produces −3×10¹⁰. This is a **stiff** problem.

**Fix: Backward (Implicit) Euler**: evaluate slope at the *future* point:
$$y_{n+1} = y_n + h f(t_{n+1}, y_{n+1}) \implies y_{n+1} = \frac{y_n}{1 - h\lambda}$$
Because $|1 - h\lambda| > 1$ for all $h > 0$ when $\lambda < 0$, Backward Euler is **unconditionally stable** for any step size $h$!

### Example by hand: solving $y' = -100y, \quad y(0) = 1$ with $h = 0.1$
Exact solution at $t = 0.1$: $y(0.1) = e^{-10} \approx 4.54 \times 10^{-5}$.
1. **Forward Euler**:
   $$y_1 = y_0 + h(-100 y_0) = 1 + 0.1(-100)(1) = 1 - 10 = \mathbf{-9.0}$$
   Result exploded in the wrong direction with inverted sign!
2. **Backward Euler**:
   $$y_1 = y_0 + h(-100 y_1) \implies y_1(1 + 10) = 1 \implies y_1 = \frac{1}{11} \approx \mathbf{0.0909}$$
   Stable exponential decay preserved, zero explosion, regardless of large step size $h$.

### Where it's used
- **Chemical kinetics & combustion simulation**: reactions with fast-decaying intermediate radicals alongside slow main reactions (timescales differ by $10^6$).
- **Heat transfer & SPICE circuit simulation**: RC network time constants spanning microseconds to seconds.

---

## Symplectic integrators — energy conservation in physics

Standard RK4 slowly drifts energy (dissipates or adds artificial energy over millions of orbits).
**Symplectic (Semi-Implicit) Euler** updates velocity with old position, then position with the **new** velocity:
$$v_{n+1} = v_n + h \cdot a(x_n), \quad x_{n+1} = x_n + h \cdot v_{n+1}$$

### Example by hand: Harmonic oscillator ($x' = v, \quad v' = -x$) with $h = 1.0$, start $(x_0, v_0) = (1, 0)$
Initial total energy: $E_0 = \frac{1}{2}(x_0^2 + v_0^2) = \frac{1}{2}(1 + 0) = \mathbf{0.50}$.
1. **Explicit Forward Euler**:
   - $x_1 = x_0 + h v_0 = 1 + 1(0) = 1$
   - $v_1 = v_0 + h (-x_0) = 0 + 1(-1) = -1$
   - $E_1 = \frac{1}{2}(1^2 + (-1)^2) = \mathbf{1.00}$ ($\mathbf{100\% \text{ energy gain!}}$ Trajectory spirals outward to infinity).
2. **Symplectic Euler**:
   - $v_1 = v_0 + h (-x_0) = 0 + 1(-1) = -1$
   - $x_1 = x_0 + h v_1 = 1 + 1(-1) = \mathbf{0}$ (used new $v_1$!)
   - $E_1 = \frac{1}{2}(0^2 + (-1)^2) = \mathbf{0.50}$ ($\mathbf{\text{Energy exactly preserved!}}$).

### Where it's used
- **Astrophysics & Solar System N-body simulators**: simulating planetary orbits over billions of years without planets falling into the sun.
- **Molecular Dynamics (Verlet algorithm)**: simulating protein folding where artificial energy gain breaks chemical bonds.

## Gotchas

- Oscillating growing error at decreasing h: that's instability, not a bug in f.
- Stiffness is often hidden (chemical kinetics, circuits, control loops) — if `RK45` crawls at tiny h while the solution looks smooth, suspect stiffness and switch to `BDF`.
- Event handling (bounce, collision) needs root-finding per step (day 2) — don't detect events by sign-checking after the step.
- Chaotic systems (double pendulum, Lorenz): tiny perturbations grow exponentially — no solver gives pointwise accuracy long-term; statistics are the meaningful output.
- `h` too small also hurts: roundoff accumulates per step (1/h steps × ε each) — the U-curve again.

## Exercises

**1.** Simulate y' = −2y, y(0)=1 with Euler at h = 0.4, 1.0, 1.2 out to t=4. At which h does it break, and does the stability formula predict it?

**2.** Implement the harmonic oscillator y'' = −y as a 2-D first-order system (v = y', v' = −y), simulate with Euler and RK4 for 100 periods, and plot/print total energy E = (y²+v²)/2 at the end. Which conserves energy better? (Bonus: look up semi-implicit/symplectic Euler and try it.)

**3.** Use Richardson thinking: run RK4 at h and h/2, and combine to get a 6th-order-ish estimate of e. How close is (16·R(h/2) − R(h))/15 to e?

<details>
<summary><b>Solutions</b></summary>

**1.** Stability boundary: |1 − 2h| ≤ 1 → h ≤ 1. At h=1.0 the iterate is stuck (−1)ⁿ oscillation — no decay; at 1.2 it explodes; at 0.4 it decays. The formula predicts all three.

**2.** Euler spirals *outward* (energy grows — Euler adds energy to oscillators; watch it), RK4 loses energy slowly, symplectic Euler (update v first, then y with the *new* v) keeps energy bounded indefinitely — bounded error, no drift. This is the symplectic superpower.

**3.** R(0.1)=2.7182797…, R(0.05)≈2.7182818…; (16·R(h/2)−R(h))/15 matches e to ~1e-12 from just 30 f-evaluations total.

</details>

## Deep dives

- Heath, *Scientific Computing*, ch. 9 — ODEs; the stability-region pictures are the ones to internalize.
- **Hairer, Nørsett & Wanner, *Solving ODEs I*** (non-stiff) — the bible; vol. II covers stiff. First chapters freely previewable via Springer.
- Cleve Moler's NCM (free: https://www.mathworks.com/moler) ch. 7 — ode23/ode45 internals explained by their author.
- 3Blue1Brown — *"Differential equations"* series for the visual intuition of slope fields: https://www.youtube.com/watch?v=p_di4Zn4wz4
- Stiffness demo worth reading: the "flame propagation" example in MATLAB docs (`ode15s` vs `ode45`).
