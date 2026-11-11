# Module 1 practical — Call telemetry and a request-level baseline

This practical turns the Lesson 1.4 instrumentation into a working baseline for your spine project. You extend the `LLMClient` from Lesson 1.2 so that every model call leaves a structured record, run real traffic through two backends, and summarize it at the level that matters: the user-facing request, not the single call. The table you produce is the first thing every later module compares against.

## Task

Extend your `LLMClient` so every call emits a `CallRecord` containing `request_id`, `call_site`, tokens in and out, cost, TTFT, total latency and `finish_reason`. Run 50 requests across two backends and produce a summary table with calls per request, p50/p95 request latency, mean cost per request and total spend.

## Deliverable

`telemetry/calls.jsonl` plus the summary table, and one paragraph on anything in the numbers that surprised you.

## How it is assessed

| Criterion | Full credit |
|---|---|
| Record completeness | All fields present, including `request_id` and `call_site` |
| Request-level grouping | Metrics grouped by `request_id`, not just per call |
| Percentiles | p50 and p95 reported, not only means |
| Cost computation | From a config price table, not hardcoded |
| Reflection | Identifies something concrete — e.g. token counts differing across backends for identical input, or TTFT/total latency diverging |

**Common failure:** reporting per-call metrics only. If calls per request is not in the table, the main point of the exercise has been missed.
