# Content moderation (Qwen3.8 27B) (`spinf/moderation-qwen`)

The same 58 questions, 22 categories and three decision modes as `spinf/moderation`, recalibrated for `spinf/qwen3.8-27b-int4` through the pack's own request (about 1,623 question tokens per input).

Model: `spinf/qwen3.8-27b-int4`. Media: text. Versions: 0.1.0 (latest 0.1.0).

## Contexts

How the content is presented to the model (`{X}` is the content); the first is the default.

- `ai`: `A user sent the following message to an AI assistant.\n\nUser: {X}`
- `forum`: `The following message was posted by a user on an online platform:\n\n\"{X}\"`

## Categories

| id | label | default threshold |
|---|---|---|
| `sexual` | Sexual content | 0.6551 |
| `child` | Child sexual exploitation and grooming | 0.3969 |
| `violence` | Violence and threats | 0.7226 |
| `selfharm` | Self-harm and suicide | 0.6114 |
| `hate` | Hate against protected groups | 0.6491 |
| `harassment` | Harassment and bullying | 0.6641 |
| `extremism` | Extremism and terrorism | 0.7063 |
| `weapons` | Weapons | 0.6342 |
| `drugs` | Illegal drugs | 0.68 |
| `crime` | Other crime | 0.7806 |
| `fraud` | Fraud and scams | 0.6916 |
| `cyber` | Hacking and malware | 0.5273 |
| `privacy` | Private personal information | 0.5289 |
| `spam` | Spam and advertising | reported only |
| `profanity` | Profanity | 0.4572 |
| `graphic` | Gore and graphic violence | 0.5071 |
| `gibberish` | Gibberish | reported only |
| `alcohol` | Alcohol (topic) | reported only |
| `tobacco` | Tobacco and vaping (topic) | reported only |
| `gambling` | Gambling (topic) | reported only |
| `medication` | Medication (topic) | reported only |
| `general` | Overall harmfulness (general reads) | 0.8329 |

## Decision

Default mode `micro_layer`; modes available: `max`, `per_category`, `micro_layer`; fallback `per_category`; critical categories: `selfharm`, `child`.

Questions: 58; reads: 224; answers read from `logprob`; calibrated on engine `fp_317e5116c2`.

## Measured

On the benchmark's 9 sets its micro-layer scores 76.2 mean macro F1 (the Gemma pack: 74.4), 75.5 over 11 sets. TODO: confirm these numbers may be published.

## Files

- `<version>.json`: the pack file (format: `docs/pack-format.md`).
- `fixtures/<version>.json`: eight inputs with their raw query results and the expected categories and decision in every mode; an evaluator must reproduce them.
- samples: `packs/moderation/samples/`.
