# spinf/products: product listings

One read of a product listing (a title with its description and bullet points, a product page, a feed row, or a
title alone) gives everything a catalog or a marketplace wants to know before the listing goes live: where it
belongs in the Shopify Standard Product Taxonomy (the vertical and the subcategory), its common attributes (colour,
pattern, material, gender, age group, condition, power source), its language, what kind of offer it is, which
information it gives and which it lacks, its quality problems, the claims it makes, and whether it offers a
prohibited or restricted product. The pack returns a score and a normal-traffic percentile for every category, flags
for the deciding ones, and a decision: does this listing need review before publication?

Model: `spinf/gemma-4-12b-fp8`. Media: text. Contexts: `listing` (default), `page` (the text of a product page),
`feed` (one item of a product feed), `title` (a title alone): pick the one that matches what you send.

## What it scores

| Group | Categories | How to read them |
|---|---|---|
| `category.*` | 22 verticals of the Shopify Standard Product Taxonomy (Apparel & Accessories, Electronics, Home & Garden, Food, Beverages & Tobacco…) | The **vertical**: the highest score. The score is the model's probability of the vertical, so it says how sure the model is. Reported only. |
| `sub.<vertical>.*` | the 227 level-2 categories, under their vertical | The **subcategory**: the highest `sub.<vertical>.` score within the chosen vertical. Probability within the vertical. Each label carries the taxonomy path. |
| `attr.color.*`, `attr.pattern.*`, `attr.material.*`, `attr.gender.*`, `attr.age.*`, `attr.condition.*`, `attr.power.*` | the taxonomy's values (19 colours, 14 patterns, 25 materials, gender, age group, condition, power source), each with "not applicable" or "none" | The **attribute value**: the highest score in the group; "not applicable" when the attribute does not apply. Probabilities. |
| `lang.*` | English, German, Spanish, French, Italian, Portuguese, other | The language of the listing. Probability. |
| `kind.*` | bundle, digital, service, handmade, customizable, subscription | What is offered. The score is the model's probability of "yes". Reported only. |
| `info.*` | brand, dimensions, material, contents, warranty, price, compatibility | Which information the listing gives: the probability of "yes". Coarse (most listings score 0.4 to 0.9 on every item): compare listings, or use the percentile. Reported only. |
| `quality.*` | thin, keyword-stuffed, off-platform contact, spelling errors, placeholder text | Listing quality. `quality.contact` (a phone number, an email, a website, "buy elsewhere") decides. |
| `claims.*` | health claim, certification claim, exaggerated claim, pressure tactics | What the listing claims. `claims.health` (cures, treats or prevents a condition) decides. |
| `restricted.*` | weapons, drugs, tobacco, alcohol, adult, pharma, counterfeit, hazardous, animals, gambling, surveillance | Prohibited or restricted products. All decide. |

## Outputs you build from it

- **Category path**: `argmax category.*`, then `argmax sub.<vertical>.*`; the labels give the taxonomy names.
- **Attributes**: the argmax of each `attr.<name>.` group, skipped when it is "not applicable".
- **Needs review**: the pack's decision (`per_category`): any deciding category over its threshold. With thirteen
  deciding categories each flagging 0.5% of normal listings, about 5% of everyday listings are flagged: tune the
  thresholds to your queue.
- **Listing completeness**: the `info.*` scores, read as a checklist.

## Calibration

Normal traffic is about 4,500 real listings of two public Amazon product sets (Amazon Berkeley Objects, CC BY 4.0,
and the Shopping Queries Dataset, Apache 2.0), titles with bullet points and descriptions. The single-choice groups
(category, attributes, language) and the `kind.*` and `info.*` questions are on the probability scale and need no
fit; the product types, colours and materials of the first set were used to check them: the vertical is the labelled
one for about four listings in five, and the misses are mostly taxonomy conventions (tools and faucets read as Home &
Garden rather than Hardware). The yes/no categories are standardised on the normal traffic, so
a score's percentile says how unusual the listing is for that category among everyday listings; the deciding ones
have no labelled set yet: their thresholds flag 0.5% of normal traffic, and their scores are on the generic scale.
Set your own thresholds on a labelled sample (`decision.thresholds`) when the false-positive cost matters.

The datasets were used to calibrate, not bundled; the shipped samples are written by us. The category names are
those of the Shopify Standard Product Taxonomy (MIT), eleven of them shortened in the questions to fit the option
length.

## Versions

- 0.1.0: first calibrated version.
