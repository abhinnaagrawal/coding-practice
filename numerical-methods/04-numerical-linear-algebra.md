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

### Example by hand: failure without pivoting on a 3-digit system
Solve $\begin{pmatrix} 10^{-4} & 1 \\ 1 & 1 \end{pmatrix} \begin{pmatrix} x_1 \\ x_2 \end{pmatrix} = \begin{pmatrix} 1 \\ 2 \end{pmatrix}$ in 3-digit decimal arithmetic. Exact answer is $x_1 = \frac{1}{0.9999} \approx 1.0001, x_2 = \frac{0.9998}{0.9999} \approx 0.9999$.
1. **Without pivoting** (pivot is $10^{-4}$):
   - Multiplier $m = 1 / 10^{-4} = 10^4$.
   - Row 2 becomes:
     - $A_{22} = 1 - 10^4 \times 1 = -9999 \xrightarrow{\text{3-digit round}} -1.00 \times 10^4$.
     - $b_2 = 2 - 10^4 \times 1 = -9998 \xrightarrow{\text{3-digit round}} -1.00 \times 10^4$.
   - Back-substitution:
     - $x_2 = (-1.00 \times 10^4) / (-1.00 \times 10^4) = \mathbf{1.00}$.
     - $x_1 = (1 - 1 \times x_2) / 10^{-4} = (1 - 1.00) / 10^{-4} = 0.0 / 10^{-4} = \mathbf{0.0}$ $\implies \mathbf{100\% \text{ error!}}$
2. **With partial pivoting** (swap Row 1 and Row 2):
   - System: $\begin{pmatrix} 1 & 1 \\ 10^{-4} & 1 \end{pmatrix} \begin{pmatrix} x_1 \\ x_2 \end{pmatrix} = \begin{pmatrix} 2 \\ 1 \end{pmatrix}$.
   - Multiplier $m = 10^{-4} / 1 = 10^{-4} \le 1$.
   - Row 2 becomes:
     - $A_{22} = 1 - 10^{-4} \times 1 = 0.9999 \xrightarrow{\text{3-digit round}} 1.00$.
     - $b_2 = 1 - 10^{-4} \times 2 = 0.9998 \xrightarrow{\text{3-digit round}} 1.00$.
   - Back-substitution: $x_2 = 1.00 / 1.00 = \mathbf{1.00}$, $x_1 = (2 - 1.00) / 1 = \mathbf{1.00}$. Both correct!

### Where it's used
- **SPICE circuit simulation**: nodal analysis matrices have large conductances next to tiny leakage currents; pivoting prevents voltage solution collapse.
- **Structural finite element analysis**: stiffness matrices for trusses and beams.

---

## LU factorization — solve many b's against the same A

Elimination factors $A = P L U$ where $P$ is a permutation matrix, $L$ is unit lower-triangular, and $U$ is upper-triangular.
To solve $A\mathbf{x} = \mathbf{b}$:
1. Solve $L\mathbf{y} = P\mathbf{b}$ via **forward-substitution** ($O(n^2)$).
2. Solve $U\mathbf{x} = \mathbf{y}$ via **back-substitution** ($O(n^2)$).
Cost: $O(n^3/3)$ once to factor, then $O(n^2)$ per subsequent right-hand side.

```python
# Never do this:  x = np.linalg.inv(A) @ b      — slower AND less accurate
# Always:         x = np.linalg.solve(A, b)
```

`inv()` costs ~3× a solve and multiplies errors. There is essentially no correct reason to materialize an inverse.

### Example by hand: factoring and solving $A\mathbf{x} = \mathbf{b}$
Let $A = \begin{pmatrix} 2 & 1 \\ 6 & 8 \end{pmatrix}, \quad \mathbf{b} = \begin{pmatrix} 5 \\ 23 \end{pmatrix}$.
1. **Factor $A = LU$**:
   - Pivot row 1: multiplier for row 2 is $m_{21} = 6 / 2 = 3$.
   - Row 2 becomes: $(6, 8) - 3(2, 1) = (0, 5) \implies U = \begin{pmatrix} 2 & 1 \\ 0 & 5 \end{pmatrix}$.
   - Multipliers form $L = \begin{pmatrix} 1 & 0 \\ 3 & 1 \end{pmatrix}$. Check: $LU = \begin{pmatrix} 1\cdot 2 & 1\cdot 1 \\ 3\cdot 2 & 3\cdot 1 + 1\cdot 5 \end{pmatrix} = \begin{pmatrix} 2 & 1 \\ 6 & 8 \end{pmatrix} = A$.
2. **Step 1: Forward-substitute $L\mathbf{y} = \mathbf{b}$**:
   $$\begin{pmatrix} 1 & 0 \\ 3 & 1 \end{pmatrix} \begin{pmatrix} y_1 \\ y_2 \end{pmatrix} = \begin{pmatrix} 5 \\ 23 \end{pmatrix} \implies y_1 = 5, \quad 3(5) + y_2 = 23 \implies y_2 = 8 \implies \mathbf{y} = \begin{pmatrix} 5 \\ 8 \end{pmatrix}$$
3. **Step 2: Back-substitute $U\mathbf{x} = \mathbf{y}$**:
   $$\begin{pmatrix} 2 & 1 \\ 0 & 5 \end{pmatrix} \begin{pmatrix} x_1 \\ x_2 \end{pmatrix} = \begin{pmatrix} 5 \\ 8 \end{pmatrix} \implies 5x_2 = 8 \implies x_2 = \mathbf{1.6}, \quad 2x_1 + 1.6 = 5 \implies x_1 = \mathbf{1.7}$$

### Where it's used
- **Dynamic time-stepping in physics/PDEs**: matrix $A = I - \Delta t K$ stays constant while load vector $\mathbf{b}(t)$ changes every millisecond; factor once, solve millions of times.
- **Power grid load-flow calculations**: repeated iterative linear solves on the grid admittance matrix.

---

## Norms and the condition number

- Vector norms: ‖x‖₁ (sum |xᵢ|), ‖x‖₂ (Euclidean), ‖x‖∞ (max |xᵢ|).
- Matrix norm (induced): max amplification of a unit vector.
- **Condition number**: $\kappa(A) = \|A\| \cdot \|A^{-1}\| \ge 1$.
- **Precision rule**: $\text{digits of precision lost} \approx \log_{10}(\kappa(A))$.

```python
A = np.array([[1, 1], [1, 1.0001]])
print(np.linalg.cond(A))          # ~4e4 — lose ~4 digits

b  = np.array([2, 2.0001])
b2 = np.array([2, 2.0002])        # perturb b by 1e-4 (5e-5 relative)
x1, x2 = np.linalg.solve(A, b), np.linalg.solve(A, b2)
print(x1, x2)                     # [1. 1.] vs [0.5 1.5] — 50% output change!
```

### Example by hand: condition number sensitivity
Let $A = \begin{pmatrix} 1 & 1 \\ 1 & 1.01 \end{pmatrix}$. Its inverse is $A^{-1} = \frac{1}{0.01}\begin{pmatrix} 1.01 & -1 \\ -1 & 1 \end{pmatrix} = \begin{pmatrix} 101 & -100 \\ -100 & 100 \end{pmatrix}$.
- Using $\infty$-norm: $\|A\|_\infty = \max(1+1, 1+1.01) = 2.01$.
- $\|A^{-1}\|_\infty = \max(101+100, 100+100) = 201$.
- $\kappa_\infty(A) = 2.01 \times 201 \approx \mathbf{404}$.
- Now solve $A\mathbf{x} = \begin{pmatrix} 2 \\ 2.01 \end{pmatrix} \implies \mathbf{x} = \begin{pmatrix} 1 \\ 1 \end{pmatrix}$.
- Perturb $\mathbf{b}$ by only $0.01$ to $\mathbf{b}' = \begin{pmatrix} 2 \\ 2.02 \end{pmatrix}$ ($\approx 0.5\%$ change).
- New solution: $\mathbf{x}' = A^{-1}\mathbf{b}' = \begin{pmatrix} 101(2) - 100(2.02) \\ -100(2) + 100(2.02) \end{pmatrix} = \begin{pmatrix} 202 - 202 \\ -200 + 202 \end{pmatrix} = \begin{pmatrix} \mathbf{0} \\ \mathbf{2} \end{pmatrix}$.
- The solution changed by $100\%$! An amplification factor of $\approx 200\times$, directly bounded by $\kappa(A) \approx 404$.

---

## The factorization menu (know what each is for)

| Method | Factorization | Use when | Cost |
|--------|---------------|----------|------|
| LU | A = PLU | general square solve | n³/3 |
| Cholesky | A = LLᵀ | A symmetric positive-definite — 2× faster than LU | n³/6 |
| QR | A = QR | least squares (min ‖Ax−b‖), avoids squaring κ like normal equations | ~2n³ |
| SVD | A = UΣVᵀ | rank-deficient, pseudoinverse, PCA, "what rank is this really?" | heavier |
| Eigen | A = XΛX⁻¹ | dynamics, stability, powers of A | heavier |

### Example by hand: Cholesky factorization $A = LL^T$
Let $A = \begin{pmatrix} 4 & 2 \\ 2 & 10 \end{pmatrix}$ (symmetric and positive-definite).
We want lower triangular $L = \begin{pmatrix} l_{11} & 0 \\ l_{21} & l_{22} \end{pmatrix}$ such that $LL^T = A$:
$$LL^T = \begin{pmatrix} l_{11}^2 & l_{11}l_{21} \\ l_{11}l_{21} & l_{21}^2 + l_{22}^2 \end{pmatrix} = \begin{pmatrix} 4 & 2 \\ 2 & 10 \end{pmatrix}$$
1. $l_{11}^2 = 4 \implies l_{11} = \mathbf{2}$.
2. $l_{11}l_{21} = 2 \implies 2l_{21} = 2 \implies l_{21} = \mathbf{1}$.
3. $l_{21}^2 + l_{22}^2 = 10 \implies 1^2 + l_{22}^2 = 10 \implies l_{22} = \sqrt{9} = \mathbf{3}$.
4. $L = \begin{pmatrix} 2 & 0 \\ 1 & 3 \end{pmatrix}$. No row swapping needed, zero pivoting overhead, half the FLOPs of LU.

#### Where it's used
- **Sampling from correlated Gaussians**: to draw $X \sim \mathcal{N}(\boldsymbol{\mu}, \Sigma)$, compute Cholesky $\Sigma = LL^T$ and return $\boldsymbol{\mu} + L\mathbf{z}$ where $\mathbf{z} \sim \mathcal{N}(0, I)$.
- **Kalman Filters (Square-Root formulation)**: storing $L$ rather than $\Sigma$ guarantees numerical positive-definiteness across millions of radar updates.

---

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

### Example by hand: 2 steps of power iteration
Let $A = \begin{pmatrix} 3 & 1 \\ 0 & 2 \end{pmatrix}$ (eigenvalues $\lambda_1 = 3, \lambda_2 = 2$). Start with $\mathbf{x}_0 = \begin{pmatrix} 1 \\ 1 \end{pmatrix}$.
- **Step 1**:
  - Multiply: $\mathbf{y}_1 = A \mathbf{x}_0 = \begin{pmatrix} 3(1) + 1(1) \\ 0(1) + 2(1) \end{pmatrix} = \begin{pmatrix} 4 \\ 2 \end{pmatrix}$.
  - Normalize ($\infty$-norm): $\mathbf{x}_1 = \frac{1}{4}\begin{pmatrix} 4 \\ 2 \end{pmatrix} = \begin{pmatrix} 1 \\ 0.5 \end{pmatrix}$.
- **Step 2**:
  - Multiply: $\mathbf{y}_2 = A \mathbf{x}_1 = \begin{pmatrix} 3(1) + 1(0.5) \\ 0(1) + 2(0.5) \end{pmatrix} = \begin{pmatrix} 3.5 \\ 1.0 \end{pmatrix}$.
  - Normalize: $\mathbf{x}_2 = \frac{1}{3.5}\begin{pmatrix} 3.5 \\ 1.0 \end{pmatrix} = \begin{pmatrix} 1 \\ 1/3.5 \end{pmatrix} \approx \begin{pmatrix} 1 \\ 0.2857 \end{pmatrix}$.
- Notice the second component shrinks by factor $\frac{\lambda_2}{\lambda_1} = \frac{2}{3}$ each iteration: $1 \to 0.5 \to 0.2857 \to \dots \to 0$.
- The vector converges to the true dominant eigenvector $\begin{pmatrix} 1 \\ 0 \end{pmatrix}$ with eigenvalue $\lambda_1 = \mathbf{3}$.

### Where it's used
- **Google PageRank**: iterating probability flow on web-graph stochastic transition matrices.
- **Principal Component Analysis (PCA)**: finding highest-variance projection vectors on data covariance matrices.
- **Vibration resonance analysis**: computing the fundamental harmonic mode of mechanical structures.

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
