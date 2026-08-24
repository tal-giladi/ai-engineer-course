# Lesson 2.4 — Prompts as Versioned Artifacts (and How Caching Changes the Math)

**Module 2: Structured Output & Prompting as Engineering** · ~13 min
**Objective:** M2-LO3 — Treat prompts as versioned artifacts — diffable, testable, cacheable — and explain when prompt caching changes cost and latency.

---

## Introduction

Somewhere in your codebase right now there is a triple-quoted string that determines how your system behaves. It has been edited six times. Nobody reviewed the edits. There is no record of which version produced last week's results, and no way to roll back to the version that worked without reading git blame and guessing.

If that string were a database connection pool size, a feature flag, or a retry policy, your team would have opinions about it. It is more impactful than any of those, and it is being managed worse than all of them.

This lesson fixes that, and then shows how the same structural discipline — knowing exactly what your prompt contains and in what order — unlocks a cost and latency optimization that is essentially free.

---

## Key Concept 1: A prompt is production configuration

Prompts have every property that makes something deserve version control:

- **They change system behavior.** More than most config values.
- **They are edited frequently**, often by more than one person.
- **They regress.** An edit that fixes one case breaks another.
- **They need rollback**, and you need to know what you are rolling back *to*.
- **They should be reviewed**, because a one-word change can alter refusal behavior across your whole traffic.

The move is mechanical. Prompts go in files:

```
spine/prompts/
  answer_v3.md
  repair_v1.md
  judge_pairwise_v2.md
```

Each file carries a header:

```markdown
---
id: answer
version: 3
model_family: any
changed: "Added explicit refusal instruction for out-of-scope questions"
supersedes: 2
---
You are answering questions about {{product}} documentation...
```

Three properties this buys immediately.

**Diffable.** A prompt change shows up in review as a diff, not as a modified string literal buried in a 400-line service file.

**Attributable.** The version ID goes into `CallRecord` — the `prompt_version` field you added in Lesson 1.4 and have not used yet. Now a cost or quality change can be traced to a specific prompt version, which is the difference between "costs went up in March" and "costs went up when answer_v3 shipped on March 12."

**Testable.** In Module 3 you will run evaluations against specific prompt versions and gate merges on the result. That requires a version identity to exist.

---

## Key Concept 2: One change per evaluation

A discipline that costs nothing and saves weeks: **change one thing between evaluation runs.**

The temptation is to fix three things at once — clarify the instruction, add an example, tighten the output format. You run the eval, the score moves up four points, and you have learned nothing. Which change helped? Did one of them hurt while the others carried it? You cannot tell, and you will keep carrying whichever one is dead weight forever.

This matters more than in ordinary software because the effects are non-monotonic and interacting. Adding a few-shot example can *lower* quality if the example is unrepresentative. A stricter instruction can raise format compliance and lower answer quality simultaneously. There is no compiler to tell you.

Version each change. Evaluate each version. Keep what wins.

---

## Key Concept 3: Templates with typed variables

A prompt file is a template, and its variables deserve the same treatment as function parameters:

```python
class AnswerPromptVars(BaseModel):
    product: str
    context_chunks: list[Chunk]
    question: str
    conversation_summary: str | None = None

prompt = render("answer", version=3, vars=AnswerPromptVars(...))
```

Why bother: a missing variable in an f-string produces `None` rendered into your prompt as the literal text "None", which the model then reasons about. Typed variables turn that into a `ValidationError` before the call. It is the Lesson 2.1 principle applied to the other direction of the boundary — you validate what comes *out* of the model, so validate what goes *in* too.

Also: **never build prompts by string concatenation across files.** Every fragment that gets appended somewhere is a fragment nobody can see when reading the prompt. If your prompt is assembled from four places, its effective content exists nowhere and cannot be reviewed.

---

## Key Concept 4: Prefix caching, and why order matters

Now the payoff for knowing exactly what your prompt contains.

Providers cache the processed representation of a prompt **prefix**. If two requests share a leading segment, the second can reuse the first's prefill work — which means substantially cheaper input tokens on the cached portion and a lower TTFT, because prefill is where TTFT comes from (Lesson 1.3).

The requirement is exact: the cached portion must be a **byte-identical prefix**. One character different at position 12 and everything after it is a cache miss.

This gives you a single, actionable rule:

> **Order your prompt from most stable to least stable.**

```
1. System instructions          ← identical every request
2. Tool definitions             ← identical every request
3. Few-shot examples            ← identical every request
--- cacheable prefix ends here ---
4. Retrieved context            ← varies per request
5. Conversation history         ← varies per request
6. The user's question          ← varies per request
```

The common anti-pattern is putting the user's name, the current date, or a request ID at the top of the system prompt. That single variable token at position 20 destroys the cache for everything below it. If you need the date, put it after the stable block.

**The portability catch.** This is a Lesson 1.2 tier-three feature. Cache mechanisms differ — some providers cache automatically, some require an explicit marker, minimum cacheable lengths differ, TTLs differ, and self-hosted vLLM has its own prefix-caching behavior that is not the same thing. So:

- The *ordering* discipline is universal and costs nothing. Do it everywhere.
- The *mechanism* lives behind your capability seam, and requesting it on a backend that lacks it must raise (Lesson 1.2's rule).

---

## Worked Example 1: Attribution in practice

A team sees monthly spend rise 40%. With `prompt_version` in telemetry:

```sql
SELECT prompt_version, COUNT(*), AVG(tokens_in), AVG(cost_usd)
FROM calls WHERE call_site = 'rag.answer' GROUP BY prompt_version;
```

| prompt_version | calls | avg tokens_in | avg cost |
|---|---|---|---|
| answer:2 | 41,203 | 1,840 | $0.0061 |
| answer:3 | 38,910 | 3,110 | $0.0102 |

Version 3 added four few-shot examples. They improved quality by two points and raised input tokens by 69%. That is now a *decision* — is two points worth 69%? — rather than a mystery. Without the version field, this investigation does not start.

## Worked Example 2: The reorder, measured

Same prompt, same content, two layouts. 1,000 requests each.

**Cache-hostile** — session ID and timestamp first, then instructions, then context:

| Backend | Cache hit rate | Avg input cost | p50 TTFT |
|---|---|---|---|
| Frontier | 0% | $0.0094 | 780 ms |
| Local vLLM | 0% | n/a | 610 ms |

**Cache-friendly** — instructions, tools, and examples first; variable content last:

| Backend | Cache hit rate | Avg input cost | p50 TTFT |
|---|---|---|---|
| Frontier | 82% | $0.0036 | 310 ms |
| Local vLLM | 71% | n/a | 340 ms |

Input cost down 62%, TTFT down 60%. No content changed — only order.

And the honest third case, a backend with no prefix caching at all:

| HF endpoint | 0% | $0.0094 | 760 ms |

Identical before and after. Which is exactly why the mechanism sits behind a capability check and the *ordering* is done unconditionally: the reorder never hurts, and on two backends out of three it is one of the largest wins available for zero quality risk.

---

## Practice

**1. Extract every prompt.** Move all prompts from Modules 1 and 2 into `spine/prompts/` with versioned filenames and frontmatter headers. Include the repair prompt from Lesson 2.3 — that one is invisible and matters.

**2. Build the renderer.** Implement `render(id, version, vars)` with a Pydantic model per prompt for its variables. Make a missing or mistyped variable raise.

**3. Stamp the version.** Populate `prompt_version` in every `CallRecord`. Verify by running 20 calls and grouping your telemetry by version.

**4. Reorder and measure.** Take your largest prompt, identify the stable prefix and the variable tail, reorder so the stable part leads. Run 200 requests before and after on each backend and report cache hit rate, input cost, and p50 TTFT. Expect one backend to show nothing — record that too.

**5. Guard the capability.** Confirm that requesting explicit caching on a backend without it raises `UnsupportedCapability` rather than silently proceeding.

---

## Visual Aid

*Two vertical prompt layouts side by side. Left, "cache-hostile": a thin band at top labeled `session_id, timestamp` shaded red, then large blocks for system instructions, tools, and examples, then context and question. A bracket down the entire left edge is labeled "cacheable: 0 tokens." Right, "cache-friendly": system instructions, tools, and examples at top shaded green with a bracket labeled "cacheable prefix: ~2,400 tokens," then a dashed line marked "cache boundary," then the variable content below. Caption: "Same tokens. Same content. One red band at the top is the whole difference."*

---

## Key Takeaways

- **Prompts are production configuration** — versioned files with headers, reviewed as diffs, and rollable back.
- **Stamp the version into telemetry.** It converts "costs rose in March" into "costs rose when answer:3 shipped," which is the difference between a decision and a mystery.
- **Change one thing per evaluation run.** Effects are non-monotonic and interacting; batched changes teach you nothing.
- **Type your template variables.** A missing f-string variable renders as literal "None" into the model's context.
- **Order prompts stable-to-variable.** The ordering discipline is free and universal; the caching mechanism is provider-specific and belongs behind the capability seam. On backends that support it, expect large cuts to input cost and TTFT for zero quality risk.

---

## Next

You have typed contracts, bounded repair, and versioned prompts, all instrumented. That is a genuinely solid foundation — and you still have no idea whether any of it produces good answers.

Everything so far has been about *shape*: valid objects, correct fields, sensible costs. Not one thing you have built can tell you whether the content is right. Every improvement you have made, you have asserted rather than proven.

**Module 3** is where that changes, and it is the module that hiring managers use to separate people who have shipped LLM systems from people who have read about them. You will build a golden dataset, choose metrics that fit your task, calibrate an LLM judge against human labels, and put a regression gate in CI — before you build anything worth evaluating, so that everything you build afterward is measured.
