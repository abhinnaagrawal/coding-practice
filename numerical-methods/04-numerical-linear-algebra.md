[← back to index](/numerical-methods/README.md)

# Day 4 — Numerical Linear Algebra

The day numpy earns its keep: `pip install numpy`. Everything here is about solving **Ax = b** and friends — the inner loop of simulation, ML, and optimization.

## Gaussian elimination, and why pivoting is not optional

Solve by eliminating variables — exact same algorithm you learned by hand:

```python
import numpy as np

def gauss_no_pivot(A, b):
    A = A.astype(float).copy(); b = b.astype(float).copy()
    n = len(A)
    for k in range(n):                       # eliminate column k
        for i in range(k+1, n):
            factor = A[i,k] / A[k,k]         # ← divides by the pivot
            A[i,k:] -= factor * A[k,k:]
            b[i]   -= factor * b[k]
    x = np.zeros(n)                          # back substitution
    for i in range(n-1, -1, -1):
        x[i] = (b[i] - A[i,i+1:] @ x[i+1:]) / A[i,i]
    return x

A = np.array([[1e-20, 1.0], [1.0, 1.0]])
b = np.array([1.0, 2.0])                     # true solution ≈ (1, 1)
print(gauss_no_pivot(A, b))                  # [0. 1.] — x₁ is WRONG
```

What happened: the tiny pivot 1e-20 forces factor = 1e20; subtracting 1e20 × (row) annihilates the original row data (absorption, day 1). **Partial pivoting** — always swap the largest-magnitude entry in the column into the pivot position — fixes it:

```python
def gauss(A, b):
    A = A.astype(float).copy(); b = b.astype(float).copy()
    n = len(A)
    for k in range(n):
        p = k + np.argmax(np.abs(A[k:,k]))   # ← pivot: biggest entry below
        A[[k,p]] = A[[p,k]]; b[[k,p]] = b[[p,k]]
        for i in range(k+1, n):
            factor = A[i,k] / A[k,k]
            A[i,k:] -= factor * A[k,k:]
            b[i]   -= factor * b[k]
    x = np.zeros(n)
    for i in range(n-1, -1, -1):
        x[i] = (b[i] - A[i,i+1:] @ x[i+1:]) / A[i,i]
    return x

print(gauss(A, b))                           # [1. 1.] — correct
```

That's the entire lesson: **the algorithm's stability depends on a choice the math says is irrelevant** (row order doesn't change the exact solution, but changes everything in floating point). `np.linalg.solve` does LU with partial pivoting.

## LU factorization — solve many b's against the same A

Elimination factors A = P·L·U (permutation, unit lower-triangular, upper-triangular). Cost: O(n³) once to factor, then O(n²) per new right-hand side. This is why code that solves Ax=b in a loop should factor once — and why you never compute A⁻¹.

```python
# Never do this:  x = np.linalg.inv(A) @ b      — slower AND less accurate
# Always:         x = np.linalg.solve(A, b)
```

`inv()` costs ~3× a solve and multiplies errors. There is essentially no correct reason to materialize an inverse.

## Norms and the condition number

- Vector norms: ‖x‖₁ (sum |xᵢ|), ‖x‖₂ (Euclidean), ‖x‖∞ (max |xᵢ|).
- Matrix norm (induced): max amplification of a unit vector.
- **Condition number**: κ(A) = ‖A‖·‖A⁻¹‖ ≥ 1. It is the answer to "how much can solving Ax=b amplify input error?" Rule of thumb: **you lose ≈ log₁₀(κ) digits of accuracy**. κ = 10¹² → expect ~4 correct digits out of 16.

```python
A = np.array([[1, 1], [1, 1.0001]])
print(np.linalg.cond(A))          # ~4e4 — lose ~4 digits

b  = np.array([2, 2.0001])
b2 = np.array([2, 2.0002])        # perturb b by 1e-4 (5e-5 relative)
x1, x2 = np.linalg.solve(A, b), np.linalg.solve(A, b2)
print(x1, x2)                     # [1. 1.] vs [0.5 1.5] — 50% output change!
```

The **problem** is sensitive (κ large) — no solver can do better. The Hilbert matrix is the extreme demo: `np.linalg.cond(np.array([[1/(i+j+1) for j in range(n)] for i in range(n)]))` hits 10¹⁸ by n=13.

## The factorization menu (know what each is for)

| Method | Factorization | Use when | Cost |
|--------|---------------|----------|------|
| LU | A = PLU | general square solve | n³/3 |
| Cholesky | A = LLᵀ | A symmetric positive-definite — 2× faster than LU | n³/6 |
| QR | A = QR | least squares (min ‖Ax−b‖), avoids squaring κ like normal equations | ~2n³ |
| SVD | A = UΣVᵀ | rank-deficient, pseudoinverse, PCA, "what rank is this really?" | heavier |
| Eigen | A = XΛX⁻¹ | dynamics, stability, powers of A | heavier |

Least squares golden rule: `np.linalg.lstsq(A, b)` (QR/SVD-based), **never** the textbook normal equations `(AᵀA)⁻¹Aᵀb` — squaring A squares κ.

```python
x, *_ = np.linalg.lstsq(A, b, rcond=None)   # overdetermined: min ||Ax - b||₂
```

## Eigen intuition via power iteration

The dominant eigenvector is whatever direction A keeps amplifying: iterate x ← Ax/‖Ax‖:

```python
A = np.array([[2.0, 1.0], [1.0, 2.0]])     # eigenvalues 3, 1
x = np.array([1.0, 0.0])
for _ in range(30):
    x = A @ x
    x = x / np.linalg.norm(x)
lam = x @ A @ x                             # Rayleigh quotient ≈ eigenvalue
print(x, lam)                               # [0.707 0.707], 3.0
```

This is PageRank's core (dominant eigenvector of the link matrix) and why PCA works (top eigenvectors of the covariance matrix = SVD of centered data).

## Gotchas

- `np.linalg.inv` in production code is a red flag in review — always `solve`.
- Singular/near-singular A: `solve` may return *something* without complaint if it's just ill-conditioned. Check `np.linalg.cond(A)` when results look insane.
- Row swaps change the *sign* of determinants — don't reuse pivoted factors blindly for `det`.
- Sparse matrices: dense O(n³) factorization destroys sparsity (fill-in). Use `scipy.sparse.linalg` — different universe, mention it exists.
- Cholesky fails loudly (non-positive pivot) when A is not SPD — that failure is information: your matrix wasn't SPD (often a bug upstream, e.g. a non-symmetrized covariance).

## Exercises

**1.** For the 6×6 Hilbert matrix, solve Ax = b with b = A·(1,1,…,1) (true answer is all ones) using `np.linalg.solve`. Print the error and `cond(A)`. Repeat for n = 10.

**2.** Show numerically that normal equations lose precision: take A as a 50×3 Vandermonde-like matrix with columns [1, x, x²] for x ∈ [0,1], fit noisy data, and compare `lstsq` vs solving AᵀA c = Aᵀy directly against the known coefficients. Compare `cond(A)²` vs `cond(AᵀA)`.

**3.** Power iteration on a matrix whose two top eigenvalues are equal in magnitude (e.g. diag([3, −3, 1])) — what happens and why?

<details>
<summary><b>Solutions</b></summary>

**1.**

```python
n = 6
H = np.array([[1/(i+j+1) for j in range(n)] for i in range(n)])
x_true = np.ones(n)
b = H @ x_true
err = np.linalg.norm(np.linalg.solve(H, b) - x_true, np.inf)
print(n, np.linalg.cond(H), err)   # n=6: κ≈1.5e7, err≈1e-9; n=10: κ≈1.6e13, err≈0.1+
```

Digits lost ≈ log₁₀κ matches: ~7 digits lost at n=6, ~13 at n=10 (nothing left).

**2.** `cond(A.T @ A)` ≈ `cond(A)**2` numerically; the normal-equations solution shows visibly larger error when columns are near-collinear (x² ≈ x on [0,1] in correlation).

**3.** It oscillates/flips between the two eigenvectors (or a combination) — the ratio test never converges because |λ₁/λ₂| = 1 gives no separation. Power iteration needs a strict spectral gap; this is why real eigensolvers (QR algorithm) exist.

</details>

## Deep dives

- **Trefethen & Bau, *Numerical Linear Algebra*** — the text for this chapter; Lectures 20–22 (LU/pivoting), 12 (conditioning), 7+ (QR). University course mirrors have the lecture notes.
- Heath, *Scientific Computing*, ch. 2–3.
- **Gilbert Strang's MIT 18.06 lectures** (free on MIT OpenCourseWare/YouTube) — for rebuilding geometric intuition of what A actually *does*: https://ocw.mit.edu/courses/18-06-linear-algebra-spring-2010/
- *Immersive Linear Algebra* (free interactive book): https://immersivemath.com
- Higham's blog posts on pivoting and growth factors: https://nhigham.com
