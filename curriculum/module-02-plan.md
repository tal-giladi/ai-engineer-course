# Module 2 Plan — Structured Output & Prompting as Engineering

**Duration:** 50 min · **Lessons:** 4 · **Depends on:** Module 1
**Outline reference:** `curriculum/course-outline.md` §8, Module 2

---

## Prerequisites

**From Module 1:** a working `LLMClient` with telemetry (`spine/llm_client.py`).
**From the student:** Pydantic or equivalent schema-validation experience, comfort with JSON Schema, familiarity with retry and idempotency patterns from ordinary distributed systems work.

---

## Module Learning Objectives

**M2-LO1** — Define a typed contract between model and application using JSON Schema and Pydantic, and enforce it with schema-constrained generation.
**M2-LO2** — Implement a validate-repair-retry loop with an explicit failure budget, and decide when to fail loudly instead of retrying.
**M2-LO3** — Treat prompts as versioned artifacts — diffable, testable, cacheable — and explain when prompt caching changes cost and latency.

---

## Lesson 2.1 — Why Free-Form Text Is a Bad API

*11 min · serves M2-LO1*

**Concept.** The boundary problem: a model returns a string, an application needs a value. Parsing prose with regex is the single most common source of silent failure in early LLM systems, because the model's output distribution drifts with every prompt edit, model version, and unusual input — and regex fails *quietly*, producing a plausible wrong value rather than an exception. Reframe the model as an untrusted upstream service: you would not accept unvalidated JSON from a third-party API, and the same rule applies here with more force, because this upstream is non-deterministic. Introduce the contract mindset — define the shape first, then make the model fill it.

**Example.** A sentiment classifier that works on 95 of 100 inputs and fails on the five where the model prefixed its answer with "Sure! Here's the classification:". Walk the regex fix, then the second failure mode it introduces, then the third — demonstrating that patching the parser is a losing game.

**Practice.** Students take a supplied prose-parsing extractor with a hidden 8% failure rate, run it over the provided input set, and catalogue every distinct failure shape they find. No fixing yet — the point is to feel the surface area.

**Summary.** The model is an untrusted, non-deterministic upstream. Parse defensively or, better, do not parse at all — constrain the output shape instead.

**Key terms:** output contract, brittle parsing, silent failure, untrusted upstream.
**Visual aid:** A "failure gallery" — one prompt, eight real output variants, each annotated with which parser assumption it breaks.
**Hook:** "You catalogued the ways it breaks. Now stop parsing and start constraining."

---

## Lesson 2.2 — Structured Outputs and Schema-Constrained Generation

*13 min · serves M2-LO1*

**Concept.** Three mechanisms, in increasing order of guarantee: prompt-and-hope (ask for JSON in the prompt), tool/function-call shaping (declare a schema the provider validates against), and constrained decoding (the sampler is restricted to tokens that keep the output grammatically valid). Which are available depends on backend — this is exactly the portability seam from Lesson 1.2, so the module builds a capability-detecting abstraction rather than pretending all backends are equal. Cover schema design as its own skill: flat beats nested, enums beat free strings, required-plus-nullable beats optional, and field descriptions are prompt real estate the model actually reads. Cover the honest limit: a valid schema guarantees *shape*, never *correctness*.

**Example.** The same extraction task implemented all three ways against the Module 1 backends, with a table of format-failure rates per approach per backend. Then a schema-design before/after: a deeply nested schema at 71% field accuracy, flattened and enum-constrained to 94%, with no prompt change.

**Practice.** Students define a Pydantic model for the spine project's answer contract — answer text, citation list, confidence, refusal flag — and implement it with the strongest mechanism each backend supports, recording the fallback ladder.

**Summary.** Constrain at the strongest level the backend allows, degrade explicitly, and design the schema as carefully as the prompt. Valid shape is not correct content — that gap is Module 3's job.

**Key terms:** structured output, JSON Schema, function/tool-call shaping, constrained decoding, grammar-constrained sampling, capability detection.
**Visual aid:** Ladder diagram of the three mechanisms — guarantee strength on one axis, backend availability on the other.
**Hook:** "Strongest available still isn't always. What happens on the call where it fails?"

---

## Lesson 2.3 — Validation, Repair Loops, and Failure Budgets

*13 min · serves M2-LO2*

**Concept.** The three-tier response to invalid output: **validate** (Pydantic parse), **repair** (return the validation error to the model and ask for a correction), **fail** (raise, with the failed payload preserved). The engineering content is in the third tier — teams instinctively retry forever, which converts a visible failure into an invisible cost explosion and a latency tail. Introduce the failure budget: a bounded repair count, a token ceiling on the repair path, and a distinction between *retryable* faults (malformed JSON, missing field) and *terminal* ones (the model refuses, the input is genuinely ambiguous, the schema is wrong). Connect directly to Module 1's per-request cost lesson — repair loops are the classic reason per-request cost diverges from per-call cost.

**Example.** A repair loop instrumented with Module 1's telemetry, showing that repair attempt 1 recovers 85% of failures, attempt 2 recovers 9%, and attempts 3+ recover almost nothing while consuming a third of the token budget — the empirical argument for a budget of two.

**Practice.** Students implement `validate_repair_call()` with a configurable budget, run it against an input set seeded with adversarial cases, and produce a table of recovery rate and marginal cost per attempt. They then set their own budget and justify the number.

**Summary.** Repair once or twice, then fail loudly with the payload attached. An unbounded repair loop is a cost incident that looks like a working system.

**Key terms:** validate-repair-retry, failure budget, retryable vs terminal fault, error feedback prompt, dead-letter payload.
**Visual aid:** Flowchart of the loop with the budget counter, the terminal-fault escape hatch, and the dead-letter path drawn explicitly.
**Hook:** "That loop's behavior depends entirely on the prompt inside it — a prompt nobody has version-controlled yet."

---

## Lesson 2.4 — Prompts as Versioned Artifacts (and How Caching Changes the Math)

*13 min · serves M2-LO3*

**Concept.** A prompt is production configuration that changes system behavior, so it needs what other production config gets: version control, review, a test that runs on change, and the ability to roll back. Cover the practical shape — prompts as files not string literals, template variables with explicit typing, a version identifier stamped into every `CallRecord`, and the discipline of one change per evaluation run. Then prompt caching: providers cache a stable prefix, so *ordering* the prompt so the invariant part comes first (system instructions, tool definitions, few-shot examples) before the variable part (the user's question, retrieved context) can cut cost and TTFT substantially. Note the portability catch flagged in Lesson 1.2: caching semantics and minimum cacheable lengths differ per backend, so this optimization lives behind the adapter seam.

**Example.** A prompt file with a version header, referenced by ID in telemetry, so a cost regression can be traced to the exact commit that caused it. Then the same prompt reordered for cache friendliness, with a before/after table of TTFT and input-token cost across two backends — including one where it makes no difference, and why.

**Practice.** Students move every prompt from Modules 1–2 into `spine/prompts/` with version headers, stamp the version into `CallRecord`, then reorder one prompt for cache friendliness and measure the change on their own backends.

**Summary.** Prompts are versioned config. Stamp the version into telemetry so behavior changes are attributable, order the prompt so the stable part comes first, and keep caching behind the adapter seam.

**Key terms:** prompt versioning, prompt template, prefix caching, cache-friendly ordering, config-as-code.
**Visual aid:** Two prompt layouts side by side — cache-hostile and cache-friendly — with the cacheable prefix shaded in each.
**Hook:** "You now have typed outputs, bounded repairs, and versioned prompts. You still have no idea whether any of it is any good. That's Module 3, and it's the module that separates people who have shipped from people who haven't."

---

## Spine Increment

**Deliverable:** `spine/contracts.py` and `spine/prompts/` — typed answer contract, validate-repair-retry with a justified budget, versioned prompt files stamped into telemetry.

**Acceptance criteria:**
1. The answer contract is enforced at the strongest mechanism each backend supports, with an explicit documented fallback ladder.
2. Repair is bounded; terminal faults raise with the failed payload preserved.
3. Every `CallRecord` carries a prompt version identifier.
4. The repair-attempt recovery table is committed as evidence for the chosen budget.

---

## Standalone Exercise

Build an extractor that pulls a strict schema from messy invoice text — mixed formats, missing fields, two languages, one scanned-OCR mess — and never returns an invalid object. Deliverable: `exercises/m2-invoice-extractor/` with the schema, the repair budget and its justification, and a failure log showing what it refused and why.

---

## Dependencies

| Lesson | Requires |
|---|---|
| 2.1 | Module 1 complete |
| 2.2 | 2.1 |
| 2.3 | 2.2, plus Lesson 1.4 telemetry |
| 2.4 | 2.3, plus Lesson 1.2 adapter seam |

**Downstream:** Module 3 evaluates these contracts. Module 5 extends the answer contract with citations and groundedness. Module 6 reuses the same schema-enforcement machinery for tool-call arguments — an agent's tool call is a structured output with consequences.

---

## Assessment Focus (Phase 4)

Diagnosis and design: given a repair-attempt cost table, choose and defend a budget; given a schema at 71% field accuracy, propose three design changes and predict which helps most; given two prompt layouts, identify the cacheable prefix and estimate the saving; distinguish a case where repair is correct from one where failing loudly is.
