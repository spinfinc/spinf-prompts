# The prompt pack format

A prompt pack is one JSON file: the scoring request a server takes, the way each answer becomes a *read*, the way reads
become calibrated *category scores*, and the way category scores become one *decision*. The file is self-contained and
model-specific: its constants were measured on the model it names.

The reference evaluator is the `spinf_serve.packs` module of [spinf-serve](https://github.com/spinfinc/spinf-serve)
(pure Python). The server runs it for `scoring.packs` requests, spinf-client carries a verbatim copy for local runs, and
every pack version in this repository ships a fixture that pins its output (see [Fixtures](#fixtures)). Using packs from
a client is described in [prompt-packs.md](prompt-packs.md); writing questions in [writing-questions.md](writing-questions.md).

## The file

| field | type | meaning |
|---|---|---|
| `pack` | string | the id, `<namespace>/<name>`: lower-case letters, digits, `.`, `_`, `-` (`spinf/moderation`) |
| `version` | string | the version (`0.1.3`); digits, letters, `.`, `+`, `-` |
| `model` | string | the model id the pack is calibrated on (`spinf/gemma-4-12b-fp8`); a server offers the pack only when it serves that model |
| `media` | list | the content kinds the pack accepts: `text` plus any of `image`, `audio`, `video`; absent = `["text"]` |
| `answers` | `"p"` \| `"logprob"` | where the option probabilities come from (below); absent = `"p"` |
| `engine` | string | informational: the engine fingerprint the calibration was measured with |
| `parameters` | object | optional: values the caller passes per input, substituted into the context |
| `contexts` | object | `{name: template}`: how the content is presented to the model; at least one |
| `scoring` | object | the scoring object of `POST /v1/score`, as the API takes it |
| `reads` | list | one read per (query, combination) used: the calibration of that answer |
| `categories` | object | `{id: {...}}`: the scores the pack reports |
| `decision` | object | the modes, their thresholds and the micro-layer |

Other fields are kept and ignored. A pack file is at most 16 MB; the ones here are a few hundred KB.

### Contexts and parameters

A context is a template with exactly one `{X}`, the content. The text before `{X}` and the text after it become content
parts around the caller's content, so an image or a video is wrapped the same way as a text. The caller picks a context
by name (`"context"` in the request); the first one is the default.

```json
"contexts": {
  "ai":    "A user sent the following message to an AI assistant.\n\nUser: {X}",
  "forum": "The following message was posted by a user on an online platform:\n\n\"{X}\""
}
```

`parameters` declares values the caller passes with each input, written into the context as `{name}`:

```json
"parameters": { "brand": { "label": "Brand of interest", "description": "the brand the questions are about", "required": true } },
"contexts":   { "page": "Brand of interest: {brand}\n\nThe following is the text of a web page.\n\n{X}" }
```

Each parameter has a `label`, an optional `description`, `required` (default `true`) and an optional `default`. A
missing required parameter is an error; a missing optional one takes its `default`, else `""`. A parameter name is an
identifier; `X` is reserved. Parameters go into the context only, never into `scoring.queries`, which a server shares
across the inputs of a call: a question refers to the value through a handle the context defines ("the brand of
interest"). Not part of the lock.

### Scoring

`scoring` is the API's scoring object: `queries` (templates with `{?}`, their options, and `combinations` or `values`
for a `{F}` placeholder), `empty_floor` and `case_insensitive`. Every query must be a scored one: a template with `{?}`
and at least two options; `group` is not set in the file. When a server or a client runs the pack, the queries go out in
their own query group `<pack>@<version>` with the pack's `empty_floor` on each, so a pack's answers do not depend on the
caller's own queries or on the request's floor setting.

### Answers

For each query and combination the API returns an option probability `p` (rounded to 6 decimals) and a `logprob`. The
pack's `answers` says which the reads use: `"p"`, or `"logprob"` = exp(logprob − logsumexp over the query's options),
full precision (a rare option's probability is not stuck on the 1e-6 grid). The constants fitted for one source do not
fit the other, so changing it is a new version.

### Reads

A read is one number taken from one answer. `p` below is the answer's option probabilities, each clipped to at least
1e-9 and renormalised over the query's options; `mu` is a per-option baseline measured on normal traffic.

| `kind` | fields | value |
|---|---|---|
| `pair` | `sign` (1 or −1; default 1); the query has two options | `sign · logit(p[0] / (p[0] + p[1]))`, the ratio clipped to [1e-6, 1 − 1e-6] |
| `option` | `option` (an index), scalar `mu` | `ln p[option] − mu` |
| `any` | `mu` per option; option 0 is the "none" option | with `a = ln p − mu`: `logsumexp(a[1:]) − a[0]` |
| `set` | `mu` per option, `unsafe` (indices, not all) | `logsumexp(a[unsafe]) − logsumexp(a[rest])` |

Every read has an `id`, the `query` id, `F` when the query has combinations (the combination's value), the `category`
it feeds, an optional `view`, and `mean` and `sd` (> 0) measured on normal traffic. Its standardised value is
`z = (value − mean) / sd`.

```json
{ "id": "tax.ask:sexual", "query": "tax.st:ask", "F": "sexual content", "kind": "pair", "sign": 1,
  "category": "sexual", "mean": -0.22112, "sd": 0.36745 }
```

### Categories

```json
"sexual": {
  "label": "Sexual content",
  "platt": [0.83218, -0.80776],
  "threshold": 0.7065,
  "normal_quantiles": { "levels": [0, 0.1, 0.2, "…", 0.99, 0.999, 1], "values": [0.0012, 0.0038, "…", 0.71, 0.93, 1.0] }
}
```

| field | meaning |
|---|---|
| `label` | the display name |
| `platt` | `[a, b]`: `score = sigmoid(a · macro + b)`, where `macro` is the mean `z` of the category's reads |
| `aggregate` | `"views"`: the macro is the mean of the per-`view` means (reads tagged with a `view`), so a view with many wordings does not outweigh one with few |
| `threshold` | a number in [0, 1], or `null` = reported only: the category gets a score and a percentile but never decides |
| `normal_quantiles` | `{levels, values}`, two increasing lists: the score at the given quantiles of normal traffic; `percentile` = the score interpolated in it (clamped at both ends) |
| `requires` | a list of other categories: this one counts as flagged only while every category it lists is flagged (a brand complaint requires the brand to be mentioned) |
| `decides` | `false`: the category has a threshold only to serve as such a gate; it never enters the decision |
| `scale` | informational, `"probability"`: the reads have `mu` 0, `mean` 0, `sd` 1 and `platt` [1, 0], so the score is the model's probability of the option (an argmax over sibling categories is the model's choice) |
| `uncalibrated`, `fit_platt` | informational: the Platt map is the generic one, not fitted on labels |

A category's `flagged` is `score >= threshold` (`null` when the threshold is `null`), then its `requires` is applied.
A category must be fed by at least one read.

A pack may **report only**: every threshold `null`, no deciding category. Its categories still get a score and a percentile, and
its decision (mode `per_category`) is always safe. That is the shape of an indicator pack, a set of reads whose consumer does its
own aggregation (a news index, a signal), with the pack carrying the questions and their baselines.

### Decision

```json
"decision": {
  "mode": "micro_layer",
  "fallback": "per_category",
  "critical": ["selfharm", "child"],
  "max": { "threshold": 0.72 },
  "per_category": {},
  "micro_layer": { "inputs": ["c:sexual", "c:child", "…", "r:gen.m:topic"], "hidden": 0, "dtype": "float32",
                   "weights": "<base64>", "lock": "sha256:…" }
}
```

The *deciding* categories are those with a threshold (the pack's or the call's) and not `decides: false`.

| `mode` | the decision |
|---|---|
| `max` | `score` = the highest score among the deciding categories (`category` names it); `unsafe` = `score >= max.threshold`. No deciding category: safe, score 0. |
| `per_category` | `categories` = the deciding categories that are flagged; `unsafe` = any. |
| `micro_layer` | `score` = the layer's output over `inputs`: `c:<category>` is that category's macro, `r:<read id>` that read's `z`; `unsafe` = `score >= threshold` (the layer's own, or the call's). Used only while the lock matches; otherwise the `fallback` mode (`max` or `per_category`, default `per_category`) decides and the result carries a `warning`. |

Then, in every mode, a flagged category listed in `critical` forces `unsafe`; the result carries `critical: [...]`
and, when that changed the outcome, `by: "critical"`. `critical` and the thresholds are editable: not part of the lock.

**The micro-layer.** `weights` is base64 of little-endian float32. `hidden: 0`: `[w (one per input), b, threshold]`,
output `w · x + b`. `hidden: h`: `[W1 (inputs × h, row-major), b1 (h), w2 (h), b2, threshold]`, output
`w2 · tanh(x W1 + b1) + b2`. `dtype` is `float32`. Other fields (`activation`, `l2`, `trained_on`, `n_train`, `n_valid`)
are informational.

**The lock.** `lock` is `"sha256:" + sha256` of the canonical JSON (keys sorted, no whitespace) of
`{"queries": scoring.queries, "reads": reads, "platt": {category: platt}, "inputs": micro_layer.inputs}`. An evaluator
recomputes it on load; after any edit to a question, an option, a read's calibration or a Platt map the layer no longer
applies and the fallback decides.

### The result

Scoring a content with a pack gives the `packs[]` entry of the API's result (numbers rounded to 6 decimals):

```json
{ "id": "spinf/moderation", "version": "0.1.3", "context": "forum",
  "categories": { "cyber": { "score": 0.81, "percentile": 0.993, "flagged": true },
                  "sexual": { "score": 0.04, "percentile": 0.41, "flagged": false } },
  "decision": { "unsafe": true, "mode": "micro_layer", "score": 1.7, "threshold": 0.4 } }
```

`decision` carries `unsafe`, `mode` (the mode used), and per mode `score`, `threshold` and `category` (max), `categories`
(per_category), `score` and `threshold` (micro_layer); plus `critical`, `by` and `warning` as above. The caller can pick
another `mode`, one `threshold` (max or micro_layer), per-category `thresholds` (`null` = reported only) and another
`critical` list per request; the categories, their scores and percentiles do not change.

### Editing a pack

Thresholds, the `critical` list, labels, contexts, `media` and `parameters` are editable in place. Changing a question,
an option, a read's constants or a Platt map invalidates the lock (the micro-layer is off, the fallback mode decides)
and means the edited reads' constants must be measured again: that is a new version. Adding or removing queries,
reads or categories follows the same rule.

### Validation

```sh
python -m spinf_serve.packs validate <folder>      # every pack file in the folder, and its prompt-packs.json
```

A pack file is valid when: the ids and the version match the patterns above; every context has exactly one `{X}` and
only declared parameters as placeholders; every query is a scored one without `group`; every read names a query of the
pack, a combination value the query has, a kind with its fields (`option` below the option count, `mu` of the right
shape, `unsafe` indices not covering every option), numeric `mean` and `sd > 0`, and a category of the pack; every
category has a `platt` pair, a threshold in [0, 1] or `null`, well-formed `normal_quantiles`, `requires` naming other
categories, and at least one read; the decision names a mode, a valid fallback, a `max.threshold` where `max` is used,
`critical` categories of the pack, and a micro-layer whose inputs exist, whose weights decode to the expected count
and which records a lock (the command also reports whether the lock matches the file). A workbench loads a pack *draft* (no fitted constants yet) with `draft=True`, which checks
everything but the constants.

## The repository manifest: `prompt-packs.json`

A pack repository (this one, or any GitHub repository or folder a server is pointed at) describes its packs in one
manifest at its root:

```json
{
  "format": 1,
  "namespace": "spinf",
  "name": "spinf prompt packs",
  "homepage": "https://github.com/spinfinc/spinf-prompts",
  "license": "Apache-2.0",
  "packs": [
    {
      "id": "spinf/moderation",
      "name": "Content moderation",
      "description": "22 categories of user-generated text with calibrated scores and a safe / unsafe decision.",
      "tags": ["trust-and-safety", "moderation"],
      "media": ["text"],
      "parameters": [],
      "readme": "packs/moderation/README.md",
      "samples": "packs/moderation/samples/",
      "versions": [
        { "version": "0.1.3", "file": "packs/moderation/0.1.3.json", "model": "spinf/gemma-4-12b-fp8",
          "status": "available", "fixture": "packs/moderation/fixtures/0.1.3.json" },
        { "version": "0.1.1", "file": "packs/moderation/0.1.1.json", "model": "spinf/gemma-4-12b-fp8",
          "status": "available", "note": "the first public calibration" }
      ]
    }
  ]
}
```

| field | meaning |
|---|---|
| `format` | `1` |
| `namespace` | the namespace every pack id in the manifest starts with. `spinf/` is reserved for the official repository; a server lists another source's `spinf/` packs only when told to |
| `name`, `homepage`, `license` | shown in the server's pack list |
| `packs[].id`, `name`, `description`, `tags` | the listing; `name` and `description` appear in `GET /v1/packs` |
| `packs[].media`, `parameters` | must agree with the pack files (`parameters` lists the names) |
| `packs[].readme`, `samples` | paths in the repository; they must exist when given |
| `versions[].version`, `file`, `model` | must agree with the file's `version` and `model`; the file must exist |
| `versions[].status` | `available` (served), `deprecated` (served, not the default), `coming_soon` (listed, not served) |
| `versions[].fixture`, `note` | the golden fixture's path (must exist when given); a free note |

The default version of a pack is the first `available` one in the manifest's order; list the newest first. A folder
without a manifest is scanned: every `*.json` with `pack` and `scoring` fields is a pack version, the newest version
is the default, and nothing is listed as coming soon. The same pack id from two sources is a configuration error.

## Fixtures

Every version here ships `packs/<name>/fixtures/<version>.json`: the pack's samples scored on the server, with the raw
`/v1/score` result per sample and the expected output in every mode the pack supports:

```json
{ "pack": "spinf/brand", "version": "0.1.0", "engine": "fp_…", "usage": { "…": "the call's usage block" },
  "cases": [ { "id": "s1", "text": "…", "context": "page", "params": { "brand": "Kestrel" },
               "result": { "input_id": "0", "queries": [ "…" ] },
               "expected": { "max": { "categories": { "…": {} }, "decision": {} }, "per_category": { "…": {} } } } ] }
```

spinf-serve's tests load every pack of the repository, recompute each case from `result` and compare with `expected`
within the fixture's rounding (2e-6). A client that implements the evaluator can do the same.
