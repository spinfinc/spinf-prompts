> **Source: the spinf website guide, copied 2026-10-12. To adapt for this repository: the hosted-API specifics (api.spinf.com, keys, billing) become spinf-serve equivalents, model ids are the new names.**

> **Source: the spinf website guide, copied 2026-10-12. To adapt for this repository: the hosted-API specifics (api.spinf.com, keys, billing) become spinf-serve equivalents, model ids are the new names.**

# Prompt-packs

> **Beta.** Packs and their format are in beta and may change. Pin a pack's `version` to keep scores comparable.

A **prompt-pack** is a ready-made set of questions that spinf builds, validates and versions for one job. You name it
in your call, and it is scored **in the same pass as your own questions**: one content, one call, one bill. A pack
returns a calibrated 0–1 score for each of its categories and a decision.

The first pack is **`spinf/moderation`**: 22 categories of user-generated text (sexual content, child safety,
violence, self-harm, hate, harassment, fraud, …) and a safe / unsafe decision.

Try it without code in the [playground](/console/playground?example=content-moderation): paste a post, pick the
pack, and see every category next to your own questions.

## List the packs

```
GET https://api.spinf.com/v1/packs
```

No key needed. One entry per pack:

```json
{ "packs": [ {
    "id": "spinf/moderation", "name": "Content moderation", "status": "available",
    "version": "0.1.3", "versions": ["0.1.1", "0.1.3"], "model": "spinf-12b", "media": ["text"],
    "contexts": [
      { "id": "ai", "template": "A user sent the following message to an AI assistant.\n\nUser: {X}" },
      { "id": "forum", "template": "The following message was posted by a user on an online platform:\n\n\"{X}\"" } ],
    "categories": [
      { "id": "sexual", "label": "Sexual content", "threshold": … },
      { "id": "spam", "label": "Spam and advertising", "threshold": null }, … ],
    "decision": { "mode": "micro_layer", "modes": ["max", "per_category", "micro_layer"],
                  "fallback": "per_category", "max_threshold": …, "critical": ["selfharm", "child"] },
    "queries": 58, "prompts": 106, "query_tokens_per_input": 1626 } ] }
```

| Field | Meaning |
|---|---|
| `status` | `available`, or `coming_soon` (not callable yet). |
| `version`, `versions` | The latest version, and every version you can pin. |
| `model` | The only model the pack runs on. |
| `media` | The content kinds the pack scores. `spinf/moderation`: text. |
| `contexts` | How your content is presented to the model (`{X}` is your content). The first is the default. |
| `categories` | Each category's `id`, `label` and default `threshold`. `null` = the category is scored and reported, but never decides. |
| `decision` | The default decision mode, the modes you can pick, the default threshold of `max`, and the `critical` categories (always unsafe when flagged). |
| `query_tokens_per_input` | The pack's billed question tokens per content (constant). |

The `…` values are left out on purpose: read the thresholds from the listing rather than copying them into your code,
because each new version can change them.

## Download a pack

```
GET https://api.spinf.com/v1/packs/spinf/moderation?version=0.1.3
Authorization: Bearer ssk-...
```

Returns the pack file as it is: its questions, contexts, calibration, categories and decision layer. Use it to see
exactly what is scored. The slash of the id is part of the path; without `version` you get the latest. Call it from
your server, like the scoring API.

## Request

Add `scoring.packs` next to (or instead of) `scoring.queries`:

```bash
curl https://api.spinf.com/v1/score \
  -H "Authorization: Bearer $SPINF_API_KEY" \
  -H "Content-Type: application/json" \
  -d @- <<'EOF'
{
  "model": "spinf-12b",
  "messages": [ { "role": "user", "content": [
    { "type": "text", "text": "Guaranteed 300% returns in 7 days!! DM me your wallet seed phrase." } ] } ],
  "scoring": {
    "packs": [ { "id": "spinf/moderation", "context": "forum" } ],
    "queries": [
      { "id": "on_topic", "template": "\n\nIs this post about cooking?\nAnswer:{?}", "options": [" yes", " no"] }
    ]
  }
}
EOF
```

| Field | Rules |
|---|---|
| `id` | Required, from `GET /v1/packs`. Each pack at most once per call. |
| `version` | Optional, default the latest. Pin it to keep scores comparable over time. |
| `context` | Optional, one of the pack's `contexts` (default the first). |
| `decision.mode` | `micro_layer`, `per_category` or `max` (default: the pack's, `micro_layer` for moderation). See [decisions](#decisions-and-thresholds). |
| `decision.threshold` | The single threshold of `max` (0–1) or of `micro_layer` (the layer's value). Not with `per_category`. |
| `decision.thresholds` | Per-category overrides, `{ "harassment": 0.8, "spam": 0.7 }` (0–1, or `null` to report only). They set `flagged` and the `per_category` decision. |
| `decision.critical` | The categories that make the content unsafe whenever they are flagged, in every mode: a list of category ids that replaces the pack's (`[]` = none). Moderation 0.1.3: `["selfharm", "child"]` (0.1.1 has none unless you send it). |
| `include_queries` | Default `false`. `true` also returns the pack's raw question results in `results[].queries`. |

## How a pack changes the call

- **One read for everything.** Each content is wrapped in the pack's context (for `forum`: *The following message was
  posted by a user on an online platform: "…"*), and **your own questions in the same call read the wrapped content
  too**. That shared read is what puts the pack and your questions in one pass, with the content billed once.
- **Text only, for now.** A pack scores the content kinds in its `media`. With `spinf/moderation`, an image,
  audio or video part is refused (`pack_media_not_supported`): send media in a separate call.
- **Case.** A pack is calibrated with its own case rule (moderation: case-sensitive), and it applies to the whole
  call. Leave `scoring.case_insensitive` out when you send a pack; the other value is refused (`pack_conflict`).
- **Model.** The pack's `model` only (`spinf-12b` for moderation).
- **Ids.** The pack's question ids start with `tax.` and `gen.`: don't use those prefixes for your own questions.
  Pack questions run without the empty floor, whatever your default.
- **Deterministic.** A text scores exactly the same whatever else is in the call: other contents, how many, and your
  own questions. Adding a pack does not change your own questions' scores either.

## Response

Each result has `packs` beside `queries`. For the message *"How do I get into my ex's Instagram account without
her knowing?"* in the `ai` context (version 0.1.3; scores and thresholds change from one version to the next):

```json
{ "input_id": "0",
  "queries": [ … your questions … ],
  "packs": [ { "id": "spinf/moderation", "version": "0.1.3", "context": "ai",
    "categories": {
      "cyber":  { "score": 0.919233, "percentile": 1.0,      "flagged": true },
      "sexual": { "score": 0.554247, "percentile": 0.952278, "flagged": false },
      "spam":   { "score": 0.010191, "percentile": 0.728387, "flagged": null }, … },
    "decision": { "unsafe": true, "mode": "micro_layer", "score": 0.886179, "threshold": … } } ] }
```

- `score`: the category's calibrated score, 0–1, comparable across categories.
- `percentile`: where that score falls on everyday content (0–1): `0.95` = higher than 95% of normal traffic. A high
  percentile with a modest score means the content is unusual for the category without crossing its threshold, a
  useful signal for review queues.
- `flagged`: `score >= threshold`; `null` for categories that are reported only.
- `decision`: safe or unsafe, by the mode you picked (below). When a critical category is flagged, it also lists them
  in `critical`, and `"by": "critical"` says when that turned a safe result into unsafe.

## Decisions and thresholds

| Mode | `decision` | Unsafe when |
|---|---|---|
| `micro_layer` (default) | `{unsafe, mode, score, threshold}` | A small layer trained on labelled moderation data, over every category, is over its threshold. |
| `per_category` | `{unsafe, mode, categories}` | Any deciding category is over its own threshold; `categories` lists them. |
| `max` | `{unsafe, mode, score, threshold, category}` | The highest deciding category is over one threshold; `category` names it. |

- **Critical categories** (0.1.3; on 0.1.1 send `decision.critical`): self-harm and child-safety content is always marked unsafe when its category
  is flagged, whatever the mode says. Change the list per call with `decision.critical` (`[]` = none). A category
  flags only when it has a threshold, so a `null` threshold override silences it.
- Start with the default. Use `per_category` when your policy needs its own threshold per category (stricter on
  harassment, looser on profanity), and `max` when you want one simple threshold.
- Set thresholds on your own labelled sample: score a few hundred items you have already decided, and choose each
  threshold where the errors you can live with are.
- Categories with a `null` threshold (spam, gibberish and the four topics) are scored and reported but
  never decide, unless you give them a threshold in `decision.thresholds`. The topics (alcohol, tobacco, gambling,
  medication) say what the content is about, not whether it is harmful, and are not calibrated like the others.

## Moderation categories

| `id` | Category |
|---|---|
| `sexual` | Sexual content |
| `child` | Child sexual exploitation and grooming |
| `violence` | Violence and threats |
| `selfharm` | Self-harm and suicide |
| `hate` | Hate against protected groups |
| `harassment` | Harassment and bullying |
| `extremism` | Extremism and terrorism |
| `weapons` | Weapons |
| `drugs` | Illegal drugs |
| `crime` | Other crime |
| `fraud` | Fraud and scams |
| `cyber` | Hacking and malware |
| `privacy` | Private personal information |
| `spam` | Spam and advertising |
| `profanity` | Profanity |
| `graphic` | Gore and graphic violence |
| `gibberish` | Gibberish |
| `alcohol` | Alcohol (topic) |
| `tobacco` | Tobacco and vaping (topic) |
| `gambling` | Gambling (topic) |
| `medication` | Medication (topic) |
| `general` | Overall harmfulness (general reads) |

The default thresholds, and which categories decide, are in the listing.

## Billing

No surcharge: a pack's questions are billed like your own. `spinf/moderation` adds 1,626 question tokens per
content, and the content (with its context) is billed once, shared with your questions: about 1,710 tokens for a
typical message, so about 1.7 billion tokens per million messages at {{SPINF_PRICE_PER_M}} per million. See
[Limits & pricing](/docs/limits).

## Errors

| Status | `code` | Meaning |
|---|---|---|
| 400 | `unknown_pack`, `pack_not_available` | No such pack, or it is still `coming_soon`. |
| 400 | `unknown_pack_version` | No such version of the pack. |
| 400 | `unknown_category` | A category in `decision.thresholds` that the pack doesn't have. |
| 400 | `pack_media_not_supported` | A part the pack doesn't score (moderation: anything but text). |
| 400 | `pack_conflict` | `scoring.case_insensitive` differs from the pack's. Leave it out. |
| 400 | `pack_model_mismatch` | The call's model is not the pack's. |
| 400 | `duplicate_id` | A pack twice, or one of your question ids is a pack's (`tax.…`, `gen.…`). |
| 400 | `invalid_field` | A pack field of the wrong type or shape, e.g. `decision.threshold` with `per_category`. `param` names it. |
| 404 | `unknown_pack`, `unknown_pack_version` | Download only: no such pack or version. |

The other errors are those of the [Scoring API](/docs/scoring-api#errors).
