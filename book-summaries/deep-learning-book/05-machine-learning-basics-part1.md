# Lesson 5 — What Learning Actually Means

Based on: Goodfellow, Bengio, Courville — *Deep Learning*, Chapter 5 "Machine Learning Basics" (original explanation, not excerpted)

Lessons 2–4 gave you the machinery: vectors and matrices, probability and loss, gradient descent. What we haven't covered is what that machinery is *in service of*: what does it mean for a model to "learn," and how do you know it worked? Back to our incident classifier from Lesson 1.

**Before I explain — guess:** suppose you train the incident classifier on the last 6 months of labeled minutes, and it scores 99.9% accuracy — on those same 6 months. Would you trust it to catch tomorrow's incident? Why or why not?

## A Precise Definition of "Learning": T/E/P

"The model learned to detect incidents" sounds obvious until you try to make it precise. Tom Mitchell's decomposition forces the precision: a program learns if you can name its **task**, its **experience**, and its **performance measure**.

- **Task (T)**: given a minute's metrics (latency, error rate, CPU, request rate), output a label — incident or normal. Defined independent of *how* you solve it; a human on-call engineer could do this task too.
- **Experience (E)**: months of historical minutes, each already labeled incident/normal (by past postmortems, alerts, or manual review). This is "supervised" experience — more in Lesson 6.
- **Performance measure (P)**: accuracy, or better, precision/recall, computed on minutes the classifier never trained on.

That last clause — measured on data the model *didn't* train on — is what your guess above was circling. If P is measured on the same minutes E came from, the framework is circular: you could "win" by simply memorizing which of those specific minutes were labeled incidents, without learning anything about what makes a minute incident-like in general. That's the seed of the whole lesson.

The same triple reframes cleanly for a regression version of the same instinct: predicting a request's response time from its features. T = output a real number, not a label. E = (request features, observed latency) pairs from your access logs. P = mean squared error on a held-out day of logs — MSE rather than accuracy because there's no discrete right/wrong, only a distance, and it penalizes big misses harder than small ones. Task and P changed; the discipline — evaluate only on data the model didn't touch — didn't.

## Training Set vs. Test Set: Why the Split Isn't Optional

You've lived this bug already, just not in ML clothing: a bug report lands with three repro inputs, you fix the code, write a test that re-runs exactly those three inputs, get a green check, ship it. Two days later a fourth, slightly different input — same underlying bug — takes the service down again. You'd verified you'd memorized the fix for the *literal examples you were handed*, not that you'd fixed the general class of bug.

ML has the identical failure mode, so the field has a formal ritual against it: never evaluate a model on the data it was fit on. Split your labeled minutes into disjoint sets before training:

- **Training set** — historical minutes the learning algorithm is allowed to adjust its parameters against.
- **Test set** — historical minutes held back, untouched, seen once at the end purely to report a number.

Say you have 10,000 historical minutes, each labeled incident or normal. You hold out the most recent 2,000 as test and train only on the older 8,000. If your classifier has enough capacity, it can achieve 0% error on the 8,000 training minutes — including correctly flagging every quirky one-off incident, like the specific afternoon a bad config push spiked errors for exactly six minutes. That looks perfect on paper. Now run it on the 2,000 held-out minutes. If it fit the *general shape* of an incident (error creeping while resources stay calm), it does well. If it fit the *specific 8,000 minutes*, including which exact minutes happened to be incidents and why, it does badly on the 2,000 it's never seen — because tomorrow's incident won't be caused by that same six-minute config push. Training accuracy tells you how well the model fits the minutes you gave it; test accuracy is your only honest estimate of how it does on next week's minutes.

## Generalization Is the Actual Goal

Say this one clearly: **the goal is not to fit the training minutes. It's to perform well on minutes the model has never seen.** Fitting training data is a means, not the end — and past a point, fitting it *harder* makes the real goal worse.

Think of your 8,000 training minutes as a sample from a much larger, unobservable population: every minute your service will ever produce, past and future. Your job isn't to explain the sample; it's to approximate the process generating it. A classifier that generalizes has found something true about what an incident actually looks like. One that doesn't has found something true about your particular 8,000 minutes — including their noise and one-off quirks — and mistaken that for the pattern.

## Capacity, Underfitting, and Overfitting

**Before I explain — guess:** if you gave the incident classifier more capacity — more features, more nuanced combinations of signals, enough flexibility to distinguish every historical minute individually — would that make it strictly better at catching tomorrow's incidents, or could it backfire?

"Capacity" is roughly how complicated a function a model can represent. Picture three classifiers of increasing capacity, fit to the same historical minutes (plotted here as a noisy scatter of "how anomalous" a minute looks vs. time, with real incidents clustering above a rising trend):

```
UNDERFITTING            GOOD FIT              OVERFITTING
(ignores signals,     (tracks the real       (memorizes each
 flags nothing/all)    incident pattern)      historical minute)

  .    ___             .    _.-'                . /\  .  /\
 . ----------.         . _-'  .                 ./  \/\ / \  .
._----   .   ---       _'   .                   /  . \/ .\ /\
   .        .          .        .               .        `  .
```

- **Degree-0 capacity**: a classifier that ignores the metrics and always predicts the majority class ("normal," since most minutes are normal). Consistently wrong whenever an incident actually happens — it never reacts to the real signal at all.
- **Degree-1 capacity**: a simple linear boundary over the four metrics — roughly "flag when this weighted combination crosses a threshold." Tracks the genuine incident pattern (error rate climbing while resources stay calm) reasonably well, without being anchored to any single historical minute.
- **Very high capacity**: enough flexibility to carve out precisely the historical minutes that were labeled incidents — including the one caused by a bad config push, another by a flaky dependency, each stitched in as its own special case. It hits 0% training error. But on a new minute tomorrow, caused by something it's never specifically seen, this classifier has no reason to generalize — it never learned "error creeping + resources calm," it learned "these exact historical minutes, for these exact reasons."

This maps to your guess: more capacity isn't free. The degree-0 classifier **underfits** — too simple, wrong on training and new minutes alike, also called **high bias**: its assumptions are wrong no matter how much data you feed it. The maximum-capacity classifier **overfits** — near-zero training error, poor on new minutes, also called **high variance**: retrain it on a different 8,000-minute window and you get a completely different set of memorized special cases. Plotted against capacity, test error is U-shaped — falls, bottoms out, rises again — while training error just keeps monotonically falling, which is exactly why training error alone is a misleading signal.

## The Bias-Variance Tradeoff, Stated Plainly

Bias and variance are two ends of one dial: capacity.

- **Bias**: how wrong the model's assumptions are on average, regardless of data volume. A model too simple to represent "error creeping while resources calm" has irreducible bias — more historical minutes don't fix a shape limitation.
- **Variance**: how much the fitted model would change if retrained on a different sample from the same population.

Picture retraining the same classifier from scratch on 5 different 8,000-minute historical windows (same service, different months). A **high-bias, low-capacity** classifier (our degree-0 one) produces 5 versions that all land in roughly the same wrong place — consistently, boringly wrong the same way each time. A **high-variance, high-capacity** classifier produces 5 versions that look nothing alike — each one has memorized its own window's specific one-off incidents, so one flags Tuesday-morning traffic ramps as incidents, another doesn't, a third flags something else entirely. Bias is how far the *average* version sits from the truth; variance is how much the versions *disagree with each other*. Pushing capacity down to kill variance raises bias; pushing it up to kill bias raises variance — you manage the tradeoff, you don't escape it.

## No Free Lunch: There's No Universally Best Algorithm

The **No Free Lunch theorem**: averaged across *all possible* data-generating problems, every learning algorithm performs identically — including one that guesses randomly. No algorithm is simply "the best," independent of the problem.

This isn't as discouraging as it sounds — it just means every algorithm carries an **inductive bias**, a built-in assumption about what patterns to expect. A convolutional network (Part II) assumes nearby pixels are more related than far-apart ones — excellent for recognizing cats, no better than simpler methods for spotting fraud in a spreadsheet of transaction amounts and merchant categories, where there's no spatial neighborhood to exploit. Our incident classifier lives in that second world: four independent metrics per minute, tabular, no spatial structure — a boosted tree or a modest linear/logistic model over engineered features is the right inductive bias here, not a CNN. Matching an algorithm's assumptions to your actual problem is the job; "best algorithm" in the abstract isn't a coherent target.

## Regularization: Deliberately Handicapping the Model

**Before I explain — guess:** an L2 regularization penalty is added directly to the loss the optimizer minimizes. What do you think it actually penalizes — the model being *wrong*, or something else entirely?

If overfitting comes from having more capacity than the data responsibly supports, one fix is fewer parameters. A subtler fix: keep capacity high, but penalize the optimizer for using it in overfitting-shaped ways. That's **regularization** — add a term proportional to the size of the model's weights (sum of squares, the squared L2 norm from Lesson 2) to the loss.

So the answer to the guess: it doesn't penalize wrongness directly — it penalizes *weight magnitude*, as a proxy for the kind of fragile fit that's usually chasing noise. Concretely: suppose our classifier weighs four metrics (say, three of them after some encoding), and two different weight vectors both achieve the same low training error:

- **w_a = [50, -48, 62]**
- **w_b = [2, -1, 3]**

Weights that large and closely-canceling are typically doing something delicate — nearly cancelling on average while reacting violently to small differences between individual training minutes. That's exactly the behavior of a classifier that's learned this particular 8,000-minute sample's noise rather than a robust signal: a tiny shift in tomorrow's metrics and w_a's prediction swings wildly, because the huge offsetting coefficients amplify small input changes. w_b's modest weights move more gently and predictably as inputs change — the weight-vector analog of the smooth degree-1 boundary rather than the wiggly high-capacity one. The L2 penalty makes reaching w_a's territory cost something the optimizer only pays if it buys a real reduction in error — otherwise it settles for something closer to w_b. In our vocabulary: regularization deliberately trades a bit of bias for a meaningful cut in variance.

## Vocabulary

| Term | Plain meaning |
|---|---|
| Task / Experience / Performance measure (T/E/P) | Mitchell's framing for precisely defining "learning": what the system does, what data it learns from, and how you score it — always on unseen data. |
| Training set | Data the model is allowed to adjust its parameters against. |
| Test set | Held-out data used only to report performance on inputs never seen — your honest estimate of real-world behavior. |
| Generalization | Performing well on new, unseen data — the actual goal, as opposed to fitting the training data itself. |
| Capacity | Roughly, how complex a function a model can represent — how flexible its family of possible fits is. |
| Underfitting / high bias | Too simple to capture the real pattern; poor even on training data; consistently wrong the same way across resamples. |
| Overfitting / high variance | Complex enough to fit noise as signal; great on training data, poor on new data; wildly different fits across resamples. |
| Bias-variance tradeoff | Reducing one (via capacity) tends to increase the other; measured as correctness-on-average (bias) vs. consistency-across-resamples (variance). |
| No Free Lunch theorem | No algorithm is best across all possible problems; every algorithm's inductive bias suits some problems and not others. |
| Regularization | Penalizing a model (e.g. L2 on weight magnitude) to trade a little bias for less variance. |

---

## Quick check

Your incident classifier gets a new feature: "seconds since last deploy." You add it, and capacity clearly went up — training accuracy jumps to 99.9%. Test accuracy on held-out minutes barely moves, maybe ticks down slightly. In your own words: what's happening here (bias, variance, both, neither), and would you keep the feature or drop it?

## Where we'll go next

**Lesson 6 — How Models Actually Learn: Estimators and the Path to Deep Learning.** Today was the *goal* (generalization) and the *failure modes* (under/overfitting). Next: what it formally means to "estimate" something from data, why minimizing cross-entropy (Lesson 3) is secretly maximum likelihood estimation, why training uses random mini-batches instead of the whole dataset, and how every ML algorithm — including deep networks — is built from the same four interchangeable parts, still tracking our incident classifier. Last lesson in the series.

Answer the check above (even roughly), then reply **ok** to continue.
