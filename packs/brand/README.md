# spinf/brand: brand perception and web content

Give it a web page and a brand, and one read answers the two sets of questions a marketing or SEO team asks of a
page. **About the brand:** is it mentioned, is it the subject or an aside, is the page the brand's own, is the brand
recommended, criticised, the top pick or the alternative, compared with others, called expensive or good value,
reliable or faulty, trusted, innovative or outdated, is there bad news about it, a deal, a how-to, a customer
complaint, a job. **About the page:** the search intents it serves (informational, navigational, transactional,
commercial), what kind of page it is (news, blog, reference, research, how-to, recipe, review, comparison, forum,
help, sales, legal, company, job, creative, transcript), its topic, audience, the level it assumes, how hard it
reads, its language, and the content signals search engines are said to reward or punish (first-hand experience,
sources, a named author, time-sensitive, thin, boilerplate, keyword stuffing, clickbait, promotional, sponsored,
generic, a call to action, local, prices, steps, Q&A, opinion). The decision is a **brand alert**: the page
criticises the brand, reports bad news about it or carries a complaint.

The same questions run on a **search query** (context `query`): the pack then describes the page the searcher is
looking for, which gives keyword intent, the kind of result expected and whether the keyword is branded.

Model: `spinf/gemma-4-12b-fp8`. Media: text. Contexts: `page` (default: any web page), `article` (a news article or
a blog post), `query` (a search query).

## The brand parameter

This pack takes a **parameter**: `brand`, the brand, company, product or site the brand questions are about. It is
passed with each input and substituted into the context, before and after the page text; the questions refer to it
as "the brand of interest". For a plain content analysis with no brand in mind, pass the site's own name. The parameter is
required.

```python
client.score_pack("spinf/brand", text=page_text, context="page", params={"brand": "Kestrel"})
```

## What it scores

Every category is on the probability scale except two: the score is the model's probability of the answer, and the
percentile says how unusual that score is among everyday web pages. `brand.mentioned` and `brand.official` are
fitted on labelled pages instead (pages that do or do not name the brand, pages on the brand's own domain), so their
scores are calibrated probabilities of the fact.

| Group | Categories | How to read them |
|---|---|---|
| `brand.mentioned` | | Whether the page names or refers to the brand (two wordings; fitted). The rest of the brand block is meaningful only where this is high. |
| `brand.view.*` | positive, negative, neutral, mixed | The page's **view of the brand**, conditional on the mention: a single choice, the four sum to about one. |
| `brand.*` | subject, passing, recommended, criticised, top_pick, alternative, compared, official, expensive, value, quality, poor, trusted, innovative, outdated, bad_news, deal, howto, complaint, hiring | Yes/no perception items, all gated on the mention (`requires`): they count as flagged only when `brand.mentioned` is. `criticised`, `bad_news` and `complaint` decide; `official` is fitted. |
| `intent.*` | informational, navigational, transactional, commercial | The **search intents** the page serves. Yes/no items, not a single choice: a comparison page is both informational and commercial. |
| `type.*` | news, blog, reference, research, howto, recipe, review, comparison, forum, help, sales, legal, company, job, creative, transcript | The **kinds of page** it is, as yes/no items; the argmax is the page type, the others say what else it is. |
| `topic.*` | 22 topics: arts and entertainment, automotive, business and finance, careers, education, family and parenting, fashion and beauty, food and drink, health, hobbies, home and garden, law and politics, pets, real estate, religion, science, shopping, sports, technology, travel, video games, adult | The main topic: a single choice. |
| `audience.*` | the general public, professionals, developers, students, children | Who the page is written for. |
| `level.*` | beginner, intermediate, expert | The knowledge the page assumes. |
| `reading.*` | elementary, intermediate, advanced | The reading level of the text. The middle option wins the argmax on most pages; the `elementary` and `advanced` probabilities (or their percentiles) order texts by difficulty. |
| `lang.*` | English, German, Spanish, French, Italian, Portuguese, other | The language (asked twice). |
| `quality.*` | experience, sources, author, timely, thin, boilerplate, stuffing, clickbait, promotional, sponsored, generic, cta, local, prices, steps, qa, opinion | **Content signals**: first-hand experience, cites sources, a named author, time-sensitive rather than evergreen, thin, site furniture rather than content, keyword stuffing, clickbait, promotional, discloses sponsorship or affiliate links, generic or machine-sounding, a call to action, about a place, states prices, step-by-step, questions and answers, the author's opinion. |

## Outputs you build from it

- **Brand verdict** per page: `mentioned`, then the `view` argmax and the items above 0.5; across a batch of pages
  (search results for a keyword, a crawl of a site, a list of citations an AI answer gave), the share of pages that
  recommend the brand, name it as the top pick, or criticise it.
- **AI-visibility and share of voice**: run the same pages once per brand of interest (yours, each competitor) and
  compare `recommended`, `top_pick` and `alternative`.
- **Brand alert**: the pack's decision (`per_category`): `criticised`, `bad_news` or `complaint` with probability
  0.6 or more, on a page that mentions the brand. Lower the thresholds for recall, raise them for precision.
- **Keyword intent**: score keywords with the `query` context; `intent.*` and `type.*` say what the searcher wants
  and what kind of page would satisfy them; `brand.mentioned` says whether the keyword is branded.
- **Content audit**: `type.*`, `topic.*`, `audience.*`, `level.*` and `reading.*` classify a site's pages;
  `quality.thin`, `quality.boilerplate`, `quality.stuffing`, `quality.generic` and `quality.clickbait` rank the ones
  to fix, `quality.experience`, `quality.sources` and `quality.author` the ones that show expertise.

## Calibration

Normal traffic is 3,000 cleaned English web pages from a public web crawl (ODC-BY), in which the brand of interest
is a well-known brand the page names (40%) or a random one (60%). The pack ships the model's probabilities, the
percentiles of each category on that traffic, the fitted scaling of `brand.mentioned` and `brand.official`, and a
0.6 threshold on the three deciding categories. The page types were checked against a corpus of web pages with
register annotations, the view of the brand against headlines with entity-level sentiment, the reading level
against texts rewritten at three levels, clickbait against crowd-judged posts, query intents against manually
labelled queries and topics against a news set; no threshold or constant comes from those checks.

Known limits: the brand items are yes/no probabilities of a 12B model and sit at 0.3–0.5 for absent properties on
a page that mentions the brand, 0.5–0.9 for present ones; a page that lists a product's problems in a how-to reads
as a "complaint", and a comparison that criticises the brand's weaker side flags `criticised`, correctly but
noisily. The mention is read well on pages up to a few thousand characters and less well on very long ones: cut a
long page to its main content. Short inputs (a search query, a headline) push every yes/no towards 0.5, so the
alert on queries is not reliable; use the brand block on pages. The view of the brand is read from the page's tone:
a neutral report of bad news comes out neutral with `bad_news` high, and neutral headlines lean positive. Among
the page types, news, how-to, forum, research, sales, legal, job and transcript are read well; blog against
reference against help is blurry on messy pages. The topic is a single choice over 22 options and leans to
"business and finance" on short snippets. `quality.clickbait` and `brand.official` are weak signals, for ranking
by percentile rather than for thresholds. The calibration traffic is English; the language categories were checked
on a handful of non-English samples only.

The datasets were used to calibrate and check, not bundled; the shipped samples are written by us, with fictional
brands.

## Versions

- 0.1.0: first calibrated version.
