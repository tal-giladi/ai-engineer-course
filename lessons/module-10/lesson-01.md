---
id: "10.1"
module: 10
minutes: 13
practice_minutes: 90
prerequisites: ["09.4"]
objectives:
  - "Design and build a streaming FastAPI service around an agent, handling concurrency, cancellation, and backpressure."
volatility: implementation
sources: []
last_verified: "2026-08-24"
---

# 10.1 · Designing the Service: FastAPI, Async, Streaming

This is the module where your existing seniority converts most directly into hireability. Roughly 93% of AI-first roles require skills beyond generative AI — Python, FastAPI, Docker, CI/CD, observability — and hiring managers screen for "production instincts" in the project deep-dive. Most people who finish an AI course arrive with a notebook. You are going to arrive with a traced, cached, dockerized service that meets a stated SLO.

So this lesson assumes you can build an HTTP service and focuses only on what is different when the thing behind the endpoint is an LLM agent. Four things are different, and each has a failure mode that will not appear in load testing unless you look for it.

---

## Key Concept 1: What makes an LLM service unusual

**Requests take seconds to minutes.** A normal API request is measured in milliseconds; your agent runs for six seconds on a good day and ninety on a bad one. Every timeout, connection pool, and load balancer setting in your usual stack assumes otherwise.

**Requests are I/O-bound on a rate-limited upstream.** Your service spends nearly all its time waiting on a provider that will start returning 429s. Your capacity is not your CPU; it is your provider quota.

**Output arrives incrementally.** Users expect to see tokens appear. A buffered response wastes the single most effective perceived-latency improvement available.

**Users abandon mid-flight.** And here is the part that is genuinely new: **an abandoned LLM request is still costing you money** until something stops it. In a normal service, a disconnected client means a wasted CPU cycle. Here it means paying for tokens nobody will read.

---

## Key Concept 2: Async everywhere, or nothing works

One blocking call anywhere in the request path serializes an entire worker.

This is the most common performance bug in LLM services, and it is easy to introduce because the blocking call is usually not the model call — it is a synchronous database query, a `requests` call in a tool, or a CPU-bound reranker.

```python
@app.post("/ask")
async def ask(req: AskRequest):
    chunks = await store.search(req.query)      # async DB driver
    result = await agent.ainvoke(...)           # async model calls
    return result
```

Rules:

- Async HTTP client, async database driver, async everything in the path
- Genuinely CPU-bound work — a local cross-encoder reranker, tokenization of a large document — goes to a thread or process pool, not the event loop
- Audit your tools (Module 6). One tool using a synchronous HTTP library will serialize the whole service, and it will look like a mysterious concurrency ceiling

---

## Key Concept 3: Stream events, not just tokens

For a plain LLM call, streaming means tokens. For an **agent**, the interesting things are not tokens — they are the steps.

A user waiting nine seconds while an agent runs four tool calls should see what is happening. So your wire format needs an event type from the start:

```python
async def event_stream(req):
    async for ev in agent.astream_events(...):
        match ev.type:
            case "tool_start":
                yield sse({"type": "tool_start", "tool": ev.name})
            case "tool_end":
                yield sse({"type": "tool_end", "tool": ev.name,
                           "summary": ev.summary})
            case "token":
                yield sse({"type": "token", "text": ev.text})
            case "citation":
                yield sse({"type": "citation", "source": ev.source})
    yield sse({"type": "done", "usage": usage, "cost": cost})
```

Design notes:

- **Type every event.** Retrofitting a type field onto a token-only stream means a breaking change for every client.
- **Send the final usage and cost** in the terminal event. Your Lesson 1.4 telemetry should reach the client too, at least in development.
- **Errors are events**, not dropped connections. A stream that stops silently is indistinguishable from a network failure.
- **Heartbeats.** A long tool call with no output can trip an intermediary's idle timeout. Send a keepalive.

---

## Key Concept 4: Cancellation must reach the model

Here is the failure that costs real money and is invisible in ordinary load testing.

A user closes the tab at second three of a nine-second agent run. Without cancellation, your agent keeps calling tools and generating tokens for six more seconds. You pay for all of it.

At 20% abandonment on a $0.03 agent run, that is 20% of your spend producing nothing.

Cancellation must propagate through **three** layers:

```python
@app.post("/ask")
async def ask(req: AskRequest, request: Request):
    task = asyncio.create_task(run_agent(req))
    try:
        async for ev in stream_with_disconnect_check(task, request):
            yield ev
    except asyncio.CancelledError:
        task.cancel()                    # 1. stop the agent loop
        await agent.abort(req.trace_id)  # 2. stop the in-flight model call
        record_cancelled(req.trace_id)   # 3. record the partial spend
        raise
```

1. **The agent loop** stops taking steps — this is the `CANCELLED` termination reason you defined in Lesson 6.4.
2. **The in-flight model call** is aborted. Closing the HTTP connection to the provider is what actually stops the meter.
3. **Record it.** Cancelled requests with partial spend must appear in your telemetry, or your cost-per-successful-request metric (Lesson 1.4) is wrong.

And note the interaction with checkpointing: with Module 7's checkpointer, a cancelled run is *resumable*. For expensive long-running work, offer to resume rather than discarding.

---

## Key Concept 5: Backpressure and long-running work

**Your capacity is your provider quota, not your CPU.** Accepting unlimited concurrent requests does not increase throughput; it increases the queue, and every request gets slower until they all time out.

Bound it explicitly:

```python
semaphore = asyncio.Semaphore(MAX_CONCURRENT)

async def guarded(req):
    try:
        await asyncio.wait_for(semaphore.acquire(), timeout=2.0)
    except asyncio.TimeoutError:
        raise HTTPException(429, "at capacity, retry shortly")
```

An explicit fast 429 is far better than an implicit slow timeout: the client can retry or back off, and your p95 stays meaningful.

**Anything longer than your request timeout becomes a job.** Return a job ID, run in the background, let the client poll or subscribe. Module 7's checkpointer makes these jobs durable and resumable, which is the reason that lesson came first.

---

## Worked Example: Naive versus correct, under load

Same agent, two implementations, load-tested at 5, 20, and 50 concurrent users, with 20% of clients disconnecting after 3 seconds.

**Naive** — sync endpoint, buffered response, no cancellation, unbounded concurrency:

| Concurrency | p50 | p95 | Errors | Wasted spend |
|---|---|---|---|---|
| 5 | 6.2 s | 9.1 s | 0% | 19% |
| 20 | 24.8 s | 61.0 s | 12% | 21% |
| 50 | timeout | timeout | 78% | 24% |

**Async streaming with cancellation and a bounded queue:**

| Concurrency | TTFT p50 | p95 total | Errors | Wasted spend |
|---|---|---|---|---|
| 5 | 0.4 s | 8.8 s | 0% | **2%** |
| 20 | 0.5 s | 10.2 s | 0% | **2%** |
| 50 | 0.6 s | 14.1 s | 8% (explicit 429) | **3%** |

Three things to read out of this:

**The naive version collapses at 20 concurrent users** — not from CPU, but from one synchronous database call in the retrieval path serializing every worker. This is the bug from Key Concept 2, and it looks like a mysterious ceiling rather than a blocking call.

**TTFT is 0.4 s versus 6.2 s.** The work takes the same time; the user's experience is completely different.

**Wasted spend went from 21% to 2%.** At $4,000/month, that is roughly $760/month recovered by propagating cancellation. **This line does not appear in any standard load-test report** — you have to measure it deliberately, which is exactly why it is worth knowing about.

At 50 concurrent, the 8% "errors" are explicit 429s returned in under two seconds, which is a correct outcome rather than a failure.

---

## Practice

**1. Build the streaming endpoint.** `service/api.py`, FastAPI, SSE, typed events for tool start/end, tokens, citations, and a terminal event carrying usage and cost.

**2. Audit for blocking calls.** Check every tool from Module 6, your vector store client, and any reranker. Move CPU-bound work to a thread pool. This audit will probably find something.

**3. Wire cancellation through all three layers.** Agent loop, in-flight model call, telemetry record. Verify by disconnecting a client mid-stream and confirming — in your provider dashboard or telemetry — that generation actually stopped.

**4. Measure wasted spend.** Load test with 20% of clients disconnecting at 3 seconds. Report spend on cancelled requests before and after your cancellation work. **This number is a strong portfolio line.**

**5. Add a bounded queue** with an explicit fast 429. Set `MAX_CONCURRENT` from your provider's actual rate limit, not a guess.

**6. Load test at three concurrency levels.** Report TTFT p50, total p95, error rate, and wasted spend at each.

**7. Add a job endpoint** for long-running work — job ID, background execution, status polling, backed by the Module 7 checkpointer so it survives a restart.

**8. Add heartbeats** so a long tool call does not trip an idle timeout.

---

## Visual Aid

*A request lifecycle diagram with four horizontal lanes: Client, Service, Agent Loop, Model Provider. A request flows down through all four. At 3 seconds, a red "disconnect" marker on the Client lane, with a heavy red arrow propagating downward through every lane in turn — annotated at each: "1. cancel task," "2. abort agent loop → CANCELLED," "3. close provider connection — meter stops," "4. record partial spend." Beside the diagram, a shaded region on the Model Provider lane from 3 s to 9 s is labeled "what you pay for without this — 21% of spend." Below, a small before/after bar pair: "wasted spend 21% → 2%."*

---

## Key Takeaways

- **LLM services differ in four ways:** long requests, I/O-bound on a rate-limited upstream, incremental output, and abandonment that costs money.
- **One blocking call serializes a worker,** and it is usually not the model call — audit your tools, database driver, and reranker.
- **Stream typed events, not just tokens.** For an agent, tool start/end and citations are what the user needs to see; add the event type from day one.
- **Cancellation must reach the provider connection,** not just your loop. Recovering 20% of spend by propagating cancellation is a real and commonly-missed win — and it does not show up in standard load-test reports.
- **Capacity is your provider quota.** Bound concurrency and return a fast explicit 429; anything past your request timeout becomes a durable job.

---

## Next

Your service is solid, and every token it produces still comes from someone else's API. You have your own fine-tuned model from Module 9 sitting in a registry.

**Lesson 10.2** serves it with vLLM behind the same OpenAI-compatible interface you built in Lesson 1.2, explains why continuous batching and PagedAttention make self-hosting viable at all, and gives you the capacity model — plus the break-even calculation that tells you whether self-hosting is actually cheaper for your volume.
