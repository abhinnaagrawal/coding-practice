# Lesson 6 — How Models Actually Learn: Estimators and the Path to Deep Learning

Based on: Goodfellow, Bengio, Courville — *Deep Learning*, Chapter 5 "Machine Learning Basics" (original explanation, not excerpted)

Lesson 5 established the goal (generalization) and the two ways to miss it (underfitting, overfitting). This lesson gets concrete about the mechanism: what does it actually mean, mathematically, to "estimate" a model's parameters from data, and why does the cross-entropy loss from Lesson 3 turn out to be the same idea in disguise? Then we close out Part I by tying gradient descent (Lesson 4) to how training happens at real-world scale, and by looking at where classical methods hit a wall — which is exactly the wall deep learning was built to get past.

## Point Estimation: A Rule for Guessing an Unknown Number

Suppose you want to know the average latency of one of your services. You can't measure every request that will ever hit it — that population doesn't fully exist yet. What you *can* do is sample some recent requests, compute their average latency, and use that as your guess for the true (unknowable) average latency of the service.

That's a **point estimator**: a rule that takes in data and spits out a single best guess for some quantity you can't observe directly. "Average of the sample" is an estimator for "true population mean." It's not guaranteed to be exactly right — a different sample would give you a slightly different number — but it's a principled, reproducible way to convert data into a guess.

Let's make that concrete with numbers small enough to hold in your head. Say you sample 5 requests to one endpoint and record their latencies in milliseconds:

```
[120, 135, 118, 142, 125]
```

The point estimate for the true average latency is just the sample mean:

```
(120 + 135 + 118 + 142 + 125) / 5 = 640 / 5 = 128 ms
```

So your point estimate is **128 ms**. Notice what this number is and isn't. It isn't "the true average latency of the service" — it's your best guess at that unknown quantity, built entirely from this one sample of 5 requests. If you'd happened to sample a different 5 requests — say one of them landed during a GC pause and came back at 210ms instead of 142ms — your estimate would shift: `(120+135+118+210+125)/5 = 708/5 = 141.6 ms`. Same underlying service, same true (unknown) average, different sample, different point estimate.

This is exactly the notion of **variance of an estimator** you first ran into with random variables in Lesson 3: an estimator computed from a random sample is itself a random variable, and it has spread. A sample of 5 has a lot of spread — one weird outlier swings the mean by 13ms. A sample of 10,000 requests barely moves if you swap a handful of points in or out; its variance is far lower. This is precisely why, in practice, you'd never trust an average built from 5 data points to represent a production SLA — not because the *formula* (sample mean) is wrong, but because a small sample gives a high-variance estimator, and high variance means "this specific number you got is not very trustworthy as a stand-in for the truth." Everything from here forward is this same idea, scaled up: training a model's parameters is exactly this, just with more parameters and a fancier data-generating process. When you fit a linear regression, you're not directly observing the "true" slope and intercept — you're using an estimator (a formula, or an optimization procedure) to guess them from a finite sample of (input, output) pairs, and a different training sample would hand you slightly different weights. Everything from a single average to a million-parameter neural network's weights is, formally, a point estimate, and all of it inherits this same sample-dependent wobble.

## Maximum Likelihood Estimation: Picking Parameters That Make the Data Look Least Surprising

There are many ways to design an estimator, but one framework — **Maximum Likelihood Estimation (MLE)** — underlies most of what modern ML does, including deep learning, whether or not the practitioner thinks about it explicitly. The idea, in plain language:

> Choose the model parameters that make the data you actually observed as probable as possible, under your model.

Concretely: imagine your model assigns a probability to every possible outcome (this is why Lesson 3's probability distributions matter). Different parameter settings assign different probabilities to the training data you actually collected. MLE says: pick the parameters that give the *highest* probability to the data you actually have. Data that would have been wildly improbable under some parameter setting is evidence against that parameter setting; MLE formalizes "prefer parameters the data doesn't contradict" into an optimization problem.

### A worked toy example: the 7-heads coin

Here's the smallest possible non-trivial MLE problem, worked by hand with intuition instead of calculus. You have a coin with an unknown bias — call the true probability of heads `p`. You flip it 10 times and observe **7 heads, 3 tails**. What's your best estimate of `p`?

Your gut answer is probably "0.7" — 7 out of 10. MLE is the formal justification for why that gut answer is correct, and you can see why without doing any calculus, just by comparing candidate values of `p` and asking "how surprising would 7-heads-out-of-10 be, if this candidate were the true bias?"

- **If p = 0.5 (fair coin):** heads and tails are equally likely on each flip, so the *most typical* outcome of 10 flips is around 5 heads. Getting exactly 7 is possible but requires more luck than a fair coin's typical behavior — a fair coin doesn't usually stray this far from 50/50 in only 10 flips. So p = 0.5 makes the data you actually saw somewhat surprising.
- **If p = 0.9 (heavily biased toward heads):** now the coin should come up heads almost every time — you'd expect 9 or 10 heads out of 10 as the typical outcome. Seeing only 7 (with a full 3 tails) is *also* surprising under this hypothesis — 3 tails is more than a 0.9-biased coin would usually produce.
- **If p = 0.7:** now 7 heads out of 10 is exactly the typical, expected outcome — it's the outcome this hypothesis predicts most strongly. Nothing about the data needs explaining away as luck.

So among these three candidates, p = 0.7 is the one under which the observed data (7 heads, 3 tails) is the *least surprising* — equivalently, the *most probable*. That is maximum likelihood estimation, in full: you're not doing anything mystical, you're just asking "which parameter value makes what I actually saw look the most unremarkable," and picking that one. It turns out (and this is the part calculus confirms rigorously, though you don't need it to trust the intuition) that for a simple coin-flip model, the maximum-likelihood estimate of `p` is always exactly the observed fraction of heads — 7/10 = 0.7 here. The formula is trivial; the *justification* for why that formula is the "right" one to use is MLE.

### Making the cross-entropy connection concrete

Here's the connection you should walk away from this lesson remembering, because it ties Lesson 3 and everything since directly together: **minimizing cross-entropy loss for a classifier is mathematically the same operation as maximum likelihood estimation.** Let's make that concrete instead of just asserting it, with a tiny 3-example classification case.

Suppose you're training a binary spam classifier and, at some point during training, your model assigns these probabilities of "spam" to 3 emails, all of which are actually spam (true label = spam for all three):

| Email | Model's predicted P(spam) | Cross-entropy loss for this example | 
|---|---|---|
| A | 0.9 | −log(0.9) ≈ 0.105 |
| B | 0.6 | −log(0.6) ≈ 0.511 |
| C | 0.2 | −log(0.2) ≈ 1.609 |

Cross-entropy loss per example is `−log(probability the model assigned to the true label)`. Total loss across the 3 examples is the sum: `0.105 + 0.511 + 1.609 ≈ 2.225`. Training pushes parameters to make this sum smaller — the optimizer wants email C's predicted probability to move up from 0.2 toward 1.0, since that email is confidently wrong right now.

Now look at the *same* 3 predictions through the MLE lens instead. If we treat the model's predictions as independent, the **joint likelihood** of the model producing all 3 correct labels is the product of the individual probabilities it assigned to the true label: `0.9 × 0.6 × 0.2 = 0.108`. MLE wants to choose parameters that make this joint probability as *large* as possible.

Here's the mechanical link: taking the logarithm of a product turns it into a sum — `log(0.9 × 0.6 × 0.2) = log(0.9) + log(0.6) + log(0.2)`. Flip every sign, and that sum of negative logs is *exactly* the total cross-entropy loss above (2.225). So "maximize the joint likelihood `0.108`" and "minimize the total cross-entropy loss `2.225`" are literally the same optimization problem, related by a monotonic transformation (log, then negate) that doesn't change which parameter values are optimal. Every gradient step your classifier's optimizer takes to reduce cross-entropy is, underneath, a gradient step toward the maximum-likelihood parameters. Every time you've trained a classifier with cross-entropy loss, you were doing maximum likelihood estimation, whether or not anyone called it that. This is one of those moments where two things you learned separately turn out to be the same thing wearing different names.

## Supervised vs. Unsupervised Learning

Two broad flavors of "experience" (recall the E in T/E/P from Lesson 5), distinguished by whether the data comes with answers attached:

- **Supervised learning**: every training example comes with a label — the "correct answer" you want the model to reproduce. Spam classification is the running example: each training email is labeled spam or not-spam by a human, and the model learns a mapping from email to label. Most of what you'd recognize as "classic ML" — classification, regression — is supervised.
- **Unsupervised learning**: the data has no labels; you're asking the model to find structure on its own. There isn't one flavor of this — here are two, so you can see the shape of the category rather than one narrow example:
  - *Clustering*: given logs of how users interact with your product (pages visited, session length, feature usage), group users into behavioral segments without anyone having told you in advance what the groups are or how many there should be. There's no "correct" cluster assignment to check against — success is judged by whether the discovered structure turns out to be useful (e.g., the clusters correspond to meaningfully different usage patterns you can act on).
  - *Anomaly detection*: given a stream of server metrics — CPU, memory, request rate, error rate — with no labels at all (nobody hand-tags "this minute was an incident"), learn what "normal" looks like from the bulk of the data, and flag points that don't fit that learned pattern. Nobody tells the model in advance what an anomaly looks like; there's no labeled training set of "here are 500 past incidents." Instead the model builds an implicit notion of the typical shape of your metrics — the ranges and correlations they normally fall into — and anything that deviates far enough from that shape (a latency spike, a memory leak's characteristic slow climb, a request-rate cliff) gets flagged simply for being unlike everything else it's seen. This is the same "find structure with no answer key" spirit as clustering, just aimed at "does this look like the rest of the data" instead of "which group does this belong to."

The boundary is genuinely blurry in practice — a lot of modern deep learning (including the pretraining stage of large language models) uses unsupervised or self-supervised setups where the "label" is manufactured from the data itself (like predicting the next word in a sentence you already have). But for now, the useful distinction is simply: do your training examples come with an answer key or not.

## Stochastic Gradient Descent: Why You Don't Compute the Exact Gradient

Lesson 4 covered gradient descent: at each step, compute the gradient of the loss function and move parameters a little in the direction that reduces it. There's a detail we glossed over that matters enormously in practice: computing the *exact* gradient of your loss requires evaluating it across your *entire* training set — every single example — for every single parameter-update step.

Here's the cost problem in numbers you already have intuition for from data engineering. Say you want the average latency across 10 million requests logged today, and you want it *exactly*. That means scanning all 10 million rows, summing, dividing — a full-table aggregation. It's correct, but it's also the most expensive way to get the number, and you pay that full cost again if you want to recompute it a moment later on slightly fresher data. Compare that to sampling 1,000 of those requests at random and averaging just those: dramatically cheaper — roughly 10,000x fewer rows touched — and, so long as your sample isn't pathologically biased, the resulting average is usually close enough to the true 10-million-row average to act on. You don't scan a 50-billion-row table to get a "good enough" estimate of a metric right now — you sample a manageable slice, accept some noise in the estimate, and get your answer fast enough to actually act on it. Waiting for the exact answer over the full dataset is often strictly worse than an approximate answer you get 10,000x faster, especially when you're going to recompute it again shortly anyway.

**Stochastic Gradient Descent (SGD)** applies exactly that idea to training: instead of computing the gradient over the whole training set (the "10 million requests, exact average" case), compute it over a small, randomly chosen **mini-batch** — maybe 32, 256, or a few thousand examples (the "sample 1,000, approximate average" case) — and take a step based on that. It's a noisy, approximate estimate of the true gradient, not the exact one. But it's dramatically cheaper per step, which means you can take far more steps in the same wall-clock time, and in practice more-but-noisier steps beat fewer-but-exact ones.

There is one place the analogy needs an extra layer to fully match deep learning, and it's worth being explicit about it: if you only ever needed *one* approximate average, sampling 1,000 requests once and calling it done would be the whole story. But training doesn't take one gradient step and stop — it takes thousands or millions of them, each on a *different* random mini-batch. What you need isn't one lucky approximate estimate; you need a long sequence of independent noisy estimates whose errors are uncorrelated with each other, so that they average out over the course of training the way independent measurement noise averages out over repeated trials. A single mini-batch's gradient might point in a slightly wrong direction, but the next mini-batch's noise points a different, unrelated slightly-wrong direction, and over thousands of steps those errors cancel while the genuine signal (the direction that actually reduces loss) accumulates. This is why SGD isn't just "one approximate query instead of one exact query" — it's closer to "many independent approximate queries, each cheap, whose combined effect over time converges to something at least as useful as the expensive exact answer, and often faster in wall-clock terms." There's a second, non-obvious benefit too: the noise itself helps. A little randomness in the gradient estimate makes it harder for the optimizer to get stuck sitting exactly at a bad local flat spot, in the same way a load balancer that adds a little jitter to retry timing avoids everything piling up in lockstep. Every deep network you've heard of was trained with some variant of SGD, precisely because "exact gradient over everything" simply doesn't scale to the dataset sizes deep learning depends on.

## Four Ingredients: Most ML Algorithms Are the Same Recipe

Here's a genuinely clarifying way to look back at everything from this lesson and the last: nearly every ML algorithm, classical or deep, is built from the same four interchangeable parts:

1. **A dataset** — the experience (E).
2. **A cost/loss function** — what "doing well" means, numerically (often cross-entropy, or squared error, plus optionally a regularization term from Lesson 5).
3. **A model** — a family of functions with adjustable parameters that map inputs to outputs (a line, a decision tree, a neural network).
4. **An optimization procedure** — how you search for parameter values that minimize the cost function (gradient descent / SGD from Lesson 4, or in some simple cases, a closed-form formula).

Swap any one of these four and you often get a different named algorithm, even though the recipe is identical. Let's instantiate all four slots with actual numbers and run one real update step, so this stops being an abstract framework and becomes a computation you've watched happen.

**1. The dataset.** Three tiny (input, output) pairs:

```
x = 1, y = 2
x = 2, y = 4
x = 3, y = 5
```

(If it were exactly `y = 2x` this would be a boring, noise-free line; the third point being 5 instead of 6 is deliberately a bit of realistic noise, so there's no perfect-fit answer, just a best-fit one — the same situation any real dataset puts you in.)

**2. The model**, explicitly: `y = w·x + b` — a single weight `w` (slope) and bias `b` (intercept). Start with an initial guess of `w = 1, b = 0` (as if we hadn't learned anything yet).

**3. The cost function**, explicitly: mean squared error, `MSE = (1/n) · Σ (y_i − ŷ_i)²`, where `ŷ_i = w·x_i + b` is the model's prediction for example `i`.

**4. The optimization procedure**: gradient descent from Lesson 4 — compute the gradient of MSE with respect to `w` and `b`, then step against it.

Now trace through one step by hand. With `w = 1, b = 0`, the model's predictions are `ŷ = x`, so:

| x | y (actual) | ŷ (predicted) | error (y − ŷ) |
|---|---|---|---|
| 1 | 2 | 1 | 1 |
| 2 | 4 | 2 | 2 |
| 3 | 5 | 3 | 2 |

Current cost: `MSE = (1² + 2² + 2²) / 3 = (1 + 4 + 4) / 3 = 9/3 = 3`.

The gradients of MSE with respect to `w` and `b` (standard calculus, stated here without derivation since the point is the mechanics, not the derivation) are:

```
∂MSE/∂w = −(2/n) · Σ (y_i − ŷ_i)·x_i
∂MSE/∂b = −(2/n) · Σ (y_i − ŷ_i)
```

Plugging in the numbers: `Σ (y_i − ŷ_i)·x_i = (1·1) + (2·2) + (2·3) = 1 + 4 + 6 = 11`, so `∂MSE/∂w = −(2/3)·11 ≈ −7.33`. And `Σ (y_i − ŷ_i) = 1 + 2 + 2 = 5`, so `∂MSE/∂b = −(2/3)·5 ≈ −3.33`.

Gradient descent moves *against* the gradient, with some learning rate — use `0.1` here:

```
w_new = w − 0.1 × (−7.33) = 1 + 0.733 = 1.733
b_new = b − 0.1 × (−3.33) = 0 + 0.333 = 0.333
```

One update step, and the model moved from `w=1, b=0` to `w≈1.73, b≈0.33`. Check that it actually helped: recompute predictions with the new weights — `ŷ = 1.733·x + 0.333` gives `2.067, 3.8, 5.533` for `x = 1, 2, 3`, against actual `2, 4, 5`. New errors are `−0.067, 0.2, −0.533`, and the new MSE is `(0.0044 + 0.04 + 0.284) / 3 ≈ 0.11` — down from `3` to about `0.11` in a single step. That drop, repeated over many steps (and, at real scale, over many random mini-batches per the SGD section above), is the entire training loop. Nothing about training a deep network changes this recipe — it changes slot 3 (the model becomes a deep composition of layers instead of `w·x + b`) and usually slot 4 (SGD instead of plain full-batch gradient descent), but the loop of "predict, measure error via the cost function, compute the gradient, step against it" is identical. Once you see the four slots, most of "which ML algorithm should I use" reduces to "which model, cost function, and optimizer fit my problem" rather than needing to learn each named algorithm as an unrelated fact.

## Where Classical Methods Hit a Wall

Given four swappable parts, why not just keep using simple models (linear regression, or classical methods like kernel machines and decision trees) with simple costs and simple optimizers, forever? The chapter's closing argument — and the actual motivation for everything the rest of this book covers, past where this lesson series stops — is that classical methods scale poorly on a specific class of hard problems, for a specific reason: the **curse of dimensionality**.

As the number of input dimensions grows (raw pixels in an image easily number in the hundreds of thousands), the volume of possible input configurations grows exponentially, while your training data grows nowhere near that fast. Here's a way to feel that growth as an actual number instead of just accepting the word "exponential." Imagine you want to divide each input dimension into just 10 bins (a coarse grid, not even a fine one), and you want at least a rough sense of what happens in every cell of the resulting grid:

```
1 dimension:  10 bins    = 10 cells
2 dimensions: 10 × 10    = 100 cells
3 dimensions: 10 × 10 × 10 = 1,000 cells
10 dimensions: 10^10     = 10,000,000,000 cells (10 billion)
```

A classical model that essentially interpolates between nearby training examples needs, roughly speaking, at least a handful of examples landing in most of those cells to say anything reliable about that region of the input space. Ten billion cells, at even one example per cell, is already a training set larger than most organizations will ever collect — and real problems (image pixels, audio samples) routinely have thousands to millions of dimensions, not 10. For genuinely high-dimensional, complex-structure problems like vision or speech, that requirement is not just impractical, it's astronomically impossible. No amount of more data or bigger machines fixes a model whose approach fundamentally doesn't scale its expressiveness with the size and complexity of the problem — you'd run out of atoms in the observable universe before you filled the grid for a modestly-sized image.

This is exactly the gap deep networks were built to close. Recall Lesson 1's central idea: instead of one flat function trying to map raw pixels straight to an answer, a deep network learns a *hierarchy* of representations — simple features composing into more complex ones, layer by layer, learned from data rather than hand-designed. That hierarchical structure is what lets deep models generalize well in high-dimensional spaces where classical flat models can't: they don't need an example near every point in the space, because they've learned reusable, composable building blocks that recombine to cover regions they never directly saw. We won't go further into *how* that works here — that's Part II's job (feedforward networks, convolutional networks, recurrent networks) — but you now understand precisely *why* it was necessary.

## Vocabulary

| Term | Plain meaning |
|---|---|
| Point estimator | A rule that converts observed data into a single best guess for some unknown quantity (e.g., sample average as an estimate of a true population mean). A different sample gives a different estimate — that spread is the estimator's variance. |
| Maximum Likelihood Estimation (MLE) | Choosing model parameters that make the observed training data as probable as possible under the model (e.g., a coin observed as 7/10 heads has its MLE bias estimate at p=0.7); mathematically equivalent to minimizing cross-entropy loss for classification. |
| Supervised learning | Learning from labeled examples — each input comes with a known correct output. |
| Unsupervised learning | Learning from unlabeled data — finding structure (e.g., clustering users, or flagging anomalous server metrics) without an answer key. |
| Mini-batch | A small, randomly sampled subset of the training set used to compute one approximate gradient step. |
| Stochastic Gradient Descent (SGD) | Gradient descent using mini-batch gradient estimates instead of the exact full-dataset gradient — trades exactness for speed, the same tradeoff as sampling-based approximate analytics, but relies on many independent noisy steps averaging out over the course of training. |
| Four ingredients of an ML algorithm | Dataset, cost function, model, and optimization procedure — the reusable structure underlying most ML algorithms, classical or deep. |
| Curse of dimensionality | The exponential blow-up in the space of possible inputs as dimensionality increases (e.g., a 10-bin grid goes from 10 cells in 1D to 10 billion cells in 10D), which breaks classical methods that rely on having training examples "near" every point they need to predict on. |

---

## You've completed Part I

You now have the foundational vocabulary this whole field is built on: linear algebra as the mechanics of a layer, probability/information theory as the language of loss functions, numerical computation as gradient descent, and this chapter's core idea — generalization is the actual goal, not fitting training data.

Part II of the book (Deep Feedforward Networks, Regularization, Optimization, CNNs, RNNs) builds directly on all of this. If/when you want to continue, say the word and we'll pick a new lesson plan for it.
