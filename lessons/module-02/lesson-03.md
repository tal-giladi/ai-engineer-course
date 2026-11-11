---
id: "02.3"
module: 2
minutes: 13
practice_minutes: 90
prerequisites: ["02.2"]
objectives:
  - "Implement a validate-repair-retry loop with an explicit failure budget, and decide when to fail loudly instead of retrying."
volatility: implementation
sources:
  - title: "Instructor (validation, max_retries, from_provider)"
    url: https://github.com/567-labs/instructor
  - title: "Pydantic validators"
    url: https://docs.pydantic.dev/latest/concepts/validators/
  - title: "Anthropic structured outputs (output_config.format)"
    url: https://platform.claude.com/docs/en/build-with-claude/structured-outputs
  - title: "vLLM structured outputs"
    url: https://docs.vllm.ai/en/latest/features/structured_outputs.html
last_verified: "2026-11-11"
---

# 02.3 · Validation, Repair Loops, and Failure Budgets

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

## Key Concept 5: Where the repair loop still belongs, and the library for it

Lesson 2.2 moved shape enforcement into the decoder or the provider. That shrinks this lesson's loop; it does not remove it.

- **With constrained decoding or native structured outputs**, the format failures in the worked example below (malformed JSON, missing fields, bad enum values) mostly disappear. What remains is **semantic validation**: cross-field rules, references that must exist, totals that must add up. Write them as Pydantic validators, and the repair loop handles only those.
- **On a backend with no enforcement**, the loop still does everything.

You do not have to hand-write the plumbing. **Instructor 1.17** (`pip install "instructor==1.17.0"`, repo `567-labs/instructor`, checked 2026-11-11) wraps a provider client, validates the response against your Pydantic model including its validators, and re-asks with the validation error:

```python
from datetime import date

import instructor
from pydantic import BaseModel, model_validator

class Booking(BaseModel):
    start_date: date
    end_date: date

    @model_validator(mode="after")
    def end_after_start(self):
        if self.end_date <= self.start_date:
            raise ValueError("end_date must be after start_date")
        return self

client = instructor.from_provider("openai/<model-id>")
booking = client.chat.completions.create(
    response_model=Booking, messages=messages, max_retries=2,
)
```

Use the library for the plumbing and keep the engineering: **`max_retries` is only the first of the three budget dimensions.** It does not classify refusals or `finish_reason == "length"` as terminal, does not cap tokens or wall-clock, and does not write your dead letters. Wrap it, classify before calling it again, and record every attempt in `CallRecord`, or you have rebuilt the Lesson 1.4 cost incident with a nicer API.

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

**6. Re-run with shape enforced (20 minutes).** Switch the adversarial run to rung 3 from Lesson 2.2 (native structured outputs or vLLM `structured_outputs`) and add one semantic validator to your schema. Then implement the same loop once more with Instructor 1.17 inside your `validate_repair_call()` wrapper, keeping your classification and budgets. **You know it worked when** your recovery table shows format failures near zero, the remaining repairs are all semantic, and total tokens are lower than in step 3.

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
- **Enforce shape first, repair semantics second.** With native structured outputs or constrained decoding, the repair loop exists for rules a schema cannot express; a library such as Instructor handles the plumbing, but `max_retries` is not a failure budget.

---

## Sources

Checked 2026-11-11.

- Instructor (validation, `max_retries`, `from_provider`): https://github.com/567-labs/instructor — checked 2026-11-11 (Instructor 1.17.0, released 2026-09-09)
- Pydantic validators: https://docs.pydantic.dev/latest/concepts/validators/ — checked 2026-11-11
- Anthropic structured outputs (`output_config.format`): https://platform.claude.com/docs/en/build-with-claude/structured-outputs — checked 2026-11-11
- vLLM structured outputs: https://docs.vllm.ai/en/latest/features/structured_outputs.html — checked 2026-11-11 (vLLM 0.33.0)

---

## Next

Your repair loop is bounded, classified, and instrumented. Its entire behavior — whether the model complies on attempt one, whether the error feedback is useful — depends on a prompt that currently lives as a string literal somewhere in your code, has no version, is not reviewed, and cannot be rolled back.

**Lesson 2.4** fixes that, and then uses the same reordering discipline to cut your input costs through prompt caching.
