# spinf/reviews: reviews and customer feedback

One read of a review (a product review, an app-store review, a review of a restaurant or a hotel, a free-text survey
answer) answers, in parallel, everything a feedback pipeline wants to know: the overall opinion and the star rating
the text implies, what the review is about, which aspects it praises or criticises, whether it asks for something
(a feature, a fix, an answer), what the reviewer intends (a refund, a repurchase, a switch, a recommendation, a
warning), what it mentions (a competitor, a previous version, how long the product has been used, a gift), its tone,
whether it reads like a promotion or an incentivised review, and the risks it carries (a safety report, a legal
threat, abuse, personal data). The pack returns a probability and a normal-traffic percentile for every category,
flags for the deciding ones, and a decision: does this review need a follow-up?

Reviews are short, about 80 tokens, so this is the pack where one read answers the most questions per token of
input: 105 prompts on a few sentences.

Model: `spinf/gemma-4-12b-fp8`. Media: text. Contexts: `review` (default, a product or a service), `app` (an app-store
review), `place` (a restaurant, a hotel, a shop), `survey` (a free-text survey answer): pick the one that matches the
source, it tells the model what it is reading.

## What it scores

Every category is on the probability scale: the score is the model's probability of the answer (0.5 means as likely
as not), and the percentile says how unusual that score is among everyday reviews.

| Group | Categories | How to read them |
|---|---|---|
| `overall.*` | positive, negative, mixed, neutral | The **overall opinion**: a single choice, the four sum to about one. |
| `stars.*` | 1, 2, 3, 4, 5 | The **rating the text implies**: the five probabilities; their expected value is a rating on the 1–5 scale. |
| `subject.*` | the product itself, the seller, the delivery, the customer support, something else | What the review is **mainly about**: the argmax. |
| `aspect.<a>.mentioned` | 16 aspects: price, quality, durability, design, fit, usability, performance, reliability, features, comfort, delivery, packaging, service, accuracy, cleanliness, atmosphere | Whether the review says anything about the aspect (two framings averaged). |
| `aspect.<a>.*` | positive, negative, mixed, neutral per aspect | The **reading**, conditional on the mention: praised, criticised, both, or mentioned without judgement. Meaningful only where `mentioned` is high; the model almost always picks praised or criticised, so treat `positive` against `negative` as the signal and `mixed` and `neutral` as the rest. |
| `lang.*` | English, German, Spanish, French, Italian, Portuguese, other | The language of the review (asked twice). |
| `request.*` | feature, bug, question | What the reviewer asks for. `request.question` decides. |
| `intent.*` | refund, repurchase, switch, recommend, warn | What the reviewer says they will do. `intent.refund` decides. |
| `mention.*` | competitor, previous, duration, gift, update | Facts worth extracting next: a competitor named, a comparison with an earlier version or purchase, how long they have used it, bought for someone else, an update to an earlier review. |
| `tone.*` | angry, disappointed, enthusiastic, sarcastic | The reviewer's state. Sarcasm is the weakest read in the pack. |
| `quality.*` | promotional, incentivized, offtopic | The review's own trustworthiness: reads like an advertisement, says it was free or discounted in exchange for the review, is not about the product it is attached to. |
| `risk.*` | safety, legal, public, abuse, pii | A safety report (an injury, a reaction, a fire), a legal threat (legal action, a chargeback, a complaint to an authority), a threat to go public, abuse of a person, personal data in the text. `risk.safety` and `risk.legal` decide. |

Every yes/no question is asked in two formats and the category is the mean log-odds; that is what makes the scores
of a 12B model on a three-sentence text stable enough to threshold.

## Outputs you build from it

- **Aspect scorecard**: for each aspect, `mentioned` and, where it is above your cut (0.5 is a fair default; the
  right cut differs by aspect and by how long your reviews are), the reading with the highest probability. Across a
  batch of reviews, the share of mentions that are `negative` per aspect is the scorecard.
- **Top complaint**: the aspect with the highest `mentioned × negative` in a review, or the most frequent one in a
  batch.
- **Rating**: `Σ k · P(stars.k)`, when the source has no stars (survey answers, social posts), or to spot reviews whose
  text and stars disagree.
- **Needs follow-up**: the pack's decision (`per_category`): a safety report, a legal threat, a question or a refund
  request with probability over 0.5.
- **Route**: `subject.*` sends a review about the delivery or the seller to a different queue from one about the
  product; `request.bug` and `request.feature` go to the product team.
- **Trust filter**: `quality.incentivized`, `quality.promotional`, `quality.offtopic` and `risk.pii` before publishing.

## Calibration

Normal traffic is a mix of real product reviews from a clothing retailer (CC0) and from a general marketplace
(Apache 2.0), about 3,000 reviews. There are no fitted constants on the probability scale: the pack ships the model's
probabilities, the percentiles of each category on that traffic, and a 0.5 threshold on the four deciding categories.
The aspect readings were checked against reviews with aspect annotations (laptops and restaurants), the stars and the
overall opinion against the ratings the reviewers gave, and the recommendation against a "would recommend" flag; no
threshold or constant comes from those checks. The deciding categories have no labelled set: the 0.5 threshold is
the model's own "more likely yes than no"; set your own on a labelled sample (`decision.thresholds`) when the
false-positive cost matters.

Known limits: the yes/no probabilities of a 12B model on a short text are shallow (an absent property sits at
0.1–0.4, a present one at 0.5–0.9), so the 0.5 threshold flags a few percent of ordinary reviews and more of
one-sentence inputs, where everything drifts towards 0.5; on single sentences, use the percentile or a higher
threshold. The `mentioned` scores miss aspects a long review touches in passing, and `accuracy` is read as
satisfaction on a glowing review. `tone.sarcastic`, `quality.promotional`, `quality.offtopic` and `mention.update`
are weak signals, for ranking by percentile rather than for thresholds. The calibration traffic is English; the
language categories were checked on a handful of non-English samples only.

The datasets were used to calibrate and check, not bundled; the shipped samples are written by us.

## Versions

- 0.1.0: first calibrated version.
