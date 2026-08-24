# Module 1 Quiz — The AI Engineer Role & a Provider-Agnostic Baseline

**8 questions · ~25 min plus the practical**
Weighted toward application and diagnosis. Answer key with explanations follows the questions.

---

## Q1 — Multiple choice

A posting reads: *"MS/PhD preferred. Deep understanding of learning-to-rank. Experience with distributed training and feature stores. Publications a plus."*

Should a software engineer transitioning into AI engineering apply?

A. Yes — most AI roles list a degree preference they do not enforce
B. No — this is an ML engineering role; the requirements are model science, not systems work
C. Yes, but only after completing the fine-tuning modules
D. Cannot tell without knowing the company

---

## Q2 — Short answer

You migrate a service from provider A to provider B. Both speak the OpenAI-compatible API. Name **three** things that can break despite the interface being identical, and say what each breaks.

---

## Q3 — Diagnosis

A team reports these numbers for an extraction endpoint:

```
Calls: 4,102          Mean cost per call: $0.0021
Requests: 1,000       Mean cost per request: $0.0086
                      p95 cost per request: $0.0294
```

What is happening, which telemetry field would confirm it in one query, and what is the likely root cause?

---

## Q4 — Multiple choice

You need to cut the p95 latency of a summarization endpoint. Input is ~4,000 tokens; output is ~800 tokens. Which change most reduces latency?

A. Reduce input to 2,000 tokens by trimming context
B. Constrain output to ~200 tokens
C. Switch from streaming to a buffered response
D. Set `temperature=0` for determinism

---

## Q5 — True/false with justification

*"Setting `temperature=0` makes model output deterministic, so you can write exact-match assertions in your test suite."*

True or false, and why? Name two mechanisms responsible.

---

## Q6 — Design

Your `LLMClient` supports three backends. One supports prompt caching; two do not. A developer calls `client.complete(messages, cache_prefix=True)` against a backend without it.

Describe the correct behavior and explain why the alternative is worse, in terms of what the team would observe.

---

## Q7 — Applied judgment

You benchmark one prompt across three backends:

| Backend | Cost/request | p95 latency | Quality (rough) |
|---|---|---|---|
| Frontier | $0.0180 | 2.1 s | excellent |
| Hosted 8B | $0.0021 | 1.4 s | good |
| Local 7B (Docker) | $0.0000* | 3.8 s | acceptable |

*\*excluding hardware*

For each of these, recommend a backend and give one sentence of reasoning:
(a) an interactive customer-facing assistant at 50k requests/day
(b) a nightly batch job classifying 2M documents
(c) a prototype being demoed to stakeholders next week

---

## Q8 — Practical

Extend your `LLMClient` so every call emits a `CallRecord` containing `request_id`, `call_site`, tokens in/out, cost, TTFT, total latency, and `finish_reason`. Run 50 requests across two backends and produce a summary table with calls per request, p50/p95 request latency, mean cost per request, and total spend.

**Deliverable:** `telemetry/calls.jsonl` plus the summary table, and one paragraph on anything in the numbers that surprised you.

---
---

# Answer Key

## Q1 — **B**

Every requirement is model science: learning-to-rank algorithms, distributed *training*, feature stores, publications. None is systems integration. This is an ML engineering role.

Recognizing the distinction quickly is a practical skill — applying to roles that require years of research background wastes effort and produces discouraging screens. The signal to look for: does the posting ask you to *train* models or to *integrate* them?

## Q2 — Any three of:

- **Tokenization differs**, so the same text is a different token count → cost model and context budgeting are both wrong
- **Prompt caching semantics differ or are absent** → cost and TTFT change, silently
- **Structured-output enforcement differs** (strict schema vs tool-shaping vs prompt-only) → format failure rate can jump substantially
- **System prompt weighting differs** → instruction-following behavior changes
- **Tool-calling quality differs** → selection accuracy drops even though the API works
- **Context window size differs** → requests that fit before now fail
- **Safety filtering differs** → hosted providers filter; self-hosted models do not
- **Reasoning/thinking parameters are provider-specific** → silently ignored elsewhere

**Full credit** requires naming the *consequence*, not just the difference. "Tokenization differs" alone is half an answer.

## Q3

**What is happening:** each user request is producing ~4.1 model calls when it should produce one, and the p95 request is 14× the mean per-call cost.

**The field:** `attempt`. One query grouping by it shows the distribution.

**Likely root cause:** an unbounded validate-repair-retry loop. Roughly 70% of requests fail validation on the first attempt and retry, with many exhausting the retry budget and failing anyway.

The teaching point: **per-call metrics looked perfectly healthy.** Only the per-request view exposes it, which is why `request_id` must exist from the first module.

## Q4 — **B**

Output tokens are generated serially (decode), one forward pass each. Input tokens are processed in parallel (prefill). Cutting 800 output tokens to 200 removes 600 sequential generation steps.

Why the others are wrong:
- **A** halves prefill, which is parallel and fast — a real but much smaller gain, and it may cost quality
- **C** makes perceived latency *worse* by removing TTFT benefit; total latency is unchanged
- **D** does not affect latency at all

## Q5 — **False**

`temperature=0` selects the highest-probability token at each step, which is not the same as determinism.

Two mechanisms (any two):
- **Floating-point non-associativity in GPU kernels** — results vary with batch composition
- **Continuous batching at serving time** — your request is computed alongside whichever others arrived, changing the arithmetic
- **Model version updates behind a stable alias**
- **Mixture-of-experts routing** in some architectures

Consequence: evaluation must use tolerance bands and semantic comparison, not exact-match assertions. This is why Module 3 gates on measured noise rather than exact values.

## Q6

**Correct behavior:** raise `UnsupportedCapability` immediately, naming the profile and the capability.

**Why the alternative is worse:** most SDKs accept an unknown keyword and the server ignores it. The call succeeds, the answer looks correct, and nothing anywhere reports a problem. The team observes: cost higher than their model predicts, TTFT higher than expected, and no error to investigate. They will look at the model, the prompt, and the network before they look at a parameter that silently did nothing.

**The general principle:** a silently degraded capability is more expensive to debug than a loud failure, because there is no signal pointing at it.

## Q7

**(a) Interactive assistant, 50k/day → Hosted 8B, with frontier fallback.**
1.4 s p95 fits interactive use, cost is $105/day rather than $900/day, and quality is adequate for most traffic. Route hard queries to frontier.

**(b) Nightly batch, 2M documents → Local.**
Latency is irrelevant for an overnight job, and 2M documents at even $0.0021 each is $4,200 per run. This is exactly the volume where self-hosting wins.

**(c) Prototype demo next week → Frontier.**
Quality matters most, volume is trivial, and engineering time is the scarce resource. Optimizing cost on a prototype is premature.

**Full credit** requires the reasoning to reference the constraint that dominates each case — latency and per-request cost, total volume, and time-to-demo respectively.

## Q8 — Practical rubric

| Criterion | Full credit |
|---|---|
| Record completeness | All fields present, including `request_id` and `call_site` |
| Request-level grouping | Metrics grouped by `request_id`, not just per call |
| Percentiles | p50 and p95 reported, not only means |
| Cost computation | From a config price table, not hardcoded |
| Reflection | Identifies something concrete — e.g. token counts differing across backends for identical input, or TTFT/total latency diverging |

**Common failure:** reporting per-call metrics only. If calls-per-request is not in the table, the exercise's main point has been missed.
