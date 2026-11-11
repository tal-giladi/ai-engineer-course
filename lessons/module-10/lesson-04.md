---
id: "10.4"
module: 10
minutes: 12
practice_minutes: 90
prerequisites: ["10.3"]
objectives:
  - "Instrument end-to-end tracing and read an agent run's trace to find where time and money went."
volatility: implementation
sources: []
last_verified: "2026-08-24"
---

# 10.4 · Tracing: Seeing Inside an Agent Run

Back in Lesson 1.4 you added a field called `request_id` to a telemetry record, and the justification was a promise: it would matter later. This is later.

One user question in your system now produces a query rewrite, three embeddings, two retrievals, a rerank, four to eight agent steps each with a model call and a tool call, a validation call, and possibly a judge call. Twenty-plus operations, nested, some concurrent, spanning several services.

Logs cannot represent that. A log line is a point in time with no structural relationship to any other line. What you need is a **trace**, and the identifier that makes it possible has been in your code since the first module.

---

## Key Concept 1: Traces and spans

**A trace** is one user-facing request, end to end.
**A span** is one operation within it, with a start time, a duration, a parent, and attributes.

Spans nest, forming a tree that mirrors what actually happened:

```
trace: req_8f3a  (8,400 ms, $0.0094)
├─ retrieve                                   3,565 ms
│  ├─ query_rewrite            [model]          420 ms  $0.0002
│  ├─ embed (×3)               [embedding]      180 ms  $0.0000
│  ├─ vector_search            [pgvector]        40 ms
│  ├─ bm25_search              [postgres]        25 ms
│  └─ rerank                   [cross-encoder] 2,900 ms
├─ agent_loop                                 2,415 ms
│  ├─ step_1                   [model]          890 ms  $0.0021
│  ├─ tool: search_docs                         310 ms
│  ├─ step_2                   [model]          720 ms  $0.0024
│  └─ step_3                   [model]          495 ms  $0.0019
└─ validate_citations          [model]        2,420 ms  $0.0028
```

That tree *is* the budget decomposition from Lesson 10.3, produced automatically for every request rather than assembled by hand for a few.

**The LLM-specific span attributes** are what make this useful rather than generic:

```python
span.set_attributes({
    "llm.model": "gpt-x",
    "llm.profile": "frontier",
    "llm.tokens.input": 1_840,
    "llm.tokens.output": 210,
    "llm.cost_usd": 0.0031,
    "llm.prompt.version": "answer:v3",       # Lesson 2.4
    "llm.finish_reason": "tool_calls",
    "llm.attempt": 1,                         # Lesson 2.3
    "rag.chunks_retrieved": 5,
    "rag.chunk_ids": ["c_1194", "c_0031"],
    "agent.step": 2,
    "agent.tool": "search_docs",
    "agent.termination_reason": "completed",  # Lesson 6.4
})
```

Every one of these is a field you already record. Tracing changes where they go and how they relate, not what you collect — which is why this lesson is an afternoon rather than a week.

---

## Key Concept 2: The tooling

**OpenTelemetry** is the vendor-neutral standard: instrument once, export anywhere. Use it for the general structure — HTTP handlers, database calls, service boundaries.

**LLM-native tracers** (Langfuse, LangSmith, and peers) add what generic APM does not:

- Full prompt and response capture per span, so you can read what the model actually saw — including what a framework assembled for you (Lesson 7.1)
- Cost rollups by model, prompt version, user, and endpoint
- **Attaching eval scores to production traces**, which is Key Concept 3
- Capturing production examples into datasets with one click
- Framework integrations, so LangGraph nodes become spans automatically

Use both: OpenTelemetry for infrastructure, an LLM tracer for the model layer. Many LLM tracers accept OTel data, so this is usually one pipeline.

**Use `request_id` as the trace ID.** It is already your `CallRecord` correlation ID and your Module 7 checkpoint thread ID. One identifier now ties telemetry, traces, and checkpoints together — which means from a trace you can jump to the checkpoint and time-travel debug it (Lesson 7.2).

---

## Key Concept 3: Online evaluation closes the loop

This is the capability tracing unlocks that matters most, and it completes the three-surface model from Lesson 3.1.

Your offline eval measures 60 curated examples. Production sees inputs you never imagined. **Online evaluation** samples real traffic and scores it with the same calibrated judge from Lesson 3.4:

```python
if random() < SAMPLE_RATE:              # 5–10%
    await queue_online_eval(trace_id, question, answer, context)
```

The judge runs asynchronously, off the request path, and attaches scores to the trace.

Three things this gives you:

**The offline/online gap.** If offline faithfulness is 0.87 and online is 0.79, your golden set does not represent production. That gap is a finding, and closing it means adding the missing cases.

**Drift detection.** Input distributions change — new product features, new user segments, seasonal patterns. Watch online scores over time and watch the escalation rate of your Lesson 9.3 router, which is an early indicator.

**Dataset capture.** Low-scoring production traces are exactly the examples your golden set is missing. One click adds them. This is how the dataset stays honest over years, and it is the mechanism that makes the Lesson 3.2 claim — that the dataset outlives everything — actually true.

---

## Key Concept 4: Traces contain everything, including PII

A trace holds the user's question, the retrieved documents, the model's reasoning, the tool arguments, and the final answer. That is potentially the most sensitive store in your system, and it is the one nobody classifies.

Note it now, and handle it in Lesson 11.3:

- **Redact at capture**, not after storage
- **Set retention** — traces are usually not needed for years
- **Restrict access** like a production database, because that is what it is
- **Check your provider's terms** if using a hosted tracing service
- **Sample rather than capture everything** — 10% of traces answers most questions at a tenth of the exposure

---

## Worked Example 1: The repeated span

Production p95 is 2.4× the staging figure. Logs show nothing unusual. Traces show this:

```
trace: req_a419  (7,200 ms)
└─ agent_loop
   ├─ step_1              [model]     780 ms
   ├─ tool: search_docs             2,900 ms   ← 
   │  ├─ embed                         60 ms
   │  ├─ vector_search                 38 ms
   │  ├─ vector_search                 41 ms
   │  ├─ vector_search                 39 ms
   │  ├─ ... (×8 total)
   │  └─ rerank                     2,100 ms
```

Eight vector searches inside one tool call, and a rerank on the union of all eight.

The cause: the multi-query retrieval from Lesson 5.2 was generating three paraphrases, and a caching layer added later was keyed on the *original* query rather than the rewritten one. Every cache miss re-ran the full multi-query expansion, and a retry wrapper was re-invoking the whole tool.

Invisible in logs — every individual operation succeeded quickly. Obvious in a span tree in about four seconds.

## Worked Example 2: The offline/online gap

| | Offline (golden set) | Online (10% sample) | Gap |
|---|---|---|---|
| Faithfulness | 0.87 | **0.79** | −0.08 |
| Answer relevance | 0.89 | 0.85 | −0.04 |
| Refusal accuracy | 0.86 | **0.61** | −0.25 |

Refusal accuracy is 25 points worse in production. Filtering low-scoring traces and clustering the questions shows why: a large group asking about **pricing and billing**, which the corpus does not cover and the golden set's out-of-scope stratum never included — every out-of-scope example was a technical question.

The model was answering pricing questions from general knowledge, confidently.

**Two fixes, in order.** Immediately: a scope classifier for billing topics routing to a refusal. Structurally: 15 real pricing questions added to the golden set as a new out-of-scope sub-stratum, so the gap cannot silently reappear.

Two weeks later, online refusal accuracy 0.84. And the golden set is now permanently better because production taught it something.

**That loop — production surfaces a gap, the gap becomes test cases, the test cases prevent regression — is what a mature evaluation practice looks like**, and it is a strong thing to describe in an interview.

---

## Practice

**1. Instrument the stack.** OpenTelemetry for HTTP and database spans, an LLM tracer for model spans. Export to a local collector in Docker Compose.

**2. Use `request_id` as the trace ID.** Verify one agent run appears as a single trace with a correct span tree, and that the trace links to its Module 7 checkpoint thread.

**3. Add LLM attributes to every model span** — model, profile, tokens, cost, prompt version, finish reason, attempt, plus RAG and agent attributes. All of these already exist in your `CallRecord`.

**4. Read one full trace end to end.** Confirm the span durations reproduce your Lesson 10.3 budget decomposition. If they disagree, one of the two is wrong and finding out which is the exercise.

**5. Add online evaluation** at 5–10% sampling, running asynchronously off the request path, attaching scores to traces.

**6. Compare online against offline** on every metric you track. Report the gap per stratum.

**7. Investigate the largest gap.** Filter to low-scoring traces, cluster the inputs, and find what your golden set is missing. **Add at least three real production failures to the golden set** and re-run the offline suite.

**8. Set up two alerts:** p95 latency above your SLO, and online quality dropping below a threshold on any metric.

**9. Note the PII surface.** Add trace storage to `docs/data-inventory.md` with a provisional retention period. Lesson 11.3 audits it.

---

## Visual Aid

*A waterfall span-tree view of one agent trace. Horizontal bars indented by nesting depth, widths proportional to duration, each labeled with operation name, duration, and cost where applicable. The `rerank` bar is conspicuously the widest and highlighted amber. Inside the `search_docs` group, eight thin identical `vector_search` bars are stacked and outlined in red with a callout: "8 identical calls — cache keyed on the wrong query." Along the top, the trace header shows total duration, total cost, and an online-eval score badge reading "faithfulness 0.62" in red, with a small link icon labeled "add to golden set."*

---

## Key Takeaways

- **A trace is one request; spans are its nested operations.** For a twenty-operation agent run this is the only representation that shows structure, and logs cannot substitute.
- **The `request_id` from Lesson 1.4 becomes your trace ID** — and it is already your checkpoint thread ID, so telemetry, traces, and time-travel debugging share one key.
- **Add LLM-specific span attributes:** tokens, cost, prompt version, finish reason, attempt, chunk IDs, agent step, termination reason. You already collect all of them.
- **Online evaluation closes the loop from Lesson 3.1.** Sample production, score with your calibrated judge, and read the offline/online gap as a statement about what your golden set is missing.
- **Production failures should flow back into the golden set.** That loop is what keeps an evaluation practice honest over years.
- **Traces are your most sensitive store.** Redact at capture, set retention, restrict access, sample rather than capture everything.

---

## Next

Your system is fast, observable, and evaluated in production. It runs on your machine, from a set of commands you would struggle to reconstruct in a month, and a reviewer cloning your repository would not get past the first step.

**Lesson 10.5** packages the whole stack — API, vector store, model server, tracing collector, cache — into one command, externalizes the configuration that changes most often, and wires the CI pipeline that gates on quality *and* cost. Reproducibility is not housekeeping here; it is what makes your portfolio repository readable in Module 12.
