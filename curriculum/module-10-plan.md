# Module 10 Plan — Serving, Deployment & Performance

**Duration:** 60 min · **Lessons:** 5 · **Depends on:** Modules 5, 7, 9
**Outline reference:** `curriculum/course-outline.md` §8, Module 10

---

## Prerequisites

**From Module 5:** the RAG pipeline. **From Module 7:** the LangGraph agent with checkpointing. **From Module 9:** a served adapter. **From Modules 1 and 3:** telemetry with correlation IDs, and the eval harness — every optimization here must be proven not to cost quality.
**From the student:** FastAPI or equivalent async web framework experience, Docker Compose, CI. This is the module where existing seniority converts most directly into hireability, so it teaches the AI-specific parts and assumes the rest.

---

## Framing

Research on postings is unambiguous that ~93% of AI-first roles require skills beyond GenAI — Python, FastAPI, Docker, CI/CD, observability — and that "production instincts" are what hiring managers screen for in a project deep-dive. Most course graduates in this field arrive with a notebook. Arriving with a traced, cached, dockerized service that meets a stated SLO is the differentiator, and it is the part this audience is already 70% equipped for.

---

## Module Learning Objectives

**M10-LO1** — Design and build a streaming FastAPI service around an agent, handling concurrency, cancellation, and backpressure.
**M10-LO2** — Serve an open-weights model with vLLM behind the same OpenAI-compatible interface used since Module 1.
**M10-LO3** — Meet a stated latency and cost SLO using caching, batching, routing, and context trimming — and show the measurement.
**M10-LO4** — Instrument end-to-end tracing and read an agent run's trace to find where time and money went.
**M10-LO5** — Package the system as reproducible Docker Compose with externalized config and a CI pipeline.

---

## Lesson 10.1 — Designing the Service: FastAPI, Async, Streaming

*13 min · serves M10-LO1*

**Concept.** What makes an LLM service different from a normal API service: requests take seconds to minutes rather than milliseconds, they are I/O-bound on an upstream that rate-limits, they produce output incrementally, and users abandon them mid-flight. Cover the shape that follows. **Async throughout** — one blocking call in the path serializes the whole worker, and this is the most common performance bug in the category. **Streaming** via SSE or chunked responses, with the agent-specific wrinkle that an agent emits *events* (tool started, tool finished, token) rather than just tokens, so the wire format needs an event type from the start. **Cancellation** as a first-class path: when the client disconnects mid-generation you are still paying for tokens, so disconnect must propagate to the upstream call and to the agent loop's termination check from Lesson 6.4. **Long-running work** — anything past request timeout becomes a job with an ID, which the Module 7 checkpointer already makes resumable. **Backpressure** — provider rate limits are the real capacity constraint, so a bounded queue with an explicit 429 beats an unbounded one that silently grows latency.

**Example.** The naive endpoint — synchronous, buffered, no cancellation — load-tested at 20 concurrent users and collapsing. Then the async streaming version under the same load, with p95 and cost-per-abandoned-request compared. The abandoned-request line is the one that surprises people.

**Practice.** Students build `service/api.py` with a streaming agent endpoint emitting typed events, wire cancellation through to the agent loop, and add a bounded queue. They load-test at three concurrency levels and report p50/p95, error rate, and wasted spend on abandoned requests.

**Summary.** Async everywhere, stream events not just tokens, propagate cancellation, and bound the queue. Abandoned requests cost real money unless cancellation actually reaches the model call.

**Key terms:** async/await, SSE, chunked transfer, event stream, client disconnect, cancellation propagation, job queue, backpressure, bounded queue.
**Visual aid:** Request lifecycle diagram — client, service, agent loop, model — with the cancellation signal traced back through every layer.
**Hook:** "The service is solid and every token still comes from someone else's API. You have your own model from Module 9 — time to serve it properly."

---

## Lesson 10.2 — Serving Open-Weights Models with vLLM

*12 min · serves M10-LO2*

**Concept.** The payoff for the Lesson 1.2 seam: vLLM exposes an OpenAI-compatible server, so the fine-tuned model from Module 9 drops into the existing client as a config change. Cover what vLLM actually does and why it beats a naive `transformers` loop by an order of magnitude — **continuous batching** (new requests join the running batch instead of waiting for a batch boundary) and **PagedAttention** (KV-cache memory managed in pages, eliminating the fragmentation that otherwise wastes most of the GPU). Cover the operational knobs that matter: GPU memory utilization, max model length, tensor parallelism for multi-GPU, and multi-adapter serving from Lesson 9.4. Cover the capacity mental model this audience needs and rarely has: throughput is governed by KV-cache capacity, which is governed by context length times concurrency — so doubling context halves concurrency, which is a scheduling fact, not a tuning detail. Cover the honest self-hosting economics: self-hosting wins at sustained high volume and loses badly at low or spiky volume once idle GPU time is counted.

**Example.** The Module 9 adapter served under vLLM in Docker, called through the unchanged Module 1 client, with a throughput comparison against naive generation. Then the capacity table — context length against concurrent requests against tokens per second — and the break-even calculation against the hosted API at three volume levels.

**Practice.** Students serve their adapter with vLLM in Docker Compose, point their existing client at it by changing configuration only, and produce a capacity table plus a break-even analysis for their spine workload.

**Summary.** vLLM makes self-hosting viable and keeps the interface unchanged. Capacity is KV-cache-bound, and self-hosting only pays above a volume you should calculate rather than guess.

**Key terms:** vLLM, continuous batching, PagedAttention, KV cache, GPU memory utilization, tensor parallelism, multi-adapter serving, break-even volume.
**Visual aid:** KV-cache capacity chart — context length against concurrency — with the workload's operating point plotted.
**Hook:** "Both backends are fast now. The system is still slow, because it makes eleven model calls per question."

---

## Lesson 10.3 — Caching, Batching, and the Latency Budget

*12 min · serves M10-LO3*

**Concept.** Optimization framed as budget allocation rather than trickery. Start by **decomposing the latency budget** from a real trace: retrieval, reranking, prefill, decode, and the tool calls in between — because optimizing the wrong stage is the most common wasted week in this field, and the trace from Lesson 10.4 is what prevents it. Then the toolkit, each with its trade. **Caching** at four levels: exact-match response cache (trivial, low hit rate on natural language), semantic cache keyed on embedding similarity (high hit rate, and a real correctness risk when the threshold is loose — a near-miss returns a confidently wrong cached answer), embedding cache (free and always worth it), and provider prefix caching from Lesson 2.4. **Batching** for offline and bulk paths, where it is a large win, versus interactive paths where it adds latency. **Routing** — a small or fine-tuned model for the common case, escalating to the frontier model on a complexity or confidence signal, which is Lesson 9.3's routing pattern applied at the system level. **Context trimming** — fewer retrieved chunks and shorter history reduce prefill cost directly, and Module 5 already proved more context is not always better. Every one of these is gated by the Module 3 eval suite, because each can silently cost quality.

**Example.** A p95 of 8.4 seconds decomposed by trace — 60% of it in two sequential reranker calls nobody suspected — then four optimizations applied one at a time with quality checked after each. The semantic cache lands the largest win *and* is the one that introduces a wrong answer at a loose threshold, which is shown rather than glossed.

**Practice.** Students set an explicit latency and cost SLO for the spine service, decompose the current budget from traces, then apply optimizations one at a time, running the full eval suite after each. They report a table of latency, cost, and quality per step, and must revert at least one change that costs quality.

**Summary.** Measure the budget before optimizing it, change one thing at a time, and re-run the evals — an optimization that quietly costs quality is a regression with better metrics.

**Key terms:** latency budget, exact-match cache, semantic cache, cache threshold, false cache hit, embedding cache, batching, model routing, escalation signal, context trimming, SLO.
**Visual aid:** Stacked latency-budget bar per pipeline stage, before and after, with the quality score printed above each bar.
**Hook:** "Everything you just did depended on knowing where the time went. Here's how you know."

---

## Lesson 10.4 — Tracing: Seeing Inside an Agent Run

*12 min · serves M10-LO4*

**Concept.** A single user question can produce twelve model calls, six tool calls, and three retrievals — logs cannot represent that, and traces can. Cover the model: a **trace** is one user request, **spans** are nested operations, and every span carries timing plus the LLM-specific attributes that make it useful — model, tokens in and out, cost, prompt version from Lesson 2.4, tool name, retrieved chunk IDs. The Module 1 correlation ID becomes the trace ID, which is why it was introduced in the first lesson. Cover the tooling landscape: OpenTelemetry as the general standard and LLM-native tracers (Langfuse, LangSmith and their peers) which add prompt/response capture, cost rollups, eval score attachment, and dataset capture from production. Cover what traces enable beyond debugging — **online evaluation**, where production traffic is sampled and scored by the Lesson 3.4 judge, closing the loop from Lesson 3.1's three evaluation surfaces; **drift detection**, since input distributions change; and **capturing production failures back into the golden dataset**, which is how the eval set stays honest. Cover the privacy constraint plainly: traces contain full prompts and responses, so PII policy is a Module 11 concern that starts here.

**Example.** One agent trace visualized — the full span tree with time and cost per span — where a retrieval span called eight times instead of once reveals a caching bug invisible in logs. Then an online eval dashboard where faithfulness on production traffic sits four points below the offline set, and the query cluster responsible for the gap.

**Practice.** Students instrument the full stack with tracing carrying LLM attributes, verify one complete agent run end to end in the UI, then sample 10% of traffic for online judging and compare online against offline scores. They add at least one production failure they find to the golden set.

**Summary.** Traces are the debugging surface for agents and the source of honest production quality data. The trace ID has been in the code since Lesson 1.4 for exactly this.

**Key terms:** trace, span, span attributes, OpenTelemetry, LLM-native tracing, online evaluation, sampling rate, drift detection, dataset capture, trace PII.
**Visual aid:** Waterfall span tree for one agent run, with the anomalous repeated retrieval highlighted and cost annotated per span.
**Hook:** "It's fast, observable, and it runs on your machine. Now make it run on any machine, the same way, every time."

---

## Lesson 10.5 — Packaging: Docker Compose, Configuration, CI

*11 min · serves M10-LO5*

**Concept.** Familiar ground for this audience, taught for what is AI-specific. Cover the **service topology** that has accumulated across the course — API, vector store, model server, tracing collector, cache — as one Compose file with health checks and dependency ordering, and the reason model containers are different: multi-gigabyte images, GPU device passthrough, model weights as mounted volumes rather than baked layers, and long warm-up periods that make health-check timing a real problem. Cover **configuration**: model choice, prompt version, retrieval-k, rerank-k, budgets, and SLO thresholds are all runtime config, because they are the things that change most often and the things a rollback must be able to change without a rebuild. Cover **secrets** — provider keys never in the image. Cover **CI** as the assembly of everything already built: lint, unit tests, the Module 3 smoke-tier evals on every PR, full evals plus a cost check on merge, and image build. Cover the AI-specific CI problem honestly — evals cost money and are non-deterministic, so the two-tier design from Lesson 3.5 is what makes this affordable. Close on **reproducibility as a portfolio requirement**: a reviewer must be able to clone, `docker compose up`, and have it work, because a repo that does not run does not get read.

**Example.** The complete `docker-compose.yml` for the spine stack walked service by service, with the GPU passthrough and warm-up health check called out. Then the CI pipeline running smoke evals on a PR and blocking a merge on a cost regression — quality and cost gated together.

**Practice.** Students produce a single-command deployment of the whole stack, externalize all configuration, wire the two-tier CI pipeline with a cost gate, and verify reproducibility by cloning into a clean directory and bringing it up from scratch.

**Summary.** One command to run the whole system, config outside the image, secrets outside the repo, and CI that gates on quality and cost. Reproducibility is a hiring signal, not housekeeping.

**Key terms:** Docker Compose, health check, GPU passthrough, model volume mount, warm-up, twelve-factor config, secret management, CI tiering, cost gate, reproducibility.
**Visual aid:** Service topology diagram with ports, volumes, dependency arrows, and health-check timing annotated on the model server.
**Hook:** "You can ship it. Module 11 is about everything that happens when someone who isn't a friendly user finds it."

---

## Spine Increment

**Deliverable:** `service/` and `docker-compose.yml` — a streaming FastAPI service with cancellation and backpressure, vLLM serving the fine-tuned adapter, caching and routing tuned to a stated SLO, full tracing with online evaluation, and a two-tier CI pipeline.

**Acceptance criteria:**
1. Streaming events with working cancellation; wasted spend on abandoned requests measured and reduced.
2. The Module 1 client reaches vLLM by config change alone.
3. A stated SLO, met, with a per-stage latency budget before and after and the eval suite green after each optimization — including one reverted change.
4. End-to-end traces with LLM attributes; 10% online eval sampling with online-vs-offline comparison.
5. `docker compose up` brings up the full stack from a clean clone.
6. CI runs smoke evals on PR, full evals plus a cost gate on merge.

---

## Standalone Exercise

Take a supplied working but slow agent endpoint and cut p95 latency by half without losing eval score. Deliverable: `exercises/m10-latency-halving/` with the trace-based budget decomposition, each optimization and its measured effect on latency, cost, and quality, and a note on any change that had to be reverted.

---

## Dependencies

| Lesson | Requires |
|---|---|
| 10.1 | Modules 5, 7 |
| 10.2 | Module 9 adapter, Lesson 1.2 seam |
| 10.3 | 10.1, 10.2, and Module 3 eval suite |
| 10.4 | 10.1, plus Lesson 1.4 correlation IDs |
| 10.5 | 10.1–10.4 |

**Downstream:** Module 11 hardens this service and adds fallback routing to its client. Module 12 packages it as the portfolio artifact — the reproducibility work in 10.5 is what makes the repo reviewable.

---

## Assessment Focus (Phase 4)

Production reasoning, the substance of the interview deep-dive. Given a latency breakdown, choose the optimization with the best return. Given a semantic cache returning a wrong answer, explain the mechanism and the fix. Given a service that collapses at 20 concurrent users, name three likely causes. Given a context-length change, predict the effect on vLLM concurrency. Given a trace with a repeated span, diagnose it. One practical: meet a stated SLO on a supplied service and show the measurement trail.
