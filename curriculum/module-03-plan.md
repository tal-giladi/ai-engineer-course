# Module 3 Plan — Evaluation Foundations

**Duration:** 60 min · **Lessons:** 5 · **Depends on:** Modules 1, 2
**Outline reference:** `curriculum/course-outline.md` §8, Module 3

---

## Prerequisites

**From Module 1:** `LLMClient` with telemetry.
**From Module 2:** typed contracts and versioned prompts — the things this module will measure.
**From the student:** test-suite and CI experience. The mental model being transferred is "unit tests for a non-deterministic function," which this audience already half-owns.

---

## Why This Module Is Here and Not Later

This is the most consequential ordering decision in the course. Hiring research is consistent that eval literacy — golden datasets, LLM-as-judge, judge calibration, regression gating — is the single clearest signal that a candidate has shipped LLM systems rather than watched tutorials, and that a RAG system without evaluation is a demo, not a product. Placing this module before retrieval and agents means every subsequent improvement in the course is *measured* rather than asserted, and students build the reflex of establishing a baseline before touching the system.

---

## Module Learning Objectives

**M3-LO1** — Build a golden dataset for a task — sourcing, labeling, sizing, and splitting it — and justify every choice.
**M3-LO2** — Select task-appropriate metrics and explain why accuracy is usually the wrong one.
**M3-LO3** — Implement LLM-as-judge, calibrate the judge against human labels, and quantify how far to trust it.
**M3-LO4** — Wire a regression suite into CI that blocks a prompt or model change on measured quality loss.

---

## Lesson 3.1 — Why You Evaluate Before You Build

*10 min · serves M3-LO1, M3-LO2*

**Concept.** The vibes trap: without a baseline, every change feels like an improvement, because the examples a developer checks are the ones they were already thinking about. Establish the discipline — baseline first, one change at a time, measure, keep or revert. Introduce the three evaluation surfaces and when each applies: **offline** (a fixed dataset, run in CI, the subject of this module), **online** (production traffic, sampled and scored, Module 10), and **human** (expensive, small, the ground truth that calibrates the other two). Name the failure this prevents: the six-week project where nobody can say whether it works, which is also the project that collapses under interview questioning.

**Example.** A prompt "improvement" that raises quality on the three examples the developer tried and lowers it on the held-out set of sixty — with the numbers shown, then the diff explained.

**Practice.** Students write an evaluation charter for the spine project: what the system is for, what a good answer looks like in one paragraph, what an unacceptable answer looks like, and which three failure modes would matter most to a user. One page, no code. This document drives every later choice in the module.

**Summary.** Baseline before change, one variable at a time, held-out data. Without those three, you have opinions.

**Key terms:** baseline, offline/online/human evaluation, held-out set, evaluation charter.
**Visual aid:** Two development timelines — vibes-driven vs eval-driven — with quality plotted over six weeks and the divergence point marked.
**Hook:** "You've written down what good looks like. Now you need enough examples of it to be worth measuring against."

---

## Lesson 3.2 — Building a Golden Dataset

*13 min · serves M3-LO1*

**Concept.** The dataset is the asset — it outlives every model and framework choice in the course. Cover sourcing: real user queries beat synthetic ones, but synthetic generation is legitimate for coverage of rare cases as long as it is labeled as such. Cover **stratification** — deliberately including easy, hard, adversarial, and out-of-scope cases in known proportions, because a dataset of only easy cases is a thermometer that reads room temperature. Cover sizing honestly: 50 well-chosen stratified examples beat 5,000 scraped ones, and the statistical reality that a 60-item set cannot detect a 2% improvement. Cover splitting and the leakage discipline: a held-out set you never look at, because a set you have iterated against is now training data. Cover the labeling contract — a rubric specific enough that two people agree, tested by actually having two people try.

**Example.** Building a 60-item stratified set for the docs-QA spine project, walked through category by category, including the eight out-of-scope questions that exist to test refusal — and a worked case where the rubric was ambiguous, two labelers disagreed, and the rubric got sharper.

**Practice.** Students build a 40–60 item stratified golden set for the spine project, documenting the strata and their proportions, holding out 30%, and writing the labeling rubric. They label a 10-item overlap sample twice and compute their own agreement rate.

**Summary.** Small, stratified, honestly labeled, and held out. This file is the most durable thing built in the course.

**Key terms:** golden dataset, stratification, held-out set, data leakage, labeling rubric, inter-annotator agreement.
**Visual aid:** Stratification table — category, proportion, count, example, why it's included.
**Hook:** "You have sixty labeled examples. Score them with accuracy and you'll learn almost nothing."

---

## Lesson 3.3 — Metrics That Fit the Task

*12 min · serves M3-LO2*

**Concept.** Why accuracy is usually wrong: most LLM tasks are generative and open-ended, so there is no single correct string, and even for classification the class balance in a stratified set makes raw accuracy misleading. Build a selection framework by task shape — **extraction** (per-field precision/recall/F1, exact-match after normalization), **classification** (per-class precision/recall, confusion matrix, because the aggregate hides the one class that matters), **generative** (rubric scoring, pairwise preference, groundedness — deferred to 3.4), **retrieval** (recall@k, MRR, nDCG — the direct handoff to Module 4). Cover the metrics that look rigorous and aren't: BLEU and ROUGE measure n-gram overlap, which is a poor proxy for whether an answer is right. Introduce the practical trio every system needs alongside quality: cost per successful task, latency, and refusal rate.

**Example.** The same 60-item output set scored four ways, showing accuracy at 0.88 while per-class recall on the refusal category sits at 0.20 — the aggregate hiding the failure that would actually get the system pulled from production.

**Practice.** Students choose metrics for the spine project against their Lesson 3.1 charter, implement them in `spine/eval/metrics.py`, and run the current pipeline to produce the **baseline number** the rest of the course is measured against.

**Summary.** Pick the metric from the task shape and the charter, always report a distribution rather than a single aggregate, and always carry cost and latency alongside quality.

**Key terms:** precision, recall, F1, confusion matrix, recall@k, MRR, nDCG, BLEU/ROUGE and their limits, refusal rate.
**Visual aid:** Decision tree — task shape → candidate metrics → what each hides.
**Hook:** "That covers everything you can check mechanically. 'Is this answer actually good?' isn't on the list."

---

## Lesson 3.4 — LLM-as-Judge and Judge Calibration

*13 min · serves M3-LO3*

**Concept.** For open-ended output, a model scores the output — cheap, fast, and scalable, and named in most senior postings. The part most people skip is the part that makes it credible: **calibration**. An uncalibrated judge is a random number generator with good manners. Cover judge design — a rubric with concrete anchors rather than a 1–10 vibe scale, one dimension per call, reasoning before score, and pairwise comparison instead of absolute scoring where possible because models are far better at "which is better" than "how good is this." Cover the known biases with the mitigation for each: position bias (randomize order, run both orders), verbosity bias (control for length), self-preference (use a different model family as judge than as generator). Then calibration itself: score a human-labeled subset, compute agreement, and state the resulting trust boundary — "this judge agrees with me 84% of the time, so I do not act on differences smaller than the disagreement band."

**Example.** A judge that rates every long answer highly, caught by a length-controlled calibration run showing correlation with word count rather than quality — then the rubric revision that fixes it, with agreement moving from 0.61 to 0.86.

**Practice.** Students build a pairwise judge with an anchored rubric, run it against their human-labeled held-out subset, compute agreement, and write the trust boundary into `spine/eval/judge.md`. They must also run one deliberate bias probe (position or verbosity) and report what they found.

**Summary.** An LLM judge is a measuring instrument, and an uncalibrated instrument is decoration. Anchor the rubric, prefer pairwise, probe for bias, state the agreement rate, and never act inside the noise band.

**Key terms:** LLM-as-judge, rubric anchoring, pairwise comparison, position bias, verbosity bias, self-preference bias, inter-rater agreement, trust boundary.
**Visual aid:** Scatter plot of judge score vs human score, before and after calibration, with the disagreement band shaded.
**Hook:** "You can measure quality and you know how much to trust the measurement. Now make it impossible to ship a regression."

---

## Lesson 3.5 — Regression Suites and CI Gating

*12 min · serves M3-LO4*

**Concept.** Evaluation becomes engineering when it runs automatically and can block a merge. Cover the shape of an eval suite as CI: a fast tier on every commit (a 15-item smoke set, seconds, cheap) and a full tier on merge to main or on demand (the whole golden set). Cover the non-determinism problem that makes this different from ordinary CI — a threshold that flaps wastes more time than it saves, so gate on a band derived from the judge's measured noise, not on an exact number, and pin what can be pinned (seed, model version, prompt version). Cover what the gate should report: not just pass/fail but *which stratum* regressed, because "faithfulness dropped on adversarial queries" is actionable and "score went down" is not. Cover cost as a gate condition alongside quality. Close with the artifact that matters for hiring: an eval report committed to the repo, showing the baseline, each change, and its measured effect.

**Example.** A GitHub Actions workflow running the smoke tier on PR and the full tier on merge, with a run where a prompt change passes the aggregate gate but trips the per-stratum check on out-of-scope refusal — the exact failure the aggregate would have shipped.

**Practice.** Students wire their suite into CI with both tiers, set thresholds derived from their measured judge noise, and deliberately push a regression to confirm the gate catches it and names the stratum. They commit `eval-report.md` with their baseline.

**Summary.** Two tiers, thresholds set from measured noise, per-stratum reporting, cost gated alongside quality, and a committed eval report. From here on, no module change ships without a number.

**Key terms:** regression suite, smoke tier, CI gate, flapping threshold, noise band, per-stratum reporting, eval report.
**Visual aid:** CI pipeline diagram — commit → smoke tier → merge → full tier → report, with the gate decision points and the stratum breakdown table.
**Hook:** "The harness is live and the baseline is committed. Module 4 is the first time you'll get to move that number — and the first time you'll be able to prove it."

---

## Spine Increment

**Deliverable:** `spine/eval/` — golden dataset, metrics, calibrated judge, two-tier CI suite, and a committed `eval-report.md` carrying the baseline.

**Acceptance criteria:**
1. 40–60 item stratified golden set with documented strata and a 30% held-out split.
2. Metrics chosen against the Lesson 3.1 charter, reported per stratum, with cost and latency alongside.
3. Judge calibrated against human labels, with the agreement rate and trust boundary written down.
4. CI runs both tiers and blocks on a regression outside the noise band, naming the affected stratum.
5. `eval-report.md` records the pre-retrieval baseline.

---

## Standalone Exercise

Take a supplied deliberately mediocre summarizer, build an eval that exposes its specific weakness (it drops the final third of long documents), prove the weakness with numbers, then fix it and prove the fix. Deliverable: `exercises/m3-summarizer-eval/` with dataset, metrics, before/after report, and a paragraph on why the first metric they tried failed to reveal the problem.

---

## Dependencies

| Lesson | Requires |
|---|---|
| 3.1 | Modules 1–2 |
| 3.2 | 3.1 (charter drives strata) |
| 3.3 | 3.2 (needs the dataset) |
| 3.4 | 3.2, 3.3 (needs human labels and mechanical metrics for contrast) |
| 3.5 | 3.3, 3.4 (needs metrics and judge to gate on) |

**Downstream:** every remaining module extends this harness. Module 5 adds RAG-specific metrics; Modules 8–9 measure fine-tuned models against this exact baseline; Module 10 extends offline eval into online scoring; Module 11 turns each successful red-team attack into a regression case.

---

## Assessment Focus (Phase 4)

Heavily diagnostic — this is the module most likely to be probed in interviews. Given a dataset description, identify the missing stratum. Given accuracy 0.88 and a class breakdown, explain what is hidden. Given judge scores correlating with length, name the bias and the fix. Given a flapping CI gate, propose a threshold policy. Given an eval report, state what it does and does not license you to claim.
