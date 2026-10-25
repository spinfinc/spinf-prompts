# Content moderation (`spinf/moderation`)

22 categories of user-generated text with calibrated 0-1 scores, a percentile against normal traffic, and a safe / unsafe decision in one of three modes. 58 questions, 106 prompts, about 1,626 question tokens per input, scored in one pass with the content.

Model: `spinf/gemma-4-12b-fp8`. Media: text. Versions: 0.1.3, 0.1.1 (latest 0.1.3).

## Contexts

How the content is presented to the model (`{X}` is the content); the first is the default.

- `ai`: `A user sent the following message to an AI assistant.\n\nUser: {X}`
- `forum`: `The following message was posted by a user on an online platform:\n\n\"{X}\"`

## Categories

| id | label | default threshold |
|---|---|---|
| `sexual` | Sexual content | 0.7065 |
| `child` | Child sexual exploitation and grooming | 0.6304 |
| `violence` | Violence and threats | 0.6888 |
| `selfharm` | Self-harm and suicide | 0.6028 |
| `hate` | Hate against protected groups | 0.7494 |
| `harassment` | Harassment and bullying | 0.7704 |
| `extremism` | Extremism and terrorism | 0.7743 |
| `weapons` | Weapons | 0.7037 |
| `drugs` | Illegal drugs | 0.7068 |
| `crime` | Other crime | 0.655 |
| `fraud` | Fraud and scams | 0.8064 |
| `cyber` | Hacking and malware | 0.69 |
| `privacy` | Private personal information | 0.6691 |
| `spam` | Spam and advertising | reported only |
| `profanity` | Profanity | 0.7001 |
| `graphic` | Gore and graphic violence | 0.6074 |
| `gibberish` | Gibberish | reported only |
| `alcohol` | Alcohol (topic) | reported only |
| `tobacco` | Tobacco and vaping (topic) | reported only |
| `gambling` | Gambling (topic) | reported only |
| `medication` | Medication (topic) | reported only |
| `general` | Overall harmfulness (general reads) | 0.6592 |

## Decision

Default mode `micro_layer`; modes available: `max`, `per_category`, `micro_layer`; fallback `per_category`; critical categories: `selfharm`, `child`.

Questions: 58; reads: 234; answers read from `logprob`; calibrated on engine `fp_2b5214ebdc`.

## Measured

Version 0.1.1, on 9 public two-class moderation sets (macro F1, sets never seen in training): micro_layer 74.5, max 72.5, per_category 72.5. 0.1.1 is the version the paper and the public benchmark pin; 0.1.3 has the same questions refit on the batch-invariant engine, reads full-precision log-probs and adds the critical categories (self-harm, child safety).

## Files

- `<version>.json`: the pack file (format: `docs/pack-format.md`).
- `fixtures/<version>.json`: eight inputs with their raw query results and the expected categories and decision in every mode; an evaluator must reproduce them.
- `samples/inputs.json`: sample inputs (written by spinf) for the playground and quickstarts.
