# Lesson 3 — Probability and Information Theory

Based on: Goodfellow, Bengio, Courville — *Deep Learning*, Chapter 3 (original explanation, not excerpted)

Lesson 2 gave you the mechanical vocabulary — vectors, matrices, the operations a layer actually performs. But matrix multiplication alone doesn't explain why a network's output looks like "73% cat, 27% dog" instead of a flat "cat." That's probability's job. This lesson is about the second language deep learning speaks fluently: the math of uncertainty, and — near the end — the specific piece of it (cross-entropy) that you'll see again the moment we start training real models.

## Why Probability Shows Up At All

As a backend engineer, you're used to systems where inputs map to outputs deterministically, and uncertainty is something you handle explicitly — retries, timeouts, circuit breakers — as a bolt-on to otherwise predictable logic. Deep learning is different: uncertainty isn't bolted on, it's baked into the core object the model produces.

There are two separate reasons probability shows up.

First, the *task itself* is uncertain. If you ask a model "is this email spam," there usually isn't a fact of the matter that's knowable from the text alone with 100% certainty — some emails are genuinely ambiguous. A model that's forced to output a hard 0 or 1 is pretending to know something it doesn't. A model that outputs "spam: 0.92" is being honest about its confidence, and that number turns out to be enormously useful downstream (you can set a threshold, flag borderline cases for review, combine it with other signals).

Second, *training itself* is a randomized process, in the same sense a randomized algorithm in distributed systems is. Weights start from random values. Each training step looks at a random mini-batch of examples rather than the whole dataset (stochastic gradient descent — we'll get to this properly later). Later architectures randomly disable neurons during training (dropout) purely to prevent overfitting. None of this randomness is a bug to eliminate; it's a deliberate design choice, the same way you might inject randomized jitter into retry backoff on purpose.

So probability enters deep learning at two layers: it's the *language the model's output is expressed in*, and it's a *tool used during the training process itself*.

## Random Variables and Distributions

A random variable is just a quantity whose value isn't fixed — it's drawn from some set of possibilities according to certain likelihoods. A probability distribution is the full accounting of those likelihoods: for every possible value, how likely is it?

There are two flavors, and the distinction matters mechanically:

**Discrete** — the variable takes one of a finite (or countable) list of values. Our spam classifier is a clean example: the output isn't really "spam" or "not spam" as a binary fact, it's a distribution over the two-item set `{spam, not-spam}`:

```
P(spam)     = 0.92
P(not-spam) = 0.08
```

Note the two numbers sum to 1 — that's the defining property of a valid probability distribution over discrete outcomes. If your model spits out numbers that don't sum to 1, something in your last layer is wrong (this is literally what the softmax function exists to guarantee, which we'll hit later).

**Continuous** — the variable can take any value in a range, like a real number. "How many milliseconds will this API call take?" isn't well described by a short list — it's smeared across a continuum. You can't assign a nonzero probability to *exactly* 42.0000000ms (there are infinitely many nearby values competing for that same sliver of probability mass), so instead you describe a *density* — a curve where the area under any interval tells you the probability of landing in that interval. This is the practical difference between a PMF (probability mass function, for discrete variables — direct probabilities) and a PDF (probability density function, for continuous variables — you integrate to get an actual probability).

Deep learning uses both constantly: classification outputs are discrete distributions over a fixed label set; predicting a continuous quantity (a stock price, a pixel intensity, a robot joint angle) uses continuous distributions.

## Expectation and Variance

Once you have a distribution, two numbers summarize a lot about it.

**Expectation** is the long-run average — if you sampled from this distribution forever and averaged the results, what would you converge to? For a fair six-sided die, the expectation is 3.5, even though 3.5 is never a value you can actually roll. It's a weighted average across all outcomes, weighted by how likely each one is.

**Variance** measures spread — how far, typically, do individual draws stray from that average? Two models can have identical expected accuracy — or identical expected latency — but wildly different variance, and that difference matters enormously in practice. Formally, variance is the expectation of the squared deviation from the mean: for each observed value, subtract the mean, square the result (so negative and positive deviations don't cancel out), and average those squares.

### Worked example: same average latency, very different variance

Suppose you're evaluating two latency-prediction models by comparing their predicted latency (ms) against 5 real requests. Both models' predictions average out to exactly 100ms. Here are the actual numbers:

**Model A's predictions:** 98, 102, 99, 101, 100

Mean: (98 + 102 + 99 + 101 + 100) / 5 = 500 / 5 = **100**

Deviations from the mean: −2, +2, −1, +1, 0
Squared deviations: 4, 4, 1, 1, 0
Sum of squared deviations: 4 + 4 + 1 + 1 + 0 = 10
Variance: 10 / 5 = **2**
(Standard deviation, the square root of variance: √2 ≈ **1.41ms**)

**Model B's predictions:** 40, 160, 20, 180, 100

Mean: (40 + 160 + 20 + 180 + 100) / 5 = 500 / 5 = **100**

Deviations from the mean: −60, +60, −80, +80, 0
Squared deviations: 3600, 3600, 6400, 6400, 0
Sum of squared deviations: 3600 + 3600 + 6400 + 6400 + 0 = 20000
Variance: 20000 / 5 = **4000**
(Standard deviation: √4000 ≈ **63.2ms**)

Both models have an identical mean prediction of 100ms — if you only reported "average predicted latency," they'd look indistinguishable. But Model A's variance is 2 (a tight, boring cluster of predictions within a couple milliseconds of the mean), while Model B's variance is 4000 — two thousand times larger — because it swings wildly between confidently-fast (20–40ms) and confidently-slow (160–180ms) predictions that happen to average out. In production, Model A tells you something useful and stable every time; Model B is right "on average" but is either wildly optimistic or wildly pessimistic on any *given* request, which is far more dangerous for anything downstream that sets timeouts or SLAs based on the prediction. This is the numeric version of preferring p50=100ms/p99=150ms over an equal-average service with p99=4s: the mean hides exactly the information that variance reveals.

## A Few Distributions Worth Knowing By Name

You don't need to memorize a catalog, but two distributions come up so often in deep learning that recognizing them by name and shape pays off immediately.

**Bernoulli** — the distribution of a single coin flip with a possibly-unfair coin: outcome is one of two values, with probability `p` for one and `1-p` for the other. This is exactly the shape of a binary classifier's output (spam / not-spam), and it's the simplest possible distribution — one parameter, `p`, fully describes it. The formula is barely a formula at all: `P(x) = p` if `x = 1` (the "yes" outcome), and `P(x) = 1 - p` if `x = 0` (the "no" outcome).

Concretely: suppose your organization's historical spam rate is `p = 0.3` — 30% of incoming mail is spam, before you've looked at any specific email. That prior belief *is* a Bernoulli distribution with `p = 0.3`:

```
P(spam)     = p     = 0.3
P(not-spam) = 1 - p = 1 - 0.3 = 0.7
```

That's the entire distribution — two numbers, one parameter, done. Every binary classifier you build is, at its output layer, producing the parameter `p` of a Bernoulli distribution for each example.

**Gaussian (Normal)** — the familiar bell curve, described by just two numbers: a mean (where the peak sits) and a variance (how wide the bell is). This is the single most common distribution in all of statistics and deep learning, and it's worth having real intuition for *why* it shows up everywhere, and for what its two parameters actually *do* to the shape.

The mean just slides the whole curve left or right along the axis — it doesn't change the shape, only the location of the peak. The variance is the interesting one: it controls how "confident" or "spread out" the curve is. A **small variance** produces a tall, narrow bell — most of the probability mass is crammed close to the mean, so draws from the distribution are almost always close to that central value (think: a well-calibrated sensor with tight, low-noise measurement error clustered right around zero). A **large variance** produces a short, wide, flattened bell — the mass is smeared over a much broader range, so draws are frequently far from the mean in either direction (think: a noisy sensor, or a rough initial guess before you've gathered much evidence). Same mean, same peak location — but a narrow-variance Gaussian says "I'm quite sure the answer is right here," while a wide-variance Gaussian says "the answer is somewhere in this much bigger neighborhood, don't hold me to the center."

The intuition for *why* the shape shows up everywhere, loosely: whenever some quantity you're measuring is the sum (or average) of a large number of small, mostly-independent random effects, the result tends toward a bell curve, almost regardless of what the individual effects looked like. This is the central limit theorem, and you don't need the proof to use the intuition — think of it like a load-balancing effect. If a single request's latency depends on dozens of small independent factors (cache state, network jitter, GC pauses, disk contention), no single factor dominates, and the aggregate settles into a predictable bell-shaped pattern. Nature and engineered systems are both full of quantities built from many small independent contributions — measurement noise, human heights, aggregated network delay — so the Gaussian shows up constantly as a reasonable default assumption when you don't have a specific reason to expect something else.

Deep learning leans on the Gaussian for two very practical reasons. First, when we assume "noise" in a model — the gap between a prediction and the true answer — behaves like a Gaussian, a lot of the resulting math (loss functions, likelihood calculations) simplifies beautifully; this is the assumption baked into ordinary mean-squared-error loss, which we'll meet directly in a later lesson. Second, when you initialize a network's weights before training even starts, you need *some* starting distribution, and small random values drawn from a Gaussian (centered at zero, modest variance — a narrow bell, not a wide one) is the standard, well-behaved default — it breaks the symmetry between neurons (so they don't all learn the identical thing) without starting anyone off at an extreme, unstable value.

## Bayes' Rule: Updating Belief With Evidence

Bayes' rule answers a question you already reason about informally: given some new evidence, how should I revise a belief I already held?

Concretely: your spam filter starts with some baseline belief that any given incoming email is spam — say, historically, 30% of your mail is spam, `P(spam) = 0.3`. That's your **prior**. Now you observe a specific piece of evidence: the email contains the word "free." You know from experience that spam emails use the word "free" far more often than legitimate ones do. Bayes' rule tells you exactly how to combine your prior belief with this new evidence to get an updated belief — your **posterior**: given that this email contains "free," what's the revised probability it's spam?

Bayes' rule, written out:

```
P(spam | word) = P(word | spam) × P(spam)
                 ───────────────────────────
                          P(word)
```

where the denominator, `P(word)`, is just a normalizing constant that spreads across both hypotheses — you compute it as `P(word | spam) × P(spam) + P(word | not-spam) × P(not-spam)`, so everything still sums to 1 at the end.

### Worked example

Let's plug in made-up but internally consistent numbers:

- Prior: `P(spam) = 0.3`, so `P(not-spam) = 0.7`
- From historical data: `P("free" | spam) = 0.5` (half of all spam contains the word "free")
- From historical data: `P("free" | not-spam) = 0.1` (only 10% of legitimate mail contains it)

First, compute the numerator — how likely is it that an email is *both* spam *and* contains "free"?

```
P("free" | spam) × P(spam) = 0.5 × 0.3 = 0.15
```

Next, compute the same joint quantity for the *other* hypothesis — how likely is it that an email is *both* not-spam *and* contains "free"?

```
P("free" | not-spam) × P(not-spam) = 0.1 × 0.7 = 0.07
```

Now normalize — add both joint probabilities together to get the overall probability of seeing the word "free" at all, regardless of hypothesis:

```
P("free") = 0.15 + 0.07 = 0.22
```

And finally, the posterior — the fraction of that 0.22 that came from the "spam" hypothesis:

```
P(spam | "free") = 0.15 / 0.22 ≈ 0.6818 ≈ 68.2%
```

So a prior belief of 30% jumped to a posterior belief of about 68% after a single piece of evidence — nearly doubling your confidence, even though the word "free" alone is far from proof. Feed the filter another suspicious word and it updates again from this new 68% starting point, incorporating both pieces of evidence — that's exactly how a real spam filter chains together many weak signals into a confident final probability.

This is precisely the operating pattern of a distributed system reconciling conflicting signals from multiple sensors or replicas — you don't discard your prior state and trust the newest reading blindly, and you don't ignore new readings either. You combine them, weighted by how reliable each source is. Bayes' rule is the formal version of that instinct, and it underlies a huge amount of classical machine learning (and shows up again once we discuss maximum likelihood, later in this series).

## Information Theory: Measuring Surprise

This next part is the one to sit with, because you will hit its consequence — cross-entropy loss — in essentially every classification model you build from here on.

Information theory starts from a deceptively simple idea: **rare events are more informative than common ones.** If I tell you "the sun rose this morning," you've learned almost nothing — you were already certain of it. If I tell you "it snowed in the Sahara today," you've learned a great deal, precisely because it was unlikely. We can turn this into a number: the "information content" (self-information) of an event with probability `p` is defined as `-log2(p)`, measured in bits. This is zero for a certain event (`p = 1`, since `log2(1) = 0`) and grows larger the less likely the event was (as `p` shrinks toward 0, `-log2(p)` grows without bound). Rare, surprising outcomes carry more bits of information than routine, expected ones.

**Entropy** takes this one step further: it's the *average* amount of surprise (self-information) you'd expect from a distribution, if you sampled from it over and over — formally, `-Σ p(x) × log2(p(x))`, summed over every possible outcome `x`.

### Worked example: fair coin vs. skewed coin

**Fair coin** — `P(heads) = 0.5`, `P(tails) = 0.5`:

```
Entropy = -(0.5 × log2(0.5) + 0.5 × log2(0.5))
        = -(0.5 × (-1) + 0.5 × (-1))
        = -(-0.5 - 0.5)
        = 1 bit
```

A fair coin has exactly 1 bit of entropy — the maximum possible for a two-outcome distribution, and the textbook definition of "1 bit of information." Every flip is a genuine 50/50 surprise.

**Skewed coin** — `P(heads) = 0.9`, `P(tails) = 0.1`:

```
log2(0.9) ≈ -0.152
log2(0.1) ≈ -3.322

Entropy = -(0.9 × (-0.152) + 0.1 × (-3.322))
        = -(-0.137 - 0.332)
        = 0.469 bits
```

The skewed coin's entropy (≈0.469 bits) is under half the fair coin's (1 bit) — because most of the time you already know it'll land heads, so there's much less genuine surprise on average, even though the rare tails outcome is individually more surprising than any flip of the fair coin. Entropy captures the *average* surprise across all outcomes, weighted by how often each occurs — a distribution that's totally predictable (heads 100% of the time) drives entropy all the way to zero.

Now for the part that matters most in practice: **cross-entropy.**

Suppose you have the *true* distribution of outcomes (in classification, this is usually a clean fact: this particular email really is spam, so the true distribution puts 100% on "spam" and 0% on "not-spam" — a one-hot distribution). And you have your *model's predicted* distribution. Cross-entropy measures how well your predicted distribution matches the true one — formally, `-Σ true(x) × log2(predicted(x))` — the average surprise you'd experience if you believed your model's predicted distribution, but reality kept generating outcomes according to the true distribution instead. Because the true distribution is one-hot here, every term in that sum is multiplied by 0 except the one term for the actual correct class, so the formula collapses to just `-log2(predicted probability of the correct class)`.

### Worked example: cross-entropy dropping as predictions improve

The true label is definitely class A: `true(A) = 1`, `true(B) = 0`.

**Model 1 predicts 70% A, 30% B:**

```
Cross-entropy = -(1 × log2(0.7) + 0 × log2(0.3))
              = -log2(0.7)
              = -(-0.5146)
              = 0.515 bits
```

**Model 2 predicts 95% A, 5% B** (a much more confident, and correct, prediction):

```
Cross-entropy = -(1 × log2(0.95) + 0 × log2(0.05))
              = -log2(0.95)
              = -(-0.074)
              = 0.074 bits
```

The loss drops from 0.515 bits down to 0.074 bits — about a 7x reduction — purely because the model became more confident in the *correct* answer. This is the numeric core of "cross-entropy loss goes down as predictions improve": it isn't an abstract claim, it's this exact arithmetic, repeated across every training example, millions of times over the course of training. And the flip side is just as sharp — if a model had instead predicted only 2% for the correct class, `-log2(0.02) ≈ 5.64 bits`, a huge loss spike, because confidently betting on the wrong answer is punished far more severely than honest uncertainty ever is.

This is exactly the behavior you want out of a training signal: reward confident correctness, punish confident wrongness severely, and be gentler on honest uncertainty. That's why cross-entropy loss is the default choice for training classifiers — minimizing it is mathematically the same operation as maximizing the probability the model assigns to the correct answers across your whole training set (a principle called maximum likelihood, which we're deferring to a later lesson, but the short version is: making correct answers more probable and minimizing cross-entropy are two descriptions of the identical goal).

## KL Divergence, Briefly

One closely related idea, worth a short section: **KL divergence** (Kullback-Leibler divergence) is a way to measure how different two probability distributions are from each other — how much "extra surprise" you incur by using one distribution as a stand-in for another. It's not quite a distance in the strict mathematical sense (measuring the gap from A to B isn't the same as measuring the gap from B to A), but the practical reading is simple: KL divergence is zero when two distributions are identical, and grows as they diverge. Concretely, if your model's predicted distribution ever became *exactly* the true distribution — say, predicting exactly 100% A / 0% B when the true label really is 100% A — the KL divergence between them would be exactly 0, and cross-entropy would collapse down to just the entropy of the true distribution itself (which, for a one-hot true label, is 0 bits — a certain event has zero surprise, so there's nothing left to minimize). The reason KL divergence is worth knowing by name is its relationship to cross-entropy: cross-entropy between the true distribution and your model's prediction equals the entropy of the true distribution (a fixed, unavoidable baseline of surprise) plus the KL divergence between the two distributions (the *extra* surprise caused specifically by your model's predictions being wrong). Since the true distribution's entropy doesn't change no matter what your model does, minimizing cross-entropy during training is, in effect, minimizing KL divergence — pushing your model's predicted distribution to look as close as possible to reality.

## Vocabulary

| Term | Plain meaning |
|---|---|
| Random variable | A quantity whose value is drawn according to some set of probabilities rather than being fixed. |
| Probability distribution | The full list (discrete) or curve (continuous) of how likely each possible value of a random variable is. |
| PMF / PDF | Probability mass function (discrete case — direct probabilities) vs. probability density function (continuous case — probabilities come from areas under the curve). |
| Expectation | The long-run average value of a random variable if you sampled it forever. |
| Variance | How spread out a distribution's values typically are around the expectation — the average of the squared deviations from the mean. |
| Bernoulli distribution | The distribution of a single yes/no event with some probability `p` of "yes" and `1-p` of "no." |
| Gaussian (Normal) distribution | The bell curve; described by a mean (location of the peak) and a variance (width of the bell — small variance is narrow/confident, large variance is wide/uncertain); shows up naturally whenever many small independent effects sum together. |
| Prior | Your belief about something before seeing a specific piece of new evidence. |
| Posterior | Your updated belief after incorporating that evidence, via Bayes' rule. |
| Self-information | A measure of how surprising a single event is, `-log2(p)`; higher for rarer events. |
| Entropy | The average surprise you'd expect from repeatedly sampling a distribution; a measure of how unpredictable it is; 1 bit for a fair coin, less than 1 bit for any skewed coin. |
| Cross-entropy | The average surprise of using a predicted distribution as your guide when the true distribution is actually generating the outcomes; collapses to `-log2(predicted probability of the correct class)` for one-hot labels; the standard loss function for training classifiers. |
| KL divergence | A measure of how different one probability distribution is from another; zero when the two distributions are identical; cross-entropy = entropy of the truth + KL divergence between prediction and truth. |

---

## Where we'll go next
**Lesson 4 — Numerical Computation.** Now that you know what a model is trying to minimize (cross-entropy) and what randomness is doing in training, we need to talk about the unglamorous but critical reality of doing this arithmetic on real computers — floating-point limits, numerical stability, and why some mathematically-equivalent formulas behave very differently once you actually run them.

Reply **ok** to continue, or ask anything about today's lesson first.
