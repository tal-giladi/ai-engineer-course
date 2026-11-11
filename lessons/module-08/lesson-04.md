---
id: "08.4"
module: 8
minutes: 13
practice_minutes: 90
prerequisites: ["08.3"]
objectives:
  - "Prove — or disprove — improvement against the Module 3 baseline rather than asserting it."
volatility: concept
sources: []
last_verified: "2026-08-24"
---

# 08.4 · Did It Work? Measuring Against the Baseline

Search for fine-tuning tutorials and count how many end at "the loss went down." Almost all of them.

Loss going down means the model fits the training distribution better. It does not mean it is better at your task, it does not mean it is better than the model you were already using, and it says nothing about what you might have broken. Those are three separate questions, each answerable, and none of them answered by a training curve.

This lesson answers all three, and it is the lesson that makes the previous three worth anything. It also covers the outcome practitioners avoid reporting: sometimes the fine-tune loses. Reporting that clearly is a professional strength and — in an interview — one of the most credible things you can say.

---

## Key Concept 1: Three questions, in order

**1. Did it improve on the target task?** Run the Module 3 harness against the **untouched held-out set** (Lesson 3.2, leakage-checked in Lesson 8.2). Full metric vector, per stratum.

**2. Did it break anything else?** Narrow training degrades unrelated capability. This is real, common, and invisible unless you look.

**3. Is it worth deploying?** Quality, cost, latency, and operational burden together — the Lesson 3.3 discipline, applied to a decision with real consequences.

Skipping question 2 is the most common failure. A model that formats perfectly and has lost its refusal behavior is worse overall, and only a stratified evaluation reveals it.

---

## Key Concept 2: Capability regression

Fine-tuning on a narrow distribution shifts the model's priors, and those priors were doing other work. Typical casualties:

- **Refusal behavior.** Trained on 800 examples that all produce output, the model learns that producing output is what it does. Your out-of-scope stratum catches this.
- **Instruction following.** Trained on one instruction shape, it may follow others less reliably.
- **Multi-turn coherence.** Single-turn training data degrades conversational handling.
- **Robustness to unusual input.** Narrow training narrows tolerance.
- **Reasoning depth.** Training on short outputs can suppress step-by-step reasoning.

So evaluate **off-task**, not just on-task:

- Your full stratified golden set, including out-of-scope
- A small general capability probe — 20 items of instruction-following and reasoning unrelated to your task
- The multi-turn stratum if you have one

Twenty extra items, five minutes, and it is the check that catches the failure that would otherwise reach production.

---

## Key Concept 3: The honest comparison table

Compare against the model you are actually replacing:

| | Baseline (prompted frontier) | Fine-tuned small |
|---|---|---|
| Target metric | | |
| All other strata | | |
| General capability probe | | |
| Cost per 1,000 requests | | |
| p95 latency | | |
| Operational burden | none | serving, versioning, retraining |

The characteristic result: **the fine-tune wins on the target metric, cost, and latency, and loses somewhere else.** That is a trade, not a verdict, and the verdict belongs to whoever owns the budget and the risk — informed by your Lesson 3.1 charter's priority ordering.

Three deployment options, not two:

- **Ship it** — the trade is favorable and regressions are acceptable
- **Ship with a router** — fine-tuned model for the common case, frontier model for cases where it regressed. Often the best answer, and it is the Lesson 9.3 routing pattern.
- **Do not ship** — the regression matters more than the gain, or the TCO does not clear

---

## Key Concept 4: Reporting a negative result

If the fine-tune lost, say so plainly, with numbers, and with an analysis of why.

This is worth dwelling on because the instinct runs the other way. You spent three days on the dataset and a GPU budget; there is real pressure to find a framing where it won.

Resist it, for two reasons. Practically, deploying a model that lost costs you every year of the TCO you modeled in Lesson 8.1 while making the system worse. Professionally, a portfolio containing a rigorously-evaluated negative result is *more* credible than one containing only wins — it is direct evidence that your evaluations are real rather than decorative. Interviewers notice this immediately.

Useful things to say about a loss: the dataset was too small or too narrow; the base model was already good at this and there was little headroom; the problem was actually a retrieval problem (Lesson 8.1's question 1); the regression cost more than the gain.

---

## Worked Example: A win, a regression, and a decision

Fine-tuned 8B model versus prompted frontier baseline, on the untouched held-out set.

**On-task:**

| Metric | Baseline | Fine-tuned | Δ |
|---|---|---|---|
| Format adherence | 0.92 | **0.99** | +0.07 |
| Field semantic accuracy | 0.89 | **0.96** | +0.07 |

Strong. This was the target and it worked.

**Off-task — the stratified set:**

| Stratum | Baseline | Fine-tuned | Δ |
|---|---|---|---|
| simple_factual | 0.93 | 0.91 | −0.02 |
| multi_hop | 0.86 | 0.82 | −0.04 |
| ambiguous | 0.79 | 0.74 | −0.05 |
| **out_of_scope (refusal)** | **0.88** | **0.43** | **−0.45** |
| adversarial | 0.76 | 0.71 | −0.05 |
| Answer relevance | 0.89 | 0.85 | −0.04 |
| General capability probe | 0.84 | 0.79 | −0.05 |

**Refusal accuracy halved.** Trained on 800 examples that all produced output, the model learned that output is what it produces. It now answers out-of-scope questions confidently — the exact failure from Lesson 3.1, arriving by a different route.

Without the out-of-scope stratum in the golden set, this ships.

**Cost and latency:**

| | Baseline | Fine-tuned |
|---|---|---|
| Cost / 1,000 requests | $2.10 | **$0.11** |
| p95 latency | 1,840 ms | **420 ms** |
| Operational burden | none | serving, versioning, retraining |

Nineteen times cheaper, four times faster.

**The decision.** Three options considered:

*Ship as-is* — rejected. A documentation assistant that confidently answers questions it should refuse is exactly the failure the charter ranked worst.

*Discard* — rejected. A 19× cost reduction with a 7-point on-task gain is too valuable to abandon over one fixable regression.

*Ship with a router* — **chosen.** The fine-tuned model handles extraction; a lightweight scope classifier routes out-of-scope questions to the refusal path directly, never reaching the tuned model.

Result: format adherence 0.99, refusal accuracy 0.86 (close to baseline), cost per 1,000 requests $0.19 — still 11× cheaper.

**And the honest note recorded in the eval report:** ambiguous and adversarial strata remain 5 points below baseline. Not enough to block, tracked as a known limitation, and a candidate for the preference tuning in Module 9.

The eval report entry is written this way — with the regression, the rejected options, and the residual limitation stated — because that is what makes it credible.

---

## Practice

**1. Evaluate on the untouched held-out set.** Full Module 3 harness. **If you have looked at this set repeatedly during the module, say so** and treat the numbers as optimistic.

**2. Confirm the leakage check.** Re-verify that no held-out item entered training, by exact hash and by embedding near-duplicate. Report the number.

**3. Run the off-task evaluation.** Full stratified set plus a 20-item general capability probe. **Pay specific attention to the out-of-scope stratum** — this is where the regression usually is.

**4. Measure cost and latency** for both configurations. Include the serving cost of the fine-tuned model honestly, including idle GPU time if self-hosting.

**5. Build the comparison table**, on-task and off-task, with every stratum.

**6. Make a ship recommendation** — ship, ship-with-router, or do not ship — with reasoning tied to your Lesson 3.1 charter priority ordering. If you choose ship-with-router, implement it and re-measure.

**7. Update the model card and `eval-report.md`.** Include the regression, the options you rejected and why, and any residual limitation. Write it whichever way the result fell.

**8. Check your TCO estimate.** Compare Lesson 8.1's projection against what this actually cost in GPU time, your hours, and evaluation spend. Note where you were wrong — that calibration is worth more than the estimate was.

---

## Visual Aid

*A diverging bar chart. Metrics listed down the left; bars extend right (green, fine-tuned better) or left (red, fine-tuned worse) from a centre line. Format adherence and field accuracy extend right. Every stratum extends left, with `out_of_scope` extending far left and outlined in heavy red, labeled "−0.45 — refusal collapse." Below the chart, two boxes side by side: "cost/1k: $2.10 → $0.11" and "p95: 1,840 ms → 420 ms," both green. Beneath everything, a decision strip with three options — ship / router / discard — and "router" circled, annotated "0.99 format, 0.86 refusal, $0.19/1k."*

---

## Key Takeaways

- **Loss is not quality.** Evaluate on the untouched held-out set with your full Module 3 harness, per stratum.
- **Always check off-task.** Narrow training degrades refusal, instruction-following, multi-turn coherence, and robustness — and refusal is the usual casualty. Twenty extra items catch it.
- **Compare on quality, cost, latency, and operational burden together.** The characteristic result is a win on target and cost and a loss somewhere else, which is a trade for the budget owner to resolve against the charter.
- **There are three options, not two.** Ship, ship-with-router, or discard — and the router is often the best answer.
- **Report a negative result plainly.** A rigorously-evaluated loss is more credible in a portfolio than an unexamined win, and interviewers can tell the difference.

---

## Next

Supervised fine-tuning taught your model to imitate examples. It has no way to express that one acceptable answer is better than another acceptable answer — and that comparative signal is where most of your remaining quality lives: verbosity, hedging, unwarranted confidence, and the refusal calibration you just had to fix with a router.

**Module 9** gets that signal into the model with preference tuning, then uses the same machinery for distillation — teaching a small model to do a frontier model's job on a narrow task, which is where the largest cost reductions in this field come from.
