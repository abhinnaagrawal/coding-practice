[← back to index](/numerical-methods/README.md)

# Day 6 — ODEs: Simulating Dynamics

Problem: y'(t) = f(t, y), y(t₀) = y₀ — given the slope field and a starting point, trace the trajectory. This is physics sims, control systems, epidemic models, and gradient flows.

## Euler — the prototype (and its fatal honesty)

Step along the current slope: y ← y + h·f(t, y).

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

Global error O(h): linear convergence — to halve the error, double the work. Euler is the "bisection" of ODE solvers: conceptually complete, practically weak.

## The Runge-Kutta ladder

Sample the slope field *several times per step* and blend — a higher-order local model (day 3's thread again). The classic **RK4**: four slope samples, error O(h⁴):

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

Same f-evaluations as 4 Euler steps, ~4–5 orders more accuracy. This is why nobody ships Euler.

## Stability — the part Euler honesty reveals

Accuracy isn't the killer; **stability** is. Test equation: y' = λy (decay, λ < 0). Exact solution decays. Euler gives yₙ = (1 + hλ)ⁿ — which **blows up if |1 + hλ| > 1**, i.e. h > 2/|λ|:

```python
import math
for h in (0.01, 0.05):
    approx = euler(lambda t, y: -50*y, 0, 1.0, h, round(1/h))
    print(h, approx)     # exact: e^-50 ≈ 2e-22 ≈ 0
# 0.01  3.4e-17         stable (1 + hλ = 0.5, shrinks)
# 0.05  -2.9e+10        EXPLODED (1 + hλ = -1.5, |·|>1 → growing oscillation)
```

The true solution is ≈ 0 and Euler produces −3×10¹⁰. Not inaccurate — *qualitatively wrong*. This is a **stiff** problem when it has multiple timescales (fast transient λ=−50 plus slow dynamics): explicit methods must size h to the *fast* scale even after it has decayed — ruinously small steps.

**Fix: implicit methods.** Backward Euler: y ← y + h·f(t+h, y_new) — solve an equation each step (Newton from day 2!) but unconditionally stable for decay problems:

```python
def backward_euler_exp( lam, y0, h, n):      # closed form for y' = λy
    return y0 * (1/(1 - h*lam))**n
print(backward_euler_exp(-50, 1.0, 0.05, 20))   # 2.1e-20 — fine at h that killed Euler
```

Trade-off summary:

| | Explicit (Euler, RK4) | Implicit (Backward Euler, Radau) |
|---|---|---|
| Cost/step | cheap (f evals) | expensive (solve nonlinear system per step) |
| Step limit | h bounded by **stability** | h bounded only by **accuracy** |
| Use for | smooth, non-stiff | stiff, multiple timescales |

`scipy.integrate.solve_ivp` defaults: `RK45` (adaptive RK) for general use, `BDF`/`Radau` for stiff.

## Adaptive stepping & symplectic note

Production solvers estimate error per step (embedded RK pair — two orders from shared evaluations, Richardson-style) and grow/shrink h automatically. Two more names worth recognizing: **symplectic integrators** (leapfrog/Verlet) conserve energy over long Hamiltonian simulations where RK4 slowly drains it — the reason orbital mechanics doesn't use RK4 naively.

## Where it's used

- **Games & animation**: position ← velocity integration every frame; semi-implicit (symplectic) Euler is the default because explicit Euler makes springs explode and orbits spiral out — the energy behavior in exercise 2 is felt by every game developer.
- **Robotics & control**: quadrotor trajectory simulation, PID loops (discretized ODEs), model-predictive control rolls forward dynamics with RK each control tick.
- **ML**: neural ODEs; gradient descent *is* Euler on θ' = −∇f (day 7); diffusion model sampling is reverse-time ODE/SDE integration — solver order directly buys sample quality per step.
- **Epidemiology & pharma**: SIR compartment models, drug concentration kinetics (the canonical stiff system — fast absorption, slow elimination).
- **Aerospace**: orbit propagation uses symplectic or high-order adaptive integrators; RK4 naive drains orbital energy over long horizons.
- **Electrical engineering**: SPICE circuit simulators live and die on stiff transient analysis — backward differentiation (BDF) is the workhorse.

## Dry run by hand

**1.** Euler on y' = y from (0, 1) with h = 0.5, four steps:
- y₁ = 1 + 0.5·1 = 1.5 → y₂ = 1.5 + 0.5·1.5 = 2.25 → y₃ = 3.375 → y₄ = 5.0625
- Exact e² = 7.389. Underestimate — makes sense: the slope *grows* along the true curve, but Euler keeps each step's starting slope for the whole step. Trace it on paper: piecewise-linear segments falling progressively behind the curve.

**2.** RK4's first step on the same problem, h = 0.5:
- k1 = f(0, 1) = 1
- k2 = f(0.25, 1 + 0.25) = 1.25
- k3 = f(0.25, 1 + 0.3125) = 1.3125
- k4 = f(0.5, 1 + 0.65625) = 1.65625
- y₁ = 1 + (0.5/6)(1 + 2·1.25 + 2·1.3125 + 1.65625) = 1 + (0.5/6)(7.781) = 1.6484
- Exact e^0.5 = 1.6487. One step, 4 f-evals, error 3e-4 — vs Euler's error 0.15 for the same h. Feel the midpoint samples doing the work.

**3.** Stability by hand, y' = −10y, h = 0.25: amplification factor 1 + hλ = 1 − 2.5 = **−1.5**. Starting y₀ = 1: y₁ = −1.5, y₂ = +2.25, y₃ = −3.375 — growing with alternating sign. The true answer decays to e^−2.5 ≈ 0.08. The sign flip is the fingerprint: solution crosses zero each step because the step overshoots it. That oscillation in a sim output = instability, instantly diagnosable by hand.

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
