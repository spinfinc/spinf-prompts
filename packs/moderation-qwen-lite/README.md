# Content moderation, lite (Qwen3.8 27B) (`spinf/moderation-qwen-lite`)

One category (`unsafe`, the overall score) from three questions written in the model's chat format, modes `max` and `per_category` (no micro-layer), 106 question tokens per input: about a fifteenth of the full pack's tokens, no per-category scores.

Model: `spinf/qwen3.8-27b-int4`. Media: text. Versions: 0.1.0 (latest 0.1.0).

## Contexts

How the content is presented to the model (`{X}` is the content); the first is the default.

- `ai`: `<|im_start|>system\nYou are a careful content moderator. You read a message and answer questions about it with one word.<|im_end|>\n<|im_start|>user\nA user sent the following message to an AI assistant.\n\nUser: {X}`
- `forum`: `<|im_start|>system\nYou are a careful content moderator. You read a message and answer questions about it with one word.<|im_end|>\n<|im_start|>user\nThe following message was posted by a user on an online platform:\n\n\"{X}\"`

## Categories

| id | label | default threshold |
|---|---|---|
| `unsafe` | Unsafe content (overall) | 0.3966 |

## Decision

Default mode `max`; modes available: `max`, `per_category`; fallback `per_category`.

Questions: 3; reads: 3; answers read from `logprob`; calibrated on engine `fp_317e5116c2`.

## Measured

On the benchmark's 9 sets: 77.0 mean macro F1 at its default threshold, AUROC 0.879. TODO: confirm these numbers may be published.

## Files

- `<version>.json`: the pack file (format: `docs/pack-format.md`).
- `fixtures/<version>.json`: eight inputs with their raw query results and the expected categories and decision in every mode; an evaluator must reproduce them.
- samples: `packs/moderation/samples/`.
