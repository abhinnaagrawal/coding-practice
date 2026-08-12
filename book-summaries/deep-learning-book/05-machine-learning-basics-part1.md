# Lesson 5 — What Learning Actually Means

Based on: Goodfellow, Bengio, Courville — *Deep Learning*, Chapter 5 "Machine Learning Basics" (original explanation, not excerpted)

Lessons 2–4 gave you the machinery: vectors and matrices to represent data and layers, probability and information theory to talk about uncertainty and define loss, and gradient descent to actually move a model's parameters toward better ones. What we haven't covered yet is the thing all that machinery is *in service of*: what does it actually mean for a model to "learn," and how do you know if it worked? That's this lesson and the next one — the last stretch of Part I, and arguably the most important ideas in the whole book, because they apply to literally every model you'll ever train, deep or not.

## A Precise Definition of "Learning"

"The model learned to detect spam" is a sentence that sounds obvious until you try to make it precise enough to build something out of it. There's a useful three-part decomposition, originally due to Tom Mitchell, that forces the precision: a program is said to learn if you can name its **task**, its **experience**, and its **performance measure**.

- **Task (T)**: what you're actually asking the system to do. Not "learn spam" — the task is "given the text of an email, output a label: spam or not-spam." Tasks are things like classification, regression, translation, ranking. Note the task is defined independent of how you solve it — you could imagine a human doing this task too.
- **Experience (E)**: what data the system gets to learn from, and how. In our spam example: a pile of emails, each already labeled spam/not-spam by a human. This is "supervised" experience — more on that distinction in Lesson 6.
- **Performance measure (P)**: a number that tells you how well the task is being done, measured on data the system did *not* use to learn. For spam: accuracy (fraction correctly labeled) on emails the classifier has never seen before.

Notice what this definition quietly forces on you: P has to be measured on unseen data, or the whole framework is circular — you could "achieve" perfect performance by just memorizing the labeled emails you were handed, without building anything that generalizes. This isn't a minor footnote. It's the seed of the single most important idea in this lesson, which we'll get to in a moment.

Spam detection is a classification problem, and it's easy to let T/E/P quietly become "the framework for labeling things." It isn't — it applies just as cleanly to predicting a *number*. Take something you've actually lived through as a backend engineer: predicting how long a request to your service will take to respond, given features of the request (payload size, endpoint, time of day, current queue depth, cache hit/miss). Translate it into the same triple:

- **Task (T)**: regression — given a feature vector describing an incoming request, output a single real number, the predicted response time in milliseconds.
- **Experience (E)**: your access logs. Millions of historical (request features, actual observed latency) pairs, harvested from production over the last few months.
- **Performance measure (P)**: mean squared error between predicted and actual latency, computed on a batch of requests the model never trained on — say, the request logs from a day you deliberately excluded from training.

Sit with why P has to be MSE here and not "accuracy" — there's no discrete right/wrong label to be correct or incorrect about, only a distance between a predicted number and an observed one, and MSE penalizes big misses more than small ones (a natural fit if your on-call instinct is that a 50ms prediction error is a shrug but a 5-second prediction error is a real problem for your capacity planning). Notice, too, that everything said about spam applies unchanged: if you compute your MSE using the same request logs the model trained on, you have no idea if the model captured something general about *what makes requests slow* (large payloads, cold caches, queue backpressure) versus just curve-fitted the specific noise of the exact millisecond timings it happened to see during training. The task changed from classification to regression, the performance measure changed from accuracy to MSE, but the discipline required — evaluate only on data the model didn't touch — is identical. That invariance is the whole point of stating the framework abstractly: it's not a spam-classifier recipe, it's a lens you can point at any learning system, discrete output or continuous, and ask the same three questions.

As a second sanity check: a house price predictor. T = predict price from square footage, location, etc. (regression). E = historical sales records with known prices. P = average error between predicted and actual price on sales it wasn't trained on. If you can't cleanly state a system's T/E/P, you probably don't understand yet what problem it's solving.

## Training Set vs. Test Set: Why the Split Isn't Optional

Here's a scenario every backend engineer has seen and will recognize is a bad idea instantly: you write a function, you write unit tests while writing it — tuning the code until every one of *those specific tests* passes — and then you ship, having never run the code against any input you didn't personally invent. You'd never trust that code. Not because it's necessarily wrong, but because passing tests you wrote *while looking at the implementation* proves almost nothing about how the code behaves on inputs you didn't think of. Bugs live exactly in the inputs you didn't imagine.

Push that analogy one notch further, because it maps onto ML almost embarrassingly precisely. Imagine a bug report lands with three example inputs that trigger a crash. You fix the code, then write a test that re-runs exactly those three example inputs and confirms they no longer crash. Green check, ship it. Two days later a fourth, slightly different input — same underlying bug, different specific values — takes the service down again. What happened is obvious in hindsight: your "test" only verified that you'd memorized the fix for the *literal examples someone handed you*, not that you'd understood and fixed the general class of bug those examples were instances of. A test suite built entirely from copy-pasted repro cases can hit 100% pass rate while the actual defect — the general shape of the problem — survives untouched.

Machine learning has the identical failure mode, and it's so central that the field has a formal ritual to prevent it: you never, ever evaluate a model's real performance using the same data it was fit on. You split your available labeled data into (at least) two disjoint sets before you start:

- **Training set** — the data the learning algorithm is allowed to look at and adjust parameters against.
- **Test set** — data held back, untouched, that the model only sees once, at the very end, purely to report a number.

Make this concrete with numbers small enough to hold in your head. Suppose you have a toy dataset of 10 points: 10 (x, y) pairs that roughly follow a line, each with a bit of random noise added to y. If your model has enough capacity — enough tunable knobs — it can find a curve that passes through all 10 points *exactly*, achieving 0% error on those 10, full stop, no ambiguity. That looks, on paper, like a perfect model. Now show it an 11th point, drawn from the exact same underlying process, that it has never seen. Because the curve was contorted to hit the *specific* noisy y-values of the first 10 points rather than the *straight-line trend* those 10 points were sampled around, the curve at the 11th point's x-coordinate is often nowhere near the true line — the fitted curve does whatever bending was necessary to hit point 9 and point 10 exactly, and that bending has no reason to line up with where point 11 actually falls. 0% error on the 10 memorized points, potentially large error on the 11th — and the 11th point is the only one that tells you anything about whether the model learned "these points lie roughly on a line" versus "these are the 10 specific numbers I was shown."

That's the entire justification for the training/test split in one paragraph: a model can achieve 100% accuracy on its training set and be useless in production — it may have simply memorized the specific 10,000 emails it was shown, including their typos and sender addresses, rather than learning what makes something spammy in general. Training accuracy tells you how well the model fits the data you gave it. Test accuracy is your only honest estimate of how it'll do on emails that land in an inbox tomorrow. If you only ever check the first number, you are, precisely, the engineer who ships a fix validated only against the exact repro steps in the bug report.

## Generalization Is the Actual Goal

Say this one clearly, because it's easy to let it slide past as a truism: **the goal of machine learning is not to fit the training data. It is to perform well on new data the model has never seen.** Fitting the training data is a means, and only a means, to that end — and past a certain point, fitting the training data even *harder* actively makes the real goal worse. This is counterintuitive if you're used to thinking of "loss went down" as strictly good news, so it's worth sitting with.

Think of the training set as a *sample* from some much larger, unobservable population — all possible emails anyone will ever receive, all possible photos of cats, all possible sensor readings your service will ever see, all possible requests your API will ever be sent. Your job isn't to explain the sample; it's to approximate the process that generates the population, using the sample as your only evidence about it. A model that generalizes well has found something true about the underlying pattern. A model that doesn't has just found something true about your particular sample — including its noise, quirks, and accidents — and mistaken that for the pattern.

## Capacity, Underfitting, and Overfitting

"Capacity" is the term for roughly how complicated a set of functions a model is able to represent. A straight line has low capacity — it can only represent linear relationships. A degree-20 polynomial has enormous capacity — it can bend and wiggle enough to pass exactly through almost any finite set of points you give it.

Make this concrete: imagine 8 noisy data points scattered on a chart, roughly tracing out an upward-sloping line, each nudged a little off the line by measurement noise. You have three candidate models to fit them:

- **Degree 0 (a flat constant)** — the "model" is just a single horizontal line at the average height of all 8 points, with no slope at all. This is about as little capacity as a curve-fitter can have.
- **Degree 1 (a straight line)** — one slope, one intercept, two knobs total.
- **Degree 7 (a 7th-degree polynomial)** — with exactly 8 points and 8 free coefficients' worth of flexibility, a degree-7 polynomial can be bent to pass through *all 8 points exactly*, no matter where the noise happened to push them.

Picture fitting a curve through that noisy scatter of points that roughly follow a gentle upward trend, with some random jitter around it:

```
UNDERFITTING            GOOD FIT              OVERFITTING
(degree 0, flat,      (degree 1, straight   (degree 7, wiggle,
 too simple)           line, matches trend)  chases every point)

  .    ___             .    _.-'                . /\  .  /\
 . ----------.         . _-'  .                 ./  \/\ / \  .
._----   .   ---       _'   .                   /  . \/ .\ /\
   .        .          .        .               .        `  .
```

Now ask the question that actually matters: a brand-new x-value shows up — a 9th point from the same underlying process — and you need a prediction for it. What does each model say?

- The **flat degree-0 fit** predicts the same average value no matter what x is. If the true trend is rising, this fit is systematically wrong everywhere except by coincidence near the middle of the range — it's not reacting to noise, but it's also not reacting to the actual signal. Consistently, boringly wrong.
- The **degree-1 line** gives a prediction that tracks the real upward trend reasonably well. It won't hit any single point exactly — it wasn't built to — but its error at a new point is generally small and in the same ballpark as its error anywhere else along the line.
- The **degree-7 polynomial** is the dangerous one. Between the points it was fit to, a high-degree polynomial doesn't calmly interpolate — it overshoots and undershoots, arcing wildly above and below the true trend in the gaps, because satisfying "pass exactly through these 8 specific points" with only 8 points to anchor it leaves the curve free to do almost anything in between them. A new x-value that falls between two training points — which is most new x-values — can land the prediction far off the true line, sometimes by a large margin, precisely because the polynomial was busy contorting itself to hit noisy points exactly rather than tracking the trend those points were sampled from.

The straight line **underfits**: it's too simple to capture even the real trend, so it does poorly on both the training data *and* new data. This is also called **high bias** — the model's assumptions (in this case, "the relationship is linear") are systematically wrong, no matter how much data you give it.

The wiggly high-degree curve **overfits**: it snakes through every single training point, including the noise, achieving near-zero training error. But it has fit the noise, not the trend, so on a fresh batch of points from the same underlying process it does badly — it's confidently wrong in the gaps between where it saw training data. This is called **high variance**: retrain that same overly-flexible model on a different random sample from the same population, and you'll get a wildly different-looking wiggly curve each time. The model's output is unstable, hypersensitive to exactly which noise it happened to see.

The good fit sits in between: enough capacity to capture the real trend, not so much that it chases noise. Plotted as training error and test error against increasing model capacity, you get the classic U-shape for test error — it falls as capacity increases from "too simple," bottoms out, then rises again as capacity increases past "just right" into "too complex." Training error, meanwhile, just keeps monotonically falling as capacity increases — which is exactly why training error alone is a misleading signal, and why the U-shaped curve only shows up when you plot it against a held-out test set.

## The Bias-Variance Tradeoff, Stated Plainly

Bias and variance aren't two unrelated failure modes — they're two ends of the same dial, and that dial is capacity.

- **Bias**: how wrong your model's assumptions are on average, regardless of how much data you throw at it. A linear model applied to a genuinely curved relationship has irreducible bias — no amount of additional training data fixes it, because the *shape* of what it can represent is the limitation.
- **Variance**: how much your model's predictions would change if you retrained it on a different random sample drawn from the same population. A model with too much capacity latches onto sample-specific noise, so a different sample produces a meaningfully different fitted model.

Here's the picture that makes the two concepts click side by side. Imagine you draw 5 different random samples from the same underlying population — 5 different batches of 8 noisy points, all generated by the same true (unknown) process — and for each batch you retrain the same model type from scratch. Now look at the 5 resulting fitted curves overlaid on one chart.

A **high-bias, low-capacity model** (say, our degree-0 flat line) produces 5 fits that all land in roughly the same wrong place — 5 nearly-identical flat lines, all sitting at more or less the same height, all equally failing to capture the upward trend. Consistent with each other, consistently wrong relative to the truth. If you had to describe the 5 fits in one phrase: "boringly identical, and identically off."

A **high-variance, high-capacity model** (our degree-7 polynomial) produces 5 fits that look nothing alike — each one wiggles through its own batch's specific noise, so where one curve dips sharply, another spikes, and a third does something else entirely. Overlay the 5 wiggly curves and you get a tangle, agreeing only roughly near the training points and disagreeing wildly everywhere else, especially in the gaps between points. If you had to describe these 5 fits: "wildly inconsistent with each other, and each one plausible-looking only where it was anchored by data."

That's the dual mental picture worth keeping: bias is measured by how far the *average* fit sits from the truth (correctness), variance is measured by how much the fits *disagree with each other* (consistency). A good model needs both — close to the truth on average, and stable enough that a different equally-valid sample of training data wouldn't have produced a wildly different model. Low-capacity models tend toward high bias, low variance (consistently wrong in the same way — stable, but stably inaccurate). High-capacity models tend toward low bias, high variance (capable of representing the truth, but also flexible enough to represent noise, and they don't reliably tell the two apart). You cannot independently drive both to zero by choosing capacity alone — pushing capacity down to kill variance raises bias, and pushing it up to kill bias raises variance. Everything about model selection, regularization, and getting more data is, underneath, an attempt to manage this tradeoff rather than escape it.

## No Free Lunch: There's No Universally Best Algorithm

One more piece of humility before you start reaching for algorithms: the **No Free Lunch theorem** says that, averaged across *all possible* data-generating problems, every learning algorithm performs identically — including a lookup table that predicts randomly. There is no algorithm that is simply "the best," full stop, independent of the problem.

This isn't as discouraging as it sounds. It just means every algorithm carries built-in assumptions — an **inductive bias** — about what kinds of patterns are likely to appear in the data it'll be pointed at. Linear models assume relationships are roughly linear. Convolutional networks (coming in Part II) assume nearby pixels are more related than far-apart ones. A concrete pair worth holding onto: a convolutional network tuned for image classification leans hard on the assumption that local spatial neighborhoods of pixels carry meaningful structure — exactly the assumption that makes it excellent at recognizing cats and terrible, or at best no better than simpler methods, at spotting fraudulent transactions in a spreadsheet of account age, transaction amount, and merchant category, where there is no spatial neighborhood structure to exploit at all and a boosted tree over tabular features will typically win outright. None of these assumptions are true in general — they're true *for the specific kinds of problems humans actually care about*, which is why they work well in practice despite the theorem. Practical ML is not a search for "the best algorithm" in the abstract; it's matching an algorithm's built-in assumptions to your actual problem.

## Regularization: Deliberately Handicapping the Model

If overfitting comes from a model having more capacity than the data can responsibly support, one fix is to reduce capacity outright (fewer parameters, a simpler architecture). But there's a subtler, often more effective fix: keep the model's raw capacity high, but add a penalty that discourages it from using that capacity in ways that look like overfitting. This is **regularization** — we'll go much deeper on it in Part II, but the concept belongs here, at the point where the problem it solves first appears.

The classic example: penalize large weight values. Instead of only minimizing your loss function, you minimize loss *plus* a term proportional to the size of the model's weights (say, the sum of their squares — the squared L2 norm from Lesson 2's discussion of vector norms). Make this concrete with an example you could actually compute. Suppose two different weight vectors both achieve essentially the same, very low training error on some regression task:

- **w_a = [50, -48, 62]**
- **w_b = [2, -1, 3]**

Both fit the training data about equally well — that's the premise. But think about what it takes for a model to *need* weights as large as 50, -48, and 62 to explain the training data: those large, closely-canceling values are typically doing something delicate, like nearly cancelling each other out on average while reacting very strongly to small differences between individual training examples — the kind of behavior you get when a model has learned to react to the specific noise pattern in this particular training sample rather than to a broad, robust signal. A tiny perturbation to the input, or a slightly different training sample, and a weight vector like w_a can swing the prediction wildly, because the huge, offsetting coefficients amplify small changes. w_b, with modest, non-extreme weights that aren't relying on near-perfect cancellation, produces predictions that move more gently and predictably as inputs change — it's the weight-vector equivalent of the smooth degree-1 line rather than the wiggly degree-7 polynomial. Given a choice between two models with equal training performance, the one with smaller weights is the safer bet on new data, because it isn't quietly counting on delicate cancellations that happened to work out for *this* training set.

The L2 penalty operationalizes that preference directly: it adds sum-of-squared-weights to the loss, so the optimizer pays a cost for reaching w_a's territory that it doesn't pay for staying near w_b's, and it will only accept large weights if they buy a large enough reduction in training error to be worth the penalty. A model chasing noise typically needs large, precisely-tuned weights to make its curve swing sharply near individual noisy points — think of the wiggly polynomial above, whose coefficients have to be large and delicately balanced to produce all those oscillations. Penalizing large weights makes that expensive, and the optimizer settles for a smoother fit that gives up a little training accuracy in exchange for a lot more stability on new data. In our vocabulary from above: regularization is a deliberate, controlled way to trade a bit of bias for a meaningful reduction in variance.

## Vocabulary

| Term | Plain meaning |
|---|---|
| Task / Experience / Performance measure (T/E/P) | Mitchell's framing for precisely defining "learning": what the system does, what data it learns from, and how you score it — always on unseen data. Applies equally to classification (accuracy) and regression (e.g. MSE) problems. |
| Training set | Data the model is allowed to adjust its parameters against. |
| Test set | Held-out data used only to report how the model does on inputs it has never seen — your honest estimate of real-world performance. |
| Generalization | Performing well on new, unseen data — the actual goal of learning, as opposed to fitting the training data itself. |
| Capacity | Roughly, how complex a function a model is able to represent — how "flexible" its family of possible fits is (e.g. polynomial degree). |
| Underfitting / high bias | Model too simple to capture the real pattern; does poorly even on training data; consistently wrong in the same way across resamples. |
| Overfitting / high variance | Model complex enough to fit noise as if it were signal; does great on training data, poorly on new data; wildly different fits across resamples. |
| Bias-variance tradeoff | The structural tension where reducing one (via capacity) tends to increase the other; measured as correctness-on-average (bias) vs. consistency-across-resamples (variance). |
| No Free Lunch theorem | No algorithm is best across all possible problems; every algorithm encodes assumptions (inductive biases) that make it suited to some problems (e.g. images) and poorly suited to others (e.g. tabular fraud data). |
| Regularization | Deliberately constraining or penalizing a model (e.g. an L2 penalty on weight magnitude) to trade a little bias for less variance and reduce overfitting. |

---

## Where we'll go next

**Lesson 6 — How Models Actually Learn: Estimators and the Path to Deep Learning.** Today was about the *goal* (generalization) and the *failure modes* (under/overfitting) of learning. Next we get concrete about the *mechanism*: what it formally means to "estimate" something from data, why minimizing cross-entropy loss (Lesson 3) is secretly the same thing as maximum likelihood estimation, why training uses random mini-batches instead of the whole dataset, and how every ML algorithm — including deep networks — is built from the same four interchangeable parts. It's also the last lesson in this series, closing with what classical methods can't do and why that opens the door to Part II.

Reply **ok** to continue, or ask anything about today's lesson first.
