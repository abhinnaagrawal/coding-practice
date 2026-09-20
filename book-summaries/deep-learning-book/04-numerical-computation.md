# Lesson 4 — When the Math Meets the Machine

Based on: Goodfellow, Bengio, Courville — *Deep Learning*, Chapter 4 (original explanation, not excerpted)

Lessons 2 and 3 gave you the linear algebra and probability behind our running classifier — the one that looks at a service's per-minute metrics (latency, error rate, CPU, request rate) and decides normal, warning, or incident. All of that math assumed exact, infinite-precision numbers. Your GPU doesn't have those. This lesson is the gap between the two: why that classifier's training can silently produce `NaN`, why one line of code can make or break a formula that's mathematically fine on paper, and why "gradient descent" — the thing that actually tunes the classifier's weights — is about to become the central idea of the rest of this book.

You already have the right instincts here. It's the same bug class as an `int32` counter wrapping after two billion increments, or `0.1 + 0.2` landing on `0.30000000000000004`. Finite bits standing in for infinite numbers, and someone forgetting that substitution isn't free.

## Floating Point Is Consensus With Rounding Errors

Floating point is a lot like a distributed system reaching "good enough" agreement instead of perfect agreement: a `float32` stores a compressed *approximation* (sign, exponent, fraction), dense near common magnitudes and sparse everywhere else. The failure modes have names: **underflow** (a value so small it rounds to exactly zero — then a later divide or log gives you a crash or `NaN`) and **overflow** (a value so large it becomes a special "infinity," poisoning everything computed from it). Both are common in ML because so much of it runs through `exp` and `log` — smooth on paper, brutal once clipped to a fixed number of bits.

To make "so large it can't be represented" concrete: `float64` tops out around `1.8 × 10^308`, but `exp(x)` eats through that headroom fast — it crosses the ceiling once `x` exceeds about `709` (`ln(1.8×10^308) ≈ 308 × ln(10) ≈ 709.2`). `float32`'s ceiling, `~3.4 × 10^38`, gets crossed by `exp(x)` once `x` exceeds about `88`. Keep `709` in your head — a lot of "why is my loss `NaN`" bugs quietly cross it.

## The Softmax Example, Reworked for Our Classifier

**Before I explain — guess:** our classifier just produced this minute's raw scores for its three classes — normal: `1000`, warning: `999`, incident: `998` (softmax will turn these into probabilities). All three numbers are large but close together. Do you think handing them straight to `exp()` is safe, or does something break?

Softmax turns raw scores (*logits*) into a probability distribution:

```
softmax(z)_i = exp(z_i) / sum_j exp(z_j)
```

That's exact math, and our scores `z = [1000, 999, 998]` describe a perfectly reasonable minute — the classifier leans "normal" over "warning" and "incident" by a modest, predictable margin. But `exp(1000)` is on the order of `10^434` — nowhere near representable, given the `709` ceiling above. The instant you call `exp()` on that score, you get infinity, and the sum, the division, the loss, the gradient all become `NaN`. Your dashboard doesn't say "numerical bug" — it just says the incident-classifier's loss is `nan`, and you spend an afternoon suspecting the data pipeline instead.

The fix: subtract the max logit from every entry before exponentiating.

```
m = max(z)
softmax(z)_i = exp(z_i - m) / sum_j exp(z_j - m)
```

Run it on our actual minute. `m = 1000`, so `z - m = [0, -1, -2]`. Exponentiate: `exp(0) = 1`, `exp(-1) ≈ 0.3679`, `exp(-2) ≈ 0.1353`. Sum `= 1.5032`. Divide each by the sum:

- normal: `1 / 1.5032 ≈ 0.6652`
- warning: `0.3679 / 1.5032 ≈ 0.2447`
- incident: `0.1353 / 1.5032 ≈ 0.0900`

That's `66.52%` normal, `24.47%` warning, `9.00%` incident for this minute — summing to `1.0000`, as a distribution must. It's also the *same* distribution the unshifted math would have produced: the ratio `exp(1000)/exp(999) = exp(1) ≈ 2.718` matches `0.6652/0.2447 ≈ 2.719`. Subtracting a constant `m` from every logit before exponentiating multiplies numerator and every denominator term by `exp(-m)`, which cancels in the division — same answer, but now every intermediate number lives between `0` and `1` instead of demanding `10^434`. Same math, safer arithmetic path — the whole chapter in one line of code.

If you've ever rewritten a query to dodge a column's numeric-precision ceiling, or reordered a running sum to avoid an overflow before a cast, it's the same discipline, with `exp()` standing in for `SUM()`.

## Poor Conditioning: Small Input Noise, Big Output Noise

Overflow is about one unrepresentable number. **Ill-conditioning** is subtler: a function where a *tiny* input error — the kind rounding always introduces — gets *amplified* into a large output error, even though every arithmetic step was exact. The book's **condition number** captures this for matrices: roughly, how differently a matrix stretches its most- and least-amplified directions.

Two tiny linear systems make this concrete. Well-conditioned:

```
x + y = 2        →  x = 1, y = 1
x - y = 0
```

Nudge the right-hand side from `0` to `0.0001`: solving again gives `x = 1.00005`, `y = 0.99995`. A `0.0001` input wobble produced a `0.00005` output wobble — proportional, unremarkable. Now the ill-conditioned twin, where the two equations are nearly saying the same thing:

```
x + y = 2              →  x = 1, y = 1
x + 1.0001y = 2.0001
```

Nudge the same right-hand side by the same `0.0001`, to `2.0002`: now `y = 2` and `x = 0`. A `0.005%` change in one input flipped the solution entirely. The rows `(1, 1)` and `(1, 1.0001)` are nearly parallel — nearly redundant constraints — so the system's determinant (`0.0001`) is barely off from zero, and that thin margin is all that's pinning the answer down; rounding-sized noise swamps it. This matters for deep learning because matrix inversions, decompositions, and deep stacks of layers all inherit whatever conditioning their underlying matrices have — it's positive feedback for rounding error, the numerical equivalent of a retry storm turning one slow node into a cascading outage.

## Gradient Descent: Tuning the Classifier, One Step at a Time

Strip away the architecture diagrams and our classifier is a function with many adjustable numbers (weights) and one scalar you care about: a **loss** measuring how badly it's currently separating incident minutes from normal ones. Training searches that weight space for a setting that makes the loss small — there's no algebra that hands you the answer directly, so you improve iteratively, and the **gradient** — the vector pointing toward steepest *increase* — tells you which direction *not* to go. Gradient descent: compute the gradient, step in the opposite direction, repeat, thousands of times.

The mental picture is a hiker descending a foggy mountain with no map — feel the local slope, step downhill, repeat. You never see the whole landscape, only local information, and yet you make real progress.

Let's watch it move a single classifier weight, using the toy loss `f(w) = w^2` (minimum at `w = 0`) as a stand-in for "how far off the incident-score contribution of this one weight currently is." Its slope is `2w`. Start at `w = 10`:

**Learning rate 0.1 (converges):**

| Step | w before | gradient = 2w | w after |
|---|---|---|---|
| 1 | 10.0 | 20.0 | 8.0 |
| 2 | 8.0 | 16.0 | 6.4 |
| 3 | 6.4 | 12.8 | 5.12 |
| 4 | 5.12 | 10.24 | 4.096 |
| 5 | 4.096 | 8.192 | 3.2768 |

Each value is `0.8×` the last, because `w_new = w_old - 0.1×2w_old = 0.8×w_old`. Repeated multiplication by `0.8` shrinks toward zero — after 10 steps, `w ≈ 1.07`. That's the weight settling toward a value that better separates incident from normal minutes.

**Learning rate 1.1 (diverges):**

| Step | w before | gradient = 2w | w after |
|---|---|---|---|
| 1 | 10.0 | 20.0 | −12.0 |
| 2 | −12.0 | −24.0 | 14.4 |
| 3 | 14.4 | 28.8 | −17.28 |
| 4 | −17.28 | −34.56 | 20.736 |

Here `w_new = -1.2×w_old` — a multiplier bigger than 1 in magnitude and negative, so the weight flips sign and grows every step. Same update rule, same gradient formula; only the step size changed, and it's the entire difference between the classifier's weight settling down and it oscillating further from useful every step. Too small a learning rate and training crawls; too large and it overshoots or diverges outright — one of the most consequential knobs in the whole training process.

*(For a second, geometric picture: `f(x,y) = x^2+y^2` is a symmetric bowl, and gradient descent on it always steps perpendicular to the current contour ring, straight toward the center — same fog-and-mountain intuition, just with a second axis to turn along.)*

## Saddle Points: When Two Weights Disagree

**Before I explain — guess:** suppose you're adjusting *two* of the classifier's weights at once — say, the weight on error-rate and the weight on CPU — and you reach a point where the gradient is exactly zero for both. Does that guarantee you've found the best possible setting of those two weights, or could something else be going on?

Gradient descent stalls wherever the local slope is zero — a **critical point**. The hoped-for case is a **local minimum**: every direction leads back uphill. But a critical point can also be a **saddle point** — flat right there, but curving *up* in some directions and *down* in others, like a mountain pass. Take `f(x, y) = x^2 - y^2`, gradient `(2x, -2y)`, zero only at the origin. Walk along `x` (hold `y=0`): `f = x^2`, curving up — a bowl. Walk along `y` (hold `x=0`): `f = -y^2`, curving down — a dome. So the origin is flat, but not a minimum (the `y` direction has lower loss nearby) and not a maximum either (the `x` direction has higher loss nearby). It's neither, and gradient descent slows to a crawl approaching it, because the gradient shrinks toward zero without anything worth stopping at.

Translate that to our classifier: if nudging the error-rate weight would improve separation (curves like the `x` axis, upward — worse loss elsewhere) while nudging the CPU weight in the analogous way would *also* improve things (curves like the `y` axis, downward from the current point), a point where both partial slopes read zero simultaneously isn't necessarily "done" — it can be a saddle, not a true minimum. And the reason this matters more as weight count grows: for a flat point to be a genuine minimum, the surface must curve upward in *every* one of the classifier's many weight-directions at once — increasingly unlikely as dimensions pile up. Saddle points, not bad local minima, are the more common obstacle in realistically-sized networks; the practical danger is slow crawling through a saddle region, not getting permanently trapped in a bad valley.

## A Brief Nod to Smarter Methods

Gradient descent only uses first-order information — slope. Methods using **second-order information** — curvature, captured by the **Hessian** matrix of second derivatives — can pick a smarter direction and step size in one move (**Newton's method**), instead of hand-tuning a learning rate the way we did above. The catch: computing and inverting a Hessian scales with the *square* of the parameter count, a non-starter once the classifier (or any real network) has millions of weights — which is exactly why the field leans on cheap, first-order, gradient-only methods, and why Part II devotes a whole chapter to variants of gradient descent rather than to second-order methods.

## Vocabulary

| Term | Plain meaning |
|---|---|
| Underflow / overflow | A value too small (rounds to zero) or too large (becomes infinity) to fit in a finite-precision format, often silently corrupting everything computed from it. |
| Numerically stable computation | An implementation chosen to avoid intermediate values that overflow, underflow, or blow up — same math, safer arithmetic path. |
| Condition number | How much a function (especially a matrix) amplifies small input errors into output errors; high means rounding noise can flip the answer. |
| Ill-conditioned function | A function where small input errors get amplified into large output errors, even with exact arithmetic at every step. |
| Gradient | The direction of steepest increase of a function at a point; used in reverse to decrease a loss. |
| Gradient descent | Repeatedly stepping opposite the gradient to shrink a loss toward a minimum. |
| Learning rate | The scalar sizing each gradient-descent step; too small stalls training, too large causes overshoot or divergence. |
| Critical point | A point where the gradient is zero — could be a local minimum, local maximum, or saddle point. |
| Saddle point | A critical point curving up in some directions and down in others — flat locally, not a true minimum; the more common obstacle at high dimension. |
| Hessian | The matrix of second derivatives, capturing local curvature; expensive at deep-learning scale. |
| Newton's method | A second-order optimizer using the Hessian for a smarter step, at much higher cost per step. |

---

## Quick check

Say the classifier's training has two weights sitting at a point where the gradient is exactly zero for both — but nudging the error-rate weight up would *raise* the loss, while nudging the CPU weight up would *lower* it. In your own words: is this a local minimum, and what would you expect gradient descent to do if it lands near this point?

## Where we'll go next

**Lesson 5 — Machine Learning Basics I.** With gradients and gradient descent on the table, we can finally define what "learning" formally means: tasks, performance measures, and the train/test split that turns optimization into machine learning — still using this same metrics classifier as the running example.

Answer the check above (even roughly), then reply **ok** to continue.
