---
id: "03.1"
module: 3
minutes: 10
practice_minutes: 30
prerequisites: ["02.4"]
objectives:
  - "Build a golden dataset for a task — sourcing, labeling, sizing, and splitting it — and justify every choice."
  - "Select task-appropriate metrics and explain why accuracy is usually the wrong one."
volatility: concept
sources: []
last_verified: "2026-08-24"
---

# 03.1 · Why You Evaluate Before You Build

This module is placed here deliberately, and the placement is the most consequential structural decision in the course.

You have a client, typed contracts, and versioned prompts. You do not yet have a retrieval system, an agent, or a fine-tuned model — the things worth evaluating. Building the evaluation harness now looks premature. It is not, for two reasons.

The first is practical: every module from here forward makes a change and claims it improved something. Without a baseline established before those changes, every claim is a feeling.

The second is about your career. Hiring research is unusually consistent on this point — eval literacy is repeatedly described as the single clearest signal that a candidate has actually shipped LLM systems rather than followed tutorials. Nearly every senior posting asks for experience designing eval pipelines, golden datasets, and judge workflows. And in interviews, it is the question most candidates have nothing to say about, because they built systems and assessed them by looking at a few outputs.

So: this module is not preparation for the real work. In hiring terms, it *is* the real work.

---

## Key Concept 1: The vibes trap

Here is how LLM projects actually degrade, and it is not through carelessness.

A developer changes a prompt. They try it on three examples — the ones they were already thinking about, which are usually the ones that motivated the change. It works better on all three. Ship.

Repeat forty times over six weeks.

Each individual change was verified. The system is now worse than it was in week two, and nobody can say when it turned, because there is no measurement anywhere in the loop. Worse, the team's confidence is *high*, because every change felt like an improvement at the time.

Three mechanisms drive this:

**Selection bias in your test cases.** You test the change on inputs that motivated the change. That is the definition of overfitting, performed manually.

**Non-monotonic effects.** Instructing the model to be more concise can improve summaries and destroy edge-case handling. There is no compiler warning and no type error.

**Invisible regressions.** The failure you introduced is in a category you were not looking at — refusals, long inputs, a second language, adversarial phrasing.

The discipline that prevents all three is unglamorous and total:

> **Baseline first. One change at a time. Measure on held-out data. Keep or revert.**

That is the whole of evaluation practice. Everything else in this module is implementation.

---

## Key Concept 2: Three evaluation surfaces

"Evaluation" covers three different activities with different costs and different jobs. Know which one you are doing.

**Offline evaluation.** A fixed dataset with known-good answers, run on demand and in CI. Fast, cheap, repeatable, and the only kind that can block a merge. Its weakness is that it measures your dataset, and your dataset is a guess about reality. **This module builds this.**

**Online evaluation.** Sampled production traffic, scored automatically. Measures the real input distribution, which your dataset never quite matches. Its weakness is no ground truth and a delayed signal. **Module 10 builds this**, using the same judge you build in Lesson 3.4.

**Human evaluation.** People reading outputs against a rubric. Slow, expensive, small — and the only ground truth there is. Its job is not to measure your system continuously; its job is to **calibrate the other two**. **Lesson 3.4 uses this.**

They form a chain: humans calibrate the judge, the judge scores the offline set, the offline set gates CI, and online evaluation checks whether the offline set still resembles production. Skip the human step and the entire chain is measuring nothing in particular, confidently.

---

## Key Concept 3: Evaluate outcomes, not vibes

Before writing a metric, you have to answer a question that sounds soft and is not: **what does good mean here?**

This has to be specific enough to disagree with. "Helpful and accurate" is not a specification. These are:

- "The answer must be supported by the retrieved documentation. An unsupported claim is a failure regardless of whether it is true."
- "Out-of-scope questions must be refused, not answered from general knowledge."
- "Answers must cite the section they came from."
- "A wrong answer is worse than a refusal. A refusal on an answerable question is worse than a mediocre answer."

That last one is a **priority ordering**, and it is the thing most teams never write down. Without it you cannot resolve a trade-off when one metric rises and another falls — which happens constantly. With it, the decision is already made.

This is what the practice section produces, and every later choice in this module derives from it: your strata in Lesson 3.2, your metrics in 3.3, your judge rubric in 3.4, and your CI thresholds in 3.5.

---

## Worked Example: The improvement that wasn't

A developer is unhappy with verbose answers. They edit the prompt:

```diff
- Answer the question using the provided documentation.
+ Answer the question using the provided documentation.
+ Be concise and direct. Do not pad your answers.
```

They test on three long-winded examples. All three tighten up. Obvious win.

Then it is run against a 60-item held-out set:

| Stratum | Before | After | Δ |
|---|---|---|---|
| Simple factual (20) | 0.91 | 0.93 | **+0.02** |
| Multi-step (15) | 0.84 | 0.71 | **−0.13** |
| Ambiguous (12) | 0.76 | 0.68 | **−0.08** |
| Out-of-scope (8) | 0.88 | 0.42 | **−0.46** |
| Long context (5) | 0.79 | 0.77 | −0.02 |
| **Overall** | **0.85** | **0.75** | **−0.10** |

The three examples the developer tested were all simple factual questions — the one stratum that improved.

The catastrophic column is out-of-scope. "Be concise and direct" suppressed the model's hedging, and hedging was the mechanism by which it declined to answer questions the documentation did not cover. Instructed toward directness, it started answering them confidently and wrongly.

Nobody would have predicted that. Nobody had to — the stratified set caught it in a two-minute run. Without it, this change ships and the failure surfaces months later as "the assistant makes things up."

---

## Practice

Write an **evaluation charter** for the spine project. One page, no code. This document drives the rest of the module.

**1. Purpose.** Two sentences: what the system is for and who uses it.

**2. What a good answer looks like.** One paragraph, concrete enough that someone else could apply it. Include what a good answer does when the documentation does not cover the question.

**3. What an unacceptable answer looks like.** List them explicitly. An unsupported claim. A fabricated citation. A confident answer to an out-of-scope question. Be specific — this list becomes your failure taxonomy.

**4. The three failure modes that matter most to a user.** Ranked. This determines where you invest.

**5. The priority ordering.** Complete these, in your own words:
- A wrong answer is ______ than a refusal.
- A slow correct answer is ______ than a fast wrong one.
- If accuracy rises 3 points and cost doubles, we ______.

**6. Non-goals.** What you are explicitly not measuring, and why. This is as useful as the goals — it stops scope creep in the harness itself.

Commit as `spine/eval/charter.md`. When a metric decision is unclear in the next four lessons, the answer is in this file.

---

## Visual Aid

*Two line charts sharing a y-axis labeled "held-out quality score," x-axis "week 1 → week 12." The upper line, "eval-driven," rises in visible steps with small dips where changes were reverted — annotated "reverted: −4 pts" at two points. The lower line, "vibes-driven," rises for three weeks then drifts steadily downward, with a cluster of small green checkmarks along it labeled "every change verified on 3 examples." A vertical dashed line at week 6 is annotated: "both teams believe they are improving." Caption: "The difference is not effort. It is held-out data."*

---

## Key Takeaways

- **Eval literacy is the strongest hiring signal in this field**, and the question most candidates cannot answer. This module is career-relevant work, not preparation for it.
- **The vibes trap is not carelessness.** Selection bias, non-monotonic effects, and invisible regressions degrade systems whose every change was individually verified.
- **Baseline first, one change at a time, held-out data, keep or revert.** That is the entire discipline.
- **Three surfaces:** offline (gates CI), online (measures reality, Module 10), human (calibrates both, Lesson 3.4). Skipping human calibration makes the chain confident and meaningless.
- **Write the charter before the metrics**, including a priority ordering — it is what resolves the trade-offs that will otherwise stall you.

---

## Next

You have written down what good means. To measure against it you need examples — enough of them, chosen deliberately, covering the cases you would otherwise never test.

**Lesson 3.2** builds the golden dataset: why 60 well-chosen examples beat 5,000 scraped ones, how to stratify so a whole failure category cannot hide, and the leakage discipline that keeps your Module 8 fine-tuning comparison honest.
