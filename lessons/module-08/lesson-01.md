---
id: "08.1"
module: 8
minutes: 13
practice_minutes: 60
prerequisites: ["07.4"]
objectives:
  - "Decide between prompting, retrieval, and fine-tuning for a given problem, with a cost and maintenance argument."
volatility: concept
sources: []
last_verified: "2026-08-24"
---

# 08.1 · The Fine-Tuning Decision: When Not To

Fine-tuning appears in roughly a quarter of AI engineering job postings, so you need it. It is also the technique most often applied to problems it does not solve, usually because it is the most *interesting* option available.

The valuable skill here is not running a training job — that is Lesson 8.3 and it takes twenty-five minutes on a Colab GPU. The valuable skill is the decision. In an interview, "we considered fine-tuning and chose retrieval instead, because the underlying facts changed weekly and a frozen model would have gone stale between releases" is a stronger answer than any description of a training run.

This lesson builds that judgment, and it ends with the question most practitioners never ask: not "would this help?" but **"do I want to own this for the next two years?"**

---

## Key Concept 1: Three techniques, three jobs

**Prompting changes instructions.** Free to iterate, instantly reversible, effective for format, tone, and task framing. It is where you start and where most problems end.

**Retrieval injects knowledge at request time.** The right answer for *facts*, because facts change. Your documentation updates, your prices change, your API adds a version — retrieval picks that up on the next index run.

**Fine-tuning changes behavioral priors.** It shifts what the model tends to do, absorbed into weights.

The decision follows from which of those you actually need:

| Problem | Technique |
|---|---|
| Output format is wrong | Prompting → constrained decoding (Lesson 2.2) |
| Model does not know your facts | **Retrieval** |
| Facts change frequently | **Retrieval**, definitively |
| Tone or style is off | Prompting first; fine-tuning if prompting plateaus |
| Format still fails after constrained decoding | **Fine-tuning** |
| Domain vocabulary consistently misused | **Fine-tuning** |
| A frontier model is too expensive at your volume | **Fine-tuning** a small model (or distillation, Lesson 9.3) |
| Task is easier to demonstrate than describe | **Fine-tuning** |
| Latency too high on a large model | **Fine-tuning** a small one |

---

## Key Concept 2: The anti-patterns

**Fine-tuning to teach facts.** The most common and most expensive mistake. Facts absorbed into weights are frozen at training time, cannot be updated without retraining, cannot be cited, and — worst — the model **confabulates around them**: having learned the shape of your documentation, it generates plausible content in that shape for things it never saw. A retrieval system says "not in the docs." A fact-tuned model invents a section.

**Fine-tuning before exhausting prompting.** You have converted a five-minute iteration into a forty-minute one with a GPU bill. Exhaust the free option first.

**Fine-tuning without a baseline.** You cannot tell whether it worked. You built the baseline in Module 3 precisely so this is not your situation.

**Fine-tuning to fix a retrieval problem.** If context recall is 0.42 (Lesson 5.5), the model never saw the answer. Training it on more examples does not change that.

**Catastrophic forgetting.** Narrow training degrades general capability. A model tuned hard on one format can lose refusal behavior, instruction-following, or multi-turn coherence. This is real, it is measurable, and Lesson 8.4 checks for it — which is why your stratified golden set exists.

---

## Key Concept 3: Total cost of ownership

Here is the question practitioners underweight. A fine-tuned model is not a result; it is an **artifact you now own**.

**Up front:** dataset construction (the real cost — days, not hours), training runs including the failed ones, and evaluation.

**Ongoing:**
- **Serving.** Either a hosted fine-tuning endpoint at a premium, or your own GPU. Idle GPU time is the line people forget.
- **Re-evaluation on every base-model change.** Your adapter is tied to a base model.
- **Retraining as data drifts.** Your product changes; the tuned behavior does not.
- **Versioning and provenance.** Which adapter is in production, trained on what, evaluated how, and how do you roll back? (Lesson 9.4.)

**The scheduled shock:** base models get deprecated. When yours does, you re-run the entire chain — dataset, training, evaluation, deployment — on a new base. That is not a hypothetical; it is a certainty on a one-to-two-year horizon.

Which produces the honest question: **is this worth owning for two years?** For a task that will be a large fraction of your traffic for years, yes. For a nice-to-have feature, almost never.

---

## Key Concept 4: The decision framework

Six questions, in order. Stop at the first that resolves it.

1. **Is this a knowledge problem?** → Retrieval. Stop.
2. **Has prompting been genuinely exhausted?** Constrained decoding, schema design, few-shot, prompt versioning with measurement (Modules 2–3). If not → do that first.
3. **Do I have a baseline?** No → build it (Module 3). You cannot evaluate a fine-tune without one.
4. **Do I have or can I build 500+ good examples?** No → probably not viable.
5. **What is the TCO over two years, and what does it save?** If the saving is not several times the cost, no.
6. **What breaks when the base model is deprecated?** If there is no answer, you are not ready.

---

## Worked Example: Three problems

**Problem A — the support bot invents policy details.**

Symptoms: confident, specific, wrong statements about refund windows and SLA terms.

Analysis: a knowledge problem. Policies live in documents and change quarterly. Fine-tuning would freeze this quarter's policy into weights, produce uncitable answers, and confabulate around gaps.

**→ Retrieval.** Question 1 resolves it. *Result: hallucinated policy claims 0.31 → 0.04 with grounded citations.*

**Problem B — JSON structure fails 8% of the time.**

Symptoms: constrained decoding is in use and shape is always valid, but field *semantics* are wrong 8% of the time — the model puts the customer's name in `company` for sole traders, and misassigns a subtly ambiguous field.

Analysis: not knowledge, not shape. Prompting attempts: field descriptions, few-shot examples, schema flattening — 12% → 8%, then plateaued across four measured prompt versions. The correct behavior is easier to *demonstrate* than to describe, which is fine-tuning's home ground.

**→ Fine-tuning is a legitimate candidate.** TCO below.

**Problem C — classification costs $4,100/month.**

Symptoms: a frontier model classifying 2M support tickets monthly, at 0.94 accuracy. Works well; the bill is the problem.

Analysis: a cost problem. The task is narrow and high-volume — exactly what a small tuned model handles. This is Lesson 9.3's distillation, and it is the strongest fine-tuning case of the three.

**→ Fine-tune (distil) a small model.** *Result: 0.92 accuracy at $180/month, with a router escalating low-confidence cases to recover most of the gap.*

**The TCO worked for Problem B:**

| | One-off | Annual |
|---|---|---|
| Dataset construction (3 days) | $2,400 | — |
| Training runs (incl. 2 failures) | $40 | $120 |
| Evaluation | $60 | $240 |
| Serving premium | — | $1,900 |
| Re-eval on base-model updates (2×/yr) | — | $400 |
| Retraining on drift (1×/yr) | — | $800 |
| **Total** | **$2,500** | **$3,460** |

**Two-year cost: ~$9,400.**

Value: 8% → 1% semantic error on 40,000 extractions a month. Roughly 2,800 fewer bad records monthly, each costing about 6 minutes of manual correction — around 280 hours a month, over 3,000 a year.

**Ship it.** But notice: the decision needed the value quantified, not just the improvement. "It got better" would not have supported this.

---

## Practice

**1. Work the decision matrix** on four supplied scenarios in `exercises/m8-decision/`. For each: which technique, why, and what you would need to measure to be confident.

**2. Include a "do not fine-tune" case.** At least one scenario should resolve to retrieval or prompting. State the reasoning explicitly — this is the answer that signals judgment.

**3. Apply it to your spine project.** Where, if anywhere, is fine-tuning justified? Candidates: a persistent format or semantic issue you measured in Modules 2–5, or a high-volume component you could move to a smaller model.

**4. Verify prompting is exhausted.** Before proceeding, show measured evidence: at least three prompt versions with eval results, plateauing. Your Lesson 2.4 versioning and Module 3 harness make this a real check rather than an assertion.

**5. Build the TCO model.** Both cases: fine-tune and do-not. Include serving, re-evaluation, retraining, and the base-model deprecation event. Two-year horizon.

**6. Quantify the value.** In whatever unit fits — hours saved, errors avoided, dollars. If you cannot quantify it, that is itself a finding.

**7. Write `docs/adr-finetuning.md`** with the decision, reasoning, TCO, value, and the conditions under which you would reverse it. **A well-argued "no" is a passing outcome** — but for this course, if you decide no, pick a second candidate task so you still complete Modules 8 and 9 hands-on.

---

## Visual Aid

*A decision tree read top to bottom. "Is it a knowledge problem?" → YES → green terminal "RETRIEVAL." NO → "Has prompting been exhausted, with measured evidence?" → NO → amber terminal "DO THAT FIRST — Modules 2–3." YES → "Do you have a baseline?" → NO → amber "BUILD IT — Module 3." YES → "500+ good examples available?" → NO → red "NOT VIABLE." YES → "TCO over 2 years vs quantified value?" → a final box, drawn heavier than the rest and shaded, reading: "Do you want to OWN this artifact for two years — including the base-model deprecation event?" with two exits, "ship" and "no." Caption: "Most candidates stop three boxes early."*

---

## Key Takeaways

- **Prompt for behavior, retrieve for facts, fine-tune for priors and for cost.** The decision follows from which of the three you actually need.
- **Never fine-tune to teach facts.** They freeze, they cannot be cited, and the model confabulates around the gaps — worse than not knowing.
- **Exhaust prompting first, with measured evidence.** Three versions and a plateau, not a feeling.
- **Fine-tuning is an artifact you own** — serving, re-evaluation, retraining, versioning, and a guaranteed base-model deprecation event on a one-to-two-year horizon.
- **Quantify the value, not just the improvement.** "8% → 1%" does not justify $9,400; "280 hours a month" does.
- **A well-argued "no" is a strong professional result** and a strong interview answer.

---

## Next

You have decided fine-tuning is justified and you can defend it with a cost model. From here everything depends on data quality — more than on the algorithm, the hyperparameters, or the base model.

The good news: you already have the raw material. You have been writing structured telemetry since Lesson 1.4, and dead-letter records since Lesson 2.3.

**Lesson 8.2** turns four thousand production traces into eight hundred training examples, and covers the one configuration detail that makes a training run look perfectly healthy while producing garbage.
