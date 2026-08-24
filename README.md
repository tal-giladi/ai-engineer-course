# AI Engineer: From Software Engineer to Shipping AI Systems

**12 modules · 53 lessons · 12 quizzes · 1 capstone · ~151,000 words · ~11 hours of instruction**

Built end to end from [`brief/course-brief.md`](brief/course-brief.md). Job-market research conducted before the outline; module order follows hiring demand rather than textbook order.

---

## Start Here

| If you are… | Read |
|---|---|
| Deciding whether to run this course | [`output/course-summary.md`](output/course-summary.md) |
| Assessing curriculum quality | [`output/review-notes.md`](output/review-notes.md) |
| Teaching it | [`curriculum/course-outline.md`](curriculum/course-outline.md), then the module plans |
| Taking it | [`lessons/module-01/lesson-01.md`](lessons/module-01/lesson-01.md) |
| Reading the whole thing in one file | [`output/ai-engineer-course-complete.md`](output/ai-engineer-course-complete.md) |

Use the sidebar to browse module by module, or jump straight to a lesson.

---

## Contents

### Curriculum — `curriculum/`
- `course-outline.md` — audience, outcomes, design principles, module map, job-requirement coverage, sources
- `module-01-plan.md` … `module-12-plan.md` — lesson plans with objectives, Concept→Example→Practice→Summary breakdowns, spine increments, dependencies

### Lessons — `lessons/module-NN/lesson-NN.md`

| Module | Lessons | Title |
|---|---|---|
| 01 | 4 | The AI Engineer Role & a Provider-Agnostic Baseline |
| 02 | 4 | Structured Output & Prompting as Engineering |
| 03 | 5 | Evaluation Foundations |
| 04 | 4 | Retrieval I — Embeddings, Chunking, Vector Stores |
| 05 | 5 | Retrieval II — Production RAG |
| 06 | 5 | The Agent Loop from Scratch |
| 07 | 4 | Agent Architectures — LangGraph & MCP |
| 08 | 4 | Fine-Tuning I — When, and SFT/LoRA |
| 09 | 4 | Fine-Tuning II — Preference Tuning & Distillation |
| 10 | 5 | Serving, Deployment & Performance |
| 11 | 5 | Security, Safety & Reliability |
| 12 | 4 | Capstone, Portfolio & Interview Readiness |

Every lesson: introduction, key concepts, worked examples with real numbers, practice, described visual aid, key takeaways, transition hook.

### Assessments — `assessments/`
- `module-NN-quiz.md` — 7–10 questions each with full answer keys and explanations. Weighted toward diagnosis and applied judgment rather than recall.
- `capstone-rubric.md` — 100-point rubric across six dimensions, plus a grader's four-minute pass.

### Assets — `assets/`
- `review-rubric.md` — four-minute cold review of a portfolio repository
- `deep-dive-rubric.md` — 50-point scoring for a mock project deep-dive, including the depth ladder
- `system-design-rubric.md` — 50-point scoring for a mock AI system design round
- `exercise-specs.md` — build specifications for the eight supplied exercise artifacts

---

## The Spine Project

Every module ships a piece of one system — a production agentic RAG service — and a self-contained exercise on a different problem.

```
M1  gateway + cost/latency telemetry
M2  typed contracts, bounded repair, versioned prompts
M3  golden dataset, calibrated judge, CI gate          ← the baseline everything is measured against
M4  hybrid retrieval over a chunked, metadata-rich index
M5  rewriting, reranking, grounding, validated citations, refusal
M6  hand-written agent loop with budgets and a sandbox
M7  LangGraph port, checkpointing, MCP server
M8  domain-tuned LoRA adapter, honestly evaluated
M9  preference-tuned + distilled variant, served
M10 streaming FastAPI service, vLLM, tracing, SLO met
M11 threat model, hardened tools, red-team regression suite
M12 documented portfolio repository
```

---

## Before First Delivery

Two items from the review, in priority order:

1. **Build `m5-rag-triage` and `m11-vulnerable-agent`** — specified in `assets/exercise-specs.md`. The course is teachable without them, but five practicals lose their diagnostic bite. See `output/review-notes.md` §2.
2. **Pilot Module 6 and time it.** It is the peak-difficulty module and the most likely stall point.

---

## Known Limitations

Stated deliberately — the same principle the course teaches about README limitation sections.

- Kubernetes and cloud-managed AI services are out of scope by design
- English-centric; multilingual retrieval is one benchmark axis in Lesson 4.1 and not developed further
- Multi-modal not covered
- Eight supplied exercise artifacts are specified, not built
- Job-market research dated August 2026; re-verify annually
