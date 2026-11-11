# Module 2 Quiz — Structured Output & Prompting as Engineering

**9 questions · ~27 min plus the practical**

---

## Q1 — Multiple choice

A sentiment classifier parses model output with `response.strip().lower()` and compares against `"positive"`. The model returns `"The sentiment appears mixed, leaning positive"`.

Someone proposes fixing it with a regex that takes the last word. What is wrong with that fix?

A. Nothing — it handles this case correctly
B. It is slow at scale
C. It converts an unhandled case into a confidently wrong value
D. Regex cannot match word boundaries reliably

---

## Q2 — Short answer

Explain why a *silent* parsing failure is more expensive than a parsing failure that raises an exception. Give a concrete example of how a team would fail to notice one.

---

## Q3 — Diagnosis

Two schemas describe the same extraction task. Both are enforced with constrained decoding, so format-failure rate is 0% for both.

**Version A** — field accuracy 71%:
```python
class Extraction(BaseModel):
    meta: Meta                # nested
    classification: str       # free text
    details: Details          # nested
```

**Version B** — field accuracy 94%:
```python
class Extraction(BaseModel):
    reasoning: str
    category: Literal["billing", "technical", "account", "other"]
    urgency: Literal["low", "medium", "high"] = Field(
        description="high only if the customer is currently blocked")
```

Name **four** specific design differences and explain the mechanism by which each helps.

---

## Q4 — Applied judgment

You instrument a repair loop and get this:

| Attempt | Entered | Recovered | Tokens spent |
|---|---|---|---|
| 1 | 1,000 | 847 | 412k |
| 2 | 153 | 130 | 98k |
| 3 | 23 | 14 | 21k |
| 4 | 9 | 1 | 11k |
| 5 | 8 | 0 | 12k |

What repair budget do you set, and what is your justification in one sentence?

---

## Q5 — Classification

For each failure, state whether it is **retryable** or **terminal**, and why:

(a) `ValidationError: urgency — input should be 'low', 'medium' or 'high' (got 'critical')`
(b) `finish_reason == "length"`
(c) The model responds *"I can't help with that request."*
(d) The input is a purchase order; your schema requires `invoice_total`, which purchase orders do not have
(e) `json.JSONDecodeError: Expecting ',' delimiter`

---

## Q6 — Design

Your team ships a prompt change. Two weeks later, monthly spend is up 40% and nobody can explain it.

What single field in your telemetry would let you answer this in one SQL query, and what query would you run?

---

## Q7 — Applied judgment

Below are two prompt layouts. Identify the cacheable prefix in each, and estimate which produces a higher cache hit rate and why.

**Layout A:**
```
Session: {session_id} | Date: {today}
You are a documentation assistant...
[tool definitions]
[4 few-shot examples]
Context: {retrieved_chunks}
Question: {question}
```

**Layout B:**
```
You are a documentation assistant...
[tool definitions]
[4 few-shot examples]
Session: {session_id} | Date: {today}
Context: {retrieved_chunks}
Question: {question}
```

What would you expect to happen on a backend with no prefix caching?

---

## Q8 — Multiple choice

Your extraction service calls a hosted provider's API. Every output must match a 9-field schema, and `invoice_date` must not be later than `due_date`. Format failures and date-order violations both reach production today.

Which design fits best?

A. Run the model yourself on vLLM so constrained decoding can enforce both the schema and the date rule
B. Use the provider's native structured outputs, then a validator with a bounded repair loop for the date rule
C. Keep prompt-only JSON and raise the repair budget to 5 attempts, since each repair fixes most failures
D. Use tool-call shaping and encode the date rule as a JSON Schema `minimum` constraint on `due_date`

---

## Q9 — Practical

Build an extractor for messy invoice text that **never returns an invalid object**. Inputs include mixed formats, missing fields, two languages, and one OCR-mangled scan.

**Deliverable:** `exercises/m2-invoice-extractor/` containing the schema, the repair budget with a data-backed justification, and a failure log showing what was refused and why.

---
---

# Answer Key

## Q1 — **C**

`"The sentiment appears mixed, leaning positive"` ends in `positive`, so a last-word regex parses it as a confident positive. The model expressed uncertainty; the parser recorded certainty.

This is strictly worse than the original failure. Before the fix, the value fell through to an else-branch. After it, an ambiguous case is confidently misclassified — and it looks identical to a correct result in the database.

**The general lesson:** each parser patch encodes an assumption about a distribution that will shift, and patches can convert unhandled cases into wrong ones.

## Q2

**Why it is more expensive:** an exception tells you *where* and *when*. A silent wrong value produces a row indistinguishable from a correct one. Dashboards render, error rates stay flat, tests pass on clean fixtures.

**Concrete example (any equivalent):** a ticket classifier misparses ~5% of inputs into a default category. Error-rate monitoring shows nothing, because nothing errors. The team discovers it months later when someone notices a category is under-represented, or a customer escalates a ticket that was never routed.

**Key phrase for full credit:** monitoring on error rates cannot detect a failure that does not error.

## Q3 — Four differences

**1. Flat versus nested.** Version A requires the model to maintain structural state across nesting levels while also getting content right. Flattening removes a whole class of structural error and frees capacity for the content.

**2. Enums versus free text.** `Literal["billing", "technical", ...]` is enforceable and self-documenting. `classification: str` invites the model to invent a category — and constrained decoding cannot help, because any string is valid.

**3. Field descriptions.** *"high only if the customer is currently blocked"* sits directly against the field it governs. This is prompt real estate the model reads at the point of decision, and it is more effective than the same instruction in a system prompt.

**4. Reasoning before the answer.** Generation is sequential, so a `reasoning` field emitted *before* `category` lets the model condition its answer on its own analysis. Placed after, it is post-hoc rationalization that costs tokens and buys nothing.

**Also creditable:** descriptive field names (`urgency` over `details.u`), and required-and-nullable over optional.

## Q4 — **Budget of 2**

Attempt 2 recovers 85% of remaining failures. Attempt 3 recovers 1.4% of the original population. Attempts 4 and 5 together recover **one record out of 1,000** while consuming 23k tokens — and note the tokens per attempt *grow* even as the population shrinks, because each repair carries more context.

**Full credit** requires citing the marginal recovery rate. "Two seems reasonable" is not the answer; "attempt 3 recovers 1.4% and attempts 4–5 recover 0.1% at rising token cost" is.

**Bonus insight:** with terminal classification in place, most of the 8 permanent failures would never enter the loop at all.

## Q5

**(a) Retryable.** The model chose a value outside the enum but clearly understood the task. The validation error text tells it exactly what to fix.

**(b) Terminal.** Output was truncated by the token limit. An identical retry truncates identically. The fix is raising `max_tokens` or shortening the request — retrying is pure waste.

**(c) Terminal.** A refusal will be repeated. Three retries produce three refusals and a bill.

**(d) Terminal — and it is a bug in your code.** The schema requires a field the input cannot supply. Retrying hides a schema error behind a cost increase. **This is the most important item in the question:** a high repair rate concentrated on one input type usually means your schema is wrong, not the model.

**(e) Retryable.** Malformed JSON is the canonical repairable case.

## Q6

**Field:** `prompt_version`.

**Query:**
```sql
SELECT prompt_version, COUNT(*), AVG(tokens_in), AVG(tokens_out), AVG(cost_usd)
FROM calls
WHERE call_site = 'rag.answer'
GROUP BY prompt_version;
```

This converts "spend went up in March" into "spend went up when `answer:v3` shipped, which raised average input tokens from 1,840 to 3,110 because it added four few-shot examples." That is a decision — are the examples worth 69% more input cost? — rather than a mystery.

Without the field, the investigation has no starting point.

## Q7

**Layout A cacheable prefix: zero tokens.** `{session_id}` and `{today}` vary per request and sit at position 0, so nothing below them can be a stable prefix.

**Layout B cacheable prefix: the system prompt, tool definitions, and four few-shot examples** — typically 1,500–3,000 tokens, identical on every request.

**Expected result:** Layout B achieves a high hit rate (70–85% on real traffic); Layout A achieves 0%. Expect substantially lower input cost and TTFT on B.

**On a backend without prefix caching:** no difference at all — identical cost, identical TTFT.

**Which is exactly why the ordering discipline is applied unconditionally** (it never hurts, and it is free) **while the caching mechanism sits behind a capability check** (Lesson 1.2's rule).

## Q8 — **B**

On a hosted API you cannot touch the sampler, so the provider's native structured outputs are the strongest shape guarantee available. The date rule is a cross-field semantic constraint that JSON Schema cannot express, so it belongs in a Pydantic validator with a bounded, classified repair loop (Lesson 2.3).

**A** changes the hosting model to solve a problem the hosted API already solves, and constrained decoding still cannot express "date A before date B". **C** keeps the weakest rung and spends money on attempts that the recovery table shows rarely pay off. **D** fails because JSON Schema `minimum` applies to numbers, not to a comparison between two date fields.

## Q9 — Practical rubric

| Criterion | Full credit |
|---|---|
| Never returns invalid | No code path constructs a default object on failure |
| Strongest mechanism | Uses the strongest structuring available per backend, with a documented fallback ladder |
| Terminal classification | Refusals, truncation, and impossible-field cases do not enter the repair loop |
| Budget justified | Cites the candidate's own recovery table, not a convention |
| Failure log | Records input, attempts, errors, and reason for refusal |

**Common failure:** returning a default-constructed object when the budget is exhausted. That reintroduces the Lesson 2.1 silent wrong value through the error handler itself, which is the most ironic possible way to fail this module.
