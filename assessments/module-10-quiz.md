# Module 10 quiz — Serving, Deployment & Performance

Ten scenario questions on running an agent as a production service, framed the way the interview deep-dive frames them: diagnosing a concurrency ceiling, making cancellation stop the spend, sizing a vLLM deployment from a context distribution, spending a latency budget deliberately, reading a span tree and an offline/online gap, and gating CI on cost as well as quality. They test reasoning, not recall. Most show a measurement, a trace or a CI result and ask what it means or what to do next.

**Covers**

- [10.1 · Designing the Service: FastAPI, Async, Streaming](../lessons/module-10/lesson-01.md) — blocking calls, typed event streams, cancellation through three layers, backpressure.
- [10.2 · Serving Open-Weights Models with vLLM](../lessons/module-10/lesson-02.md) — continuous batching, PagedAttention, the capacity model, break-even.
- [10.3 · Caching, Batching, and the Latency Budget](../lessons/module-10/lesson-03.md) — budget decomposition, the semantic cache threshold, one change at a time, reverts.
- [10.4 · Tracing: Seeing Inside an Agent Run](../lessons/module-10/lesson-04.md) — spans, LLM attributes, online evaluation, traces as a sensitive store.
- [10.5 · Packaging: Docker Compose, Configuration, CI](../lessons/module-10/lesson-05.md) — model containers, profiles, config, the cost gate, the clean-clone test, worktrees.

**Pass mark:** 70% (7 of 10). Unlimited retries; answers and explanations appear after you submit.

The hands-on part of this module is the [Module 10 practical](../projects/module-10-practical.md): halve the p95 latency of a slow agent endpoint without losing eval score.
