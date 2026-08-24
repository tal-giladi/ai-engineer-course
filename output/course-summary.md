# AI Engineer: From Software Engineer to Shipping AI Systems

**Course summary — the document to hand a stakeholder, a learner deciding whether to enroll, or a reviewer assessing the curriculum.**

---

## In One Paragraph

A twelve-module, ~11-hour hands-on course that takes an experienced software developer with no AI background to the point of applying credibly for professional AI engineer roles. It is built backwards from what 2026 job postings and interview processes actually screen for, not from a textbook syllabus. Students build one production agentic RAG service incrementally across every module — measured before it is built, retrieval chosen on evidence, an agent written by hand before being handed to a framework, models fine-tuned and proven, served under a stated SLO, defended by attacking it themselves — and finish with a repository a reviewer can clone and run in three minutes, plus the artifacts to defend every claim in it.

---

## Who It Is For

Working software engineers — backend, platform, full-stack — moving into AI engineering. They can already design a system, containerize it, and reason about failure. They are beginners at this specific stack, not at engineering.

**Assumed:** Python, HTTP APIs, Docker, system architecture, basic ML/LLM literacy, experience with coding agents.
**Not assumed:** PyTorch fluency, GPU experience, transformer internals, any prior production LLM work.

**Environment:** local Docker, a paid Hugging Face account, a paid Google Colab account (GPU for Modules 8–9), and API credit with one hosted provider.

---

## What a Graduate Can Do

1. Build a production retrieval system — chunking, hybrid search, reranking, grounded citations — and diagnose its failures by category rather than by guesswork.
2. Build an agent that plans, calls tools, holds state, and recovers from failure, by hand and in LangGraph, and expose or consume capabilities over MCP.
3. Measure any LLM system with a golden dataset, task-fitted metrics, and a calibrated judge, and gate changes on a regression suite in CI.
4. Decide between prompting, retrieval, and fine-tuning with a cost argument — and run SFT, preference tuning, and distillation end to end when fine-tuning is the answer.
5. Ship the system as a dockerized, streaming, traced, cost-instrumented service meeting a stated latency and cost SLO.
6. Defend it against prompt injection, tool abuse, and data exfiltration, and keep it inside a cost ceiling under failure.
7. Present the work the way hiring panels ask for it, and work an AI system design round out loud.

---

## Structure

**12 modules · 53 lessons · ~11 hours of instruction · one capstone**

| # | Module | Lessons | Focus |
|---|---|---|---|
| 1 | The AI Engineer Role & a Provider-Agnostic Baseline | 4 | The job, one interface, token/cost/latency mechanics, telemetry |
| 2 | Structured Output & Prompting as Engineering | 4 | Typed contracts, constrained generation, bounded repair, versioned prompts |
| 3 | **Evaluation Foundations** | 5 | Golden datasets, metrics, judge calibration, CI gating |
| 4 | Retrieval I — Embeddings, Chunking, Vector Stores | 4 | Benchmarking on your own corpus, chunking, pgvector vs Qdrant, hybrid |
| 5 | Retrieval II — Production RAG | 5 | Failure taxonomy, rewriting, reranking, grounding, RAG metrics |
| 6 | The Agent Loop from Scratch | 5 | Tool calling, the loop by hand, state, budgets, sandboxing |
| 7 | Agent Architectures — LangGraph & MCP | 4 | Porting to a graph, checkpointing, multi-agent, MCP |
| 8 | Fine-Tuning I — When, and SFT/LoRA | 4 | The decision, datasets from traces, LoRA on Colab, honest evaluation |
| 9 | Fine-Tuning II — Preference Tuning & Distillation | 4 | DPO/ORPO, reward hacking, distillation economics, shipping adapters |
| 10 | Serving, Deployment & Performance | 5 | Streaming service, vLLM, latency budgets, tracing, packaging |
| 11 | Security, Safety & Reliability | 5 | Injection, permissions, PII, red-teaming, cost controls |
| 12 | Capstone, Portfolio & Interview Readiness | 4 | Shipping, documenting, the deep-dive, the system design round |

Every module ends with running code, a measured result, and a self-contained exercise on a different problem.

---

## Five Design Decisions

**1. Evaluation comes third, before anything worth evaluating exists.** Hiring research is consistent that eval literacy is the clearest signal a candidate has shipped LLM systems rather than followed tutorials — and it is the question most candidates cannot answer. Placing Module 3 before retrieval and agents means every subsequent improvement is *measured* rather than asserted, and students build the reflex of establishing a baseline first.

**2. Mechanism before framework.** Students hand-roll the agent loop, the retrieval pipeline, and the eval harness before touching LangGraph or RAGAS. Frameworks are then taught as a mapping onto something already understood — so when the abstraction leaks in production, it is debuggable. This also produces the strongest available interview answer: *"I wrote it myself first; here is what the framework adds and what it hides."*

**3. Provider-agnostic by construction.** Every lab speaks an OpenAI-compatible API through a gateway. A frontier model, a Hugging Face endpoint, and a locally-served fine-tuned model are a config change apart. The seam introduced in Lesson 1.2 pays off in Modules 9, 10, and 11 — and it is how mature teams actually build.

**4. Every abstract concept gets a measured example.** Reranking is not a definition; it is a specific document moving from rank 23 to rank 2, with the latency cost shown. The course contains no claim a student cannot reproduce.

**5. Production instincts throughout.** Cost, latency, failure handling, and observability are introduced in Module 1 and carried in every module — because that is what interviewers probe for, and because retrofitting them is how LLM projects become unexplainable.

---

## The Spine Project

One system, built incrementally: **a production agentic RAG service over technical documentation.**

Gateway with cost accounting (M1) → typed contracts (M2) → eval harness and CI gate (M3) → hybrid retrieval (M4) → reranking, grounding, citations, refusal (M5) → hand-written agent loop (M6) → LangGraph + MCP server (M7) → domain-tuned model (M8) → preference-tuned and distilled variant (M9) → dockerized FastAPI service with tracing (M10) → threat model, hardened tools, red-team suite (M11) → documented portfolio repository (M12).

**Assessment:** the capstone repository, graded against a published 100-point rubric across reproducibility, evaluation rigor, retrieval quality, agent robustness, production readiness, and security posture. Evaluation rigor is the heaviest dimension at 25 points, because it is the strongest hiring signal.

---

## Job-Requirement Coverage

| Requirement | Modules |
|---|---|
| RAG, chunking, embeddings, hybrid retrieval, reranking | 4, 5 |
| Vector databases (pgvector, Qdrant, Pinecone-class) | 4 |
| Agents: function calling, state, multi-agent orchestration | 6, 7 |
| LangChain / LangGraph · MCP | 7 |
| Evaluation, golden datasets, LLM-as-judge, RAGAS | 3, 5 |
| Prompt engineering, structured outputs, caching | 2 |
| Fine-tuning: SFT, LoRA/QLoRA, DPO, distillation | 8, 9 |
| Python, FastAPI, async services | 1, 10 |
| Docker, Compose, CI/CD | 10 |
| Serving and inference optimization (vLLM, quantization) | 9, 10 |
| Observability, tracing, cost monitoring | 1, 10 |
| Prompt-injection defense, guardrails, PII | 11 |
| Cost optimization and model routing | 1, 9, 10, 11 |
| Portfolio, deep-dive, AI system design interview | 12 |

---

## Stated Limitations

Included deliberately, on the same principle the course teaches about README limitation sections.

- **Kubernetes and cloud-managed AI services are out of scope.** This audience already deploys containers; the time is invested in AI-specific depth. Module 10 notes the porting path.
- **English-centric.** Multilingual retrieval appears as an embedding-selection axis in Lesson 4.1 and is not developed further. A real limitation for international cohorts.
- **Multi-modal is not covered.** The strongest candidate for a future module.
- **Voice agents, browser agents, graph databases, and RLHF/RFT internals** are covered as landscape where relevant, not taught hands-on.
- **Eight supplied exercise artifacts are specified but not yet built** (`assets/exercise-specs.md`). The course is teachable without them; five practicals lose diagnostic bite until they exist. See `output/review-notes.md` §2.
- **The job-market research is dated August 2026** and should be re-verified annually. Stale percentages would be a poor look in a course about evidence-based decisions.

---

## Materials

```
brief/          course brief
curriculum/     outline + 12 module plans
lessons/        53 lessons across 12 module directories
assessments/    12 quizzes with answer keys + capstone rubric
assets/         3 scored rubrics + exercise build specs
output/         review notes, this summary, compiled course
reference/      (unused — research is cited inline in the outline)
```

**Roughly 155,000 words of instructional material.**

---

## Recommended Delivery

Self-paced, one module per session, with the spine project carried between sessions. Modules 8 and 9 need Colab GPU access. Module 6 is the peak-difficulty module and the most likely stall point — pilot it and time it before scaling.

A cohort variant works well for Module 12, where the cold review and the mock deep-dive both benefit from a second person.
