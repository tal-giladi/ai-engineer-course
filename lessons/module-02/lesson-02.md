---
id: "02.2"
module: 2
minutes: 13
practice_minutes: 75
prerequisites: ["02.1"]
objectives:
  - "Define a typed contract between model and application using JSON Schema and Pydantic, and enforce it with schema-constrained generation."
volatility: implementation
sources:
  - title: "vLLM structured outputs (structured_outputs, StructuredOutputsParams, backends, guided_* removed in v0.12.0)"
    url: https://docs.vllm.ai/en/latest/features/structured_outputs.html
  - title: "Outlines README and quickstart"
    url: https://github.com/dottxt-ai/outlines
  - title: "OpenAI Structured Outputs guide (Responses API text.format, responses.parse)"
    url: https://developers.openai.com/api/docs/guides/structured-outputs
  - title: "Anthropic structured outputs (output_config.format, GA; output_format deprecated)"
    url: https://platform.claude.com/docs/en/build-with-claude/structured-outputs
  - title: "Gemini structured output (Interactions API response_format)"
    url: https://aistudio.google.com/docs/structured-output
  - title: "Instructor"
    url: https://github.com/567-labs/instructor
last_verified: "2026-11-11"
---

# 02.2 · Structured Outputs and Schema-Constrained Generation

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

Guarantee: total, for shape. Available on self-hosted stacks (vLLM's `structured_outputs`, Outlines, llama.cpp grammars) and, as of 2026, on all three major hosted APIs through their native structured-output modes (Key Concept 5). This is the strongest rung, and the reason your Module 10 self-hosted deployment gains a capability rather than losing one.

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

## Key Concept 5: The tools in 2026, and which one to reach for

*Versions and parameter names checked 2026-11-11.* Three families of tools, matching the rungs above.

**Constrained decoding — when you run the model.** The schema is compiled into a grammar and the sampler cannot leave it.

- **vLLM 0.33.** On the OpenAI-compatible server, send the standard `response_format` with a JSON schema; vLLM-specific constraints (choice, regex, grammar) go in `extra_body={"structured_outputs": {...}}`. Offline, pass `SamplingParams(structured_outputs=StructuredOutputsParams(json=schema))`. The backend is chosen with `--structured-outputs-config.backend` (`auto`, `xgrammar`, or `guidance`; `auto` is the default). **The old `guided_json`, `guided_choice`, `guided_regex`, and `guided_grammar` parameters were removed in vLLM 0.12** — tutorials that use them no longer run.
- **Outlines 1.3.** A library that applies the same idea to a model you load yourself: `model = outlines.from_transformers(...)`, then `model(prompt, Ticket)` returns output that matches the Pydantic type.

```python
# vLLM 0.33 OpenAI-compatible server — standard field, enforced by constrained decoding
resp = client.chat.completions.create(
    model=MODEL,
    messages=messages,
    response_format={"type": "json_schema",
                     "json_schema": {"name": "ticket", "schema": Ticket.model_json_schema()}},
)
```

**Native structured outputs — when a provider runs the model.** You cannot touch the sampler, so you ask the provider to enforce the schema:

| Provider | Parameter (2026-11-11) | Pydantic helper |
|---|---|---|
| OpenAI (Responses API) | `text={"format": {"type": "json_schema", "name": ..., "schema": ..., "strict": True}}` | `client.responses.parse(..., text_format=Ticket)` → `response.output_parsed` |
| Anthropic (GA) | `output_config={"format": {"type": "json_schema", "schema": ...}}` | `client.messages.parse(..., output_format=Ticket)` → `response.parsed_output` |
| Google Gemini (Interactions API) | `client.interactions.create(..., response_format={"type": "text", "mime_type": "application/json", "schema": ...})` | parse `interaction.output_text` with `Ticket.model_validate_json` |

Two traps in older material: Anthropic's raw-request **`output_format` parameter is deprecated** (it now needs a beta header and returns 400 without it — use `output_config.format`; the `output_format=` argument of the SDK's `parse()` helper is a different thing and is fine), and Gemini examples built on `generationConfig.responseSchema` describe the older `generateContent` path rather than the current Interactions API.

**Validate-and-retry — for rules a schema cannot express.** "`end_date` is after `start_date`", "the cited chunk ID exists", "the total equals the sum of the lines" are not JSON Schema. A library such as **Instructor 1.17** validates the output against your Pydantic validators and re-asks with the error. Lesson 2.3 covers how to bound that loop.

**The decision rule:**

- **You control the inference stack** (vLLM, a model you load) → **constrained decoding**.
- **You call a hosted API** → **the provider's native structured outputs first**, then **validate-and-retry only for the semantic rules** the schema cannot express.
- Either way, the capability ladder from Key Concept 2 stays: declare which mechanism each profile supports, and log when you fall back.

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

**5. Wire the current mechanisms into rung 3 (30 minutes).** For your hosted profile, implement `_constrained()` with that provider's native structured outputs from Key Concept 5 (pin the SDK version you used in `requirements.txt`). For a self-hosted profile, use vLLM 0.33's `response_format` or Outlines 1.3. Then search your code and any snippets you copied for `guided_`, a raw `output_format=` request field, or `responseSchema` and replace them. **You know it worked when** the Answer contract above returns zero format failures over 100 records on rung 3 for every profile that declares it, and nothing in your repository still uses a removed or deprecated parameter.

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
- **Pick the mechanism by who runs the model.** Your own stack → constrained decoding (vLLM `structured_outputs`, Outlines); a hosted API → native structured outputs first, validate-and-retry only for semantic rules.

---

## Sources

Versions and parameter names checked 2026-11-11.

- vLLM structured outputs (`structured_outputs`, `StructuredOutputsParams`, backends, `guided_*` removed in v0.12.0): https://docs.vllm.ai/en/latest/features/structured_outputs.html — checked 2026-11-11 (vLLM 0.33.0)
- Outlines README and quickstart: https://github.com/dottxt-ai/outlines — checked 2026-11-11 (Outlines 1.3.3)
- OpenAI Structured Outputs guide (Responses API `text.format`, `responses.parse`): https://developers.openai.com/api/docs/guides/structured-outputs — checked 2026-11-11
- Anthropic structured outputs (`output_config.format`, GA; `output_format` deprecated): https://platform.claude.com/docs/en/build-with-claude/structured-outputs — checked 2026-11-11
- Gemini structured output (Interactions API `response_format`): https://aistudio.google.com/docs/structured-output — checked 2026-11-11
- Instructor: https://github.com/567-labs/instructor — checked 2026-11-11 (Instructor 1.17.0)

---

## Next

You are now producing valid, typed output at the strongest level each backend supports. On the calls where the strongest level is still not available — or where the content itself makes compliance impossible — you will get an invalid object anyway.

**Lesson 2.3** covers what happens next: the validate-repair-retry loop, why the natural instinct to retry until it works is a cost incident wearing a helpful disguise, and how to pick a repair budget from your own data rather than from a blog post.
