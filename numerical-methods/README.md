# Numerical Methods — Intuition & Code for Engineers

Live site: **https://abhinnaagrawal.github.io/coding-practice/#/numerical-methods/**
Top-level site: [Home](/README.md)

Audience: engineers who studied numerical analysis years ago and remember none of it. Goal: rebuild **intuition** (why methods work, when they break) with runnable Python code — not theorem-proving. Shareable — no personal context.

## How to use this
- One chapter per day; ~1 hour of reading + 30–60 min running the code.
- Every concept: **why it matters → minimal example → runnable Python → gotchas → deep-dive references**.
- Code uses plain Python where possible so every operation is visible; `numpy` appears only where it *is* the lesson (chapter 4).
- Run everything: `python3 <file>.py` — no dependencies except chapters 3–4, 7 (numpy/scipy snippets).

## Pacing plan (one week)

| Day | Chapter | Core question |
|-----|---------|---------------|
| 1 | [Number Representation](/numerical-methods/01-number-representation.md) | Why is `0.1 + 0.2 != 0.3`, and when does it actually hurt? |
| 2 | [Errors & Root Finding](/numerical-methods/02-errors-and-root-finding.md) | How do we find x where f(x)=0, and how fast? |
| 3 | [Interpolation & Approximation](/numerical-methods/03-interpolation-and-approximation.md) | How do we reconstruct a function from samples — and why can more points be *worse*? |
| 4 | [Numerical Linear Algebra](/numerical-methods/04-numerical-linear-algebra.md) | Why is "just solve Ax=b" a minefield, and what's a condition number? |
| 5 | [Differentiation & Integration](/numerical-methods/05-differentiation-and-integration.md) | How do computers compute derivatives and areas without calculus? |
| 6 | [ODEs](/numerical-methods/06-ordinary-differential-equations.md) | How do we simulate f'(t) = f(t), and why do some simulations explode? |
| 7 | [Numerical Optimization](/numerical-methods/07-numerical-optimization.md) | Gradient descent through numerical-analysis eyes: step sizes, curvature, conditioning |

## The five intuitions to keep after the week

1. **Floating point is not real arithmetic.** It's a finite, non-uniform grid. Addition isn't associative; subtraction of near-equal numbers destroys precision (*catastrophic cancellation*). Every algorithm is judged by how it behaves on this grid.
2. **Conditioning vs stability are different.** Conditioning = how sensitive the *problem* is to input perturbations (intrinsic). Stability = how much extra error the *algorithm* adds (our choice). A stable algorithm on an ill-conditioned problem still gives a bad answer.
3. **Convergence rate matters more than convergence.** Bisection halves the interval each step (linear); Newton squares the error each step (quadratic) — 10 bisection steps ≈ 3–4 Newton steps.
4. **Step size is a three-way tradeoff.** Small h: truncation error shrinks, roundoff error grows, cost grows. Every numerical method lives at the minimum of that U-curve.
5. **Local polynomial models drive everything.** Taylor series truncated at k terms → finite differences, Newton's method, Runge-Kutta, quadrature rules. When the function is locally polynomial-like, these shine; when it isn't (kinks, discontinuities, stiffness), they fail predictably.
