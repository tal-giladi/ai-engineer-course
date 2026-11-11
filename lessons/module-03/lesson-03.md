---
id: "03.3"
module: 3
minutes: 12
practice_minutes: 75
prerequisites: ["03.2"]
objectives:
  - "Select task-appropriate metrics and explain why accuracy is usually the wrong one."
volatility: concept
sources: []
last_verified: "2026-08-24"
---

# 03.3 · Metrics That Fit the Task

You have 60 labeled examples. The natural next move is to run the system, count how many it got right, and divide. That number is accuracy, and it is almost always the wrong metric — not slightly wrong, but wrong in a way that actively conceals the failures you built the dataset to find.

This lesson is about picking metrics from the shape of the task, reading them as distributions rather than scores, and producing the baseline that every remaining module measures against.

---

## Key Concept 1: Why accuracy misleads

Three reasons, and they compound.

**Most LLM tasks have no single correct string.** For "explain the difference between these two endpoints," there are hundreds of correct answers. Exact match scores zero on all of them. This alone rules accuracy out for generative work.

**Aggregates hide categories.** This is the serious one. An aggregate is a weighted average, and a small stratum with a catastrophic score barely moves it. A system at 0.88 overall can be failing 80% of refusals — and refusals may be the failure that gets the system pulled from production.

**Class imbalance distorts it.** If 85% of your inputs are one class, a model that always predicts that class scores 0.85 and has learned nothing.

The replacement is not one better metric. It is a **metric vector reported per stratum**, plus cost and latency alongside — because a quality gain that triples cost is a decision, not a win, and your charter's priority ordering already tells you how to resolve it.

---

## Key Concept 2: Pick from the task shape

**Extraction** — pulling structured fields from text.

Use **per-field precision, recall, and F1**, plus **exact match after normalization** (case, whitespace, date format). Per-field is essential: an aggregate of 0.91 across seven fields can be six fields at 0.98 and one at 0.42, and the broken one may be the one downstream code depends on.

**Classification** — assigning a label.

Use **per-class precision and recall** and a **confusion matrix**. The matrix is what tells you *which* classes get confused, which is the actionable part — "urgent is being classified as normal" points at a fix; "accuracy is 0.84" does not.

**Retrieval** — finding relevant documents.

- **recall@k** — is the needed document in the top k? This is the ceiling on everything downstream: what retrieval misses, generation cannot use.
- **MRR** — how highly is the first relevant document ranked? Sensitive to ranking quality.
- **nDCG@k** — graded relevance with position discounting. Use when documents have degrees of relevance rather than binary.

You built the dataset for these in Lesson 3.2 and you will use them heavily in Module 4.

**Generative** — open-ended text.

Mechanical metrics mostly fail here. Use **checkable requirements** (`must_contain`, `must_not_contain`, `must_cite` from your dataset — these are cheap, deterministic, and catch a surprising amount), plus **rubric scoring** and **pairwise preference**, which need a judge. That is Lesson 3.4.

**Agentic** — multi-step trajectories, from Module 6.

**Task success rate** (did it accomplish the goal), **steps to completion**, **tool selection accuracy**, and **cost per successful task**. Note that the final answer being right is not sufficient — an agent that reaches the right answer in 19 steps and $2.40 has failed differently.

---

## Key Concept 3: The metrics that look rigorous and are not

**BLEU and ROUGE** measure n-gram overlap with a reference. They come from machine translation and summarization research, they are easy to compute, and they appear in a lot of tutorials.

They are poor proxies for whether an answer is *right*. A response that says the opposite of the reference while reusing its vocabulary scores well. A correct answer phrased differently scores badly. For most LLM engineering work — QA, agents, extraction — they add a number without adding information, which is worse than having no number, because teams act on them.

Use them only where n-gram overlap is genuinely what you mean. That is rare.

**Perplexity** measures how surprised a model is by text. It is a training diagnostic, not a task metric. A model can have excellent perplexity and be useless at your task.

---

## Key Concept 4: Quality is not the only axis

Every evaluation in this course reports three things together:

**Quality** — the metric vector for your task.
**Cost** — per request, from your Lesson 1.4 telemetry.
**Latency** — p50 and p95, per request, end to end.

And one composite that matters more than any of them individually: **cost per successful task**. A system at 60% success and $0.02 per request costs $0.033 per useful answer. A system at 85% success and $0.028 per request costs $0.033 per useful answer too — identical, despite the second looking more expensive per call. Without this metric you will optimize the wrong one.

Add **refusal rate** as a fourth, tracked in both directions: refusals on answerable questions (over-refusal) and answers to unanswerable ones (missed refusal). These move in opposite directions and a single number hides the trade.

---

## Worked Example: 0.88 that should not have shipped

A support-ticket classifier, evaluated on the 60-item stratified set. Headline result:

```
Accuracy: 0.88
```

Comfortably above the 0.85 target. Ship it.

Now per class:

| Class | Support | Precision | Recall |
|---|---|---|---|
| billing | 22 | 0.95 | 0.95 |
| technical | 20 | 0.90 | 0.90 |
| account | 13 | 0.85 | 0.85 |
| **escalate** | **5** | **0.50** | **0.20** |

The `escalate` class — the one where a mistake means an angry customer waits three days — has 20% recall. Four out of five escalations are being misrouted.

It barely dented the aggregate because it is 8% of the set. And notice: with 5 items, the difference between 0.20 and 0.40 is one item, which is exactly the resolution problem from Lesson 3.2. The right response is both to fix the classifier *and* to grow that stratum.

The confusion matrix says where they went:

| | pred: billing | technical | account | escalate |
|---|---|---|---|---|
| **true: escalate** | 0 | 3 | 1 | 1 |

Three of four misroutes went to `technical`. That is a specific, fixable finding — the prompt does not distinguish "a technical problem" from "a technical problem the customer has already reported twice." An aggregate accuracy of 0.88 contains none of that.

**Cost and latency, for completeness:**

| | Quality (macro F1) | Cost/request | p95 latency | Cost per success |
|---|---|---|---|---|
| Baseline | 0.80 | $0.0021 | 1.8 s | $0.0024 |

That last row is what Module 4 onward will be compared against.

---

## Practice

**1. Choose your metrics.** Against your Lesson 3.1 charter and your task shape, pick the metric vector. Write down for each: what it measures, what it hides, and why you chose it over the alternative. Two or three sentences each in `spine/eval/metrics.md`.

**2. Implement them** in `spine/eval/metrics.py`. Every function takes predictions and labels and returns per-stratum results, never a single scalar:

```python
def evaluate(preds, labels, strata) -> EvalResult:
    """Returns {metric: {stratum: value}} plus an 'overall' key.
    Never return a bare float."""
```

Include the mechanical checks from your dataset — `must_contain`, `must_not_contain`, `must_cite`. They are free and they catch real failures.

**3. Wire in cost and latency.** Pull from your `CallRecord` JSONL, grouped by `request_id`. Compute cost per successful task.

**4. Run the baseline.** Against your current pipeline — which is still just a model call with a prompt, and that is fine, that is the point of a baseline. Produce the full table: every metric, every stratum, plus cost and latency.

**5. Commit it as `eval-report.md`.** Structure it as: date, system version, prompt version, dataset version, the table, and a short "what this tells us" paragraph. Every module from here appends to this file. By Module 12 it is the single most differentiating artifact in your portfolio.

---

## Visual Aid

*A decision tree. Root: "What shape is the task?" Four branches — Extraction, Classification, Retrieval, Generative — each leading to a box of recommended metrics. Under each box, a red-bordered sub-box headed "what this hides": under Classification, "aggregate accuracy hides small critical classes"; under Retrieval, "recall@k hides ranking quality — pair with MRR"; under Extraction, "aggregate F1 hides one broken field"; under Generative, "mechanical checks hide everything soft — needs a judge (3.4)." A separate lane along the bottom, spanning all branches, reads: "Always, regardless of shape: cost/request, p50/p95 latency, cost per successful task, refusal rate (both directions)."*

---

## Key Takeaways

- **Accuracy is usually wrong** — most tasks have many correct answers, aggregates conceal small critical strata, and class imbalance inflates it.
- **Pick metrics from the task shape:** per-field F1 for extraction, per-class precision/recall plus a confusion matrix for classification, recall@k and MRR for retrieval, checkable requirements plus a judge for generative, task success and cost-per-success for agents.
- **BLEU, ROUGE, and perplexity look rigorous and mostly are not.** A number without information is worse than no number, because people act on it.
- **Report a vector, per stratum, with cost and latency.** Cost per successful task is the metric that survives to your interview.
- **The baseline is the deliverable.** Everything in Modules 4–11 is measured against the table you produce today.

---

## Next

Your metric vector handles everything mechanically checkable. It cannot answer the question that matters most for a documentation assistant: *is this answer actually good?* — well-reasoned, appropriately hedged, genuinely responsive to what was asked.

**Lesson 3.4** uses a model to score that, and then does the part almost everyone skips: proves the model's scores agree with a human's, and states exactly how far they can be trusted.
