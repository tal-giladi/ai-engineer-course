---
id: "01.1"
module: 1
minutes: 12
practice_minutes: 30
prerequisites: []
objectives:
  - "Distinguish AI engineering from ML engineering and data science, and map a real job posting onto the modules of this course."
volatility: concept
sources: []
last_verified: "2026-08-24"
---

# 01.1 · What an AI Engineer Actually Does (and What They Don't)

You already know how to build software. You can design a service, containerize it, reason about failure, and argue about database choices. The reason you are here is that a category of job appeared with "AI" in the title, the postings look intimidating, and it is genuinely hard to tell from the outside how much of it is mathematics you never learned and how much is engineering you already do.

Here is the answer up front, so the rest of the course has somewhere to stand: **most of it is engineering you already do.** The gap is real, but it is smaller and far more specific than the job descriptions suggest. This lesson locates the gap precisely, so you spend the next eleven modules closing it rather than studying things that will not matter.

---

## Key Concept 1: Three roles that get confused

The confusion in job postings is not accidental — the titles genuinely overlap and companies use them inconsistently. But three distinct jobs exist underneath.

**The data scientist** answers questions with data. They work in notebooks, run experiments, and produce analyses and recommendations. Their output is usually a decision or a report, not a running system.

**The ML engineer** builds and trains models. They need real depth in statistics and optimization — loss functions, architecture choices, training dynamics. They own model artifacts, training pipelines, and the infrastructure that produces them. When a company needs a custom recommendation model trained on its own data, this is the person.

**The AI engineer** integrates and deploys existing models into products. They rarely train a model from scratch — they call one, retrieve context for it, give it tools, evaluate its behavior, and make the whole thing fast, cheap, safe, and observable. Their output is a running system.

The reason this distinction matters to you specifically: the first two require years of mathematical background you may not have. The third requires distributed systems, API design, data pipelines, observability, and cost discipline — which is your existing job with one unfamiliar dependency added.

> **The one-line version:** ML engineers make models. AI engineers make products out of models.

---

## Key Concept 2: The job is mostly not the model

Analysis of AI engineering job postings finds that roughly **93% of AI-first roles require skills beyond generative AI**. The most common requirements across those postings are Python, cloud platforms, Docker, CI/CD, Postgres, REST APIs, and FastAPI — a stack that describes a competent backend engineer.

Read a posting carefully and you will typically find it decomposes something like this:

- **~40% ordinary software engineering** — services, APIs, data pipelines, deployment, testing, monitoring. You have this.
- **~40% the AI-specific middle** — retrieval, agents, evaluation, prompting, model serving, cost and latency management. This is the gap, and it is what this course teaches.
- **~20% context and judgment** — knowing when a problem needs an LLM at all, what it will cost, and how it fails. This is learnable and is what separates a good hire from a keyword match.

The practical consequence: you are not starting from zero. You are starting from roughly 40%, plus transferable instincts that make the middle 40% land faster than it does for someone coming from a research background.

---

## Key Concept 3: The four pillars

The AI-specific middle is not an unbounded list. Hiring research is consistent that the most-screened skills cluster into four areas, and this course is organized around them:

**Retrieval.** Giving a model the right information at request time. RAG — retrieval-augmented generation — is the single most common pattern in applied LLM roles, appearing in a large share of postings. Modules 4 and 5.

**Agents.** Systems where the model decides what to do next, calls tools, holds state, and recovers from failure. Interviewers probe specifically for whether you have built one. Modules 6 and 7.

**Evaluation.** Measuring whether any of it works. This is repeatedly described as the clearest signal that a candidate has actually shipped LLM systems rather than followed tutorials. Module 3, and then every module after it.

**Production.** Serving, observability, cost, latency, security. Modules 10 and 11 — and the pillar where your existing experience converts most directly.

Modules 8 and 9 cover fine-tuning, which sits slightly apart: it appears in roughly a quarter of postings, so skipping it leaves a visible gap, but it is a smaller part of the job than its prominence in online discussion suggests.

---

## Worked Example: Two postings, annotated

**Posting A — "ML Engineer, Ranking"**

> *Requirements: MS/PhD in CS or related field. Deep understanding of learning-to-rank algorithms. Experience with PyTorch, distributed training, and feature stores. Publications a plus.*

Every line is model science. Training infrastructure, algorithm depth, formal background. This is not the job this course prepares you for, and recognizing that quickly saves you from applying to roles where you would be outmatched on the first technical screen.

**Posting B — "AI Engineer, Platform"**

> *Requirements: 5+ years software engineering. Python, FastAPI, Docker, AWS. Experience building RAG pipelines and agentic workflows. Familiarity with LangChain/LangGraph, vector databases, and evaluation frameworks. Ability to optimize for latency and cost.*

Line by line: *5+ years software engineering* — you have it. *Python, FastAPI, Docker, AWS* — you have it. *RAG pipelines* — Modules 4 and 5. *Agentic workflows* — Modules 6 and 7. *LangChain/LangGraph, vector databases* — Modules 7 and 4. *Evaluation frameworks* — Module 3. *Latency and cost* — Modules 1 and 10.

That is the entire posting, mapped. Nothing on it requires a new degree.

---

## Practice

Find three real job postings for AI engineering roles you would genuinely apply to. Use whichever job board you actually use.

For each posting, create a table with every stated requirement on its own row, tagged:

- **HAVE** — you can speak to this today with a concrete example
- **LEARNING** — this course covers it (note the module)
- **GAP** — neither of the above

Save this as `job-map.md`. Two things will happen. First, the HAVE column will be longer than you expect, which is the point of doing this on day one rather than at the end. Second, you will return to this file in Module 12 and convert it into resume bullets — so keep it.

If a posting comes out mostly GAP, look at whether it is actually an ML engineering role wearing an AI engineering title. That is a useful filter to develop early.

---

## Visual Aid

*Venn diagram — three overlapping circles labeled Software Engineering, Machine Learning, and Product/Systems. The AI engineering region sits in the overlap of software engineering and ML, shaded and containing twelve numbered dots for this course's modules. Modules 1, 10, and 12 sit close to the software engineering edge; Modules 8 and 9 sit closest to the ML edge; Modules 3 through 7 cluster in the center. A caption reads: "The center is the gap. The edges are where your existing experience already reaches."*

---

## Key Takeaways

- **ML engineers build models; AI engineers build products out of models.** The second is a systems job with a probabilistic dependency.
- **Around 93% of AI-first roles require skills beyond generative AI** — Python, Docker, CI/CD, cloud, APIs. You are starting from roughly 40% of the job, not from zero.
- **The gap is four pillars:** retrieval, agents, evaluation, and production. Fine-tuning sits alongside them, smaller than its reputation.
- **Evaluation is the differentiator.** It is the clearest signal that someone has shipped, and it is where most self-taught candidates have nothing to say.
- **Read postings as a decomposition, not a wall.** Most requirements on an AI engineering posting are things you either have or will have by Module 12.

---

## Next

You now know what the job is and roughly where you stand against it. Every posting on your list assumes you can call a model — that part is trivial. Almost none of them assume you can swap one model for another on a Friday afternoon without rewriting your application, and that is a real and immediate engineering decision.

**Lesson 1.2** builds the client that every remaining lab in this course runs on: one interface, three very different backends, and an honest accounting of where the abstraction leaks.
