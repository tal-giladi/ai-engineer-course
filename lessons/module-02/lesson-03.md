# Lesson 2.3 — Validation, Repair Loops, and Failure Budgets

**Module 2: Structured Output & Prompting as Engineering** · ~13 min
**Objective:** M2-LO2 — Implement a validate-repair-retry loop with an explicit failure budget, and decide when to fail loudly instead of retrying.

---

## Introduction

In Lesson 1.4 you saw a system where per-call metrics looked perfect and per-request cost was fourteen times higher at p95. The cause was a repair loop nobody had bounded. Seventy percent of requests were failing validation, retrying up to four times, and most of them were failing anyway — burning the entire budget to arrive at the same failure.

That system was not built by careless engineers. It was built by reasonable ones who wrote `while not valid: retry()` because it was obviously the right thing to do, and who never looked at the marginal return of attempt three.

This lesson is about the third tier of the response — the one everyone skips. Validation is easy. Repair is easy. **Deciding to stop** is the engineering.

---

## Key Concept 1: Three tiers

Every invalid output goes through the same three-tier response.

**Validate.** Parse into your Pydantic model. This either succeeds totally or raises with a structured error naming the field and the violation.

**Repair.** Feed the error back to the model and ask for a correction. This works far better than re-asking blind, because the model gets a specific, actionable message:

```
Your previous response failed validation:
  urgency: Input should be 'low', 'medium' or 'high' (got 'critical')
Return the corrected object.
```

Include the failed output and the exact validation error. Do not paraphrase it — the structured error text is more useful to the model than your summary of it.

**Fail.** Raise, preserving the failed payload, the validation error, and the `request_id`. This is where the engineering lives, because "fail" competes against "try once more" and always feels worse in the moment.

---

## Key Concept 2: Retryable versus terminal

Not every validation failure is worth a retry, and treating them uniformly is the root of most bad repair loops.

**Retryable** — the model can plausibly do better with feedback:
- Malformed JSON, truncated output
- Missing required field
- Wrong type in a field
- Value outside an enum where a valid neighbour clearly applies

**Terminal** — retrying is waste, or worse:
- **The model refused.** It will refuse again. Retrying a refusal three times produces three refusals and a bill.
- **The input is genuinely ambiguous.** No amount of re-asking resolves a ticket that does not state a priority. The correct output is a null or a refusal, and if your schema cannot express that, your schema is wrong.
- **`finish_reason == "length"`.** The output was truncated by the token limit. Retrying identically truncates identically. Raise `max_tokens` or shorten the request — a retry is pure waste.
- **The schema is impossible.** You asked for a field the input cannot supply. This is a bug in your code, and retrying hides it.

That last category deserves emphasis. **A repair loop is an excellent way to hide your own bugs behind a cost increase.** If your schema requires `invoice_total` and half your documents are purchase orders with no total, you will see a 50% repair rate and a large bill, and the actual fix is a schema change. Classify before you retry, and you find these.

```python
def classify_failure(err, response) -> Literal["retryable", "terminal"]:
    if response.finish_reason == "length":
        return "terminal"          # truncation — fix max_tokens
    if looks_like_refusal(response.text):
        return "terminal"
    return "retryable"
```

---

## Key Concept 3: The failure budget

A budget has three dimensions, and most implementations set only the first.

**Attempt count.** How many repairs before giving up. Derived from data — see the worked example. Usually one or two.

**Token ceiling.** Total tokens across all attempts for one logical operation. This catches the pathological case where each repair grows the context (previous output plus error plus original prompt) and attempt four costs several times attempt one.

**Wall-clock ceiling.** Because three sequential repairs on a slow backend is a user-visible timeout, and a timeout that arrives after nine seconds of invisible retrying is a worse experience than an error at two seconds.

And a fourth thing that is not a budget but belongs beside them: **what happens when the budget is exhausted.** Options, in rough order of preference:

1. Return a typed failure the caller can handle (`Result.failure(reason, payload)`)
2. Fall back to a simpler schema — sometimes three fields succeed where seven fail
3. Fall back to a stronger model, once, for that request
4. Raise, with the payload preserved to a dead-letter store

What you must not do is return a default-constructed object. A `Sentiment(label="neutral", confidence=0.0)` fabricated on failure is Lesson 2.1's silent wrong value, reintroduced by your own error handling.

---

## Key Concept 4: The dead letter is the dataset

When you fail, write the whole thing down: the input, every attempt's raw output, every validation error, the model, the prompt version, the `request_id`.

This costs almost nothing and pays back three times over the rest of the course:

- **Now:** it is your debugging surface. Patterns in the dead-letter store name your schema bugs.
- **Module 3:** these are your hardest evaluation cases. Real failures beat invented ones.
- **Module 8:** these are fine-tuning candidates. A model that persistently violates a format is precisely the case where fine-tuning is justified.

A repair loop that discards its failures is throwing away the most valuable data your system produces.

---

## Worked Example: Where to stop, from data

An extraction pipeline over 1,000 records, instrumented per attempt with the `attempt` field from Lesson 1.4:

| Attempt | Entered | Recovered | Recovery rate | Cumulative recovered | Tokens spent | % of total tokens |
|---|---|---|---|---|---|---|
| 1 (initial) | 1,000 | 847 | 84.7% | 84.7% | 412k | 51% |
| 2 (repair 1) | 153 | 130 | 85.0% | 97.7% | 98k | 12% |
| 3 (repair 2) | 23 | 14 | 60.9% | 99.1% | 21k | 3% |
| 4 (repair 3) | 9 | 1 | 11.1% | 99.2% | 11k | 1% |
| 5 (repair 4) | 8 | 0 | 0.0% | 99.2% | 12k | 1.5% |

Read the last two rows. Attempts 4 and 5 recovered **one record out of 1,000** and consumed 23k tokens doing it — and note that the tokens grow per attempt even as the population shrinks, because each repair carries more context.

Now classify the 8 records that never succeeded:

- 5 were refusals (the documents were unreadable scans)
- 2 hit `finish_reason == "length"`
- 1 required a field the document genuinely did not contain

**All eight were terminal from attempt one.** With classification in place, they would never have entered the repair loop at all, and the budget of 2 would have cost nothing extra.

The decision this table supports: **budget of 2, with terminal classification before the first repair.** Not because two is a nice number, but because attempt 3 recovers 1.4% and attempts 4–5 recover 0.1%.

Your table will differ. Build it.

---

## Practice

**1. Implement the loop.**

```python
def validate_repair_call(messages, schema, budget: RepairBudget) -> Result:
    ...
```

It must: classify failures as retryable or terminal before repairing; respect all three budget dimensions; record `attempt` on every `CallRecord`; return a typed result rather than raising into the caller's face; and write a dead-letter record on exhaustion.

**2. Run the adversarial set.** `exercises/m2-adversarial-inputs/` contains 200 records seeded with truncation cases, genuine refusals, ambiguous inputs, and ordinary hard cases. Run with a generous budget of 5, no classification.

**3. Build your table.** Reproduce the columns above from your own telemetry. Include the token column — it is what makes the argument.

**4. Set your budget and defend it.** Write two or three sentences in `docs/repair-budget.md` citing your own recovery rates. Then enable terminal classification and re-run: report the change in total tokens and in final success rate. Success should be flat; tokens should drop.

**5. Check your dead letters.** Read them. If more than a quarter share one cause, you have found a schema bug, not a model problem. Fix it and note the finding.

---

## Visual Aid

*Flowchart. `call model` → `validate` → on success, exit green. On failure → `classify` diamond. The "terminal" branch goes straight to a red `dead letter + typed failure` box, bypassing repair entirely — drawn prominently, since this is the path most implementations lack. The "retryable" branch goes to a `budget exhausted?` diamond with three inputs drawn feeding it (attempts, tokens, wall-clock); "no" loops back to the model with an `+error feedback` annotation, "yes" joins the terminal path. A counter badge sits on the loop edge. Caption: "Two paths reach the red box. Only one of them costs money first."*

---

## Key Takeaways

- **Validate, repair, fail — and the third tier is the engineering.** Unbounded repair converts a visible failure into an invisible cost problem with a long latency tail.
- **Classify before you retry.** Refusals, truncation, and genuinely ambiguous inputs are terminal; retrying them is pure waste, and it hides your own schema bugs behind a bill.
- **A budget has three dimensions** — attempts, tokens, wall-clock — and an explicit exhaustion path that is never a default-constructed object.
- **Set the budget from your own recovery table.** Marginal recovery collapses fast; the tokens per attempt do not.
- **Dead letters are an asset.** Debugging surface now, evaluation cases in Module 3, fine-tuning candidates in Module 8.

---

## Next

Your repair loop is bounded, classified, and instrumented. Its entire behavior — whether the model complies on attempt one, whether the error feedback is useful — depends on a prompt that currently lives as a string literal somewhere in your code, has no version, is not reviewed, and cannot be rolled back.

**Lesson 2.4** fixes that, and then uses the same reordering discipline to cut your input costs through prompt caching.
