---
id: "01.4"
module: 1
minutes: 12
practice_minutes: 60
prerequisites: ["01.3"]
objectives:
  - "Record a minimum telemetry record (request_id, attempt, tokens, cost, latency) for every model call, and aggregate cost and latency per user request rather than per call."
volatility: implementation
sources: []
last_verified: "2026-08-24"
---

# 01.4 · Instrumenting Cost and Latency From Day One

There is a predictable moment in the life of every LLM project. Somewhere around week eight, someone from finance asks why the bill tripled. The team looks at the code, sees nothing obviously wrong, and starts guessing. Was it the new prompt? The retrieval change? More users? A retry loop? Nobody can tell, because the system logs "request completed" and nothing else.

That moment is entirely preventable, and preventing it costs about forty lines of code written now rather than a week of archaeology written later. This lesson builds those forty lines. One of the fields in them — a correlation ID that does nothing useful today — is the reason your Module 10 traces will be readable at all.

---

## Key Concept 1: Why this is scaffolding, not polish

Instrumentation gets deferred because it feels like an operational concern for later. For LLM systems that instinct is wrong, for three specific reasons.

**Cost is emergent, not designed.** Nobody decides "this feature will cost $1,400 a month." It emerges from a prompt someone lengthened, a retrieval-k someone raised from 5 to 20, and a repair loop someone added. Each change was individually reasonable. Without per-change attribution, the total is unexplainable.

**The feedback loop is otherwise absent.** In ordinary software, a slow function shows up in a profiler. In an LLM system, a prompt that is 30% more expensive looks and behaves identically to one that is not. There is no natural signal.

**Your telemetry becomes training data.** In Module 8 you will build a fine-tuning dataset from production traces. In Module 10 you will sample production traffic for online evaluation. Both require that you have been recording inputs, outputs, and outcomes all along. Telemetry started in Module 1 is a dataset by Module 8; telemetry started in Module 8 is a plan.

---

## Key Concept 2: The minimum viable record

Every model call should emit one structured record. Here is the full field list, with the reason each one exists:

```python
@dataclass
class CallRecord:
    # identity
    request_id: str        # THE correlation ID — one per user-facing request
    call_id: str           # unique per model call
    timestamp: str

    # what was called
    profile: str           # frontier / hf / local — from Lesson 1.2
    model: str
    prompt_version: str    # populated in Module 2

    # where from
    call_site: str         # "rag.answer", "agent.plan", "eval.judge"

    # the numbers
    tokens_in: int
    tokens_out: int
    cost_usd: float
    ttft_ms: float
    total_ms: float

    # what happened
    finish_reason: str     # stop / length / tool_calls / error
    error: str | None
    attempt: int           # 1 for first try — matters in Module 2
```

Two fields deserve specific attention.

**`request_id` is the whole game.** It is a single ID generated at the entry point of a user-facing request and threaded through every model call that request causes. Today, with one call per request, it looks pointless. By Module 6 one request will produce fifteen calls, and this field is the only thing that groups them. In Module 10 it becomes your **trace ID**, and the entire observability story depends on it existing from the beginning. Add it now.

**`call_site` is what makes aggregation actionable.** "Total spend: $840" tells you nothing. "`agent.plan`: $610, `rag.answer`: $180, `eval.judge`: $50" tells you exactly where to look.

Write these as JSONL to a local file for now. That is genuinely sufficient through Module 9. Module 10 swaps the sink for a real tracing backend without changing the shape.

---

## Key Concept 3: Per-call is the wrong unit

Here is the trap that produces the week-eight surprise.

You measure a call: 800 input tokens, 150 output tokens, $0.0031. Reasonable. You reason about your system in those terms. Then you ship, and the bill does not match your model — because the unit that matters to a user, and to finance, is the **request**, and one request is many calls.

By the end of this course, a single user question in your system will trigger:

- 1 query rewrite (Module 5)
- 3 retrieval embeddings (Module 5)
- 1 reranking pass (Module 5)
- 4–8 agent steps, each a model call (Module 6)
- 1 output validation, sometimes a repair (Module 2)
- 1 sampled evaluation judge call (Module 10)

Twelve to fifteen calls. Each one "reasonable." Together, an order of magnitude more than your mental model.

So define your metrics at the request level from the start:

- **Cost per request** — sum of `cost_usd` grouped by `request_id`
- **Calls per request** — the count, which is the number that surprises people
- **End-to-end latency** — first call start to last call end, not the sum of individual latencies
- **Cost per *successful* request** — cost divided by requests that produced a usable answer

That last one is the metric that survives to the end of this course. A system with a 60% success rate and $0.02 per call costs far more per useful answer than the arithmetic suggests, and it is the number a hiring manager will ask you for in Module 12's deep-dive.

---

## Worked Example: When the two units disagree

A team instruments an extraction endpoint and reports healthy numbers:

```
Calls: 2,914
Mean cost per call: $0.0021
Mean latency per call: 1.9 s
```

Then they group by `request_id`:

```
Requests: 1,000
Mean calls per request: 2.9        ← expected 1.0
Mean cost per request: $0.0061     ← 3× the per-call figure
p95 cost per request: $0.0294      ← 14× the per-call figure
```

The `attempt` field explains it immediately:

```
attempt=1: 1,000 calls   (100% of requests)
attempt=2:   704 calls   (70% of requests needed a repair)
attempt=3:   612 calls
attempt=4:   598 calls   ← hitting the retry ceiling and failing anyway
```

Seventy percent of requests are failing validation and entering a repair loop, and roughly 60% are burning the full retry budget and *still* failing. The per-call metrics were flawless. The system was broken.

This is the exact scenario Module 2 addresses, and it is only visible because `request_id` and `attempt` were recorded. Without them, the team sees "cost is a bit higher than expected" and moves on.

---

## Practice

**1. Add the record.** Implement `CallRecord` and a decorator or context manager that emits one per model call from your Lesson 1.2 client. Write JSONL to `telemetry/calls.jsonl`.

```python
@instrumented(call_site="demo.summarize")
def summarize(text: str) -> str:
    return client.complete([...]).text
```

**2. Thread the request ID.** Generate one `request_id` at your entry point and make it available to every call in that request — a `contextvars.ContextVar` is the cleanest approach in async Python and will still work when Module 10 makes everything concurrent. Do not pass it as an argument through ten layers.

**3. Generate a baseline.** Run 50 requests across two backends with a mix of prompt sizes. Then write a small aggregation script producing:

| Metric | Frontier | Local |
|---|---|---|
| Requests | | |
| Calls per request | | |
| p50 / p95 latency (request) | | |
| Mean cost per request | | |
| Total spend | | |

**4. Commit it.** Save `telemetry/calls.jsonl` and the summary as the first artifact of your spine project, and add a `make baseline` target that regenerates the table. Every module from here on will compare against something, and this is the first something.

---

## Visual Aid

*An annotated JSONL record, pretty-printed, with a callout line from each field to the module that consumes it. `request_id` → "Module 6: groups an agent's 15 calls / Module 10: becomes the trace ID." `prompt_version` → "Module 2: attributes a cost regression to a commit." `attempt` → "Module 2: reveals repair-loop cost." `call_site` → "Module 10: locates the expensive stage." `tokens_in` / `cost_usd` → "Module 8: the cost case for fine-tuning." Caption: "None of these fields are useful today. All of them are load-bearing by Module 10."*

---

## Key Takeaways

- **Instrument before you build.** LLM cost is emergent from many small reasonable decisions, and without per-change attribution it becomes unexplainable.
- **One structured record per model call**, with identity, configuration, numbers, and outcome. JSONL to a local file is genuinely enough through Module 9.
- **`request_id` is the most important field.** It groups an agent's many calls today and becomes your distributed trace ID in Module 10. Adding it later means rewriting everything that touches it.
- **Measure per request, not per call.** One user question becomes a dozen model calls by the end of this course; per-call metrics will look healthy while the real bill does not.
- **Cost per *successful* request is the metric that lasts** — and the one you will be asked for in an interview.

---

## Next

You can call any model through one interface, you know what each call costs in tokens, time, and money, and you record all of it in a form you can aggregate. That is the entire foundation of Module 1, and every remaining module builds directly on it.

But there is a gap sitting between the model and your application that none of this addresses. The model returns a paragraph of text. Your code needs a typed object with fields you can rely on. In between is where most LLM projects quietly break — and where an unbounded repair loop like the one in this lesson's worked example comes from.

**Module 2** closes that gap: typed contracts, schema-constrained generation, bounded repair, and prompts treated as the versioned production config they actually are.
