# Lesson 6 — How Models Actually Learn: Estimators and the Path to Deep Learning

Based on: Goodfellow, Bengio, Courville — *Deep Learning*, Chapter 5 "Machine Learning Basics" (original explanation, not excerpted)

Lesson 5 established the goal (generalization) and the two ways to miss it (underfitting, overfitting). This lesson gets concrete about the mechanism, using our incident classifier the whole way through: what does it actually mean, mathematically, to "estimate" a model's parameters from data, and why does the cross-entropy loss from Lesson 3 turn out to be the same idea in disguise? Then we close out Part I by tying gradient descent (Lesson 4) to how training happens at real scale, and by looking at where classical methods hit a wall — the exact wall deep learning was built to get past.

## Point Estimation: A Rule for Guessing an Unknown Number

Suppose you want to know the average latency of one of your services. You can't measure every request that will ever hit it. What you *can* do is sample some recent requests, average their latency, and use that as your guess for the true, unknowable average. That's a **point estimator**: a rule that converts data into a single best guess for a quantity you can't observe directly.

Sample 5 requests, latencies in ms: `[120, 135, 118, 142, 125]`. Point estimate: `(120+135+118+142+125)/5 = 640/5 = 128 ms`. That number isn't "the true average" — it's your best guess from this one sample. Swap one request for a GC-pause outlier (210ms instead of 142ms) and the estimate shifts to `(120+135+118+210+125)/5 = 141.6 ms`. Same true average, different sample, different estimate.

That wobble is the **variance of an estimator** from Lesson 3: an estimator is itself a random variable, with spread. 5 samples wobble a lot; 10,000 samples barely move when you swap a few points. This is exactly why you'd never trust an incident classifier trained on 5 labeled minutes — not because "average the examples" is the wrong idea, but because a small sample makes for a high-variance estimate of whatever the classifier is trying to learn. Every parameter in every model, from a single average up to a million-weight network, is formally a point estimate with this same sample-dependent wobble — including, as we'll get to, the classifier's own weights.

## Maximum Likelihood Estimation: Picking Parameters the Data Doesn't Contradict

**Maximum Likelihood Estimation (MLE)** is the framework underlying most of modern ML, deep learning included, whether or not anyone names it explicitly:

> Choose the parameters that make the data you actually observed as probable as possible, under your model.

**Before I explain — guess:** suppose your incident monitor auto-flagged 10 minutes over the past month, and a human later confirmed 7 of those were true incidents and 3 were false alarms. What would you guess is the "best" estimate of the flag's true incident rate — and do you think there's a rigorous reason that guess is correct, or is it just common sense dressed up?

Your gut says 0.7 — 7 out of 10. MLE is the formal justification, and you can see why without calculus, just by asking "how surprising would 7-true-out-of-10-flagged be, under each candidate rate `p`?"

- **p = 0.5:** if only half of flags were real, the typical outcome over 10 flags is around 5 confirmed incidents. Seeing 7 requires more luck than a 50% rate typically produces — somewhat surprising.
- **p = 0.9:** if 90% of flags were real, you'd expect 9 or 10 confirmed incidents as typical. Seeing only 7 (with 3 false alarms) is *also* surprising — more false alarms than a 0.9 rate usually yields.
- **p = 0.7:** now 7-out-of-10 is exactly the outcome this hypothesis predicts most strongly. Nothing needs explaining away as luck.

p = 0.7 is the candidate under which your actual data is *least surprising* — equivalently, *most probable*. That's MLE, in full: ask which parameter makes what you saw look unremarkable, and pick it. It turns out (calculus confirms this rigorously, but you don't need it to trust the intuition) that for this kind of yes/no rate, the maximum-likelihood estimate is always exactly the observed fraction — 7/10 = 0.7. The formula is trivial; MLE is the justification for why that formula is the *right* one.

### The cross-entropy connection, made concrete

Here's the fact worth remembering above all else in this lesson: **minimizing cross-entropy loss for a classifier is mathematically the same operation as maximum likelihood estimation.** Suppose at some point during training, your incident classifier scores 3 minutes that are all confirmed true incidents (true label = incident, for all three):

| Minute | Model's predicted P(incident) | Cross-entropy loss |
|---|---|---|
| A | 0.9 | −log(0.9) ≈ 0.105 |
| B | 0.6 | −log(0.6) ≈ 0.511 |
| C | 0.2 | −log(0.2) ≈ 1.609 |

Cross-entropy per example is `−log(probability assigned to the true label)`. Total loss: `0.105 + 0.511 + 1.609 ≈ 2.225`. Training pushes minute C's score up from 0.2 — it's confidently wrong.

Now view the same 3 scores as MLE. The **joint likelihood** of the model getting all 3 labels right (treated as independent) is the product: `0.9 × 0.6 × 0.2 = 0.108`. MLE wants that number maximized. Taking logs turns the product into a sum: `log(0.9) + log(0.6) + log(0.2)`. Flip the signs, and that's *exactly* the total cross-entropy above (2.225). "Maximize joint likelihood 0.108" and "minimize total cross-entropy 2.225" are the same optimization problem, related by a transformation (log, negate) that never changes which parameters are optimal. Every gradient step your incident classifier takes to reduce cross-entropy is, underneath, a step toward the maximum-likelihood parameters — two things you learned separately, turning out to be the same thing wearing different names.

## Supervised vs. Unsupervised Learning

Two flavors of "experience" (the E from Lesson 5's T/E/P), distinguished by whether the data comes with answers attached:

- **Supervised learning**: every example has a label. This is exactly Lesson 1's setup — past minutes hand-labeled "incident" or "normal" by an on-call human, and the model learns metrics → label.
- **Unsupervised learning**: no labels; the model finds structure on its own. Two flavors, so you see the shape of the category:
  - *Clustering*: group users by behavior (pages visited, session length) with no predefined groups. Success is judged by whether the discovered structure is useful, not by checking against a right answer.
  - *Anomaly detection*: this is the unsupervised twin of our exact classifier — same four metrics (latency, error rate, CPU, request rate), but now with no labels at all, nobody hand-tagging past incidents. The model learns what the *bulk* of normal minutes looks like — the typical ranges and correlations — and flags anything that deviates far enough, simply for being unlike everything else. Same "find structure with no answer key" spirit as clustering, aimed at "does this look like the rest of the data" instead of "which group."

So our running example actually spans both flavors: labeled-incident training is supervised; "learn what normal looks like and flag deviations" is unsupervised. Same metrics, same goal, two different assumptions about what your training data hands you. The boundary blurs further in modern deep learning — pretraining large language models manufactures labels from the data itself (predict the next word) — but for now: does your training data come with an answer key, or not.

## Stochastic Gradient Descent: Why You Don't Compute the Exact Gradient

Lesson 4 covered gradient descent: compute the loss's gradient, step against it. The detail we glossed over: computing the *exact* gradient requires evaluating the loss across your *entire* training set, every single example, for every single update step.

**Before I explain — guess:** if your incident classifier trains on months of per-minute metrics across every service — tens of millions of rows — and gradient descent needs the exact gradient over the *entire* set for every one of possibly millions of update steps, how much of that data do you think gets touched per step, under the "exact" approach?

All of it, every step — the same cost problem you already have instincts for from data engineering. Want the average latency across 10 million logged requests, exactly? Scan all 10 million, sum, divide — correct, but the most expensive way to get the number, and you pay it again the moment the data goes stale. Sample 1,000 requests at random instead: ~10,000x fewer rows touched, and (unless your sample is pathologically biased) close enough to act on.

**Stochastic Gradient Descent (SGD)** applies exactly that to training: instead of the full-dataset gradient, compute it over a small random **mini-batch** — 32, 256, a few thousand examples — and step based on that. Noisy, approximate, dramatically cheaper per step — which means far more steps per unit of wall-clock time, and more-but-noisier steps beats fewer-but-exact ones in practice.

One place the analogy needs an extra layer: training doesn't take one gradient step and stop, it takes thousands or millions, each on a *different* random mini-batch. You don't need one lucky approximate average — you need a long sequence of independent noisy estimates whose errors cancel out over the run, the way independent measurement noise averages out over repeated trials. A single mini-batch's gradient might point slightly wrong; the next one's error points a different, unrelated way; over thousands of steps the noise cancels while genuine signal accumulates. There's a bonus too: a little randomness in the gradient makes it harder for the optimizer to get stuck sitting exactly on a bad flat spot — the same way a load balancer adding jitter to retry timing avoids everything piling up in lockstep. This is why every deep network you've heard of trains with some SGD variant: "exact gradient over everything" doesn't scale to the dataset sizes deep learning needs — including the months of per-minute rows our own classifier would train on.

## Four Ingredients: Most ML Algorithms Are the Same Recipe

Nearly every ML algorithm, classical or deep, is built from four interchangeable parts: **a dataset** (the experience), **a cost/loss function** (what "doing well" means — often cross-entropy or squared error, plus optionally regularization), **a model** (a family of functions with adjustable parameters), and **an optimization procedure** (how you search for parameters that minimize the cost — gradient descent/SGD, or sometimes a closed-form formula). Swap one slot, get a different named algorithm, same recipe.

Let's run one real update step for a simplified stand-in for our incident classifier: one input feature (a scaled error-rate-trend value — the same kind of signal Lesson 1's layer-1 units computed), predicting a raw risk score a human assigned in hindsight (not yet squashed to a 0–1 probability — that's a Part II detail).

**1. Dataset** — three (trend feature, hindsight risk score) pairs: `x=1,y=2` / `x=2,y=4` / `x=3,y=5` (the third point being 5 instead of 6 is deliberate noise — no perfect fit, just a best fit, same situation any real dataset puts you in).

**2. Model**: `y = w·x + b`, start at `w=1, b=0`.

**3. Cost function**: mean squared error, `MSE = (1/n)·Σ(y_i − ŷ_i)²`.

**4. Optimizer**: gradient descent — compute ∂MSE/∂w and ∂MSE/∂b, step against them.

With `w=1, b=0`: `ŷ = x`, so predictions are 1, 2, 3 against actual 2, 4, 5 — errors 1, 2, 2. `MSE = (1+4+4)/3 = 3`.

Gradients: `∂MSE/∂w = −(2/n)·Σ(y_i−ŷ_i)·x_i = −(2/3)·(1·1+2·2+2·3) = −(2/3)·11 ≈ −7.33`. `∂MSE/∂b = −(2/3)·(1+2+2) = −(2/3)·5 ≈ −3.33`.

Step with learning rate 0.1: `w_new = 1 − 0.1×(−7.33) = 1.733`, `b_new = 0 − 0.1×(−3.33) = 0.333`.

Check it helped: new predictions `1.733·x + 0.333` give 2.067, 3.8, 5.533 against 2, 4, 5 — errors −0.067, 0.2, −0.533 — new `MSE ≈ 0.11`, down from 3 in one step. Repeated over many steps (and, at real scale, over many random mini-batches), that drop is the entire training loop. Swap slot 3 for a multi-layer network taking all four of Lesson 1's raw metrics, and slot 4 for SGD over months of logged minutes, and this exact loop — predict, measure error, compute gradient, step against it — is how the incident classifier's weights actually update. Nothing about deep learning changes the loop; it changes which slots you plug in.

## Where Classical Methods Hit a Wall

Given four swappable parts, why not keep using simple models forever? The chapter's closing argument — and the real motivation for the rest of the book, past where this series stops — is that classical methods scale poorly on a specific class of problem, for a specific reason: the **curse of dimensionality**.

As input dimensions grow, the volume of possible input configurations grows exponentially, while training data grows nowhere near that fast. Divide each dimension into just 10 bins:

```
1 dimension:   10 bins         = 10 cells
2 dimensions:  10 × 10         = 100 cells
3 dimensions:  10 × 10 × 10    = 1,000 cells
10 dimensions: 10^10           = 10,000,000,000 cells
```

Our classifier's four metrics live in a friendly 4D space. A real observability stack tracking thousands of per-endpoint, per-pod series — exactly the "real version" Lesson 1 gestured at — pushes this grid past anything collectable. A classical model that essentially interpolates between nearby training examples needs several examples landing in most cells to say anything reliable; 10 billion cells at one example each is already unreachable, and real problems (image pixels, audio) run to thousands or millions of dimensions, not 10. No amount of more data or bigger machines fixes an approach that doesn't scale its expressiveness with the problem's complexity.

This is the exact gap deep networks close. Recall Lesson 1: instead of one flat function mapping raw input straight to an answer, a deep network learns a *hierarchy* of representations, layer by layer, from data instead of hand design. That hierarchy is what lets deep models generalize in high-dimensional spaces classical flat models can't — they don't need an example near every point, because they've learned reusable, composable building blocks. How that works is Part II's job (feedforward networks, CNNs, RNNs) — you now understand precisely *why* it was necessary.

## Vocabulary

| Term | Plain meaning |
|---|---|
| Point estimator | A rule converting observed data into a single best guess for an unknown quantity (e.g. sample average estimating a true mean). A different sample gives a different estimate — that spread is the estimator's variance. |
| Maximum Likelihood Estimation (MLE) | Choosing parameters that make the observed data as probable as possible under the model (e.g. 7/10 confirmed-true flags gives MLE rate p=0.7); mathematically equivalent to minimizing cross-entropy loss. |
| Supervised learning | Learning from labeled examples — each input comes with a known correct output. |
| Unsupervised learning | Learning from unlabeled data — finding structure (clustering users, flagging anomalous metrics) with no answer key. |
| Mini-batch | A small, randomly sampled subset of the training set used to compute one approximate gradient step. |
| Stochastic Gradient Descent (SGD) | Gradient descent using mini-batch gradient estimates instead of the exact full-dataset gradient — trades exactness for speed, relying on many independent noisy steps averaging out over training. |
| Four ingredients of an ML algorithm | Dataset, cost function, model, optimization procedure — the reusable structure underlying most ML algorithms, classical or deep. |
| Curse of dimensionality | The exponential blow-up in possible input configurations as dimensionality grows, which breaks classical methods that rely on training examples "near" every point they must predict on. |

---

## Quick check

Your team's incident monitor flagged 20 minutes last month. A human later reviewed all 20 and confirmed 15 were true incidents, 5 were false alarms. In your own words: (a) what's the MLE estimate of the flag's true-incident rate, and why is that the "correct" estimate rather than just a reasonable guess; and (b) is reviewing those 20 flagged minutes itself a supervised or unsupervised learning setup, and what would make it the other kind?

## You've completed Part I

You now have the foundational vocabulary this whole field is built on: linear algebra as the mechanics of a layer, probability/information theory as the language of loss functions, numerical computation as gradient descent, and this lesson's core idea — that fitting a model is estimation, and cross-entropy training is maximum likelihood in disguise.

Answer the check above (even roughly) first. Then, when you're ready, Part II of the book (Deep Feedforward Networks, Regularization, Optimization, CNNs, RNNs) builds directly on all of this — say the word and we'll pick a new lesson plan for it.
