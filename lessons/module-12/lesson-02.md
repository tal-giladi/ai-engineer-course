---
id: "12.2"
module: 12
minutes: 13
practice_minutes: 90
prerequisites: ["12.1"]
objectives:
  - "Document the work the way hiring panels read it."
volatility: concept
sources: []
last_verified: "2026-08-24"
---

# 12.2 · Documenting Like a Hire: Model Card, Eval Report, Architecture Doc

Hiring research is consistent on one point that engineers find mildly insulting and should take seriously: **a well-documented project with a clear write-up beats five undocumented notebooks, every time.**

That is not because documentation is intrinsically virtuous. It is because documentation is the only evidence of judgment a reviewer can access in four minutes. They cannot read your code deeply enough to see that you chose pgvector for good reasons. They can read the decision record where you said so.

Four artifacts do the work. You have already written most of them.

---

## Key Concept 1: The README as a four-minute pitch

Structure it to answer the reviewer's questions in the order they ask them:

**1. What it does** — two sentences, plain language, no framework names. *"A documentation QA agent that answers questions about a product's docs with verifiable citations, and refuses when the docs don't cover it."*

**2. Show it** — a short GIF or three screenshots. This is the highest-value element in the file. A reviewer who can see it working reads further; one who cannot, guesses.

**3. Run it** — the exact commands, verified by Lesson 12.1's clean-clone test. Under five minutes, CPU-only path first.

**4. Architecture** — one diagram plus a short paragraph. Enough to understand the pipeline without reading code.

**5. Results** — a table with real numbers. Quality, cost, latency, before and after. This is where you separate from most portfolios.

**6. Known limitations** — four to six honest entries.

**7. Links** — eval report, decision records, model cards, security report.

Two rules that govern all of it.

**Every claim carries a number or gets deleted.** "Improved retrieval quality" is noise. "recall@5 0.61 → 0.87 by switching to structure-aware chunking with contextual headers" is evidence. If you cannot produce the number, either measure it or remove the claim.

**Limitations argue for you, not against you.** Stated limits read as judgment; their absence reads as unawareness. "Untested above ~50k documents; pgvector filtered p95 would likely become the bottleneck" tells a reviewer you understand your own system's edges — and it pre-empts a question they were going to ask anyway.

---

## Key Concept 2: The eval report — the differentiating artifact

This is the document almost no candidate has, and it is worth more than any feature you could add.

It is a running log, appended to since Lesson 3.3, of every change and its measured effect:

```markdown
## 2026-09-14 — Cross-encoder reranking

**Change:** retrieve 50 → rerank 5 (was: retrieve 5 directly)
**Prompt version:** answer:v3 (unchanged) · **Dataset:** golden v1.2 (60 items)

| Stratum | Before | After | Δ |
|---|---|---|---|
| multi_hop | 0.71 | 0.86 | +0.15 |
| ... | | | |
| **Overall** | 0.812 | **0.871** | +0.059 |

Cost $0.0019 → $0.0031 (+63%) · p95 2.1 s → 3.4 s

**Decision:** kept. Multi-hop gain is well outside the judge's noise band
(±0.04, see judge.md) and multi-hop is our second-priority failure mode
per the charter. Cost accepted; Module 10 addresses latency.
```

Four things make an entry good: **the change is isolated**, **results are per stratum**, **cost and latency sit alongside quality**, and **the decision is stated with reasoning tied to the charter**.

Include:

- The baseline and every change since
- Metric definitions and the judge's calibration and trust boundary (Lesson 3.4)
- Cost and latency throughout
- **The changes you reverted**

That last point deserves emphasis. **The reverted entries are the strongest pages in your repository.** "We added semantic caching at threshold 0.90; latency dropped 60% and faithfulness dropped 8 points on near-miss queries; we swept the threshold to 0.97 and kept the win" is direct evidence that your evaluation is real and that you act on it. Almost nobody has this.

---

## Key Concept 3: Model cards

For every fine-tuned artifact from Modules 8 and 9. You wrote the shape in Lesson 8.3 — now make sure each is complete:

- Base model and **exact commit**
- Method and hyperparameters
- Data provenance, with a link to the datasheet
- Evaluation results, with a link to the eval report
- **Intended use**
- **Out-of-scope use**
- **Limitations**

The last three are what make it read as professional, and they are the sections a reviewer looks for to gauge whether you understand what you built.

---

## Key Concept 4: Architecture decision records

You have been writing these since Module 4 without necessarily thinking of them as portfolio material:

- `adr-embedding-model.md` (4.1)
- `adr-vector-store.md` (4.3)
- `adr-agent-architecture.md` (7.3)
- `framework-tradeoff.md` (7.1)
- `adr-finetuning.md` (8.1)
- `adr-distillation.md` (9.3)
- `serving-economics.md` (10.2)

Collect them in `docs/` and link from the README. Each should carry: the decision, the alternatives considered, the evidence, and **the conditions under which you would revisit it**.

That final element is what distinguishes a decision from a preference — and, as Lesson 12.3 will show, it is exactly what the third rung of an interviewer's depth ladder is asking for.

---

## Key Concept 5: Write it up publicly

A public write-up multiplies a project's reach — hiring guidance is explicit that documenting a project and writing about it substantially increases the chance it is seen.

One post, structured like the deep-dive from Lesson 12.3: the problem, the architecture and why, one interesting trade-off in depth, the measurements, and what you would do differently. Link the repository.

The most-read version of this post is usually not the overview — it is **one specific thing you measured that contradicts common advice.** "More retrieved context made our answers worse: here is the sweep" is a post people read and share. Your eval report contains several such findings already.

---

## Worked Example: One README, rewritten

**Before:**

```markdown
# AI Doc Assistant

An AI-powered documentation assistant using RAG and agents.

## Features
- Vector search with embeddings
- LangGraph agent orchestration
- Fine-tuned model support
- Docker deployment
- Observability

## Setup
See requirements.txt
```

A reviewer learns: this person used the standard stack. Nothing about whether it works, how well, or whether they understood any choice they made.

**After:**

```markdown
# Documentation QA Agent

Answers questions about product documentation with verifiable citations,
and refuses when the docs don't cover the question. Runs against a
hosted API or a self-hosted fine-tuned model.

![demo](docs/demo.gif)

## Run it (3 minutes, no GPU required)
    git clone … && cd docs-qa-agent
    cp .env.example .env      # add one API key
    docker compose --profile hosted up
    open http://localhost:8000

## Architecture
![architecture](docs/architecture.png)
Hybrid retrieval (dense + BM25, RRF) → cross-encoder rerank → agent loop
with tool calling → grounded answer with validated citations. Full
tracing, two-tier eval gate in CI. [Decision records](docs/)

## Results
| | Baseline | Final | |
|---|---|---|---|
| Faithfulness | 0.71 | **0.89** | +0.18 |
| Context recall | 0.61 | **0.91** | +0.30 |
| Refusal accuracy (out-of-scope) | 0.42 | **0.86** | +0.44 |
| p95 latency | 8.4 s | **2.9 s** | −66% |
| Cost / request | $0.0094 | **$0.0026** | −72% |

Measured on a 60-item stratified held-out set. Judge calibrated against
human labels (86% agreement, κ=0.71). [Full eval report](eval-report.md)
— includes two changes that were reverted and why.

## Known limitations
- Untested above ~50k documents; pgvector filtered p95 would likely
  become the bottleneck first
- Multi-hop questions spanning more than 3 documents degrade (0.86 → 0.71)
- English only; the embedding model was benchmarked on English docs
- Direct system-prompt extraction succeeds; accepted, no secrets in the
  prompt ([security report](docs/security-report.md))
```

Same system. The second version is evidence; the first is a claim.

Notice the last limitation: an *accepted security finding*, stated in the README. That is a candidate who red-teamed their own system, made a judgment call, and is comfortable showing it. Few things signal seniority faster.

---

## Practice

**1. Rewrite the README** to the seven-section structure. Ruthlessly: every claim carries a number or gets deleted.

**2. Record a demo GIF.** Twenty seconds, showing a real question, tool calls, and a cited answer. Highest-value element in the file.

**3. Draw the architecture diagram.** One image, readable at a glance. Render your LangGraph graph (Lesson 7.1) if it is clear enough, or draw the pipeline by hand.

**4. Assemble `eval-report.md`** from your running log. Verify it contains the baseline, every change with per-stratum results, cost and latency, metric definitions, the judge's trust boundary, and **at least one reverted change**.

**5. Complete every model card**, including intended use, out-of-scope use, and limitations.

**6. Collect the decision records** into `docs/` and confirm each states the conditions under which you would revisit it.

**7. Write the limitations section** — four to six entries, including at least one accepted risk.

**8. Get a cold review.** Swap repositories with someone, or use the rubric in `assets/review-rubric.md`. Four minutes, no explanation from you. Write down every question they could not answer from the documentation, then fix those.

**9. Optional but recommended: write the public post.** Lead with the finding that contradicts common advice.

---

## Visual Aid

*An annotated README wireframe. Each section is a labeled block with two annotations: the reviewer question it answers, and a time budget. Title + one-liner ("what is this?" — 10 s), demo GIF ("does it work?" — 20 s), setup ("can I run it?" — 30 s), architecture ("do they understand systems?" — 45 s), results table ("is any of this real?" — 60 s), limitations ("do they know their own weaknesses?" — 45 s), links ("is there depth here?" — 30 s). A running timer down the side reaches 4:00 at the end. Beside the results block, a callout: "the eval report link is the artifact almost no candidate has."*

---

## Key Takeaways

- **Documentation is the only evidence of judgment a reviewer can access in four minutes.** That is why it outweighs additional features.
- **Structure the README for the reviewer's question order:** what, show, run, architecture, results, limitations, links. Every claim carries a number or gets deleted.
- **The eval report is the differentiating artifact,** and its reverted entries are its strongest pages — direct evidence that your evaluation is real and acted upon.
- **Model cards need intended use, out-of-scope use, and limitations.** Those three sections are what read as professional.
- **Decision records should state when you would revisit the decision** — that is what turns a preference into a decision, and it answers the third rung of the interview depth ladder in advance.
- **Stated limitations argue for you.** Including an accepted security risk in your README signals seniority quickly.

---

## Next

The repository speaks for you until someone asks you to speak for it. That happens in the round hiring research identifies as central — a project deep-dive with follow-up probes, forty-five minutes, where every claim gets pushed until it breaks or holds.

**Lesson 12.3** structures that answer and drills the depth ladder — the five-rung probe most candidates fail at rung three, and which every decision record you wrote in this course exists to answer.
