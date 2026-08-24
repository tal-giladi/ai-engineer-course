# Course Outline — AI Engineer: From Software Engineer to Shipping AI Systems

> **Phase 1 deliverable.** Source brief: `brief/course-brief.md`. Pipeline: `CLAUDE.md`.

---

## 1. Course Identity

| Field | Value |
|---|---|
| **Title** | AI Engineer: From Software Engineer to Shipping AI Systems |
| **Audience** | Experienced software developers moving into AI engineering |
| **Level** | Advanced (in software) / Intermediate (in AI) |
| **Modules** | 12 |
| **Lessons** | 53 |
| **Duration** | ~11 hours of instruction (50–60 min per module), plus project time |
| **Format** | Self-paced, hands-on. Every module ends with running code. |
| **Outcome** | A portfolio-ready production AI system and the vocabulary to defend it in an interview |

---

## 2. Target Audience

Working software engineers — backend, platform, full-stack — who can already design a system, containerize it, and reason about failure. They are **not** beginners; they are beginners *at this specific stack*. The course never explains what an API is, and never assumes they have trained a model.

**What they bring:**

- Basic machine learning and LLM understanding (they know what a token and an embedding are)
- Docker and local deployment experience
- Experience working with coding agents
- System architecture and microservices design

**Environment assumed:**

- Local Docker
- Paid Hugging Face account
- Paid Google Colab account (GPU access for the fine-tuning modules)
- API credit with at least one hosted model provider

**What is deliberately *not* assumed:** PyTorch fluency, GPU/CUDA experience, transformer internals, or any prior production LLM work.

---

## 3. Course-Level Learning Outcomes

After this course, a student can:

1. **Build** a production-grade retrieval system — chunking, hybrid search, reranking, grounded citations — and diagnose its failures by category rather than by guesswork.
2. **Build** an agent that plans, calls tools, holds state, and recovers from failure — first by hand, then in LangGraph — and expose or consume capabilities over MCP.
3. **Measure** any LLM system with a golden dataset, task-appropriate metrics, and a calibrated LLM judge, and gate changes on a regression suite in CI.
4. **Decide** whether a problem needs prompting, retrieval, or fine-tuning — and if fine-tuning, run SFT/LoRA and preference tuning end to end and prove the result against a baseline.
5. **Ship** the system as a dockerized, streaming, traced, cost-instrumented service that meets a stated latency and cost SLO.
6. **Defend** the system against prompt injection, tool abuse, and data exfiltration, and keep it inside a cost ceiling under failure.
7. **Present** the work the way hiring panels ask for it: architecture and why, trade-offs, evaluation metrics, and what they would improve.

Every outcome maps to a skill that appears in current AI engineer job descriptions (see §9).

---

## 4. Design Principles

**Built backwards from job posts.** The brief asks that graduates "stand in the job requirements," so the module order follows hiring demand, not textbook order. Research on 2026 AI engineer postings and interview practice produced six findings that shaped this outline:

| Finding | Where it lands |
|---|---|
| RAG, agents, and evaluation are the three most-screened skills; RAG is the single most common pattern in applied LLM roles | Modules 3–7 — five of twelve modules, the core of the course |
| Eval literacy is the strongest signal separating people who have shipped from people who have not | Module 3, placed *before* RAG and agents; every later module extends the same harness |
| ~93% of AI-first roles require skills beyond GenAI — Python, FastAPI, Docker, CI/CD, Postgres, cloud | Modules 1, 10, 12 — and this is where the audience's existing strength is leveraged, not re-taught |
| Recruiters keyword-screen on named tools: LangGraph, LlamaIndex, MCP, Qdrant/pgvector/Pinecone, RAGAS, tracing tools | Module 7 (LangGraph, MCP), Module 4 (vector DBs), Modules 3 and 5 (RAGAS), Module 10 (tracing) |
| Fine-tuning appears in roughly a quarter of postings (SFT/LoRA, DPO, RFT) | Modules 8–9, a full two-module track |
| Interviews center on a project deep-dive and an AI system design round | Module 12, rehearsed against real job descriptions |

**Five further principles:**

- **Eval-first, always.** Module 3 comes before anything worth evaluating exists, on purpose. From then on, no module ships a change without a number attached to it.
- **Mechanism before framework.** Students hand-roll the agent loop and the retrieval pipeline before touching LangGraph. Frameworks are taught as a mapping onto something they already understand, so leaks are debuggable.
- **Provider-agnostic by construction.** All labs speak an OpenAI-compatible API through a gateway. A frontier model, a Hugging Face endpoint, and a local vLLM server are a config change apart — which is also how mature teams actually build.
- **Every abstract concept gets a running example.** Reranking is not a definition; it is a measured jump in context precision on their own dataset.
- **Production instincts throughout.** Cost, latency, failure handling, and observability are introduced in Module 1 and carried in every module, because that is what interviewers probe for.

---

## 5. Technical Stack

| Layer | Choice | Rationale |
|---|---|---|
| Model access | OpenAI-compatible API via a gateway (LiteLLM / OpenRouter-style) | One interface for hosted frontier models, HF Inference, and local vLLM/Ollama. Swapping providers is config, not a rewrite. |
| Language / service | Python 3.11+, Pydantic, FastAPI, async | The stack in the job posts, and familiar ground for this audience |
| Retrieval | pgvector and Qdrant (both, compared), BM25, cross-encoder reranker | Postgres-native vs. purpose-built — the real-world choice students will be asked to justify |
| Agents | Hand-rolled loop → LangGraph; MCP for tool/capability exposure | Mechanism first, then the names recruiters screen for |
| Evaluation | Hand-built harness → RAGAS-style RAG metrics, promptfoo-style regression runs | Understand the metric before importing the library that computes it |
| Fine-tuning | Hugging Face `transformers` + `peft` + `trl` on Colab GPU | Matches the assumed paid accounts |
| Serving | vLLM, Docker Compose, streaming HTTP | Local-first, cloud-portable |
| Observability | OpenTelemetry-style tracing plus an LLM-native tracer (Langfuse/LangSmith-class) | Traces are the debugging surface for agents |

---

## 6. The Spine Project

One system, built across all twelve modules: **a production agentic RAG service over a technical documentation corpus** — it answers questions with citations, calls tools when retrieval is not enough, and runs inside a cost and latency budget.

| Module | Increment |
|---|---|
| 1 | Gateway client with token, cost, and latency accounting |
| 2 | Typed request/response contracts with validation and repair |
| 3 | Golden dataset + eval harness + CI regression gate |
| 4 | Ingestion pipeline, embeddings, hybrid index |
| 5 | Query rewriting, reranking, grounded answers with citations |
| 6 | Hand-rolled agent loop with tools, budgets, and recovery |
| 7 | Ported to LangGraph, with an MCP server exposing its tools |
| 8 | Domain-tuned small model (LoRA), measured against the baseline |
| 9 | Preference-tuned and distilled variant, served as an adapter |
| 10 | FastAPI service, vLLM backend, caching, tracing, Docker Compose |
| 11 | Injection defenses, tool permissions, PII filtering, cost kill switch |
| 12 | Finished repo: model card, eval report, architecture doc, README |

**Each module also carries a self-contained exercise** on a different problem domain. This gives every concept one clean isolated repetition, and means a student who reorders or skips a module is never blocked on a broken spine.

---

## 7. Module Map

| # | Module | Duration | Lessons | Depends on |
|---|---|---|---|---|
| 1 | The AI Engineer Role & a Provider-Agnostic Baseline | 50 min | 4 | — |
| 2 | Structured Output & Prompting as Engineering | 50 min | 4 | 1 |
| 3 | Evaluation Foundations | 60 min | 5 | 1, 2 |
| 4 | Retrieval I — Embeddings, Chunking, Vector Stores | 55 min | 4 | 1, 3 |
| 5 | Retrieval II — Production RAG | 60 min | 5 | 3, 4 |
| 6 | The Agent Loop from Scratch | 60 min | 5 | 2, 3 |
| 7 | Agent Architectures — LangGraph & MCP | 55 min | 4 | 6 |
| 8 | Fine-Tuning I — When, and SFT/LoRA | 55 min | 4 | 3 |
| 9 | Fine-Tuning II — Preference Tuning & Distillation | 55 min | 4 | 8 |
| 10 | Serving, Deployment & Performance | 60 min | 5 | 5, 7, 9 |
| 11 | Security, Safety & Reliability | 60 min | 5 | 6, 10 |
| 12 | Capstone, Portfolio & Interview Readiness | 55 min | 4 | all |

**Total: 12 modules · 53 lessons · ~11 hours**

---

## 8. Modules in Detail

### Module 1 — The AI Engineer Role & a Provider-Agnostic Baseline
*50 min · 4 lessons*

Orientation for people who can already build software. Establishes what the job is, and sets up the one interface every later lab speaks.

**Learning objectives.** Students can:
1. Distinguish AI engineering from ML engineering and data science, and map a real job posting onto the modules of this course.
2. Configure a single OpenAI-compatible client that routes to a hosted frontier model, a Hugging Face endpoint, or a local Docker-served open-weights model without changing call sites.
3. Predict and measure the token cost and latency of a call, and explain how context length, sampling parameters, and streaming affect each.

**Lessons.**
1. What an AI engineer actually does — and what they don't
2. One interface, many models: the OpenAI-compatible gateway
3. The physics of an LLM call: tokens, context, sampling, streaming
4. Instrumenting cost and latency from day one

**Spine increment.** Gateway client with per-call token, cost, and latency accounting.
**Standalone exercise.** Benchmark the same prompt across three backends (frontier, hosted open-weights, local) and produce a cost/latency/quality comparison table.

---

### Module 2 — Structured Output & Prompting as Engineering
*50 min · 4 lessons*

The bridge between a probabilistic model and deterministic code. Where most "it works in the notebook" systems break.

**Learning objectives.** Students can:
1. Define a typed contract between model and application using JSON Schema and Pydantic, and enforce it with schema-constrained generation.
2. Implement a validate-repair-retry loop with an explicit failure budget, and decide when to fail loudly instead of retrying.
3. Treat prompts as versioned artifacts — diffable, testable, cacheable — and explain when prompt caching changes cost and latency.

**Lessons.**
1. Why free-form text is a bad API
2. Structured outputs and schema-constrained generation
3. Validation, repair loops, and failure budgets
4. Prompts as versioned artifacts (and how caching changes the math)

**Spine increment.** Typed request/response contracts with validation and repair.
**Standalone exercise.** Build an extractor that pulls a strict schema out of messy invoices and never returns an invalid object.

---

### Module 3 — Evaluation Foundations
*60 min · 5 lessons*

Placed before RAG and agents deliberately. Research on hiring says eval literacy is the clearest signal that someone has actually shipped LLM systems; it is also the only way the remaining nine modules can claim progress.

**Learning objectives.** Students can:
1. Build a golden dataset for a task — sourcing, labeling, sizing, and splitting it — and justify every choice.
2. Select task-appropriate metrics and explain why accuracy is usually the wrong one.
3. Implement LLM-as-judge, calibrate the judge against human labels, and quantify how far to trust it.
4. Wire a regression suite into CI that blocks a prompt or model change on measured quality loss.

**Lessons.**
1. Why you evaluate before you build
2. Building a golden dataset
3. Metrics that fit the task
4. LLM-as-judge and judge calibration
5. Regression suites and CI gating

**Spine increment.** Golden dataset, eval harness, and a CI gate that fails the build on regression.
**Standalone exercise.** Take a deliberately mediocre summarizer, build an eval that exposes its specific weakness, then prove a fix.

---

### Module 4 — Retrieval I: Embeddings, Chunking, Vector Stores
*55 min · 4 lessons*

The foundation under the most-demanded skill in the field.

**Learning objectives.** Students can:
1. Choose an embedding model on evidence — benchmarking candidates from Hugging Face on their own corpus rather than on a leaderboard.
2. Design and defend a chunking strategy, and demonstrate its effect on retrieval recall.
3. Stand up pgvector and Qdrant in Docker, index the same corpus in both, and articulate when each is the right call.
4. Implement hybrid retrieval combining dense vectors and BM25 with rank fusion, and measure the gain over either alone.

**Lessons.**
1. Embeddings: choosing and benchmarking a model
2. Chunking: the decision that quietly determines your recall
3. Vector stores in Docker — pgvector vs. Qdrant
4. Hybrid retrieval: dense + BM25 + fusion

**Spine increment.** Ingestion pipeline, embeddings, and a hybrid index over the docs corpus.
**Standalone exercise.** Index a codebase and measure retrieval recall across three chunking strategies.

---

### Module 5 — Retrieval II: Production RAG
*60 min · 5 lessons*

The gap between a RAG demo and a RAG system, which is exactly the gap interviewers probe.

**Learning objectives.** Students can:
1. Classify a wrong RAG answer by failure mode — retrieval miss, ranking failure, context overflow, or generation drift — and fix the right layer.
2. Improve recall with query rewriting and multi-query retrieval, and precision with a cross-encoder reranker, measuring each independently.
3. Produce grounded, cited answers and make the system refuse when the context does not support an answer.
4. Evaluate a RAG pipeline on faithfulness, context precision, and context recall, and read those numbers as a diagnosis.

**Lessons.**
1. The RAG failure taxonomy
2. Query rewriting and multi-query retrieval
3. Reranking and context assembly
4. Grounding, citations, and knowing when to refuse
5. Measuring RAG: faithfulness, context precision, context recall

**Spine increment.** Query rewriting, reranking, and grounded answers with citations — with the RAG eval suite attached.
**Standalone exercise.** Take a RAG pipeline with three seeded defects, diagnose each from the metrics alone, and fix them.

---

### Module 6 — The Agent Loop from Scratch
*60 min · 5 lessons*

Written by hand, no framework. Interviewers explicitly probe whether a candidate has built something agentic rather than something that answers prompts.

**Learning objectives.** Students can:
1. Implement tool calling end to end — schema, dispatch, result formatting, parallel calls — against the raw API.
2. Write an agent loop by hand and explain each control-flow decision inside it.
3. Manage state, memory, and a shrinking context window across a long-running task.
4. Make the loop survive failure: tool errors, malformed calls, step and token budgets, and explicit termination conditions.
5. Execute untrusted or dangerous tool calls inside a sandbox.

**Lessons.**
1. Tool calling: the protocol beneath every agent
2. Writing the agent loop by hand
3. State, memory, and context management
4. Failure recovery, budgets, and termination
5. Sandboxed execution and dangerous tools

**Spine increment.** Hand-rolled agent loop with tools, budgets, and recovery.
**Standalone exercise.** Build a research agent that plans, searches, and writes a sourced brief — and cannot exceed its step budget.

---

### Module 7 — Agent Architectures: LangGraph & MCP
*55 min · 4 lessons*

The same loop, now in the vocabulary the job market screens on.

**Learning objectives.** Students can:
1. Port a hand-rolled loop to LangGraph and name what the framework adds and what it hides.
2. Use checkpointing, interrupts, and human-in-the-loop approval for consequential actions.
3. Evaluate multi-agent patterns — supervisor, handoff, parallel workers — and argue when a single agent is the better design.
4. Build an MCP server exposing their tools and consume third-party MCP servers as a client.

**Lessons.**
1. From loop to graph: porting to LangGraph
2. Checkpointing, interrupts, and human-in-the-loop
3. Multi-agent patterns — and when they're a mistake
4. MCP: consuming and building servers

**Spine increment.** Agent ported to LangGraph, with an MCP server exposing its tools.
**Standalone exercise.** Rebuild the Module 6 research agent as a LangGraph graph with a human approval gate, and compare the two implementations.

---

### Module 8 — Fine-Tuning I: When, and SFT/LoRA
*55 min · 4 lessons*

Starts with the decision, not the technique — because the most valuable fine-tuning skill is knowing when not to.

**Learning objectives.** Students can:
1. Decide between prompting, retrieval, and fine-tuning for a given problem, with a cost and maintenance argument for the choice.
2. Build a supervised dataset from production traces, including deduplication, filtering, and held-out splits.
3. Run LoRA/QLoRA supervised fine-tuning on Colab GPU and push the adapter to Hugging Face.
4. Prove — or disprove — improvement against the Module 3 baseline rather than asserting it.

**Lessons.**
1. The fine-tuning decision: when *not* to
2. Building a dataset from production traces
3. LoRA and QLoRA supervised fine-tuning on Colab
4. Did it work? Measuring against the baseline

**Spine increment.** Domain-tuned small model (LoRA adapter), measured against the frontier baseline.
**Standalone exercise.** Fine-tune a small model to match a strict output format the base model keeps violating, and quantify the improvement.

---

### Module 9 — Fine-Tuning II: Preference Tuning & Distillation
*55 min · 4 lessons*

The half of the training track that shows up in senior postings.

**Learning objectives.** Students can:
1. Construct preference data that encodes a real quality dimension, and explain what SFT cannot teach that preferences can.
2. Run DPO or ORPO on top of an SFT checkpoint and evaluate for both gains and regressions.
3. Distill a frontier model's behavior on a narrow task into a small deployable model, and quantify the cost and latency win.
4. Ship the result: merge or serve adapters, quantize, and account for the quality cost of quantization.

**Lessons.**
1. Preference data: defining what "better" means
2. DPO and ORPO in practice
3. Distilling a frontier model into a small one
4. Shipping adapters: merging, quantizing, serving

**Spine increment.** Preference-tuned and distilled variant, served as an adapter.
**Standalone exercise.** Take an SFT model with a known tone or verbosity problem and correct it with preference tuning without losing task accuracy.

---

### Module 10 — Serving, Deployment & Performance
*60 min · 5 lessons*

Where this audience's existing engineering strength converts directly into hireability.

**Learning objectives.** Students can:
1. Design and build a streaming FastAPI service around an agent, handling concurrency, cancellation, and backpressure.
2. Serve an open-weights model with vLLM behind the same OpenAI-compatible interface used since Module 1.
3. Meet a stated latency and cost SLO using caching, batching, model routing, and context trimming — and show the measurement.
4. Instrument end-to-end tracing and read an agent run's trace to find where time and money went.
5. Package the system as reproducible Docker Compose with externalized config and a CI pipeline.

**Lessons.**
1. Designing the service: FastAPI, async, streaming
2. Serving open-weights models with vLLM
3. Caching, batching, and the latency budget
4. Tracing: seeing inside an agent run
5. Packaging: Docker Compose, configuration, CI

**Spine increment.** Dockerized FastAPI service on a vLLM backend, cached, traced, CI-built.
**Standalone exercise.** Take a working but slow agent endpoint and cut p95 latency by half without losing eval score.

---

### Module 11 — Security, Safety & Reliability
*60 min · 5 lessons*

An agent with tools is an attack surface with a budget. This module is treated as engineering, not compliance paperwork.

**Learning objectives.** Students can:
1. Explain prompt injection — direct and indirect — and why untrusted content plus tools plus exfiltration paths is the dangerous combination.
2. Design tool permissions on least-privilege lines, with sandboxing and approval gates for consequential actions.
3. Handle PII and sensitive output: detection, redaction, filtering, and what to log versus what never to log.
4. Red-team their own agent systematically and turn each successful attack into a regression test.
5. Keep the system inside a cost and reliability envelope: retries with backoff, provider fallback routing, rate limits, and runaway-agent kill switches.

**Lessons.**
1. Prompt injection: direct, indirect, and why it isn't solved
2. Tool permissions, sandboxing, and least privilege
3. PII, output filtering, and compliance basics
4. Red-teaming your own agent
5. Reliability and cost: retries, fallback routing, kill switches

**Spine increment.** Injection defenses, scoped tool permissions, PII filtering, and a cost kill switch — each with a regression test.
**Standalone exercise.** Attack a deliberately vulnerable agent, document every successful exploit, then fix and regression-test each one.

---

### Module 12 — Capstone, Portfolio & Interview Readiness
*55 min · 4 lessons*

The brief's actual goal: a graduate who can apply and hold up under questioning.

**Learning objectives.** Students can:
1. Finish and harden the spine system against a definition of done drawn from real job requirements.
2. Document the work the way hiring panels read it: model card, eval report, architecture decision record, and a README that earns a second look.
3. Deliver a project deep-dive — problem, architecture and why, trade-offs, evaluation metrics, what they would improve next.
4. Work an AI system design round out loud: requirements, retrieval strategy, agent boundaries, eval plan, cost model, failure modes.

**Lessons.**
1. Shipping the capstone
2. Documenting like a hire: model card, eval report, architecture doc
3. The project deep-dive round
4. The AI system design round

**Spine increment.** Finished, documented, public portfolio repository.
**Standalone exercise.** Take three real job postings, map the course's artifacts onto their requirements, and write the resume bullets and the gap list.

---

## 9. Job-Requirement Coverage

| Requirement seen in postings | Covered in |
|---|---|
| RAG architecture, chunking, embeddings, hybrid retrieval, reranking | 4, 5 |
| Vector databases (pgvector, Qdrant, Pinecone-class) | 4 |
| Agents: function calling, tool use, state, multi-agent orchestration | 6, 7 |
| LangChain / LangGraph | 7 |
| MCP | 7 |
| Evaluation, golden datasets, LLM-as-judge, judge calibration, RAGAS | 3, 5 |
| Prompt engineering, structured outputs, prompt caching | 2 |
| Fine-tuning: SFT, LoRA/QLoRA, DPO, distillation | 8, 9 |
| Python, FastAPI, async services, REST | 1, 10 |
| Docker, Docker Compose, CI/CD | 10 |
| Serving and inference optimization (vLLM, batching, caching, quantization) | 9, 10 |
| Observability, tracing, latency and cost monitoring | 1, 10 |
| Prompt-injection defense, guardrails, PII, safety | 11 |
| Cost optimization and model routing | 1, 10, 11 |
| Portfolio, project deep-dive, AI system design interview | 12 |

**Known gaps, accepted deliberately.** Kubernetes and cloud-specific deployment (AWS/Azure/GCP managed AI services) are named in many postings but are out of scope here — this audience already deploys containers, and the course invests that time in AI-specific depth instead. Module 10 notes the porting path. Voice and browser agents, graph databases, and RLHF/RFT training internals are mentioned as landscape in the relevant modules but not taught hands-on.

---

## 10. Assessment Plan (Phase 4 preview)

- One quiz per module, 5–10 questions, mixing multiple choice, short answer, and a practical exercise — weighted toward application and diagnosis rather than recall, per `CLAUDE.md` Rule 5.
- Diagnostic-style questions throughout: *given these eval numbers, what is broken and which layer do you fix?*
- Final assessment: the capstone repository, graded against a published rubric covering retrieval quality, agent robustness, eval rigor, deployment, security posture, and documentation.

---

## 11. Sources

Job-market research informing §4 and §9:

- [AI Engineering Field Guide — role skills (2026 hiring research)](https://github.com/alexeygrigorev/ai-engineering-field-guide/blob/main/role/02-skills.md)
- [AI Engineer Roadmap 2026: From LLM APIs to Production](https://dataskew.io/roadmaps/ai-engineering/)
- [15 AI Engineer Skills Every Hire Should Have in 2026](https://www.ayautomate.com/blog/ai-engineer-skills-2026)
- [The Agentic AI Engineer Roadmap for 2026](https://medium.com/data-science-collective/the-agentic-ai-engineer-roadmap-for-2026-skills-stack-and-order-fc1dfa17948d)
- [AI Engineering Career Path: Complete Guide for 2026](https://www.dataexpert.io/blog/ai-engineering-career-path-complete-guide-2026)
- [Ultimate Guide to AI Engineering Portfolios](https://www.dataexpert.io/blog/ultimate-guide-ai-engineering-portfolios)
- [AI Engineer Resume 2026: RAG, LLMs & MLOps Keywords](https://levstack.io/en/blog/ai-engineer-resume-2026/)
- [How to Transition Into AI Engineering in 2026](https://jobsbyculture.com/blog/transitioning-to-ai-roles-2026)

---

## 12. Next Step

Phase 2 — expand each module into a lesson plan following Concept → Example → Practice → Summary, saved to `curriculum/module-XX-plan.md`.
