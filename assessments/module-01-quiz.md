# Module 1 quiz — The AI Engineer role and a provider-agnostic baseline

Ten scenario questions on what the AI engineer job actually is, building one client that talks to many models, the mechanics that set the cost and latency of a call, and instrumenting both from day one. They test reasoning, not recall. Most describe a posting, a migration, a benchmark or a set of telemetry numbers and ask what it shows or what to do next.

**Covers**

- [01.1 · What an AI Engineer Actually Does (and What They Don't)](../lessons/module-01/lesson-01.md) — AI vs ML engineering vs data science; reading a posting as a decomposition; the four pillars.
- [01.2 · One Interface, Many Models: The OpenAI-Compatible Gateway](../lessons/module-01/lesson-02.md) — one `LLMClient` with profiles; the three tiers of portability; unsupported capabilities must raise; the local-first dev loop.
- [01.3 · The Physics of an LLM Call: Tokens, Context, Sampling, Streaming](../lessons/module-01/lesson-03.md) — model-specific tokens; prefill vs decode; the context budget; `temperature=0` is not determinism; streaming and TTFT.
- [01.4 · Instrumenting Cost and Latency From Day One](../lessons/module-01/lesson-04.md) — the `CallRecord`; `request_id` and `call_site`; per-request metrics; cost per successful request.

**Pass mark:** 70% (7 of 10). Unlimited retries; answers and explanations appear after you submit.

The hands-on part of this module is the [Module 1 practical](../projects/module-01-practical.md): call telemetry and a request-level baseline.
