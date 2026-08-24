# Module 3 Quiz — Evaluation Foundations

**10 questions · ~35 min plus the practical**

This is the most heavily probed module in interviews. The questions are weighted accordingly.

---

## Q1 — Multiple choice

A developer changes a prompt, tests it on three examples that all improve, and ships it. Held-out evaluation later shows overall quality dropped from 0.85 to 0.75.

What is the primary mechanism?

A. The model was updated between the two runs
B. Selection bias — the three examples were the ones that motivated the change
C. The held-out set is too small to be reliable
D. Prompt changes are inherently unpredictable

---

## Q2 — Diagnosis

A classifier reports **accuracy 0.88** against a 60-item stratified set. Per class:

| Class | Support | Precision | Recall |
|---|---|---|---|
| billing | 22 | 0.95 | 0.95 |
| technical | 20 | 0.90 | 0.90 |
| account | 13 | 0.85 | 0.85 |
| escalate | 5 | 0.50 | 0.20 |

(a) Should this ship? Why?
(b) What does the aggregate conceal, and why does it barely move it?
(c) Name **two** actions, one about the model and one about the dataset.

---

## Q3 — Short answer

A team builds a 5,000-item evaluation set by scraping production logs. A colleague argues that 60 hand-built stratified items would be more useful.

Give the strongest argument for the 60-item set, and then state one genuine limitation of it that the 5,000-item set does not have.

---

## Q4 — Applied judgment

For each task, name the metric you would report and one thing it hides:

(a) Extracting seven fields from invoices
(b) Routing tickets into four categories, one of which is rare and critical
(c) Retrieving documents for a RAG pipeline
(d) Generating explanatory answers about documentation

---

## Q5 — Diagnosis

You calibrate a faithfulness judge against 40 human-labeled items:

```
Agreement with human labels:              61%
Correlation with human score:             0.42
Correlation with answer word count:       0.79
```

(a) What is this judge actually measuring?
(b) What caused it?
(c) What is the fix — and is it a model change or a prompt change?
(d) Why does this matter beyond evaluation? Name a downstream consequence.

---

## Q6 — Multiple choice

Which judge design is most reliable?

A. "Rate the quality of this answer from 1 to 10."
B. "Here are two answers, A and B. Which better satisfies the rubric? Explain your reasoning, then give a verdict."
C. "Score this answer on faithfulness, relevance, tone, and completeness, from 1 to 5 each."
D. "Give a verdict, then explain your reasoning."

---

## Q7 — Design

Your CI eval gate is failing roughly one run in four with no code change. Engineers are asking to disable it.

(a) What is the underlying cause?
(b) What should the threshold policy be instead?
(c) Where does the number in your policy come from?

---

## Q8 — Diagnosis

A CI gate reports:

```
Overall: 0.847 → 0.812  (−0.035, within band)

  simple_factual   (12)  0.93 → 0.94   +0.01
  multi_hop        (10)  0.86 → 0.85   −0.01
  out_of_scope      (8)  0.88 → 0.38   −0.50
  adversarial       (6)  0.74 → 0.76   +0.02

Changed: prompts/answer.md v3 → v4
```

(a) A gate on the aggregate alone would have passed this. Why?
(b) What almost certainly changed in v4?
(c) What gate design caught it, and what additional control should exist for this specific behavior?

---

## Q9 — Short answer

You have an eval report showing your system at 0.89 faithfulness on your golden set. Your judge agrees with human labels 86% of the time.

State precisely what this licenses you to claim, and one thing it does **not**.

---

## Q10 — Practical

Take the supplied mediocre summarizer in `exercises/m3-summarizer-eval/`. It has a specific weakness.

Build an evaluation that exposes it, prove the weakness with numbers, fix it, and prove the fix.

**Deliverable:** dataset, metrics, before/after report, and a paragraph on why the first metric you tried failed to reveal the problem.

---
---

# Answer Key

## Q1 — **B**

The three examples were almost certainly the ones that motivated the change — that is manual overfitting. Compounding it: effects are non-monotonic (a change helping one case can harm another), and regressions appear in categories the developer was not examining.

**A** is possible but not the primary mechanism. **C** is wrong — 60 stratified items detected a 10-point drop, which is well outside noise. **D** is defeatist and false; the effect is measurable, which is the point.

## Q2

**(a) No.** The `escalate` class — where a mistake means an angry customer waits days — has 20% recall. Four of five escalations are misrouted.

**(b)** The aggregate is a weighted average, and `escalate` is 5 of 60 items (8%). A catastrophic score in a small class barely moves it. This is the structural reason aggregates must never be reported alone.

**(c)**
- **Model action:** examine the confusion matrix to see where escalations go. If three of four land in `technical`, the prompt does not distinguish "a technical problem" from "a technical problem already reported twice" — a specific, fixable finding.
- **Dataset action:** grow the `escalate` stratum. With 5 items, the difference between 0.20 and 0.40 is one item; the metric has no resolution.

## Q3

**Strongest argument for 60 stratified:** a scraped set is dominated by whatever was easy to collect, which is the common case. 4,800 simple lookups tell you repeatedly that simple lookups work. The failures you need to find — refusal, adversarial phrasing, near-misses — are absent or vanishingly rare, so the large set is precise about the thing you were not worried about.

**Genuine limitation of the small set:** resolution. A 60-item set cannot reliably detect a 2% change; differences below roughly 5–8 points are inside noise. The 5,000-item set could detect smaller effects.

**The professional response** is not to inflate the small set with easy items but to *know your resolution and refuse to act inside it* — which becomes the CI threshold policy in Q7.

## Q4

**(a) Extraction → per-field precision/recall/F1.** Hides: an aggregate of 0.91 across seven fields can be six at 0.98 and one at 0.42, and the broken one may be what downstream code depends on.

**(b) Classification → per-class precision and recall plus a confusion matrix.** Hides: aggregate accuracy conceals the rare critical class entirely (see Q2). The matrix is the actionable part — it names which classes are confused.

**(c) Retrieval → recall@k and MRR.** recall@k hides ranking quality (the answer at rank 9 counts the same as rank 1); MRR hides whether it was found at all. Report both.

**(d) Generative → checkable requirements plus rubric or pairwise judge scores.** Hides: mechanical checks say nothing about whether the answer is good; judge scores need calibration before they mean anything.

**Any answer** should also mention cost, latency, and refusal rate alongside — those are reported regardless of task shape.

## Q5

**(a)** It is measuring **length**. Correlation with word count (0.79) is nearly double its correlation with human judgment (0.42). It is a verbosity detector wearing a faithfulness label.

**(b)** The rubric had no anchors — "rate faithfulness from 1 to 5" gives the model nothing concrete to assess, so it falls back on a proxy: apparent thoroughness, which reads as length.

**(c)** A **prompt change**, not a model change. Add concrete anchors, decompose the procedure ("for each factual claim, determine whether the context supports it"), and state explicitly that length is not a factor. Typical result: agreement 61% → 86%, length correlation 0.79 → 0.11.

**(d) Downstream consequence:** every optimization made on this judge's guidance was selecting for longer answers. Worse — if this judge is used to label preference pairs in Module 9, verbosity gets trained into model weights, and it will show up as latency and cost, not as an eval failure.

## Q6 — **B**

Pairwise comparison with reasoning before the verdict.

- **B** is correct on three counts: relative judgment is far more reliable than absolute scoring; reasoning before verdict genuinely conditions the answer because generation is sequential; one dimension per call keeps the signal separable.
- **A** — unanchored 1–10 scales produce unstable scores clustering around 7–8.
- **C** — multiple dimensions in one call collapse toward each other; a judge that thinks an answer is good rates it well on everything.
- **D** — verdict before reasoning makes the reasoning post-hoc rationalization: cost without accuracy.

**B should also be run with position swapping** to control position bias.

## Q7

**(a)** The system is non-deterministic (Lesson 1.3), the judge has measured disagreement with humans, and a 60-item set has limited resolution. Runs vary by a few points with no code change. A gate on an exact value flaps.

**(b)** Gate on a **band relative to the committed baseline**, with:
- an overall band derived from the judge's noise
- **wider per-stratum bands** (smaller n, worse resolution)
- **absolute hard floors** for non-negotiable behaviors like refusal accuracy
- a cost ceiling gated alongside quality

**(c)** From the judge calibration in Lesson 3.4: the measured agreement rate and mean absolute difference produce a noise band, and the policy is "do not act on differences inside it." The number is measured, not chosen.

## Q8

**(a)** The overall drop is 0.035, inside the noise band. The out-of-scope stratum is 8 of 44 items shown; a 50-point collapse there is diluted to near-nothing in a weighted average.

**(b)** v4 almost certainly added an instruction toward directness or brevity — something like "be concise and direct, do not pad your answers." That suppresses hedging, and hedging was the mechanism by which the model declined to answer questions the documentation did not cover. Instructed toward directness, it began answering out-of-scope questions confidently.

**(c)** **Per-stratum thresholds** caught it. The additional control that should exist is an **absolute hard floor** on refusal accuracy — a behavior that must hold regardless of what the baseline or the aggregate says.

## Q9

**Licenses you to claim:** on this 60-item stratified set, with this judge, at this prompt and model version, faithfulness measured 0.89 — and that differences smaller than the judge's noise band are not meaningful.

**Does not license you to claim (any one):**
- That production faithfulness is 0.89. Your golden set is a guess about reality; the offline/online gap is routinely several points, and Lesson 10.4 exists to measure it.
- That the system is 89% correct. Faithfulness measures support by context, not truth.
- That a competing system scoring 0.87 is worse — that difference may be inside the noise band.
- That it will still be 0.89 after a model version update.

**Full credit** requires naming the scope conditions (this set, this judge, this version), not just the number.

## Q10 — Practical rubric

| Criterion | Full credit |
|---|---|
| Weakness identified | Correctly identifies that the summarizer drops the final third of long documents |
| Stratification | Dataset includes a long-document stratum; the weakness is invisible without it |
| Metric choice | First metric attempted plausibly failed (e.g. ROUGE, or an aggregate) and the write-up explains why |
| Proof | Before/after numbers, per stratum |
| Reflection | Explains the mechanism by which the first metric hid the problem |

**The reflection paragraph is the graded part.** "ROUGE looked fine because the summary overlapped heavily with the first two-thirds of the document, and the aggregate hid it because only 6 of 60 items were long" is the target answer.
