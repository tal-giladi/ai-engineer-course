# Module 1 Plan — The AI Engineer Role & a Provider-Agnostic Baseline

**Duration:** 50 min · **Lessons:** 4 · **Depends on:** nothing (course entry point)
**Outline reference:** `curriculum/course-outline.md` §8, Module 1

---

## Prerequisites

**From the student:** working Python, comfort with HTTP APIs and Docker, an API key for at least one hosted provider, a Hugging Face account.
**From the course:** none. This is the entry point.

**Environment check before Lesson 2:** Python 3.11+, `uv` or `pip`, Docker running, and one of Ollama or a vLLM container able to pull a small open-weights model (~4–8 GB). A fallback path using only hosted endpoints is provided for students on constrained machines.

---

## Module Learning Objectives

**M1-LO1** — Distinguish AI engineering from ML engineering and data science, and map a real job posting onto the modules of this course.
**M1-LO2** — Configure a single OpenAI-compatible client that routes to a hosted frontier model, a Hugging Face endpoint, or a local Docker-served open-weights model without changing call sites.
**M1-LO3** — Predict and measure the token cost and latency of a call, and explain how context length, sampling parameters, and streaming affect each.

---

## Lesson 1.1 — What an AI Engineer Actually Does (and What They Don't)

*12 min · serves M1-LO1*

**Concept.** The role split that confuses career-changers: ML engineers build and train models and need heavy math; AI engineers integrate and deploy models and need software engineering. Research on current postings shows ~93% of AI-first roles require skills beyond GenAI — Python, FastAPI, Docker, CI/CD, Postgres, cloud — which means this audience already holds most of the job's surface area and is missing a specific, learnable middle. Introduce the four pillars the rest of the course is organized around: **retrieval, agents, evaluation, and production**.

**Example.** Two real job postings side by side — one "ML Engineer, Ranking" and one "AI Engineer, Platform" — annotated line by line to show which requirements are model science and which are systems work. Then the same annotation applied to this course's module map.

**Practice.** Students pull three live job postings for roles they would actually apply to, tag every requirement as *have / learning here / gap*, and save the result as `job-map.md`. This file gets revisited in Module 12.

**Summary.** The job is systems engineering with a probabilistic component. The gap is not mathematics; it is retrieval, agents, evaluation, and the production discipline around a non-deterministic dependency.

**Key terms:** AI engineer, ML engineer, applied LLM role, GenAI stack.
**Visual aid:** Venn diagram — software engineering / ML / AI engineering, with the overlap labeled and this course's twelve modules positioned inside it.
**Hook:** "Every one of those postings assumes you can call a model. Almost none of them assume you can swap one out on a Friday afternoon. That's next."

---

## Lesson 1.2 — One Interface, Many Models: The OpenAI-Compatible Gateway

*14 min · serves M1-LO2*

**Concept.** The OpenAI chat-completions shape has become the de-facto wire protocol: hosted frontier providers, Hugging Face endpoints, vLLM, Ollama, and llama.cpp all speak it. Building against that interface — through a thin gateway rather than a vendor SDK — makes provider choice a config value. Cover the surface honestly: what is genuinely portable (messages, tools, streaming, temperature) and what is not (prompt caching semantics, reasoning/thinking parameters, safety-filter behavior, tokenizer differences). Introduce the adapter seam where non-portable features get isolated.

**Example.** One `LLMClient` class, three `.env` profiles — hosted frontier, HF Inference endpoint, local vLLM in Docker — and the identical call site producing an answer from each. Then a deliberate demonstration of leakage: a provider-specific parameter that silently no-ops on another backend, and where the adapter should have caught it.

**Practice.** Students build the client, wire all three profiles, and write one test that asserts identical behavior across backends on a fixed prompt — plus one that asserts the client *raises* rather than silently drops an unsupported parameter.

**Summary.** Code against the interface, isolate the differences at a seam, and provider migration becomes a config change plus a known list of exceptions.

**Key terms:** OpenAI-compatible API, gateway/proxy, base URL routing, model alias, adapter seam.
**Visual aid:** Architecture diagram — application → `LLMClient` → gateway → three backends, with the non-portable features drawn as annotations on the seam.
**Hook:** "You can now call three models with one line. Each one costs a different amount and takes a different time. If you can't say how much or how long, you can't ship it."

---

## Lesson 1.3 — The Physics of an LLM Call: Tokens, Context, Sampling, Streaming

*12 min · serves M1-LO3*

**Concept.** The mechanical facts that govern every later design decision. Tokenization and why token count is not word count and differs per model family. The asymmetry that dominates cost and latency: input tokens are processed in parallel (prefill), output tokens are generated one at a time (decode) — so output is far more expensive in wall-clock terms. Context window as a hard budget shared by system prompt, retrieved context, conversation history, and tool results, plus the practical fact that quality degrades before the limit is reached. Sampling parameters as a reproducibility problem, not a creativity dial. Streaming as a time-to-first-token optimization that changes perceived latency without changing total cost.

**Example.** A single prompt run at four context sizes, with a table showing tokens in/out, time to first token, total latency, and cost — making the prefill/decode asymmetry visible in real numbers. Then the same prompt with `temperature=0` run five times, showing that "deterministic" still isn't.

**Practice.** Students instrument their Module 1 client to report tokens in, tokens out, TTFT, total latency, and computed cost for every call, then reproduce the four-context table on their own backends.

**Summary.** Output tokens dominate latency, input tokens dominate context pressure, and temperature zero is not determinism. Every optimization in Module 10 is an application of these three facts.

**Key terms:** tokenization, prefill, decode, context window, time to first token (TTFT), temperature, top-p, streaming.
**Visual aid:** Timeline diagram of a streamed call — request, prefill block, per-token decode ticks, completion — annotated with where TTFT and total latency are measured.
**Hook:** "You have the numbers for one call. Production is a million calls, and nobody is watching them one at a time."

---

## Lesson 1.4 — Instrumenting Cost and Latency From Day One

*12 min · serves M1-LO3*

**Concept.** Why measurement is scaffolding rather than polish: an LLM system's cost and latency are emergent properties of prompts and retrieval decisions made months earlier, and retrofitting instrumentation means you cannot attribute a regression to its cause. Introduce the minimum viable record — timestamp, model, tokens in/out, cost, latency, TTFT, a call-site tag, and a correlation ID that will later tie every call in one agent run together. Distinguish per-call metrics from per-*request* metrics, which is the unit that actually matters once one user question triggers twelve model calls. Preview cost-per-successful-task as the metric that survives to Module 10.

**Example.** A `CallRecord` dataclass and a decorator that captures it, writing JSONL locally. Then a five-line aggregation showing cost per request and p50/p95 latency — and a worked case where per-call cost looks fine while per-request cost is 14× higher because of a retry loop.

**Practice.** Students add the decorator to their client, generate 50 calls across two backends, and produce a summary table: p50/p95 latency, mean cost per request, and total spend. They commit the JSONL as the first baseline artifact of the spine project.

**Summary.** Instrument before you build. The correlation ID added in this lesson is what makes the Module 10 trace readable, and cost-per-successful-task is the number a hiring manager will ask for.

**Key terms:** call record, correlation ID, p50/p95, cost per request, cost per successful task, JSONL telemetry.
**Visual aid:** Annotated JSONL record with each field labeled by the module that will consume it.
**Hook:** "Module 2: the model returns a paragraph, your code needs an object, and the space between those two things is where most LLM projects quietly break."

---

## Spine Increment

**Deliverable:** `spine/llm_client.py` — an OpenAI-compatible client with pluggable backends and automatic telemetry.

**Acceptance criteria:**
1. One call site works unchanged against hosted frontier, hosted open-weights, and local Docker backends.
2. Unsupported provider-specific parameters raise rather than silently no-op.
3. Every call emits a `CallRecord` with tokens, cost, latency, TTFT, and correlation ID.
4. `make baseline` regenerates the telemetry summary table.

---

## Standalone Exercise

Benchmark one fixed prompt across three backends and produce a comparison table of cost, p50/p95 latency, TTFT, and a subjective quality note. Deliverable: `exercises/m1-backend-benchmark.md` with the table and a one-paragraph recommendation defending a choice for a stated use case.

---

## Dependencies

| Lesson | Requires |
|---|---|
| 1.1 | — |
| 1.2 | 1.1 (framing only) |
| 1.3 | 1.2 (needs a working client) |
| 1.4 | 1.3 (needs the metrics defined) |

**Downstream:** every subsequent module calls `LLMClient`. Module 10 extends its telemetry into distributed tracing; Module 11 extends its error handling into fallback routing.

---

## Assessment Focus (Phase 4)

Application over recall: given a cost and latency table, recommend a backend and defend it; given a provider migration, identify which parameters break; given a per-call and per-request cost pair that disagree, diagnose the cause.
