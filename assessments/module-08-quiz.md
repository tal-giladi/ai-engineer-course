# Module 8 Quiz — Fine-Tuning I: When, and SFT/LoRA

**9 questions · ~35 min plus the practical**

Weighted toward the decision, because that is the part that is professionally valuable and the part interviews probe.

---

## Q1 — Applied judgment

For each scenario, choose prompting, retrieval, or fine-tuning, and give one sentence of reasoning:

(a) A support bot confidently states refund windows that are wrong; policies change quarterly
(b) A model's JSON *shape* is always valid (constrained decoding) but field *semantics* are wrong 8% of the time, after four measured prompt versions plateaued
(c) A frontier model classifies 2M tickets a month at 0.94 accuracy for $4,100/month
(d) The model uses "customer" where the company's internal vocabulary says "member", consistently, across all outputs

---

## Q2 — Short answer

Explain why fine-tuning to teach facts is worse than *not knowing* the facts. Name the specific failure mode.

---

## Q3 — Design

A team proposes fine-tuning. Before approving, name the **six** questions you would ask, in order, and state which answer at each step would stop the project.

---

## Q4 — Applied judgment

Two-year TCO for a proposed fine-tune comes to ~$9,400. The improvement is a semantic error rate falling from 8% to 1% on 40,000 extractions per month.

(a) Is this enough information to decide? What is missing?
(b) Supply the missing piece with a plausible estimate and state the decision.

---

## Q5 — Diagnosis

A training run produces this:

| Step | Train loss | Val loss |
|---|---|---|
| 250 | 0.47 | 0.68 |
| 400 | 0.29 | 0.71 |
| 550 | 0.16 | 0.79 |
| 700 | 0.08 | 0.88 |

(a) What is happening?
(b) A colleague suggests increasing LoRA rank to fit better. Why is that wrong?
(c) What are the two correct fixes?
(d) What would you expect to see in sampled outputs at step 700?

---

## Q6 — Diagnosis

A training run has a textbook loss curve — smooth descent from 1.84 to 0.41 over three epochs, validation tracking nicely. Inference output looks like this:

```
### Response: The extracted fields are... ### Instruction: ###
Response: ### Instruction:
```

(a) What is the cause?
(b) Why did the loss curve give no warning?
(c) What sixty-second check prevents this?
(d) What habit prevents this class of problem generally?

---

## Q7 — Diagnosis

A dataset funnel:

| Stage | Remaining | Removed |
|---|---|---|
| Raw traces | 4,102 | — |
| Successful outcomes | 3,318 | 784 |
| Held-out leakage check | 3,301 | **17** |
| Near-duplicate removal | 1,842 | **1,459** |
| Contains target defect | 1,690 | 152 |
| Quality filter | 1,521 | 169 |
| Category balance cap | 1,088 | 433 |

(a) Seventeen examples were removed by the leakage check. Why does such a small number matter?
(b) Near-duplicate removal cut 36% of the corpus. What would have happened without it?
(c) Which of these stages, if skipped, would produce a model that *looks* excellent in evaluation?

---

## Q8 — Applied judgment

Fine-tuned model versus prompted frontier baseline, on the untouched held-out set:

| | Baseline | Fine-tuned |
|---|---|---|
| Format adherence | 0.92 | 0.99 |
| Field semantic accuracy | 0.89 | 0.96 |
| **Out-of-scope refusal** | **0.88** | **0.43** |
| Cost / 1,000 requests | $2.10 | $0.11 |
| p95 latency | 1,840 ms | 420 ms |

(a) What happened to refusal accuracy, and why?
(b) There are three deployment options, not two. Name them.
(c) Which do you choose for a customer-facing documentation assistant, and what does it cost?
(d) What would have happened without an out-of-scope stratum in the golden set?

---

## Q9 — Practical

Fine-tune a small model to reliably produce a strict output format the base model violates ~15% of the time. Quantify the improvement **and** check whether general capability regressed.

**Deliverable:** `exercises/m8-format-adherence/` with datasheet, notebook, adapter link, and before/after evaluation including off-task strata.

---
---

# Answer Key

## Q1

**(a) Retrieval.** This is a knowledge problem and the knowledge changes quarterly. Fine-tuning would freeze this quarter's policy into weights, produce uncitable answers, and confabulate around gaps.

**(b) Fine-tuning is a legitimate candidate.** Not knowledge, not shape. Prompting has been genuinely exhausted with measured evidence across four versions. The correct behavior is easier to demonstrate than to describe, which is fine-tuning's home ground.

**(c) Fine-tune (distil) a small model.** This is a cost problem, not a quality problem, and a narrow high-volume task is exactly where a small model wins. This is Lesson 9.3's territory.

**(d) Prompting first.** A consistent vocabulary substitution is a straightforward instruction. Only escalate to fine-tuning if a measured prompt attempt plateaus — and it almost certainly will not.

## Q2

**Why it is worse than not knowing:** a retrieval system that lacks a fact says "the documentation doesn't cover this." A fact-tuned model has learned the *shape* of your documentation, so when asked about something adjacent to what it saw, it **confabulates fluently in that shape** — inventing a plausible section, a plausible parameter, a plausible limit.

**The failure mode:** confident, well-formatted, uncitable fabrication that is indistinguishable from a correct answer. Compounded by the facts being frozen at training time, so the model is confidently wrong about anything that changed since.

**Full credit** requires naming confabulation-around-gaps, not just "the facts go stale."

## Q3 — Six questions, in order

1. **Is this a knowledge problem?** → If yes, **stop**: use retrieval.
2. **Has prompting been genuinely exhausted, with measured evidence?** → If no, **stop**: do that first. Three prompt versions with eval results and a plateau, not a feeling.
3. **Do we have a baseline?** → If no, **stop**: build it. Without one you cannot evaluate the result.
4. **Can we get 500+ good examples?** → If no, **stop**: probably not viable.
5. **What is the two-year TCO, and what is the quantified value?** → If the value is not several times the cost, **stop**.
6. **What happens when the base model is deprecated?** → If there is no answer, **stop**: you are not ready to own the artifact.

**The sixth question is the one practitioners skip**, and it is where the real ongoing cost lives.

## Q4

**(a) No.** The improvement is stated as a rate change, not as value. "8% → 1%" does not tell you whether $9,400 is worth spending.

**(b)** 7 percentage points of 40,000 monthly extractions ≈ 2,800 fewer bad records per month. At roughly 6 minutes of manual correction each, that is ~280 hours per year of recovered effort — comfortably exceeding a two-year cost of $9,400.

**Decision: ship it.**

**The teaching point:** quantify the *value*, not just the improvement. Interviews probe exactly this — a candidate who converts a metric delta into hours or dollars is reasoning like an engineer who has had to justify a budget.

## Q5

**(a) Overfitting.** Validation loss bottomed around step 250 and has risen steadily since. Everything after that turn is memorization of the training set.

**(b)** Increasing rank increases adapter *capacity*, which lets the model memorize the training set more thoroughly. It makes overfitting worse, not better. This is a common wrong instinct because "it's not fitting well enough" is the intuitive reading of a rising validation loss, and it is backwards.

**(c) Fewer epochs** (stop at ~250 steps, where validation flattened) **or more data**. Both address the actual cause: too much capacity applied too many times to too few examples.

**(d)** Outputs reproducing training examples nearly verbatim — including specific values from the training data appearing in unrelated inputs. A customer name from one training example showing up in a different extraction is the canonical tell.

## Q6

**(a) Chat template mismatch.** The run used a hand-written format string instead of the tokenizer's own template. The model learned to generate the delimiter structure it was trained on, which the inference-time template neither produces nor terminates on.

**(b)** Because loss measures fit to the training distribution, and the model fit that distribution *perfectly well* — the distribution was just the wrong one. Nothing about a wrong template makes the optimization harder, so the curve looks identical to a healthy run.

**(c)** `tokenizer.apply_chat_template(...)`, then **print three formatted training examples and read them character by character, including special tokens.** Sixty seconds against a wasted GPU hour and an hour of confusion.

**(d) Sample outputs at checkpoints during training, not just curves.** This habit catches the template bug, memorization, and — in Module 9 — reward hacking. Curves are compatible with all three failures; outputs are not.

## Q7

**(a)** Because 17 leaked examples are enough to inflate the held-out comparison misleadingly — and crucially, the inflation makes the fine-tune look **better**, which is the direction nobody investigates. A result that looks excellent and is contaminated is more dangerous than one that looks poor.

**(b)** Without deduplication, one common phrasing pattern (36% of the corpus) would dominate training. The model would over-learn that pattern and generalize poorly to the rest — and evaluation on a stratified golden set would show it as weakness in the less-common strata, which teams often misdiagnose as "needs more data."

**(c) The leakage check.** Skipping it produces a model that memorized part of the test set and scores excellently against it. Every other skipped stage produces a *worse-looking* result, which prompts investigation. Only leakage produces a flattering lie.

## Q8

**(a)** Refusal accuracy halved because the model was trained on 800 examples that **all produced output**. It learned that producing output is what it does. This is capability regression from narrow training — the specific casualty is nearly always refusal, because refusal is behavior the training data by construction never demonstrates.

**(b)** Ship as-is · Ship with a router · Do not ship.

**(c) Ship with a router.** A lightweight scope classifier routes out-of-scope questions to the refusal path so they never reach the tuned model. Typical result: format adherence 0.99 retained, refusal accuracy back to ~0.86, cost per 1,000 requests ~$0.19 — still roughly 11× cheaper than baseline rather than 19×.

**The cost is the router:** an extra component to build, evaluate, version, and maintain. Worth it here because discarding a 19× cost reduction over one fixable regression would be poor judgment, and shipping a documentation assistant that confidently answers questions it should refuse would be worse.

**(d)** Without an out-of-scope stratum, refusal accuracy would not have been measured at all. The on-task metrics look excellent, the aggregate would look fine, and **this ships** — surfacing months later as "the assistant makes things up."

## Q9 — Practical rubric

| Criterion | Full credit |
|---|---|
| Leakage check | Run and reported, exact-hash and near-duplicate, count stated even if zero |
| Chat template | Uses `apply_chat_template`; formatted examples inspected before training |
| Curve interpretation | Written interpretation — healthy, overfitting, or underfitting — with the stopping point identified |
| Output sampling | Outputs generated and read at checkpoints, not curves only |
| **Off-task evaluation** | Full stratified set plus a general capability probe, with out-of-scope specifically reported |
| Comparison | Quality, cost, and latency against the prompted baseline |
| Recommendation | Ship / router / discard, with reasoning tied to the charter |

**The off-task row is the one that separates submissions.** A submission reporting only the target metric has demonstrated it can run a training job, not that it can evaluate one.
