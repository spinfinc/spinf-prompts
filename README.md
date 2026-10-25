# spinf prompt packs

The official prompt packs for [spinf-serve](https://github.com/spinfinc/spinf-serve), the open-source inference engine for
decision models, and for the hosted spinf API. A **prompt pack** is a ready-made, calibrated set of questions for one job:
named in a request, it is scored in the same pass as your own questions and returns a 0-1 score per category and a decision.

This repository holds **files only**: the pack files, their golden fixtures, sample inputs and the format specification. The
code that loads, validates and evaluates packs is spinf-serve's `packs` module (also used by spinf-client and the hosted API),
so every evaluator gives the same answer.

| pack | job | model | versions |
|---|---|---|---|
| [`spinf/moderation`](packs/moderation/) | 22 categories of user-generated text, safe / unsafe decision | `spinf/gemma-4-12b-fp8` | 0.1.3 (default), 0.1.1 |
| [`spinf/moderation-qwen`](packs/moderation-qwen/) | the same, for the Qwen model | `spinf/qwen3.8-27b-int4` | 0.1.0 |
| [`spinf/moderation-qwen-lite`](packs/moderation-qwen-lite/) | one overall unsafe score from three questions | `spinf/qwen3.8-27b-int4` | 0.1.0 |
| [`spinf/moderation-decider`](packs/moderation-decider/) | the same, asked in the decision model's own question format | `spinf/pplx-decider-v1-27b-int4` | 0.1.0 |
| [`spinf/triage`](packs/triage/) | inbound emails, chats and tickets: route, specific intent, priority, tone, risk, personal data, phishing; needs-attention decision | `spinf/gemma-4-12b-fp8` | 0.1.1 |
| [`spinf/entertainment`](packs/entertainment/) | synopses of films, series, books, games, podcasts: genres and sub-genres, mood, themes, advisories, audience, setting, style; restricted-to-adults decision | `spinf/gemma-4-12b-fp8` | 0.1.1 |
| [`spinf/products`](packs/products/) | product listings, pages, feeds, titles: Shopify taxonomy category in two levels, attributes, language, kind of offer, information given, quality, claims, restricted products; needs-review decision | `spinf/gemma-4-12b-fp8` | 0.1.0 |
| [`spinf/reviews`](packs/reviews/) | reviews and customer feedback: overall opinion and implied stars, subject, 16 aspects mentioned and praised or criticised, language, requests, intent, mentions, tone, review quality, risks; needs-follow-up decision | `spinf/gemma-4-12b-fp8` | 0.1.0 |
| [`spinf/brand`](packs/brand/) | brand perception and web content: a page (or a search query) plus a brand parameter; mentioned, the page's view of the brand and 20 perception items (recommended, criticised, top pick, bad news, complaint...), search intents, page types, topic, audience, level, reading level, language, content signals; brand-alert decision | `spinf/gemma-4-12b-fp8` | 0.1.0 |

Planned: news analysis.

## Use

With spinf-serve, this repository is loaded by default at the tag the engine pins; any tag or a local checkout can be given:

```sh
spinf-serve serve spinf/gemma-4-12b-fp8 --packs github:spinfinc/spinf-prompts@<tag>
spinf-serve serve spinf/gemma-4-12b-fp8 --packs ./spinf-prompts
```

Then `GET /v1/packs` lists what is enabled for the served model, and a request names a pack in `scoring.packs`. The hosted
API at `api.spinf.com` serves the same files under the same ids. Guides: `docs/prompt-packs.md` (using packs),
`docs/writing-questions.md` (writing questions), `docs/pack-format.md` (the file format).

## Layout

- `prompt-packs.json`: the repo manifest (format 1): namespace, licence, and per pack its name, description, tags, media, readme,
  samples and versions (file, model, status, fixture). A server lists a repo from this file without opening every pack.
- `packs/<name>/<version>.json`: the pack files. `packs/<name>/fixtures/<version>.json`: inputs with raw results and the expected
  categories and decision in every mode (an evaluator must reproduce them). `packs/<name>/samples/`: sample inputs written by spinf.
- `docs/`: the guides and the format specification.

## Publish your own packs

Any GitHub repository with a `prompt-packs.json` at its root can be loaded by spinf-serve (`--packs github:you/your-packs@tag`):
use your own namespace (by convention your GitHub owner; `spinf/` is reserved for this repository), pin a tag per release, and run
`spinf-serve packs validate .` in CI. This repository is the reference example.

## Versioning and licence

A pack version never changes once published: thresholds and calibration are refit as a new version. Pack files are licensed under
Apache 2.0 (prompts and calibration are spinf's; the data they were checked on is not bundled: `ACKNOWLEDGEMENTS.md`).
