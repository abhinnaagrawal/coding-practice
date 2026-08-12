# Lesson 4 — When the Math Meets the Machine

Based on: Goodfellow, Bengio, Courville — *Deep Learning*, Chapter 4 (original explanation, not excerpted)

Lessons 2 and 3 gave you the linear algebra and probability that deep learning is built from — vectors, matrices, distributions, expectations. All of that math was written as if numbers were exact: real numbers, infinite precision, no rounding. That's the world mathematicians live in. It is not the world your GPU lives in. This lesson is about the gap between the two, and why that gap is not a footnote — it's the reason training runs silently produce `NaN`, why certain formulas that are mathematically identical behave completely differently in code, and why "gradient descent" is about to become the single most important idea in the rest of this book.

You already have the right instincts for this topic. It's the same category of bug as an `int32` counter wrapping around after two billion increments, or a financial system that computes tax as `0.1 + 0.2` and gets `0.30000000000000004`. Different domain, identical root cause: a finite number of bits standing in for an infinite set of numbers, and someone forgetting that substitution isn't free.

## Floating Point Is Consensus With Rounding Errors

Here's a framing that should land immediately for you: floating-point arithmetic is a lot like a distributed system that reaches "good enough" agreement instead of perfect agreement. A `float32` doesn't store a real number — it stores a compressed *approximation*, spending its bits on a sign, an exponent, and a fixed-size fraction. Just like a distributed consensus protocol trades perfect global truth for a fast, practical agreement that's correct enough almost all of the time, floating point trades perfect numerical truth for a representation that's dense enough near common magnitudes and sparse everywhere else.

The word "almost" is where the bugs live. In a distributed system, the failure mode is a stale read or a split-brain during a rare partition. In floating point, the failure modes have names too:

- **Underflow** — a number so small it rounds down to exactly zero. If your code later divides by that value, or takes its logarithm, you don't get "a small error," you get a crash or a `NaN`.
- **Overflow** — a number so large it can't be represented at all, and gets replaced by a special "infinity" value. Anything computed from infinity onward is usually garbage.

Neither of these is exotic. They show up constantly in ML because a huge fraction of the operations involve exponentials (`exp`) and logarithms — functions that are extremely well-behaved on paper and extremely badly-behaved once you clip them to a fixed number of bits.

To make "so large it can't be represented" concrete: a `float64` (double precision) tops out at roughly `1.8 × 10^308`. That sounds enormous, and it is — but `exp()` grows so aggressively that it eats through that headroom faster than intuition suggests. `exp(x)` crosses the `float64` ceiling once `x` exceeds about `709`, because `ln(1.8 × 10^308) ≈ 308 × ln(10) ≈ 308 × 2.3026 ≈ 709.2`. So any computation that calls `exp()` on a number bigger than roughly 709 overflows *even in double precision*, and `float32` gives you far less room — its ceiling is around `3.4 × 10^38`, which `exp()` blows past once the input exceeds about `88`. Keep that number, 709, in your head; it's the line a lot of "why is my loss `NaN`" bugs quietly cross.

## The Softmax Example: A Bug You Can Reproduce in Your Head

This is the clearest "aha" in the whole chapter, so let's build it concretely, with real numbers you can check on a calculator.

Softmax turns a vector of raw scores (called *logits*) into a probability distribution — the last step in almost every classifier. The textbook formula for the *i*-th output, given a vector of scores `z`, is:

```
softmax(z)_i = exp(z_i) / sum_j exp(z_j)
```

That's exact math. Now watch what happens when you implement it literally. Suppose your model produces a fairly ordinary-looking logit vector: `[1000, 999, 998]`. On paper, softmax of that vector is perfectly well defined — it's just a distribution that favors the first score over the other two by a modest, predictable margin. But `exp(1000)` is astronomically past the `709` overflow line we just established — it's on the order of `10^434`, a number no standard floating-point type can hold. The moment you call `exp()` on that logit, you get infinity, and every downstream number derived from it — the sum, the division, the loss, the gradient — becomes `NaN`. Your training run doesn't error out with a helpful stack trace; the loss just goes to `nan` and stays there, and if you didn't know this chapter existed, you'd spend an afternoon suspecting your data pipeline instead of a one-line numerical bug.

The fix is one line, and it's worth internalizing because the *trick*, not just the formula, reappears constantly in numerical ML code: subtract the maximum value in the vector from every entry before exponentiating.

```
m = max(z)
softmax(z)_i = exp(z_i - m) / sum_j exp(z_j - m)
```

Let's actually run the numbers instead of waving at them. Take `z = [1000, 999, 998]`. The max is `m = 1000`. Subtracting it from every entry gives `z - m = [0, -1, -2]`. Now exponentiate:

- `exp(0) = 1`
- `exp(-1) ≈ 0.3679`
- `exp(-2) ≈ 0.1353`

Sum of those three: `1 + 0.3679 + 0.1353 = 1.5032`. Dividing each term by that sum gives the softmax probabilities:

- `1 / 1.5032 ≈ 0.6652`
- `0.3679 / 1.5032 ≈ 0.2447`
- `0.1353 / 1.5032 ≈ 0.0900`

Those three numbers sum to `1.0000` (within rounding), which is exactly what a probability distribution has to do. Notice something important: these are the *exact same probabilities* softmax was always going to produce for `[1000, 999, 998]` — you can verify it without ever computing `exp(1000)` by checking the *ratios*. The unshifted math says the ratio between output 1 and output 2 should be `exp(1000) / exp(999) = exp(1000 - 999) = exp(1) ≈ 2.718`. Check the shifted numbers: `0.6652 / 0.2447 ≈ 2.719`. Same ratio, same distribution, computed with numbers no bigger than `1`. Why is the shift legal at all? Because subtracting a constant `m` from every logit before exponentiating is algebraically the same as multiplying both the numerator and every term in the denominator's sum by `exp(-m)` — a value that appears identically top and bottom and therefore cancels out completely when you divide. You've taken a computation that used to demand representing `10^434` and rewritten it, with zero loss of mathematical meaning, to only ever need numbers between roughly zero and one. Same answer, wildly different numerical behavior. This is the whole chapter in miniature: the *mathematical* function is fine, the *literal implementation* of that function is not, and the difference is entirely about how the intermediate values happen to interact with a finite number format.

If you've ever rewritten a query to avoid an intermediate result that blows past a column's numeric precision, or reordered a running-sum calculation to dodge integer overflow before a cast, this is the same discipline, just with `exp()` in place of `SUM()`.

## Poor Conditioning: When Small Input Noise Becomes Big Output Noise

Overflow and underflow are about a single number being unrepresentable. Poor conditioning is a subtler problem: a function where a *tiny* error in the input — the kind rounding always introduces — gets *amplified* into a large error in the output, even though every individual arithmetic step was performed as accurately as the hardware allows.

Think of it as a signal-amplification problem rather than a signal-loss problem. Some functions act like a low-noise amplifier: perturb the input slightly, the output moves by roughly the same slight amount. Others act like a circuit balanced right at the edge of instability: perturb the input by a rounding error's worth of noise, and the output swings wildly. A function with that second property is called **ill-conditioned**. The book quantifies this for matrices with the **condition number** — loosely, how much the *ratio* between the matrix's largest and smallest stretching factors (its eigenvalues) differs from 1. A well-conditioned matrix stretches every direction of space by roughly similar amounts; an ill-conditioned one stretches some directions enormously and others almost not at all, so inverting it (or solving a linear system with it) turns small input rounding error into large output error.

Let's make that concrete with two tiny systems of linear equations — small enough to solve by hand, which is exactly the point.

**Well-conditioned system.** Take the equations:

```
x + y = 2
x - y = 0
```

Solving directly: adding the two equations gives `2x = 2`, so `x = 1`, and then `y = 1`. Now perturb the second equation's right-hand side by a tiny amount, from `0` to `0.0001`:

```
x + y = 2
x - y = 0.0001
```

Adding again: `2x = 2.0001`, so `x = 1.00005`, and `y = 2 - 1.00005 = 0.99995`. A change of `0.0001` in the input produced a change of about `0.00005` in the output. Small perturbation in, small perturbation out — that's what "well-conditioned" feels like. Geometrically, the two rows of this system, `(1, 1)` and `(1, -1)`, point in very different directions (they're actually perpendicular), so the system pins down `x` and `y` firmly from two genuinely independent pieces of information.

**Ill-conditioned system.** Now take a system that *looks* almost identical, except the two rows point in nearly the same direction:

```
x + y = 2
x + 1.0001y = 2.0001
```

Solve it: subtract the first equation from the second, giving `0.0001y = 0.0001`, so `y = 1`, and then `x = 2 - 1 = 1`. Same solution as before, `x = 1, y = 1`. But now perturb the second equation's right-hand side by the same tiny amount, from `2.0001` to `2.0002`:

```
x + y = 2
x + 1.0001y = 2.0002
```

Subtracting again: `0.0001y = 0.0002`, so `y = 2`, and `x = 2 - 2 = 0`. Look at what just happened: a change of `0.0001` in one input number — a relative change of about `0.005%` — moved `y` from `1` to `2` (a 100% change) and `x` from `1` to `0`. The exact same size of nudge that caused a barely-noticeable wobble in the well-conditioned system caused a total change of solution here. The reason is visible in the rows themselves: `(1, 1)` and `(1, 1.0001)` are nearly parallel — nearly redundant statements about `x` and `y` rather than two independent constraints — so the system is close to being unsolvable altogether (its determinant is `1×1.0001 - 1×1 = 0.0001`, barely different from zero, the mathematical signature of near-singularity). When two rows of a matrix are almost telling you the same thing, the tiny amount by which they differ is doing all the work of pinning down the answer, and rounding error is exactly the kind of tiny perturbation that swamps that thin margin.

This matters for deep learning very directly: many of the operations under the hood — solving least-squares problems, inverting or decomposing matrices, propagating values through many stacked layers — are exactly the kind of linear-algebra operations whose conditioning determines whether your results are trustworthy or numerically mush. It's the same intuition as a distributed retry storm: a system that's supposed to be a small negative-feedback loop but is actually configured as positive feedback turns one slow node into a cascading outage. Ill-conditioning is positive feedback for rounding error.

## Gradient Descent: The Single Idea Behind Almost All Training

Now the part that matters for everything that follows in this book.

A neural network, once you strip away the architecture diagrams, is a function with a huge number of adjustable numbers (the weights) and a single scalar output you care about: a **loss** measuring how wrong the network currently is. Training means: search over that enormous space of weight settings for one that makes the loss small. You cannot solve for the answer directly — there's no algebra that spits out the optimal weights in one shot for a realistically-sized network. So instead you improve iteratively, and the **gradient** is what tells you which direction is an improvement.

The gradient of a function, at a given point, is the vector that points in the direction of *steepest increase* of that function. This is a direct generalization of something you already know: in one dimension, the derivative tells you the slope; in many dimensions, the gradient is the multi-dimensional slope, pointing "uphill" in the input space that produces the fastest possible increase in the output.

Since we want to *decrease* the loss, not increase it, the entire algorithm is: compute the gradient of the loss with respect to the weights, then take a step in the *opposite* direction. That's it. That's gradient descent — repeated, thousands or millions of times, one small step at a time.

The standard mental picture is a hiker descending a mountain in thick fog, with no map and no view of the valley below. You can't see the destination. What you *can* do is feel the ground under your feet and sense which direction is locally steepest downhill, take a step that way, and repeat. You'll never see the whole mountain — you're navigating by local information only, and yet, step after step, you make real progress toward lower ground. A gradient at a single point in a landscape with millions of dimensions is exactly that: local slope information, and nothing more. It says nothing about what's a mile away. It's a strong hint, not a plan.

```
loss
 |        *
 |       * *
 |      *   *
 |     *     *
 |    *        *___     <- gradient descent takes small
 |   *              \       steps "downhill" from wherever
 |  *                 \_    the hiker currently stands
 +-----------------------------> weight value
```

## Learning Rate: How Big a Step the Hiker Takes, Worked in Full

Knowing which direction is downhill doesn't tell you *how far* to step, and that's a separate, equally important choice called the **learning rate**. It's a single number, usually written `α` or `lr`, that scales the size of every gradient-descent step. The update rule is always the same shape:

```
w_new = w_old - learning_rate × gradient
```

Let's take a toy one-dimensional loss, `f(w) = w^2`, whose true minimum is obviously at `w = 0` just by inspection. Its slope (derivative) at any point `w` is `2w` — that's the "gradient" in this one-variable case. Start at `w = 10` and walk through the update rule step by step for two different learning rates, so you can see convergence and divergence as actual numbers rather than as a claim.

**Learning rate 0.1 (converges):**

| Step | w before | gradient = 2w | update: w − 0.1×gradient | w after |
|---|---|---|---|---|
| 1 | 10.0 | 20.0 | 10.0 − 0.1×20.0 = 10.0 − 2.0 | 8.0 |
| 2 | 8.0 | 16.0 | 8.0 − 0.1×16.0 = 8.0 − 1.6 | 6.4 |
| 3 | 6.4 | 12.8 | 6.4 − 0.1×12.8 = 6.4 − 1.28 | 5.12 |
| 4 | 5.12 | 10.24 | 5.12 − 0.1×10.24 = 5.12 − 1.024 | 4.096 |
| 5 | 4.096 | 8.192 | 4.096 − 0.1×8.192 = 4.096 − 0.8192 | 3.2768 |

Look at the "w after" column: `10 → 8 → 6.4 → 5.12 → 4.096 → 3.2768`. Each value is exactly `0.8` times the one before it (`8/10 = 0.8`, `6.4/8 = 0.8`, `5.12/6.4 = 0.8`, and so on) — because plugging the gradient into the update rule algebraically gives `w_new = w_old - 0.1 × 2w_old = w_old × (1 - 0.2) = 0.8 × w_old`. A repeated multiplication by `0.8`, a number smaller than 1 in absolute value, shrinks toward zero forever: after 10 steps, `w ≈ 10 × 0.8^10 ≈ 10 × 0.107 ≈ 1.07`, and it keeps shrinking from there. That's convergence, made visible as arithmetic.

**Learning rate 1.1 (diverges):**

| Step | w before | gradient = 2w | update: w − 1.1×gradient | w after |
|---|---|---|---|---|
| 1 | 10.0 | 20.0 | 10.0 − 1.1×20.0 = 10.0 − 22.0 | −12.0 |
| 2 | −12.0 | −24.0 | −12.0 − 1.1×(−24.0) = −12.0 + 26.4 | 14.4 |
| 3 | 14.4 | 28.8 | 14.4 − 1.1×28.8 = 14.4 − 31.68 | −17.28 |
| 4 | −17.28 | −34.56 | −17.28 − 1.1×(−34.56) = −17.28 + 38.016 | 20.736 |

Same exercise, same algebra: `w_new = w_old - 1.1 × 2w_old = w_old × (1 - 2.2) = -1.2 × w_old`. Now the repeated multiplier is `-1.2` — bigger than 1 in absolute value, and negative. Every step it flips sign *and* grows by 20%: `10 → -12 → 14.4 → -17.28 → 20.736 → ...`, oscillating back and forth across the minimum while getting farther away each time instead of closer. That's divergence, and it's not a metaphor — it's the same update rule, the same gradient formula, with the only difference between "converges nicely" and "blows up forever" being whether `|1 - 2×learning_rate|` is less than 1 (it was `0.8`) or greater than 1 (it was `1.2`). Nothing about the loss function changed between the two runs — only the step size did, and it was the entire difference between finding the minimum and never finding it.

That's the whole trade-off in miniature. Too small a learning rate, and the hiker inches down the mountain so cautiously that training takes forever (or gets stuck making imperceptible progress). Too large, and each step overshoots the bottom of the local valley entirely, sometimes landing on worse ground than before, sometimes bouncing back and forth forever, sometimes diverging outright, exactly like the `1.1` example above. Choosing this number well — and modern optimizers spend real engineering effort adapting it automatically as training progresses — is one of the most consequential knobs in the entire training process.

## Walking Downhill in Two Dimensions

The one-dimensional example makes the arithmetic easy to follow, but it hides something the hiker picture depends on: in real problems you're not walking along a single number line, you're walking around on a landscape with many directions to choose from, and the gradient has to pick the single best one.

Take `f(x, y) = x^2 + y^2` — a perfectly symmetric bowl, minimized at `(0, 0)`. If you slice this surface at a constant height, the slices are circles centered on the origin: closer to the middle, smaller circles; farther out, bigger ones. Drawn from directly overhead, the contour map looks like this:

```
        y
        |
     ..-''''-..
   .'    ___    '.
  /   .-'   '-.    \
 |   /  .---.  \    |
 |  |  ( * )  |    |   <- concentric rings = constant-loss contours
 |   \  '---'  /    |      * marks the minimum at (0,0)
  \   '-.___.-'    /
   '.          _.'
     ''-....-''      --------> x
```

The gradient at any point on this landscape is a 2D vector that points straight *away* from the center, perpendicular to whichever contour ring you're standing on, in the direction the surface rises fastest. Gradient descent negates that vector, so it always points straight back *toward* the center, again perpendicular to the local ring. Standing anywhere on this bowl and always stepping perpendicular to the contour line under your feet, aimed inward, walks you directly to the minimum — which is exactly the fog-covered-mountain intuition from before, just now with an actual second axis to turn along instead of only "left or right" on a number line. Real loss surfaces are almost never this symmetric — they're stretched, tilted, and full of gentle ridges — which is precisely why the *learning rate* discussion above matters so much more once you're in many dimensions: a step size that's safe along one contour direction can easily overshoot along a more tightly curved one.

## Flat Spots in the Landscape: Critical Points and Saddle Points

Gradient descent stops making progress wherever the local slope is zero — these are called **critical points**. The intuitive one is a **local minimum**: a valley floor, where every direction leads back uphill. That's the outcome you're hoping for. But a critical point can also be a local *maximum* (a hilltop — rare in practice, since gradient descent moves away from those) or something stranger: a **saddle point**, shaped like a mountain pass or the middle of a Pringle chip — flat in the sense that the slope is zero right at that point, but curving *upward* in some directions and *downward* in others simultaneously.

A concrete, easy-to-hold-in-your-head example is `f(x, y) = x^2 - y^2`. Its gradient is `(2x, -2y)`, which is exactly `(0, 0)` only at the origin — so the origin is the one and only critical point. Now look at what the surface actually does near that point along each axis separately:

- Walk along the x-axis (hold `y = 0`): `f(x, 0) = x^2`, which is `0` at the origin and *increases* as you move either direction away from it — this is an upward-curving parabola, like the bottom half of a bowl.
- Walk along the y-axis (hold `x = 0`): `f(0, y) = -y^2`, which is `0` at the origin and *decreases* as you move either direction away from it — this is a downward-curving parabola, like the top of a dome.

So standing exactly at the origin, the ground is perfectly flat (gradient zero, satisfying the critical-point condition), but it rises if you step along `x` and falls if you step along `y`. It's not a local minimum, because there's a direction (`y`) that leads to *lower* loss. It's not a local maximum either, because there's a direction (`x`) that leads to *higher* loss. It's neither — a genuine saddle, and gradient descent landing near that point can slow to a crawl (the gradient shrinks toward zero as you approach it) without ever having found anything worth stopping at.

Here's the genuinely useful, still-relevant insight from this chapter: in the low-dimensional loss landscapes you can actually draw and visualize, local minima look like the scary obstacle. But real networks have millions or billions of weights, meaning the loss landscape has millions or billions of dimensions. For a flat point to be a true local minimum, the surface has to curve *upward* in every single one of those directions simultaneously — a coincidence that becomes exponentially less likely as the dimension count grows, the same way it's easy for one coordinate axis to curve down (as in the `x^2 - y^2` example) but astronomically unlikely for millions of independent axes to all curve up at once by chance. Saddle points, where some directions curve up and others curve down, become overwhelmingly more common than true local minima at that scale. Practically, this reframes the worry: the danger in high-dimensional training isn't usually "getting permanently trapped in a bad valley," it's "slowing to a crawl while traversing a saddle region," because the gradient can become very small there too, even though it isn't a real minimum.

## A Brief Nod to Smarter Methods: Curvature and Newton's Method

Gradient descent only ever uses first-order information — the slope. There's a whole family of methods that go a step further and use **second-order information** — the *curvature* of the loss surface, captured mathematically by a matrix of second derivatives called the **Hessian**. Knowing curvature, not just slope, lets an algorithm like **Newton's method** make a much smarter guess about both direction and step size in a single move, often converging in far fewer steps than plain gradient descent. Loosely, this is the difference between the `learning_rate = 0.1` example above, which had to be *chosen* through trial and error to avoid divergence, and a method that looks at the local curvature (`2` for `f(w) = w^2`) and computes the ideal step size directly rather than guessing at it.

The catch, and the reason you won't see this in most deep learning code, is cost: computing and inverting a Hessian scales with the *square* of the number of parameters, which is a non-starter when a network has millions or billions of weights. That expense is exactly why the field leans so heavily on cheaper, first-order, gradient-only methods instead — and why Part II of this book devotes an entire chapter to the many clever variants of gradient descent rather than to second-order methods.

## Vocabulary

| Term | Plain meaning |
|---|---|
| Underflow / overflow | A value too small (rounds to zero) or too large (becomes infinity) to fit in a finite-precision number format, often silently corrupting everything computed from it afterward. |
| Numerically stable computation | An implementation of a formula chosen specifically to avoid intermediate values that overflow, underflow, or otherwise blow up — same math, safer arithmetic path. |
| Condition number | A measure of how much a function (especially a matrix) amplifies small input errors into output errors; high condition number means small rounding noise can produce a wildly different answer. |
| Ill-conditioned function | A function where small input errors (like rounding) get amplified into large output errors, even with perfectly accurate individual arithmetic steps. |
| Gradient | The direction (in a many-dimensional input space) of steepest increase of a function at a given point; used in reverse to decrease a loss. |
| Gradient descent | The iterative algorithm of repeatedly stepping in the direction opposite the gradient to shrink a loss function toward a minimum. |
| Learning rate | The scalar that sets how big each gradient-descent step is; too small stalls training, too large causes overshoot or divergence. |
| Critical point | A point where the gradient is zero — could be a local minimum, local maximum, or saddle point. |
| Saddle point | A critical point that curves upward in some directions and downward in others — flat locally, but not a true minimum; the more common obstacle in high-dimensional training. |
| Hessian | The matrix of second derivatives of a function, capturing its local curvature; expensive to compute at deep-learning scale. |
| Newton's method | A second-order optimization method that uses curvature (the Hessian) to choose a smarter step direction and size than gradient descent, at much higher computational cost per step. |

---

## Where we'll go next
**Lesson 5 — Machine Learning Basics I.** With gradients and gradient descent now on the table, we can finally talk about what "learning" formally means: tasks, performance measures, and the train/test split that turns optimization into machine learning.

Reply **ok** to continue, or ask anything about today's lesson first.
