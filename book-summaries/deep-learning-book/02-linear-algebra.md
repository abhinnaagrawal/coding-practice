# Lesson 2 — The Shapes Data Comes In, and the One Operation That Moves It

Based on: Goodfellow, Bengio, Courville — *Deep Learning*, Chapter 2 (original explanation, not excerpted)

Lesson 1 ended on a claim: every layer in that pixels → edges → corners → cat chain is, mechanically, a matrix multiplication plus a small nonlinear tweak. Today we make that literal. Not a linear algebra course — just enough of the toolkit to read what a layer is actually doing when you look at its code, with actual numbers run through actual arithmetic so the mechanics stop being hand-wavy.

## Scalars, Vectors, Matrices, Tensors: It's a Question of How Many Indices You Need

These four words describe the same idea — "a container of numbers" — at increasing levels of shape complexity. The only thing that changes is how many independent indices you need to point at one number inside the container.

- **Scalar**: a single number. No index needed. A learning rate, a loss value.
- **Vector**: a 1-D list of numbers, indexed by one number. `v[3]` gets you the 4th entry.
- **Matrix**: a 2-D grid, indexed by two numbers (row, column). `M[2][5]`.
- **Tensor**: the general case — indexed by however many numbers you need. A vector is a rank-1 tensor, a matrix is rank-2. "Tensor" just means "we stopped counting dimensions and started calling it N."

Here's the part that actually matters for you: **these aren't abstract math objects in a deep learning system, they're specific things you already work with.** Let's make that literal with an actual dataset instead of describing it in the abstract.

### A concrete dataset: 4 houses, 3 features

Say you're predicting house prices from three features: square footage, number of bedrooms, and age in years. One house is a **vector**:

```
house_A = [1500, 3, 12]     # 1500 sqft, 3 bedrooms, 12 years old
```

That's a rank-1 tensor with 3 entries. `house_A[0]` is 1500, `house_A[2]` is 12. Nothing more exotic than a row in a CSV, or a `float[3]` in any language you already use.

Stack four houses and you get a **matrix** — a 4×3 grid, one row per house, one column per feature:

```
X = [[1500,  3, 12],     <- house A
     [2100,  4,  3],     <- house B
     [1200,  2, 25],     <- house C
     [1800,  3,  8]]     <- house D

shape: (4, 3)   4 rows (examples), 3 columns (features)
```

This is exactly what a **batch** means, concretely — not an abstract concept, just this: instead of feeding the network one house at a time (four separate 1×3 vectors, four separate passes through the math), you stack them into one 4×3 matrix and push all four through the same arithmetic in a single matrix multiplication. `X[0]` is house A's full feature vector; `X[:, 1]` (column 1) is "the bedroom count of every house in the batch," useful the same way pulling a single column out of a dataframe is. The batch size (4, here) is just the number of rows — in real training it's usually something like 32, 128, or 256, but the shape logic is identical whether it's 4 houses or 4 million.

Now go one level further, to a **tensor**. A batch of 64 color images, each 28×28 pixels with 3 color channels, is a rank-4 tensor with shape `(64, 28, 28, 3)` — batch, height, width, channel. You need all four indices to find one specific pixel intensity: `image_batch[5, 10, 10, 2]` is "in the 6th image of the batch, the pixel at row 10, column 10, blue channel." The house matrix and the image tensor are the same underlying idea — a schema plus a batch axis — just with more axes needed to describe one example.

```
scalar:  7.2
vector:  [1500, 3, 12]                     <- one house       shape (3,)
matrix:  [[1500, 3, 12],                   <- 4 houses,       shape (4, 3)
          [2100, 4, 3 ],
          [1200, 2, 25],
          [1800, 3, 8 ]]
tensor:  shape (64, 28, 28, 3)             <- 64 images, 28x28, 3 channels
```

If you've worked with multi-dimensional arrays, protobufs with nested repeated fields, or NumPy/pandas shapes, this is exactly that instinct — a tensor's "shape" is just its schema, and adding a batch axis is just adding a "list of" wrapper around a fixed-shape record.

## Matrix Multiplication *Is* a Layer

Here's the mechanical core of Lesson 1's diagram. When a network layer takes an input vector and produces an output vector, the transformation in between is a matrix multiplication. Let's not describe this — let's actually do the arithmetic.

Take one house, scaled to friendlier numbers so the arithmetic stays readable: square footage in thousands, bedrooms as-is, age in decades.

```
x = [1.5, 3, 1.2]      # [sqft/1000, bedrooms, age/10] for house A
```

Say the layer wants to turn those 3 input features into 2 internal signals — call them "size signal" and "age signal." The layer stores a 3×2 weight matrix `W`, where each *column* holds the weights for one output signal:

```
        size_signal   age_signal
sqft   [   0.5           -0.1  ]
bed    [   0.2            0.1  ]
age    [  -0.1            0.6  ]
```

The output is `x @ W`, a (1×3) times a (3×2), producing a (1×2). Each output number is a **dot product**: walk down the input vector and the matching column of `W` together, multiply each pair, and sum.

**size_signal** = (1.5 × 0.5) + (3 × 0.2) + (1.2 × −0.1)
&nbsp;&nbsp;&nbsp;&nbsp;= 0.75 + 0.6 − 0.12
&nbsp;&nbsp;&nbsp;&nbsp;= **1.23**

**age_signal** = (1.5 × −0.1) + (3 × 0.1) + (1.2 × 0.6)
&nbsp;&nbsp;&nbsp;&nbsp;= −0.15 + 0.3 + 0.72
&nbsp;&nbsp;&nbsp;&nbsp;= **0.87**

So `x @ W = [1.23, 0.87]`. That's it — that's the entire mechanical content of "a layer processes an input." Two output numbers means two columns of `W`, each producing one independent dot product; want a layer that produces 8 internal signals instead of 2, you just give `W` 8 columns instead of 2, and each one is computed the same way, independently.

Now watch what happens with a **batch**. Take the 2-house batch (scaled the same way):

```
X = [[1.5, 3, 1.2],     <- house A
     [2.1, 4, 0.3]]     <- house B (2100 sqft, 4 bed, 3 years old)
```

House A's row reproduces the dot products above: `[1.23, 0.87]`. House B's row runs the *exact same* `W`, independently:

**size_signal (B)** = (2.1 × 0.5) + (4 × 0.2) + (0.3 × −0.1) = 1.05 + 0.8 − 0.03 = **1.82**
**age_signal (B)** = (2.1 × −0.1) + (4 × 0.1) + (0.3 × 0.6) = −0.21 + 0.4 + 0.18 = **0.37**

```
X (2x3)          W (3x2)              output (2x2)
[[1.5, 3, 1.2]     [ 0.5 -0.1]        [[1.23, 0.87],    <- house A
 [2.1, 4, 0.3]]  x  [ 0.2  0.1]   =    [1.82, 0.37]]    <- house B
                    [-0.1  0.6]
```

Same weight matrix, same dot-product arithmetic, applied to every row independently. That's the whole content of "batching" at the linear-algebra level: stacking more rows into `X` doesn't change the computation per row at all, it just gives you more rows of output for free, computed in parallel. This is the concrete meaning of "a layer": a layer's parameters *are* a matrix (plus usually a small added vector, the bias — ignore that detail for now), and applying the layer to data means multiplying. Chain several of these together, each with its own weight matrix and a small nonlinearity squeezed in between, and you get Lesson 1's function-composition chain, now with actual mechanics attached.

## Why Multiplication, Not Just Elementwise Multiply

The natural instinct from an engineering background is "multiplying two arrays should mean multiplying corresponding entries" — that's elementwise multiplication, and it exists (it's used elsewhere in deep learning), but it is *not* what turns one representation into another. Elementwise multiply can't change the *shape* of information: a 3-feature input stays 3 numbers, no mixing occurs. It can't combine `sqft` and `age` into a single new signal — `[1.5, 3, 1.2] * [0.5, 0.2, -0.1]` elementwise gives you `[0.75, 0.6, -0.12]`, three separate numbers, not one combined "size signal."

Matrix multiplication is different specifically because it's a **weighted combination across the whole vector for each output value** — that's what the dot product does, and the worked example above shows it directly: `1.23` is not "one input scaled by one weight," it's `sqft` and `bedrooms` and `age` all pooled into a single number via three multiplications and a sum. That's the mechanism that lets a layer say "the output I care about is 0.5× the square footage, plus 0.2× the bedroom count, minus 0.1× the age" — a genuinely new representation, not a rescaled old one.

### The dimension rule, felt rather than stated

The dimension rule follows directly from what a dot product needs: to combine two vectors entry-by-entry and sum, they must be the *same length*. Look back at the batch example: `X` is (2×3), `W` is (3×2). Computing one output entry means walking across a row of `X` (length 3) against a column of `W` (length 3) — the *inner* 3's have to match, because that's literally the walk the dot product performs. The outer numbers (2 and 2) just become the output's shape; they never touch each other.

Now try to break it. Suppose instead of `W` (3×2) you had a matrix `Z` that's (2×2) — maybe you sized it for a different layer by mistake:

```
X (2x3)  x  Z (2x2)   ->  INVALID
      ^3       ^2
      inner dimensions don't match: 3 != 2
```

There's no way to even define this: computing the first output entry would mean walking across `X`'s row of length 3 against `Z`'s column of length 2 — you'd run out of entries in one before the other, with no rule for what to do with the leftover. It's not that the answer is wrong, there's no operation to perform at all. This is worth internalizing as a mental type-checker: before touching the arithmetic, check the inner dimensions line up, the same reflex as checking that a function's return type matches what the caller expects, or that a protobuf field number wasn't reused for an incompatible type.

```
(n x k)  times  (k x m)  =  (n x m)     <- valid: inner k's match
        ^inner dims must match

(n x k)  times  (p x q), k != p         <- invalid: no shared dimension to sum over
```

**Why GPUs are built for exactly this.** Look again at what computing one output matrix requires: a whole grid of independent dot products, one per (row, column) pair of the output, and *none of them depend on each other*. In the 2×2 output above, the [house A, size_signal] entry and the [house B, age_signal] entry were computed from completely disjoint slices of `X` and `W` — house B's row doesn't need to know what house A's row produced. That's about as embarrassingly parallel as numerical computation gets — it's the same shape of problem as a distributed map-reduce with no shuffle step, just independent workers each doing a small multiply-and-sum and writing to their own output slot. A CPU core is optimized for doing one complex sequential thing very fast; a GPU is thousands of small, weak cores optimized for doing the *same simple thing* to *many independent pieces of data* at once. Matrix multiplication is close to the ideal workload for that architecture, which is why it — not some other operation — became the thing GPUs got repurposed for. Scale the house batch from 2 rows to 2 million rows, and the *shape* of the parallelism story doesn't change at all — that's the whole reason training on bigger batches on a GPU doesn't require rethinking the algorithm, just farming out more independent dot products.

### A five-line sanity check in code

Since this is arithmetic you can actually run, here it is in NumPy — the two things above (a matrix multiply, and the norms coming up next) in one snippet:

```python
import numpy as np

x = np.array([1.5, 3, 1.2])          # house A: [sqft/1000, bedrooms, age/10]
W = np.array([[ 0.5, -0.1],
              [ 0.2,  0.1],
              [-0.1,  0.6]])         # 3 inputs -> 2 outputs

print(x @ W)                         # -> [1.23  0.87], matches the hand computation

v = np.array([3, 4])
print(np.linalg.norm(v, ord=1))      # L1 norm -> 7.0
print(np.linalg.norm(v, ord=2))      # L2 norm -> 5.0
```

Running this is worth doing once just to see `x @ W` spit out `[1.23, 0.87]` and confirm the hand arithmetic above wasn't a trick.

## Identity and Inverse: "Solving" a Linear System, Conceptually

Two special matrix ideas worth having, without the proof machinery:

The **identity matrix** is the matrix version of multiplying by 1 — it has 1s down its diagonal and 0s everywhere else, and multiplying anything by it leaves that thing unchanged. It's the "no-op" of matrix multiplication, and it matters mainly as a reference point for defining the inverse.

The **inverse** of a matrix `A`, written `A⁻¹`, is the matrix that "undoes" whatever `A` did: `A⁻¹ · A` gives you the identity back. Why care? Because it's the tool for solving `A · x = b` for an unknown vector `x` when you know `A` and `b` — conceptually, `x = A⁻¹ · b`. Translate that to something familiar: if `A` encodes "here's how features combine to produce outcomes" and `b` is the outcomes you observed, the inverse is the operation that reconstructs which input must have produced them. In practice, deep learning almost never inverts matrices directly — inverses are numerically fragile and expensive at scale, and not every matrix even has one — but the *concept* (there exists an operation that reverses a linear transformation) underlies why we can reason about a network's transformations as invertible or not, and shows up later when you hit determinants and why some transformations "collapse" information irreversibly.

## Norms: Measuring the "Size" of a Vector

A norm is just a rule for turning a vector into a single non-negative number that represents its "size" or "length" — the vector equivalent of `abs()` for a scalar.

Two you'll see constantly — computed side by side on the same small vector so the difference is visible, not just described:

```
v = [3, 4]

L1 norm = |3| + |4| = 3 + 4 = 7
L2 norm = sqrt(3^2 + 4^2) = sqrt(9 + 16) = sqrt(25) = 5
```

- **L1 norm**: sum of the absolute values of all entries — here, **7**. Treats every unit of movement in any direction as equally costly — think "total distance traveled if you can only move along grid lines," like a taxicab navigating city blocks: 3 blocks over, 4 blocks up, 7 blocks total.
- **L2 norm**: square root of the sum of squares — here, **5**. This is ordinary straight-line distance — the Pythagorean theorem generalized to more dimensions. `[3, 4]` isn't a random choice: it's the classic 3-4-5 right triangle, so the L2 norm is literally "how long is the hypotenuse" — the straight-line distance from the origin to the point `(3, 4)`, exactly as it would be on a map. That's the "aha" worth holding onto: L2 norm of a difference vector is just ordinary geometric distance between two points, dressed up in vector notation.

Why should a backend engineer care about a distance formula? Because both of these reappear, unchanged, as the backbone of two things you'll hit in every training pipeline:

1. **Loss functions.** "How wrong was the model's prediction?" is usually answered by taking the norm of the (prediction − actual) vector. If a model predicts `[house price signals] = [1.23, 0.87]` and the true target was `[1.0, 1.0]`, the error vector is `[0.23, -0.13]`, and its squared L2 length — `0.23² + 0.13² = 0.0529 + 0.0169 = 0.0698` — *is* the mean-squared-error-style loss you've probably heard of, scaled by however many entries you're averaging over.
2. **Regularization.** Penalizing a model for having large weights (to stop it from overfitting) is typically done by adding the L2 norm (or L1 norm) of the weight vector into the loss. L2 regularization nudges all weights to be small-ish; L1 regularization tends to push some weights all the way to zero, which is why it's associated with automatic feature selection — a nice engineering-adjacent fact: L1 gives you sparsity "for free."

## Eigenvectors and Eigenvalues: Directions a Matrix Doesn't Bend

A matrix, applied to a vector, generally both stretches *and* rotates it — the output points in a different direction than the input did. But for most matrices, there are a handful of special directions that survive unrotated: feed in a vector pointing that way, and the output points in *exactly the same direction*, just longer or shorter. That special direction is an **eigenvector**; the amount it got stretched (or shrunk, or flipped) by is its **eigenvalue**.

The simplest possible example makes this exact rather than hand-wavy. Take the diagonal matrix:

```
A = [[2, 0],
     [0, 3]]
```

Apply it to the vector `[1, 0]` (pointing straight along the x-axis):

```
A * [1, 0] = [2*1 + 0*0, 0*1 + 3*0] = [2, 0] = 2 * [1, 0]
```

The output, `[2, 0]`, points in *exactly* the same direction as the input `[1, 0]` — it just got scaled by 2. No rotation at all. So `[1, 0]` is an eigenvector of `A`, with eigenvalue **2**.

Now try `[0, 1]` (straight along the y-axis):

```
A * [0, 1] = [2*0 + 0*1, 0*0 + 3*1] = [0, 3] = 3 * [0, 1]
```

Same story: output `[0, 3]` is just the input scaled by 3, no rotation. So `[0, 1]` is an eigenvector with eigenvalue **3**. Try any *other* direction, say `[1, 1]`: `A * [1, 1] = [2, 3]`, which does *not* point the same way as `[1, 1]` — it got bent toward the y-axis, since the matrix stretches that axis harder (factor 3) than the x-axis (factor 2). `[1,0]` and `[0,1]` are the only two directions (up to sign and scale) this particular matrix leaves unrotated, and they happen to be the coordinate axes precisely because `A` is diagonal — that's not a coincidence, it's the reason diagonal matrices are the easiest case to reason about.

That's the whole intuition, now anchored to real numbers: eigenvectors are the "grain of the wood" for a matrix — the directions along which its effect is pure scaling, with no rotation mixed in. Everything else gets bent to some combination of these directions.

You don't need the full computation for non-diagonal matrices here, just the forward pointer: this idea is the machinery behind **PCA** (finding the directions along which your data varies the most, used for compressing/visualizing high-dimensional data), and it's also how researchers reason about the "shape" of the loss landscape a network is being optimized over — whether a point is a stable valley or a saddle depends on the eigenvalues of a matrix describing the local curvature there. Filed for later; we're not proving anything about the general case today.

## The SVD: Taking Any Matrix Apart Into Rotate → Scale → Rotate

Eigenvectors are defined for a limited class of matrices, and only tell the full story for square ones. The **Singular Value Decomposition (SVD)** generalizes the same "find the well-behaved directions" idea to *any* matrix, of any shape. It says every matrix, no matter how it warps space, can be broken into exactly three simpler steps chained together: rotate, stretch along the coordinate axes by some set of factors (the singular values), then rotate again. Nothing about a linear transformation, however complicated it looks written out as a matrix, is more exotic than "rotate, scale, rotate" — the SVD is the guarantee of that, and the decomposition it produces is the workhorse behind compressing matrices, denoising data, and (not coincidentally) PCA itself. We'll leave the mechanics there; the useful takeaway is that this decomposition exists and is trustworthy, not how to compute it by hand.

## Vocabulary

| Term | Plain meaning |
|---|---|
| Tensor | A container of numbers indexed by however many dimensions you need; scalar/vector/matrix are just the rank-0/1/2 special cases. |
| Batch | Multiple examples' vectors stacked into one matrix (extra leading axis), so the same operation processes all of them in one pass instead of one at a time. |
| Dot product | Multiply two equal-length vectors entry-by-entry and sum the results — the single arithmetic step that every entry of a matrix multiplication output is built from. |
| Identity matrix | The matrix "no-op" — multiplying by it changes nothing, analogous to multiplying a scalar by 1. |
| Inverse | The matrix that undoes another matrix's transformation; used conceptually to "solve for" an unknown input given a known transformation and output. |
| Norm | A single number summarizing a vector's size (L1 = sum of absolute values, L2 = straight-line length); shows up directly in loss functions and regularization terms. |
| Eigenvector / eigenvalue | A direction a matrix only stretches (never rotates), and the factor by which it stretches it. |
| SVD | A decomposition proving any matrix can be expressed as rotate → scale → rotate; underlies PCA and matrix compression. |

---

## Where we'll go next
**Lesson 3 — Probability & Information Theory.** Once a network makes a prediction, we need a principled way to say how "surprised" or "wrong" it should feel — that's where probability and entropy come in, and it's the missing piece behind loss functions like cross-entropy.

Reply **ok** to continue, or ask anything about today's lesson first.
