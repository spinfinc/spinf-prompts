# spinf/triage: inbound message triage

One read of an inbound message (an email, a support chat, a help-desk ticket) answers, in parallel, everything a
triage system wants to know before a person sees it: which team it is for and what exactly it asks, how urgent it is,
the tone, the risks it carries, whether it holds sensitive personal data, and whether it is a phishing or business
email compromise attempt. The pack returns a calibrated 0 to 1 score and a normal-traffic percentile for every
category, flags for the deciding ones, and a decision: does this message need attention before routine handling?

Model: `spinf/gemma-4-12b-fp8`. Media: text. Contexts: `email` (default), `chat`, `ticket`: pick the one that matches
the channel, it tells the model what it is reading.

## What it scores

| Group | Categories | How to read them |
|---|---|---|
| `intent.*` | 13 domains: billing, account, orders, returns, technical, sales, cancellation, legal, hr, partnerships, feedback, press, other | The **route**: the domain with the highest score. All on one scale, reported only. |
| `sub.<domain>.*` | about 40 specific intents (a double charge, a password reset, a late delivery, an outage, a quote…) | The **specific request**: the highest `sub.` category within the routed domain. |
| `action.*` | reply expected, refund asked, cancellation asked, escalation needed, deadline mentioned, repeat contact | What has to happen. `action.escalate` decides. |
| `urgency` | one category | The **priority**: use the percentile (0.98 = more urgent than 98% of normal traffic), not a fixed threshold. Reported only. |
| `tone.*` | negative, churn (threatens to leave), praise | The sender's state. Reported only. |
| `risk.*` | complaint, legal threat, regulator mention, going public | `risk.legal`, `risk.regulator`, `risk.public` decide. |
| `pii.*` | card number, bank details, ID number, health information, credentials, contact details | Sensitive data present (yes / no, not spans). All but `pii.contact` decide. |
| `phishing` | one category | Phishing, scam or business email compromise: a direct read and eight signals (impersonation, payment-detail change, credential lure, false urgency, gift cards or crypto, attachment or link, unexpected windfall, sloppy writing) averaged. Decides, and is **critical**: a flag makes the decision "needs attention" in every mode. |
| `scope.*` | unrelated to the company, automated notification | Mail that should not reach a queue. Reported only, and they feed `intent.other`. |
| `lang.*` | English, German, Spanish, French, Portuguese, Italian, other | The language of the message. Reported only. |

## Outputs you build from it

- **Route**: `argmax intent.*`, then `argmax sub.<domain>.*`.
- **Priority**: bands of the `urgency` percentile, for example P1 above 0.98, P2 above 0.9, P3 above 0.6, P4 below.
- **Needs attention**: the pack's decision (`per_category`): any deciding category over its threshold, phishing always.
- **Phishing risk**: the `phishing` score itself, calibrated: 0.5 is as likely as not a phishing attempt.

## Calibration

Normal traffic is a mix of real corporate email (the Enron corpus, first messages of threads), real support chats
(ABCD, online shopping) and synthetic help-desk tickets (CC BY 4.0), about 4,000 messages. Every read is standardised
on it, so a score's percentile says how unusual the message is for that category among everyday mail. `phishing` is
also Platt-scaled on 800 real phishing emails (the Nazario corpus, CC BY 4.0) against the normal traffic. The other
deciding categories have no labelled set yet: their thresholds flag 0.5% of normal traffic, and their scores are on
the generic scale; set your own thresholds on a labelled sample (`decision.thresholds`) when the false-positive cost
matters.

The datasets were used to calibrate, not bundled; the shipped samples are written by us.

## Versions

- 0.1.1: the team question asked twice (two phrasings, opposite option orders); the scope questions feed `intent.other`.
- 0.1.0: first calibrated version.
