# Lesson 2.2 — Structured Outputs and Schema-Constrained Generation

**Module 2: Structured Output & Prompting as Engineering** · ~13 min
**Objective:** M2-LO1 — Define a typed contract between model and application using JSON Schema and Pydantic, and enforce it with schema-constrained generation.

---

## Introduction

Last lesson ended with a rule: stop parsing, start constraining. This lesson is the mechanics of that — three different mechanisms with genuinely different guarantees, an honest map of which backends support which, and the part that surprises most engineers: **how you design the schema matters as much as which mechanism you use.**

That last point is worth stating up front, because it is counterintuitive. Two schemas describing the same data, both valid, both enforced identically, can differ by twenty points in field-level accuracy. The schema is not just a validation artifact. It is part of the prompt.

---

## Key Concept 1: Three mechanisms, three guarantees

**Mechanism 1 — Prompt and hope.** Ask for JSON in the prompt, parse the result.

```
Respond with JSON matching: {"label": "...", "confidence": 0.0}
```

Guarantee: none. Modern models comply most of the time. "Most" is doing a lot of work — the failures are markdown code fences, preambles, trailing commentary, and occasionally invalid JSON. Available on every backend, which is its only real virtue. This is your floor, not your target.

**Mechanism 2 — Tool/function-call shaping.** Declare your schema as a tool the model can call. The provider handles the shaping, and most enforce the schema server-side.

```python
tools = [{
    "type": "function",
    "function": {
        "name": "record_sentiment",
        "parameters": Sentiment.model_json_schema(),
    },
}]
```

Guarantee: strong on most hosted providers, variable elsewhere. The output arrives as structured arguments rather than as text you parse. This is the widely-available middle rung and where most production systems live.

**Mechanism 3 — Constrained decoding.** The sampler itself is restricted at each step to tokens that keep the output valid against a grammar compiled from your schema. Invalid output is not "rejected" — it is *unreachable*.

Guarantee: total, for shape. Available on self-hosted stacks (vLLM with a structured-output backend, llama.cpp grammars) and on some hosted providers under a strict structured-output mode. This is the strongest rung, and the reason your Module 10 self-hosted deployment gains a capability rather than losing one.

---

## Key Concept 2: Capability detection, not assumption

This is Lesson 1.2's seam arriving exactly where it was promised. Your backends do not all support the same rung, so your client needs a declared capability and an explicit ladder:

```python
def structured_complete(self, messages, schema: type[BaseModel]):
    if self._caps.constrained_decoding:
        return self._constrained(messages, schema)      # rung 3
    if self._caps.tool_calling:
        return self._via_tool_call(messages, schema)    # rung 2
    logger.warning("profile=%s falling back to prompt-only structuring",
                   self._profile)
    return self._prompt_json(messages, schema)          # rung 1
```

Two rules make this safe rather than dangerous:

**Degrade explicitly, never silently.** The warning matters. A system that quietly drops from rung 3 to rung 1 when someone switches profiles will produce a mysterious spike in validation failures with nothing pointing at the cause.

**Record the rung used.** Add it to the `CallRecord` from Lesson 1.4. When you compare failure rates across backends in the practice section, this field is what makes the comparison interpretable.

---

## Key Concept 3: Schema design is prompt design

Here is the part that gets overlooked. The schema is serialized into the model's context — including your field names and descriptions. The model reads it. Design accordingly.

**Flat beats nested.** Deeply nested structures degrade accuracy noticeably. The model must hold more structural state while also getting the content right. Where you need nesting, consider two calls instead of one.

**Enums beat free strings.** `Literal["billing", "technical", "account"]` is enforceable, self-documenting, and removes an entire failure class. Every free-text field is a place the model can be creative when you did not want creativity.

**Required-and-nullable beats optional.** An optional field lets the model omit it, and you cannot distinguish "not applicable" from "forgot." A required field typed `str | None` forces an explicit decision.

**Field descriptions are prompt real estate.** This is the highest-leverage and least-used lever:

```python
class Ticket(BaseModel):
    priority: Literal["low", "medium", "high"] = Field(
        description="high only if the customer is currently blocked from "
                    "working. Frustration alone is medium."
    )
```

That description does more work than three paragraphs in your system prompt, because it sits directly against the field it governs.

**Name fields the way you would explain them.** `is_blocked_from_working` outperforms `flag_2`. The model has no schema documentation beyond what you wrote.

**Ask for reasoning before the answer.** Order matters, because generation is sequential — a `reasoning` field placed *before* `label` lets the model condition its answer on its own reasoning. Placed after, it is post-hoc rationalization and adds cost without accuracy.

---

## Key Concept 4: What this does not solve

Say it plainly, because the false confidence is expensive: **constrained decoding guarantees shape, never correctness.**

A perfectly-enforced schema will happily return:

```json
{"label": "positive", "confidence": 0.97,
 "rationale": "The customer expressed satisfaction."}
```

for a furious complaint. Every constraint satisfied. The answer is wrong.

Constrained decoding eliminates parse errors, missing fields, invalid enum values, and type mismatches — the entire Lesson 2.1 failure gallery. It does nothing about wrong values. Knowing which class of problem you have solved is the difference between an engineer who ships confidently and one who ships blindly, and it is exactly why Module 3 comes next.

---

## Worked Example 1: The three mechanisms measured

Same extraction task, 200 records, three backends, format-failure rate (output that could not be validated at all):

| | Prompt-only | Tool-call | Constrained |
|---|---|---|---|
| Frontier | 3.5% | 0.5% | 0.0% |
| HF endpoint (8B) | 14.0% | 2.0% | n/a |
| Local vLLM (7B) | 22.5% | 6.5% | 0.0% |

Two observations that shape real architecture. Smaller models degrade *dramatically* on prompt-only structuring — the 22.5% figure is what makes a naive open-weights integration feel unusable. And constrained decoding flattens the difference entirely: the local 7B model produces valid output as reliably as the frontier model. That is a large part of why self-hosting is viable at all, and it is worth remembering when you reach Module 9.

## Worked Example 2: Same data, two schemas

Both enforced with constrained decoding, so format failures are zero in both. What differs is field-level accuracy.

**Version A** — nested, free-text, terse names:

```python
class Extraction(BaseModel):
    meta: Meta            # nested: {source, confidence, notes}
    classification: str   # free text
    details: Details      # nested: {urgency, tier, tags[]}
```

Field accuracy: **71%**

**Version B** — flat, enum-constrained, described, reasoning-first:

```python
class Extraction(BaseModel):
    reasoning: str = Field(description="Brief analysis before deciding.")
    category: Literal["billing", "technical", "account", "other"]
    urgency: Literal["low", "medium", "high"] = Field(
        description="high only if the customer is currently blocked."
    )
    customer_tier: Literal["free", "pro", "enterprise"] | None = Field(
        description="null if not stated in the ticket."
    )
```

Field accuracy: **94%**

Twenty-three points. The prompt was not touched.

---

## Practice

**1. Define the spine answer contract.** This is the type your RAG system will return from Module 5 onward:

```python
class Answer(BaseModel):
    reasoning: str
    answer: str
    citations: list[str]          # chunk IDs — validated in Module 5
    confidence: Literal["high", "medium", "low"]
    refused: bool
    refusal_reason: str | None
```

**2. Implement the ladder.** Add `structured_complete()` to your client with capability detection and explicit degradation. Record the rung in `CallRecord`.

**3. Measure across backends.** Run 100 records through each profile and reproduce the failure-rate table on your own numbers. Document your ladder in `docs/structured-output-ladder.md` — which backend gets which rung, and what happens on degradation.

**4. Run the schema ablation.** Take one schema, produce a deliberately poor version (nested, free-text, undescribed) and a good one, and measure field accuracy on both with constrained decoding. Report the gap. You are proving to yourself that the schema is part of the prompt.

---

## Visual Aid

*A three-rung ladder drawn vertically. Bottom rung: "Prompt and hope — guarantee: none — available: everywhere." Middle: "Tool-call shaping — guarantee: strong on most hosted providers — available: widely." Top: "Constrained decoding — guarantee: total (shape only) — available: self-hosted + strict modes." A dashed arrow labeled "explicit degradation, logged" runs downward. Off to the right, a separate boxed note in a different colour: "None of these rungs make the content correct. → Module 3."*

---

## Key Takeaways

- **Three mechanisms, three guarantees:** prompt-only (none), tool-call shaping (strong, widely available), constrained decoding (total for shape, self-hosted and strict modes).
- **Detect capability and degrade explicitly.** A silent drop from rung 3 to rung 1 produces a mysterious failure spike with no attributable cause. Log the rung used.
- **Schema design is prompt design.** Flat over nested, enums over free strings, required-and-nullable over optional, descriptions on every non-obvious field, and reasoning fields *before* the answer they justify.
- **Constrained decoding closes the gap between small and large models** on format compliance — a large part of why self-hosting is practical.
- **Shape is not correctness.** Every constraint can be satisfied by a wrong answer.

---

## Next

You are now producing valid, typed output at the strongest level each backend supports. On the calls where the strongest level is still not available — or where the content itself makes compliance impossible — you will get an invalid object anyway.

**Lesson 2.3** covers what happens next: the validate-repair-retry loop, why the natural instinct to retry until it works is a cost incident wearing a helpful disguise, and how to pick a repair budget from your own data rather than from a blog post.
