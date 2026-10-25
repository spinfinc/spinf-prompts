> **Source: the spinf website guide, copied 2026-10-12. To adapt for this repository: the hosted-API specifics (api.spinf.com, keys, billing) become spinf-serve equivalents, model ids are the new names.**

> **Source: the spinf website guide, copied 2026-10-12. To adapt for this repository: the hosted-API specifics (api.spinf.com, keys, billing) become spinf-serve equivalents, model ids are the new names.**

# Writing questions

> **Beta.**

As with any language model, how you write the questions changes the results, often by more than the choice of model.
The good news: spinf is cheap enough to try several wordings, and a few labelled examples, on a few hundred items of
your own data for cents, and to keep the version that works best. This page collects what made the biggest difference
in our benchmarks (spinf-12b, synthetic and public datasets; see the [use cases](/use-cases) for the figures). The
test sets, the exact questions and example sets, and the code to re-run them are public:
[spinf-benchmarks on GitHub](https://github.com/spinfinc/spinf-benchmarks).

## 1. Say what the content is

Start the content with a short label that says what follows: `Customer support message:`, `User post:`,
`Customer review:`, `News article:`. It tells the model how to read the text. In our moderation benchmark, dropping the
label `User post:` took the zero-shot policy category from 79% to 52% correct.

```json
{ "role": "user", "content": "Customer support message:\nHi, since this morning's SSO change nobody on our team can log in…" }
```

## 2. Name the answers in the question

A closed question works best when the question lists its answers, and the options are those words:

```json
{
  "id": "topic",
  "template": "\n\nTopic (billing, technical, account, shipping, or cancellation):{?}",
  "options": [" billing", " technical", " account", " shipping", " cancellation"]
}
```

- One or two words per answer, clearly different from each other, each with a leading space.
- For yes/no questions, put the definition in the question: `"Is this urgent (business blocked, security, legal
  threat, data loss, or a deadline within 24 hours)?\nAnswer:{?}"`.
- Keep the list manageable. A few to a few dozen answers work well; for a long list (e.g. 77 banking intents), ask
  for the broad category first, then the answer within it.
- Try two or three wordings on items you know the answer to, and keep the one that separates them best.

## 3. Add a few labelled examples

Labelled examples placed in the content, before the item to score, are the biggest lever: they show the model your
labels, your boundaries and your edge cases. With 8 examples in the prompt, our benchmarks went from:

| Question | No examples | 8 examples |
|---|---|---|
| Ticket: can a bot resolve it? | 67% | 93% |
| Moderation: policy category (7) | 79% | 98% |
| Review: sentiment per aspect | 43% | 94% |
| Public prompt-injection set | 78% | 91% |

Write each example in the same format as the question, with its answer, and separate them from the item:

```text
Customer support message:
I was charged twice for September, please refund one of the charges.

Topic (billing, technical, account, shipping, or cancellation): billing

---

Customer support message:
The export button does nothing since the last update.

Topic (billing, technical, account, shipping, or cancellation): technical

---

Customer support message:
<the ticket to score>
```

- 4 to 8 examples are usually enough; cover every answer at least once, and include the cases people get wrong.
- Never score the items you used as examples, when you measure accuracy.
- Examples are part of the content, so they are billed with every input: 8 short examples add about 1,100 tokens per
  item (about $0.12 per 1,000 tickets at {{SPINF_PRICE_PER_M}} per million tokens). Send many inputs per call with
  `inputs`, so the per-call minimum and the questions' floors are shared.

## 4. Raw or calibrated?

The floor (`floor_p`, the same question with empty content) removes the question's own bias; see
[Calibration](/docs/calibration).

- **Without examples**, read the lift over the floor for multi-choice questions: it often fixes a question that leans
  towards one answer (ticket topic: 75% raw, 87% calibrated; aspects: 43% raw, 84% calibrated).
- **With examples**, read `p` as it is: the floor is computed without your examples, so it no longer describes the
  question the model sees (aspects with 8 examples: 94% raw, 80% calibrated).

## 5. Iterate on your own data

1. Label 100–300 items of your own data (fewer if they are easy to tell apart).
2. Score them zero-shot, then with 4 and 8 examples, with two or three wordings of each question.
3. Keep the version that separates your labels best, pick your thresholds, and keep the template, the answers and the
   examples fixed from then on: scores compare over time only under the same question and `system_fingerprint`.

A round on 300 items with 8 examples is about 400,000 tokens: a few cents per variant, and well within the free tokens
of a new account.

## 6. Ask many questions at once

Each question costs only its own tokens: the content is read once per call. So add the checks you would otherwise skip
to the same call: route a message by intent, check it for prompt injection and against your content policy, all in
one read. Use placeholders (`{aspect}`, `{company}`) with `combinations` to ask the same question for many values.
