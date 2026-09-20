# Lesson 2 — The Shapes Data Comes In, and the One Operation That Moves It

Based on: Goodfellow, Bengio, Courville — *Deep Learning*, Chapter 2 (original explanation, not excerpted)

Lesson 1 ended on a claim: the raw-metrics → trend → subsystem-pattern chain is, mechanically, a matrix multiplication plus a small nonlinear tweak. Today we make that literal, with actual numbers run through actual arithmetic, still using the same incident classifier.

## Scalars, Vectors, Matrices, Tensors: How Many Indices You Need

Four words, one idea — "a container of numbers" — at increasing shape complexity. What changes is how many independent indices you need to point at one number inside it.

- **Scalar**: a single number. No index. A learning rate, a loss value.
- **Vector**: a 1-D list, indexed by one number. `v[3]` gets the 4th entry.
- **Matrix**: a 2-D grid, indexed by two numbers (row, column).
- **Tensor**: the general case, indexed by however many numbers you need. A vector is rank-1, a matrix rank-2.

Concretely, in our world: one minute's metrics reading is a **vector**.

```
minute_4 = [latency=1.2, error_rate=0.9, cpu=0.6, request_rate=4.0]
```

(same units as Lesson 1's trace: latency in hundreds of ms, error_rate in percent, cpu as a 0–1 fraction, request_rate in hundreds/min.) That's a rank-1 tensor with 4 entries — no more exotic than a `float[4]` in any language you already use.

Stack five consecutive minutes and you get a **matrix**, one row per minute, one column per metric — this is exactly Lesson 1's error-rate trace table, just with all four metrics side by side instead of one:

```
X = [[1.0, 0.1, 0.5, 3.8],    <- minute 0
     [1.0, 0.1, 0.5, 3.9],    <- minute 1
     [1.1, 0.3, 0.55, 4.0],   <- minute 2
     [1.15, 0.6, 0.58, 4.0],  <- minute 3
     [1.2, 0.9, 0.6, 4.0]]    <- minute 4

shape: (5, 4)   5 rows (minutes), 4 columns (metrics)
```

This is what a **batch** means, concretely: instead of running the network one minute at a time, you stack minutes into one matrix and push them all through the same arithmetic in one matrix multiplication. `X[4]` is minute 4's full metrics vector; `X[:, 1]` (column 1) is "the error rate at every minute in this window" — the same move as pulling a column out of a dataframe. Go one level further and stack many *services*, each contributing its own 5-minute × 4-metric matrix, and you get a rank-3 **tensor** of shape `(n_services, 5, 4)` — you now need three indices to find one number: `metrics[7, 4, 1]` is "service 7, minute 4, error rate." Same underlying idea, one more axis.

## Matrix Multiplication *Is* a Layer

Here's the mechanical core of Lesson 1's diagram. When a layer turns an input vector into an output vector, the transformation in between is a matrix multiplication.

**Before I explain — guess:** Lesson 1's layer 1 computed "trend" by subtracting `error_rate[t] − error_rate[t−2]` — one hand-picked formula for one metric. If a network had to *learn* how to turn all four raw metrics at once into two internal signals ("error-ish" and "resource-ish"), what single mathematical operation on the metrics vector would let it weigh and combine all four numbers into each output signal?

Take minute 4's vector, and say the layer wants to turn those 4 raw metrics into 2 internal signals — call them `error_signal` and `resource_signal`, standing in for Lesson 1's layer-1/layer-2 units. The layer stores a 4×2 weight matrix `W`, one column per output signal:

```
          error_signal   resource_signal
latency  [   -0.1            0.3   ]
error    [    1.0            0.0   ]
cpu      [   -0.1            0.5   ]
reqrate  [    0.05            0.2   ]
```

The output is `x @ W`, a (1×4) times a (4×2), giving (1×2). Each output number is a **dot product**: walk down the input vector and the matching column of `W` together, multiply pairwise, sum.

**error_signal** = (1.2×−0.1) + (0.9×1.0) + (0.6×−0.1) + (4.0×0.05)
&nbsp;&nbsp;&nbsp;&nbsp;= −0.12 + 0.9 − 0.06 + 0.2 = **0.92**

**resource_signal** = (1.2×0.3) + (0.9×0.0) + (0.6×0.5) + (4.0×0.2)
&nbsp;&nbsp;&nbsp;&nbsp;= 0.36 + 0 + 0.3 + 0.8 = **1.46**

So `x @ W = [0.92, 1.46]` for minute 4 — high error signal, matching the creeping-error minute from Lesson 1's trace.

Now the **batch**. Take minute 4 alongside an earlier, calmer minute:

```
X = [[1.2, 0.9, 0.6, 4.0],    <- minute 4 (creeping)
     [1.0, 0.1, 0.5, 3.8]]    <- minute 0 (normal)
```

Minute 4's row reproduces `[0.92, 1.46]` above. Minute 0 runs the *exact same* `W`, independently:

**error_signal** = (1.0×−0.1)+(0.1×1.0)+(0.5×−0.1)+(3.8×0.05) = −0.1+0.1−0.05+0.19 = **0.14**
**resource_signal** = (1.0×0.3)+(0.1×0)+(0.5×0.5)+(3.8×0.2) = 0.3+0+0.25+0.76 = **1.31**

```
X (2x4)                W (4x2)                output (2x2)
[[1.2, 0.9, 0.6, 4.0]    [-0.1  0.3]           [[0.92, 1.46],   <- minute 4
 [1.0, 0.1, 0.5, 3.8]]   [ 1.0  0.0]    x   =   [0.14, 1.31]]   <- minute 0
                         [-0.1  0.5]
                         [ 0.05 0.2]
```

Same weights, same dot-product arithmetic, per row, independently — and the numbers do exactly what Lesson 1 promised: the creeping-error minute lights up `error_signal` (0.92 vs 0.14) while `resource_signal` stays close for both, because neither minute has a resource problem. Stacking more minutes into `X` never changes the per-row computation — it just gives you more output rows, computed in parallel. This is the concrete meaning of "a layer": its parameters *are* a matrix (plus usually a small added bias vector, ignored for now), and applying the layer means multiplying.

## Why Multiplication, Not Just Elementwise Multiply

The natural instinct is "multiplying two arrays should multiply corresponding entries" — elementwise multiplication, which exists and is used elsewhere in deep learning, but it can't change the *shape* of information: it can't combine latency, error rate, cpu, and request rate into one new signal. `x * column1` elementwise gives `[-0.12, 0.9, -0.06, 0.2]` — four separate numbers, not the single `0.92` above.

Matrix multiplication is different specifically because it's a **weighted combination across the whole vector for each output value** — the dot product. `0.92` isn't "one metric scaled," it's latency, error rate, cpu, and request rate all pooled into one number via four multiplications and a sum. That's the mechanism that lets a layer express "this output is mostly error rate, a touch of latency and cpu, barely any request rate" — a genuinely new representation, not a rescaled old one.

### The dimension rule, felt rather than stated

It follows directly from what a dot product needs: two vectors of the *same length*, multiplied entry-by-entry and summed. `X` is (2×4), `W` is (4×2) — computing one output entry walks a row of `X` (length 4) against a column of `W` (length 4). The *inner* 4's must match; that's literally the walk. The outer numbers (2 and 2) just become the output's shape.

Try to break it: swap in a `Z` that's (3×2) instead of (4×2) — maybe sized for a layer with 3 inputs by mistake.

```
X (2x4)  x  Z (3x2)   ->  INVALID
      ^4       ^3
      inner dimensions don't match: 4 != 3
```

There's no operation to perform at all — you'd run out of `Z`'s column entries before finishing the walk across `X`'s row, with no rule for the leftover. Worth internalizing as a mental type-checker, the same reflex as checking a function's return type matches the caller's expectation.

```
(n x k)  times  (k x m)  =  (n x m)     <- valid: inner k's match
(n x k)  times  (p x q), k != p         <- invalid: no shared dimension to sum over
```

**Why GPUs are built for exactly this.** Every entry of that 2×2 output came from a disjoint slice of `X` and `W` — minute 4's row doesn't need minute 0's row to finish. That's about as embarrassingly parallel as numerical computation gets: no shuffle step, just independent workers each doing one small multiply-and-sum. Scale from 2 minutes to 2 million minutes across a whole fleet, and the *shape* of the parallelism doesn't change — which is why bigger batches on a GPU don't require rethinking the algorithm, just farming out more independent dot products.

### A five-line sanity check in code

```python
import numpy as np

x = np.array([1.2, 0.9, 0.6, 4.0])          # minute 4: [latency, error_rate, cpu, request_rate]
W = np.array([[-0.1, 0.3],
              [ 1.0, 0.0],
              [-0.1, 0.5],
              [ 0.05, 0.2]])                # 4 metrics -> 2 signals

print(x @ W)                         # -> [0.92  1.46], matches the hand computation

v = np.array([3, 4])
print(np.linalg.norm(v, ord=1))      # L1 norm -> 7.0
print(np.linalg.norm(v, ord=2))      # L2 norm -> 5.0
```

## Identity and Inverse: "Solving" a Linear System, Conceptually

The **identity matrix** is the matrix version of multiplying by 1 — 1s down the diagonal, 0s elsewhere, and multiplying anything by it leaves it unchanged. It matters mainly as the reference point for defining the inverse.

The **inverse** of a matrix `A`, written `A⁻¹`, "undoes" `A`: `A⁻¹ · A` gives the identity back. It's the tool for solving `A · x = b` for unknown `x`, given known `A` and `b`: `x = A⁻¹ · b`. Translate that to our world: if `A` were "how raw metrics combine into subsystem signals" and `b` were the observed signals, the inverse would reconstruct which raw metrics must have produced them. In practice, deep learning almost never inverts matrices directly — inverses are numerically fragile and not every matrix has one — but the *concept* (a linear transformation can, in principle, be reversed) matters for reasoning about whether a layer loses information, which is where determinants come in later.

## Norms: Measuring the "Size" of a Vector

A norm turns a vector into one non-negative number representing its "size" — the vector equivalent of `abs()` for a scalar.

```
v = [3, 4]

L1 norm = |3| + |4| = 3 + 4 = 7
L2 norm = sqrt(3^2 + 4^2) = sqrt(9 + 16) = sqrt(25) = 5
```

- **L1 norm**: sum of absolute values — **7**. Every unit of movement in any direction costs the same — a taxicab navigating city blocks: 3 over, 4 up, 7 blocks total.
- **L2 norm**: square root of sum of squares — **5**. Ordinary straight-line distance, Pythagoras generalized. `[3, 4]` is the classic 3-4-5 triangle: the L2 norm is literally "how long is the hypotenuse."

Why should a backend engineer care? Both reappear, unchanged, in two things every training pipeline has:

1. **Loss functions.** "How wrong was the prediction?" is usually the norm of `(prediction − actual)`. Our classifier predicted `[error_signal, resource_signal] = [0.92, 1.46]` for minute 4; say the true incident-strength target for that minute was `[1.0, 1.5]`. The error vector is `[0.08, 0.04]`, and its squared L2 length — `0.08² + 0.04² = 0.0064 + 0.0016 = 0.008` — *is* the mean-squared-error-style loss for that minute, scaled by however many entries you average over.
2. **Regularization.** Penalizing large weights (so the model doesn't overfit) is typically adding the L2 or L1 norm of the weight vector into the loss. L2 nudges all weights small; L1 tends to zero out some entirely — automatic feature selection, for free.

## Eigenvectors and Eigenvalues: Directions a Matrix Doesn't Bend

A matrix applied to a vector generally both stretches *and* rotates it. But for most matrices there are special directions that survive unrotated.

**Before I explain — guess:** when you multiply a matrix by a vector, does the output ever point in *exactly* the same direction as the input — just longer or shorter — or does multiplication always rotate it somewhere new?

Take the diagonal matrix `A = [[2, 0], [0, 3]]` and apply it to `[1, 0]`:

```
A * [1, 0] = [2*1 + 0*0, 0*1 + 3*0] = [2, 0] = 2 * [1, 0]
```

The output `[2, 0]` points in *exactly* the same direction as `[1, 0]` — just scaled by 2, no rotation. So `[1, 0]` is an **eigenvector** of `A`, with **eigenvalue** 2. Same story for `[0, 1]`: `A * [0, 1] = [0, 3] = 3 * [0, 1]`, eigenvalue 3. Try any other direction, say `[1, 1]`: `A * [1, 1] = [2, 3]`, which does *not* point the same way — it's bent toward the y-axis, since `A` stretches that axis harder. `[1,0]` and `[0,1]` are the only two directions (up to sign/scale) this matrix leaves unrotated, and they're the coordinate axes precisely because `A` is diagonal.

So your guess was right if you said "sometimes" — it depends on the direction and the matrix. Eigenvectors are the "grain of the wood": directions along which a matrix's effect is pure scaling. This is the machinery behind **PCA** — e.g. if you took every minute's 4-metric vector across a service's history and asked "which combination of latency/error/cpu/request-rate varies the most across normal-vs-incident minutes," the answer is an eigenvector of that data's covariance matrix. Filed for later; we're not computing PCA today, just recognizing the shape of the idea.

## The SVD: Any Matrix, Taken Apart Into Rotate → Scale → Rotate

Eigenvectors are only fully defined for square matrices. The **Singular Value Decomposition (SVD)** generalizes "find the well-behaved directions" to *any* matrix, of any shape: every matrix can be broken into rotate, then stretch along coordinate axes by some factors (singular values), then rotate again. Nothing about a linear transformation, however complicated it looks written out, is more exotic than "rotate, scale, rotate" — that's the workhorse behind compressing matrices, denoising data, and PCA itself. The useful takeaway is that this decomposition always exists, not how to compute it by hand.

## Vocabulary

| Term | Plain meaning |
|---|---|
| Tensor | A container of numbers indexed by however many dimensions you need; scalar/vector/matrix are the rank-0/1/2 special cases. |
| Batch | Multiple examples' vectors stacked into one matrix (extra leading axis), so one operation processes all of them at once instead of one at a time — e.g. several minutes' metrics stacked into one matrix. |
| Dot product | Multiply two equal-length vectors entry-by-entry and sum — the single arithmetic step every matrix-multiplication output entry is built from. |
| Identity matrix | The matrix "no-op" — multiplying by it changes nothing, like multiplying a scalar by 1. |
| Inverse | The matrix that undoes another matrix's transformation; used conceptually to "solve for" an unknown input given a known transformation and output. |
| Norm | A single number summarizing a vector's size (L1 = sum of absolute values, L2 = straight-line length); shows up directly in loss functions and regularization terms. |
| Eigenvector / eigenvalue | A direction a matrix only stretches (never rotates), and the factor by which it stretches it. |
| SVD | A decomposition proving any matrix can be expressed as rotate → scale → rotate; underlies PCA and matrix compression. |

---

## Quick check

In your own words: why can `x @ W` combine four different raw metrics into one "error subsystem" signal in a way that `x * w` (elementwise) can't? And if we wanted the layer to output 3 signals instead of 2, what exactly would need to change about `W`'s shape, and why?

## Where we'll go next

**Lesson 3 — Probability & Information Theory.** Once the classifier's layers produce signals like `[0.92, 1.46]`, we need a principled way to say how "surprised" or "wrong" a prediction should feel — that's probability and entropy, the missing piece behind loss functions like cross-entropy, still on the same metrics classifier.

Answer the check above (even roughly), then reply **ok** to continue.
