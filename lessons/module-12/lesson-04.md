# Lesson 12.4 — The AI System Design Round

**Module 12: Capstone, Portfolio & Interview Readiness** · ~13 min
**Objective:** M12-LO4 — Work an AI system design round out loud: requirements, retrieval strategy, agent boundaries, eval plan, cost model, failure modes.

---

## Introduction

Classic system design has a well-known playbook — estimate load, choose a database, shard, cache, handle failure. The AI system design round borrows that shape and adds three concerns that the classic playbook has no vocabulary for: **the system is non-deterministic, quality is a measured property rather than a correctness property, and every request has a variable price.**

Most candidates walk into this round with the classic playbook and stall at the moment someone asks "how would you know it works?"

This lesson gives you a seven-step framework. Its two hardest steps are the two most candidates skip entirely — and they are the two this course has spent twelve modules preparing you to answer.

---

## Key Concept 1: The seven steps

**Step 1 — Clarify requirements.** Five to eight minutes. Ask about:

- **Users and volume** — who, how many, requests per day, arrival pattern
- **Latency tolerance** — interactive or batch? p95 target?
- **The accuracy bar** — what failure rate is acceptable, and what happens when it fails?
- **The cost ceiling** — what can this cost per request or per month?
- **Data sensitivity** — can it leave the building? Multi-tenant?
- **The corpus** — size, format, update frequency, who can write to it

**Asking about the accuracy bar and the cost ceiling immediately signals production experience**, because they are the two questions people who have only built demos do not think to ask.

**Step 2 — Choose the approach.** Prompting, retrieval, fine-tuning, agent — using the Lesson 8.1 decision matrix out loud. "Facts that change weekly means retrieval, not fine-tuning" is a complete answer in one sentence.

**Step 3 — Design retrieval.** Sources and ingestion, chunking strategy and why, embedding model selection *on their corpus*, index choice with the pgvector-vs-purpose-built reasoning, hybrid if the query profile warrants it, reranking, and how updates and deletions propagate.

**Step 4 — Draw the agent boundary.** Which decisions belong to the model and which to your code. What tools exist and their blast radius. Budgets and termination. Whether it should be an agent at all — "a fixed pipeline is more predictable and cheaper here" is often the right answer and reads as judgment.

**Step 5 — Plan evaluation.** Golden dataset with strata, metrics fitted to the task, LLM-as-judge with calibration, CI gating, online sampling. **Most candidates say nothing here.** Two minutes on this is the cheapest differentiation available in the round.

**Step 6 — Model cost and latency.** Do the arithmetic out loud. Calls per request × tokens per call × price, cache hit rate, then a latency budget by stage.

**Step 7 — Name failure modes and defenses.** Hallucination and grounding, prompt injection with the three-part chain, provider outage and fallback, cost runaway and caps, and what degraded mode looks like.

---

## Key Concept 2: The cost model, done out loud

This is a two-minute calculation that very few candidates perform, and it changes how the room perceives you.

```
Per request:
  query rewrite       1 call  ×   300 tok in /  50 out
  retrieval           0 calls (embeddings cached)
  rerank              0 calls (local cross-encoder)
  generation          1 call  × 3,000 tok in / 400 out
  citation validation 0.1 calls (10% sample)
  ─────────────────────────────────────────────
  ≈ 2.1 calls, ≈ 3,700 input / 500 output tokens

At $3/M input, $15/M output:
  ≈ $0.011 + $0.0075 ≈ $0.019 per request

100k requests/day → ~$1,900/day → ~$57k/month
```

Then reason about it: *"That's above a typical budget, so I'd route — a fine-tuned small model for the ~70% simple factual queries at roughly a twentieth of the cost, escalating the rest. That gets it to around $20k/month. If volume is sustained I'd model self-hosting; break-even for this shape is somewhere around 500k requests/month."*

Two things happened there. You produced a number, and you turned it into an architectural decision. That sequence is what the round is testing.

---

## Key Concept 3: The evaluation plan, in two minutes

Equally cheap, equally rare:

> "I'd build a 60-item golden set stratified by failure mode — simple factual, multi-hop, ambiguous, out-of-scope, adversarial phrasing — deliberately over-sampling out-of-scope, because refusal is what degrades silently and never appears in normal traffic.
>
> Metrics: context recall, context precision, and faithfulness, so the metric vector localizes failures — low recall means retrieval, low precision means ranking, low faithfulness with good context means generation.
>
> Faithfulness needs an LLM judge, so I'd calibrate it against 30–50 human-labeled items, state the agreement rate, and not act on differences inside the noise band.
>
> Then a two-tier CI gate — a fast smoke set on every PR, the full set on merge — gating on quality *and* cost. And 10% online sampling in production, because the offline set is a guess about reality and the gap between them is a finding."

That is the whole of Module 3, delivered in under two minutes, and it will be the most substantive thing said in most rounds.

---

## Key Concept 4: The traps

**Jumping to a framework.** "I'd use LangChain and Pinecone" before requirements. Requirements first, always.

**No evaluation plan.** The single most common gap.

**No cost model.** Second most common. In this field cost is a first-class design constraint, not an afterthought.

**Treating the model as deterministic.** Proposing exact-match tests, or a design that assumes the same input yields the same output.

**Ignoring the data pipeline.** Retrieval requires ingestion, updates, deletions, and re-indexing. Candidates draw a vector database and never say how documents get into it or out of it.

**Over-engineering with agents.** Five agents for a task one agent with four tools handles better. Arguing *against* multi-agent, with the serialization and attribution costs named, is a strong signal.

**Ignoring multi-tenancy.** If the corpus has per-customer data, tenant scoping affects retrieval filters, cache keys, and credentials. Raising it unprompted is a strong move.

---

## Key Concept 5: The prompts you will actually get

Four archetypes, each mapping onto this course:

**A documentation or knowledge assistant** — Modules 4, 5. Retrieval-centric.
**A support agent that takes actions** — Modules 6, 7, 11. Agent boundaries and blast radius dominate.
**A high-volume document processing pipeline** — Modules 8, 9, 10. Cost and throughput dominate; batch, not interactive.
**Internal search over mixed sources** — Modules 4, 11. Permissions and multi-tenancy dominate.

Recognizing which archetype you have been given tells you where the interesting constraints are.

---

## Worked Example: A full round, abbreviated

**Prompt:** *"Design an AI assistant for our internal engineering documentation."*

**Clarifying (5 min).** Answers: ~2,000 engineers, ~5,000 queries/day, interactive with a 3 s p95 target, wrong answers are "annoying not dangerous" but confident wrong answers about security procedures are dangerous. Budget "a few thousand a month." Corpus is 40k markdown documents, updated continuously, some restricted to specific teams.

Two things stand out immediately and both came from questions: **restricted documents means permission-aware retrieval**, and **"confident wrong answers about security are dangerous" means refusal behavior is a first-class requirement**, not a nicety.

**Approach.** Retrieval, not fine-tuning — the corpus changes continuously and answers must be citable. Not an agent initially: a fixed pipeline is cheaper and more predictable, and the requirement is lookup, not action. *"I'd add an agent later only if we see multi-step questions in the logs."*

**Retrieval.** Structure-aware chunking on markdown headings with contextual header prefixes. Embedding model benchmarked on their own corpus, not a leaderboard. Hybrid dense + BM25, because engineering docs are full of function names and error codes. Cross-encoder rerank, 50 → 5. **Permission filters applied as pre-filters** — post-filtering would silently collapse recall, and worse, it is a permissions boundary. pgvector if Postgres is already operated; 40k docs is comfortably within range.

**Ingestion.** Webhook on document change, incremental re-index, tombstones for deletions. *"A deleted document that still answers queries is both a quality bug and a permissions bug."*

**Evaluation.** As in Key Concept 3, with a specific addition: *"Given the security-procedure concern, I'd make that its own stratum with a hard floor in CI — refusal accuracy on security topics can never regress, regardless of what the aggregate says."*

**Cost.** ~$0.008/request × 5,000/day ≈ $1,200/month. Inside budget. *"No routing needed initially — I'd rather keep the architecture simple and revisit if volume grows 10×."*

**Failure modes.** Hallucination → grounding plus validated citations. Permissions → pre-filtering plus a cross-tenant regression test. Injection → anyone who can edit a wiki page can plant instructions; no dangerous tools in this design, so risk is bounded to misleading answers, and an egress allowlist if tools are added later. Provider outage → fallback to a secondary, then a retrieval-only degraded mode. Cost → per-request budget and anomaly alerting.

**The curveball:** *"What if we want it to file tickets?"*

> "That changes the risk profile substantially. Filing a ticket is a write action, and now anyone who can edit a wiki page can potentially cause tickets to be filed — untrusted content plus a tool that acts. I'd split the contexts: a retrieval agent reads documents with no write tools and returns a structured summary; a separate action agent holds the ticket tool and only sees that summary. Plus an approval gate above a threshold, per-session rate limits, and a scoped credential that can only create tickets in one project. Ticket creation also needs an idempotency key, because retries and resumes would otherwise duplicate."

That answer is Modules 6, 7, and 11 arriving in forty seconds because the risk chain was already a reflex.

---

## Practice

**1. Run three timed rounds**, 40 minutes each, one per archetype: a documentation assistant, an action-taking support agent, and a high-volume processing pipeline.

**2. For each, produce:** a clarifying-question list (aim for 8, including the accuracy bar and cost ceiling), an architecture sketch, an evaluation plan, a cost model with actual arithmetic, and a failure-mode table.

**3. Say it out loud.** Record yourself. This round is a performance and silent reasoning scores zero.

**4. Time your sections.** Requirements 5–8 min, architecture 15, evaluation 5, cost 3, failure modes 5, buffer for probes. If evaluation and cost are getting squeezed to nothing, that is the habit to fix.

**5. Self-assess** against `assets/system-design-rubric.md`, **weighted toward evaluation and cost** since those are the differentiators.

**6. Practise curveballs.** Have a partner — or your own list — inject one mid-round: "the corpus includes customer-specific documents," "we need it in four languages," "volume is 50× what I said," "it needs to take actions." Each should map to a module you know.

**7. Build a one-page reference card** of the seven steps with your own prompts under each. Review it before interviews; do not read from it during.

**8. Return to `job-map.md`.** Open the file you wrote in Lesson 1.1 and re-tag every requirement. The GAP column from day one is now mostly LEARNING or HAVE — and the ones still marked GAP are your honest next steps.

---

## Visual Aid

*A seven-step reference card, rendered as a single page. Each step is a numbered band with three or four bullet prompts beneath it, and a small tag on the right naming the module that supplies its content: 1 Requirements (accuracy bar · cost ceiling · sensitivity · corpus) — "the two starred questions signal production experience"; 2 Approach [M8]; 3 Retrieval [M4–5]; 4 Agent boundary [M6–7]; 5 **Evaluation** [M3] — highlighted, tagged "most candidates say nothing"; 6 **Cost model** [M1, M9–10] — highlighted, tagged "do the arithmetic out loud"; 7 Failure modes [M11]. A timing column down the right edge shows a 40-minute budget per step.*

---

## Key Takeaways

- **Seven steps:** clarify, choose the approach, design retrieval, draw the agent boundary, plan evaluation, model cost, name failure modes.
- **Asking about the accuracy bar and the cost ceiling in the first five minutes** immediately marks you as someone who has shipped.
- **Evaluation and cost are the two steps most candidates skip** and the two you have the most to say about. Budget time for them explicitly.
- **Do the cost arithmetic out loud and turn the number into a decision.** That sequence is what the round is testing.
- **Arguing against complexity is a strong signal** — a fixed pipeline over an agent, one agent over five, simple architecture until volume justifies otherwise.
- **The curveballs map to modules.** Untrusted content plus a write tool is the Lesson 11.1 risk chain; multi-tenancy is Lessons 4.3 and 11.3.

---

## Course Complete

Twelve modules. One production system: measured before it was built, retrieval that you chose on evidence, an agent you wrote by hand before handing it to a framework, models you tuned and proved, a service that meets a stated SLO, defenses you tested by attacking your own system, and a repository someone can clone and run in three minutes.

Go back to `job-map.md` from Lesson 1.1 — the file you wrote before you knew any of this. The requirements you tagged **GAP** on day one are the ones you have spent twelve modules closing, and the evidence for every one of them is now in your eval report, your decision records, your model cards, and your security report.

What is left is the part this course cannot do for you: apply, and then be the candidate who reaches rung five.
