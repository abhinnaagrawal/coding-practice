# Lesson 3 — Probability and Information Theory

Based on: Goodfellow, Bengio, Courville — *Deep Learning*, Chapter 3 (original explanation, not excerpted)

Lesson 2 gave you the mechanical vocabulary — vectors, matrices, the operations a layer actually performs. But matrix multiplication alone doesn't explain why your incident classifier's output looks like "P(incident) = 0.83" instead of a flat "yes." That's probability's job — the second language deep learning speaks fluently, and the one that ends, near the close of this lesson, in cross-entropy: the exact quantity you'll watch go down every time you train a real model.

**Before I explain — guess:** why would a classifier output "0.83" instead of just "yes, incident" — what would you actually lose by forcing it to commit to a hard yes/no?

## Why Probability Shows Up At All

As a backend engineer, you're used to systems where uncertainty is handled explicitly — retries, timeouts, circuit breakers — as a bolt-on to otherwise deterministic logic. Deep learning is different: uncertainty isn't bolted on, it's baked into the object the model produces.

Two separate reasons. First, the *task itself* is uncertain: a minute with slightly elevated error rate and normal everything else isn't cleanly "incident" or "not" — it's ambiguous, the same way a borderline email isn't cleanly spam or not. A classifier that outputs "0.83" is being honest about that ambiguity, and the number is useful downstream — page immediately above 0.9, log-and-watch between 0.5 and 0.9, ignore below that. Force it to output a hard 0/1 and you throw away exactly the information an on-call engineer would want.

Second, *training itself* is a randomized process — weights start random, each step samples a random mini-batch, later architectures randomly disable neurons on purpose (dropout). None of this is a bug; it's a deliberate design choice, the same way you might inject randomized jitter into retry backoff on purpose.

So probability enters at two layers: it's the *language the model's output is expressed in*, and it's a *tool used during training itself*.

## Random Variables and Distributions

A random variable is a quantity whose value isn't fixed — it's drawn according to some set of likelihoods. A probability distribution is the full accounting of those likelihoods: for every possible value, how likely is it?

**Discrete** — a finite (or countable) list of values. Our incident classifier's output is exactly this: not a binary fact but a distribution over `{incident, normal}`:

```
P(incident) = 0.83
P(normal)   = 0.17
```

The two numbers sum to 1 — the defining property of a valid discrete distribution. If your model's output layer ever produced numbers that didn't sum to 1, something upstream is broken (this is literally what softmax exists to guarantee, later in the series).

**Continuous** — the variable can take any value in a range. "How many milliseconds will this request take?" isn't a short list, it's smeared across a continuum — you can't assign nonzero probability to *exactly* 42.0000ms, so you describe a *density* instead: a curve where area under an interval gives you probability. That's the practical difference between a PMF (discrete, direct probabilities) and a PDF (continuous, integrate to get a probability).

Deep learning uses both: classification outputs (like our classifier's incident/normal call) are discrete distributions; predicting a continuous quantity (predicted latency itself, a pixel intensity) uses continuous ones.

## Expectation and Variance

**Expectation** is the long-run average — sample forever, average the results, converge to this. **Variance** measures spread — how far individual draws typically stray from that average, formally the average of the squared deviation from the mean.

### Worked example: same average latency, very different variance

Two latency-prediction models, evaluated against 5 real requests, both averaging exactly 100ms predicted:

**Model A:** 98, 102, 99, 101, 100 → mean 100. Deviations: −2,+2,−1,+1,0 → squared: 4,4,1,1,0 → sum 10 → **variance 2** (σ ≈ 1.41ms).

**Model B:** 40, 160, 20, 180, 100 → mean 100. Deviations: −60,+60,−80,+80,0 → squared: 3600,3600,6400,6400,0 → sum 20000 → **variance 4000** (σ ≈ 63.2ms).

Identical mean, 2000x different variance. Model A is tight and boring near 100ms every time; Model B is "right on average" but wildly optimistic or pessimistic on any *given* request — exactly the p50=100ms/p99=150ms vs. p50=100ms/p99=4s distinction that matters for setting real SLAs. The mean hides exactly what variance reveals.

## Two Distributions Worth Knowing By Name

**Bernoulli** — a single yes/no draw with probability `p` for "yes," `1-p` for "no." One parameter, fully described: `P(x=1) = p`, `P(x=0) = 1-p`. This is exactly the shape of our incident classifier's output for one minute: if historically 12% of minutes across your fleet are incidents, your *prior* — before looking at this minute's metrics at all — is a Bernoulli distribution with `p = 0.12`. Every binary classifier you build is, at its output layer, producing the parameter `p` of a Bernoulli distribution per example (a spam filter's `P(spam)=0.3` prior is the identical shape, just a different domain).

**Gaussian (Normal)** — the bell curve, described by a mean (peak location) and a variance (width). The mean just slides the curve; the variance controls confidence — small variance is a tall narrow bell (draws cluster near the mean, like a low-noise metric), large variance is short and wide (draws scatter, like a noisy one). Loosely, whenever a quantity is the sum of many small independent effects — request latency shaped by cache state, network jitter, GC pauses, disk contention, none dominating — the result tends toward this bell shape (the central limit theorem; think of it as a load-balancing effect on randomness). Deep learning leans on it for two reasons: assuming Gaussian noise around a prediction is what makes ordinary mean-squared-error loss work cleanly (a later lesson), and weight initialization uses small Gaussian-drawn random values to break symmetry between neurons without starting anyone at an unstable extreme.

## Bayes' Rule: Updating Belief With Evidence

Bayes' rule formalizes something you already do informally: given new evidence, how should you revise a belief you already held?

**Before I explain — guess:** your classifier's prior is P(incident) = 0.12 for any random minute. You then observe evidence — sustained error-rate creep over the last 3 minutes, which historically shows up in 55% of real incidents but only 4% of normal minutes. Do you think the posterior P(incident | this evidence) lands closer to 0.12 still, or much higher — and roughly how would you combine those three numbers to find out?

Bayes' rule:

```
P(incident | evidence) = P(evidence | incident) × P(incident)
                          ────────────────────────────────────
                                    P(evidence)
```

where `P(evidence)` normalizes across both hypotheses: `P(evidence|incident)×P(incident) + P(evidence|normal)×P(normal)`.

### Worked example: the incident classifier

- Prior: `P(incident) = 0.12`, so `P(normal) = 0.88`
- `P(evidence | incident) = 0.55` (55% of real incidents show this creep pattern)
- `P(evidence | normal) = 0.04` (only 4% of normal minutes do)

Numerator (joint, incident-and-evidence): `0.55 × 0.12 = 0.066`
Other joint (normal-and-evidence): `0.04 × 0.88 = 0.0352`
Normalizer: `0.066 + 0.0352 = 0.1012`
Posterior: `0.066 / 0.1012 ≈ 0.652 ≈ 65.2%`

A 12% prior jumped to a ~65% posterior from one piece of evidence — this is precisely how a real anomaly-detection system reconciles a low base rate with a suspicious-but-not-conclusive signal, and it's exactly the pattern of reconciling conflicting readings from multiple sensors or replicas: you don't discard your prior, and you don't ignore the new reading either — you combine them, weighted by reliability.

The identical mechanism, different domain — a spam filter with prior `P(spam)=0.3`, `P("free"|spam)=0.5`, `P("free"|not-spam)=0.1` — gives numerator `0.15`, other-joint `0.07`, normalizer `0.22`, posterior `0.15/0.22 ≈ 68.2%`. Same three-step arithmetic, same shape of answer: one weak signal nearly doubles a low prior. Feed either filter a second signal and it updates again from its new posterior, chaining weak evidence into a confident final call — the basis of a lot of classical ML, and it resurfaces once we cover maximum likelihood later in this series.

## Information Theory: Measuring Surprise

This is the part to sit with — you'll hit its consequence, cross-entropy, in essentially every classifier you train from here on.

Information theory starts from: **rare events are more informative than common ones.** "The sun rose this morning" teaches you nothing; "it snowed in the Sahara" teaches you a lot, precisely because it was unlikely. Formalized: the self-information of an event with probability `p` is `-log2(p)` bits — zero for a certain event (`p=1`), unbounded as `p→0`.

**Entropy** is the *average* self-information a distribution produces if you sample it repeatedly: `-Σ p(x) × log2(p(x))`.

### Worked example: how surprised should the classifier's own output be, at itself?

Suppose one minute, before folding in evidence, your classifier's belief is a coin-flip-level `P(incident)=0.5, P(normal)=0.5`:

```
Entropy = -(0.5×log2(0.5) + 0.5×log2(0.5)) = -(0.5×(-1) + 0.5×(-1)) = 1 bit
```

Maximum possible entropy for two outcomes — genuine 50/50 uncertainty, the classifier has no useful signal yet.

Now suppose after seeing metrics it's confident: `P(incident)=0.9, P(normal)=0.1`:

```
log2(0.9) ≈ -0.152,  log2(0.1) ≈ -3.322
Entropy = -(0.9×(-0.152) + 0.1×(-3.322)) = -(-0.137 - 0.332) = 0.469 bits
```

Under half the entropy of the coin-flip case — a confident classifier is, by construction, less "surprised" by its own eventual answer on average, even though *if* it turns out wrong, that outcome individually carries far more bits (`-log2(0.1) ≈ 3.32`) than any coin flip could.

Now, **cross-entropy** — the quantity that actually trains the model. You have the *true* label (this minute really was an incident: `true(incident)=1, true(normal)=0` — one-hot) and the model's *predicted* distribution. Cross-entropy, `-Σ true(x)×log2(predicted(x))`, measures the average surprise of trusting the model's prediction while reality generates outcomes from the true label. Because the true distribution is one-hot, every term vanishes except the correct class's, collapsing the formula to `-log2(predicted probability of the correct class)`.

**Before I explain further — guess:** if the true answer is "incident" and the model predicted `P(incident)=0.02` (confidently wrong), versus predicted `P(incident)=0.5` (honestly unsure) — which do you think gets penalized more heavily, and by roughly how much more?

### Worked example: cross-entropy as the classifier gets more confident and correct

True label: this minute is an incident. `true(incident)=1`.

**Model 1 predicts P(incident)=0.7:** `Cross-entropy = -log2(0.7) = 0.515 bits`

**Model 2 predicts P(incident)=0.95** (more confident, still correct): `Cross-entropy = -log2(0.95) = 0.074 bits`

Loss drops ~7x purely from added correct confidence — the exact arithmetic behind "cross-entropy loss goes down as predictions improve," repeated millions of times over training. The flip side answers the guess above: had the model instead predicted `P(incident)=0.02` for a true incident, `-log2(0.02) ≈ 5.64 bits` — a spike roughly 76x worse than Model 1's honest-ish guess, because confidently betting wrong is punished far harder than honest uncertainty ever is. That asymmetry — reward confident correctness, punish confident wrongness severely, go easy on honest uncertainty — is why cross-entropy is the default loss for training classifiers. Minimizing it is mathematically the same operation as maximizing the probability the model assigns to correct answers across the whole training set (maximum likelihood, deferred to a later lesson).

## KL Divergence, Briefly

**KL divergence** measures how different two distributions are — the "extra surprise" from using one as a stand-in for the other. Not symmetric (A-to-B ≠ B-to-A), but simple in practice: zero when identical, grows as they diverge. Key relationship: `cross-entropy = entropy of the true distribution + KL divergence between prediction and truth`. For a one-hot true label the entropy term is 0 bits, so minimizing cross-entropy during training is, in effect, minimizing KL divergence — pushing the classifier's predicted distribution to match reality as closely as possible.

## Vocabulary

| Term | Plain meaning |
|---|---|
| Random variable | A quantity whose value is drawn according to some set of probabilities rather than being fixed. |
| Probability distribution | The full list (discrete) or curve (continuous) of how likely each possible value of a random variable is. |
| PMF / PDF | Probability mass function (discrete — direct probabilities) vs. probability density function (continuous — probabilities come from areas under the curve). |
| Expectation | The long-run average value of a random variable if sampled forever. |
| Variance | How spread out a distribution's values typically are around the expectation. |
| Bernoulli distribution | The distribution of a single yes/no event with probability `p` of "yes," `1-p` of "no" — the shape of our classifier's incident/normal output. |
| Gaussian (Normal) distribution | The bell curve; mean = peak location, variance = width (small = confident/narrow, large = uncertain/wide); emerges from sums of many small independent effects. |
| Prior | Belief about something before seeing a specific piece of new evidence. |
| Posterior | Updated belief after incorporating that evidence, via Bayes' rule. |
| Self-information | How surprising a single event is, `-log2(p)`; higher for rarer events. |
| Entropy | The average surprise expected from repeatedly sampling a distribution; 1 bit for a coin-flip-level 50/50, less for anything skewed/confident. |
| Cross-entropy | Average surprise of trusting a predicted distribution when the true distribution generates outcomes; `-log2(predicted probability of the correct class)` for one-hot labels; the standard classifier loss. |
| KL divergence | How different one distribution is from another; zero when identical; cross-entropy = entropy of the truth + KL divergence between prediction and truth. |

---

## Quick check

Your classifier outputs `P(incident) = 0.2` for a minute that turns out to be a genuine incident. In your own words: what is the cross-entropy for that prediction (rough arithmetic is fine), and why is it so much worse than if the model had instead predicted `P(incident) = 0.2` for a minute that turned out to be *normal*?

## Where we'll go next

**Lesson 4 — Numerical Computation.** Now that you know what a model is trying to minimize (cross-entropy) and what randomness is doing in training, we need to talk about the unglamorous but critical reality of doing this arithmetic on real computers — floating-point limits, numerical stability, and why some mathematically-equivalent formulas behave very differently once you actually run them.

Answer the check above (even roughly), then reply **ok** to continue.
