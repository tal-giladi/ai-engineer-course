# Module 9 Quiz — Fine-Tuning II: Preference Tuning & Distillation

**8 questions · ~30 min plus the practical**

---

## Q1 — Short answer

Two responses to the same question are both factually correct and both well-formatted. One hedges through three clauses; one is direct.

Explain what supervised fine-tuning can and cannot learn from this pair, and why preference tuning can.

---

## Q2 — Diagnosis

A preference dataset audit reports:

```
mean chosen length:   142 words
mean rejected length:  71 words
ratio: 2.00
```

The target dimension is *answer completeness*.

(a) Predict what the trained model will do.
(b) Name three costs that will show up, and where each becomes visible.
(c) What is the fix, and where does it belong — the data, the training config, or the prompt?

---

## Q3 — Applied judgment

Two DPO runs on the same task:

| | Run A | Run B |
|---|---|---|
| Final reward margin | 1.24 | **2.98** |
| Reward accuracy | 0.89 | **0.99** |
| KL divergence | 0.31 | 0.29 |

Run B is better on every training metric.

(a) Can you conclude Run B produced the better model? Why or why not?
(b) What single check would settle it?
(c) If Run B turns out to be worse, what is the phenomenon called and where did it originate?

---

## Q4 — Multiple choice

You have an SFT checkpoint you are happy with and want to fix a hedging problem. Memory is constrained but adequate. Which do you choose?

A. ORPO, because it needs no reference model
B. DPO on the SFT checkpoint, because you already have the checkpoint and want independent control of the two effects
C. RLHF with a trained reward model, for maximum fidelity
D. More SFT with better examples

---

## Q5 — Design

You are distilling a frontier model into a small one for a classification task.

Name the **four conditions** that must hold, and for each, state the failure that occurs when it does not.

---

## Q6 — Applied judgment

Distillation results:

| | Teacher | Student | Routed (12%) |
|---|---|---|---|
| Accuracy | 0.94 | 0.92 | 0.94 |
| Cost / month | $4,100 | $180 | $672 |
| p95 latency | 1,900 ms | 310 ms | 480 ms |

Escalation threshold sweep:

| Escalation rate | Accuracy | Cost |
|---|---|---|
| 0% | 0.92 | $180 |
| 5% | 0.930 | $385 |
| 12% | 0.940 | $672 |
| 25% | 0.942 | $1,160 |
| 50% | 0.943 | $2,140 |

(a) Why 12% rather than 25%?
(b) An ablation shows training without the teacher's reasoning yields 0.87 instead of 0.92. What does that change about the routing decision?
(c) Name one thing you must check before running this project at all.
(d) What production metric tells you the student needs retraining?

---

## Q7 — Diagnosis

Quantization results on the same hardware:

| | bf16 | AWQ 4-bit |
|---|---|---|
| Extraction task accuracy | 0.94 | 0.93 |
| Multi-step reasoning success | 0.84 | 0.76 |

(a) Explain the difference between the two rows.
(b) What does this mean for a blanket "we quantize everything" policy?
(c) Which artifact must you evaluate — the one you trained, or the one you serve?

---

## Q8 — Practical

Take the supplied SFT model in `exercises/m9-verbosity-fix/` with a known verbosity problem and correct it with preference tuning **without losing task accuracy**.

**Deliverable:** preference set, confound audit before and after, training run with sampled outputs at checkpoints, and a before/after evaluation covering both response length and task accuracy.

---
---

# Answer Key

## Q1

**What SFT can learn:** to produce output resembling whichever response you designate as the target. That is a *destination*.

**What SFT cannot learn:** that the direct response is *better than* the hedged one. An SFT example contains one target and no comparison, so nothing in the training signal expresses relative quality. Since the hedged response is also a plausible, well-formed output of the kind the model already produces, using the direct one as an SFT target is a weak signal.

**Why preference tuning can:** a preference pair expresses a *direction* — chosen over rejected — and that direction generalizes to inputs neither example covers. This is why preference tuning fixes verbosity, hedging, confidence calibration, and refusal calibration, which are exactly the qualities SFT leaves untouched.

## Q2

**(a)** The model will learn **"longer is better"** as the dominant signal, because length is the strongest and easiest-to-detect correlate of "chosen" in the data. Completeness may improve slightly; verbosity will improve dramatically.

**(b) Three costs:**
- **Response length up sharply** (~90% in the worked case) — visible immediately in sampled outputs
- **Latency and cost up proportionally** — visible in p95 and cost-per-request telemetry
- **Quality measurably *down*** — visible in judge scores on the held-out set, because padding dilutes

**(c) The fix belongs in the data.** Regenerate rejected responses at matched length so the pairs differ only on completeness, then re-audit until the ratio is roughly 0.9–1.1.

**Not the training config** (no hyperparameter fixes a confounded signal) and **not the prompt** (the data *is* the signal in preference tuning). This is the central discipline of Lesson 9.1.

## Q3

**(a) No.** Reward margin and accuracy measure how well the model separates chosen from rejected *in your data*. If the data contains a confound, a model that learns the confound will score better on these metrics than one that learns the intended quality. Better training metrics can indicate a *worse* model.

**(b) Sample outputs from both runs on the same held-out prompts and read them.** This is the only check that distinguishes the two cases, and it takes five minutes.

**(c) Reward hacking.** It originated in the **preference data** — a surface feature (usually length, sometimes formatting or position) correlated with "chosen," which the model exploited as a shortcut. The Lesson 9.1 confound audit exists to prevent exactly this, which is why it is mandatory rather than advisory.

## Q4 — **B**

You already have an SFT checkpoint, so ORPO's main advantage — collapsing two stages into one — buys you nothing here. DPO on top of the checkpoint gives independent control: you can adjust the preference effect via `beta` without disturbing the task capability the SFT stage produced.

**A** would mean discarding your SFT work and retraining from base. **C** is heavier, less stable, and DPO was specifically derived to avoid it. **D** is the trap this module exists to correct — SFT cannot express relative quality no matter how good the examples are.

**Note:** ORPO would be the right answer if memory were genuinely tight or if you were starting from scratch.

## Q5 — Four conditions

**1. The task must be narrow.** *Failure when violated:* a small model has limited capacity; spreading it across a broad task yields a student that is mediocre everywhere. "Be a helpful assistant" does not distil.

**2. The input distribution must match production.** *Failure when violated:* the student performs well on the clean inputs you imagined and poorly on real traffic — typos, mixed languages, truncated pastes, fragments. This is the most commonly violated condition, and the fix is sampling real inputs from telemetry.

**3. Teacher outputs must be filtered.** *Failure when violated:* the teacher's errors are distilled into the student, which reproduces them confidently. Expect to discard 5–15% using your Module 3 judge plus human spot-checks.

**4. Sufficient volume — thousands, not hundreds.** *Failure when violated:* the student underfits, because it must *generalize* to a task rather than adapt an existing capability. Volume is a cost question here, not a labor question, since the teacher generates it.

## Q6

**(a)** 12% is the knee. It recovers the **full** quality gap — 0.940, matching the teacher's 0.94 — and beyond that you are paying hundreds of dollars a month for 0.003 accuracy. Going to 25% buys 0.002 for an extra $488/month. The threshold should come from the swept curve, not from a round number.

**(b)** Reasoning distillation is worth 5 points, so **without it the student is at 0.87 and the routing threshold would need to be far higher** to reach teacher parity — materially eroding the cost saving. The ablation is not academic: it changes the economics of the whole design, which is why it is worth running.

**(c) The teacher's terms of service.** Providers differ on whether outputs may be used to train other models. Checking before the project rather than after is part of the job.

**(d) The escalation rate.** A rising escalation rate means the input distribution has drifted away from what the student was trained on — an early warning that retraining is needed, well before quality metrics degrade.

## Q7

**(a)** Quantization's quality cost is **task-dependent**. Narrow classification and extraction have wide decision margins, so reduced precision rarely flips an outcome. Multi-step reasoning compounds error across steps — a small perturbation at step 1 can change the entire trajectory — so the same precision reduction costs far more.

**(b)** A blanket policy is wrong in both directions. "We quantize everything" costs 8 points on the reasoning path; "we never quantize" wastes 65% of GPU memory and 45% of throughput on the extraction path. **Quantization is a per-model measurement, not a policy.**

**(c) The one you serve.** A quantized model is a different model from the one you trained. Evaluating the bf16 checkpoint and deploying the AWQ version means your evaluation describes an artifact that is not in production.

## Q8 — Practical rubric

| Criterion | Full credit |
|---|---|
| One dimension | Pairs vary on verbosity only; correctness, citation, and formatting matched |
| Confound audit | Length ratio reported before and after, target 0.9–1.1 |
| Judge ceiling stated | The judge's measured agreement rate acknowledged as the data's quality ceiling |
| Sampled outputs | Generated at checkpoints during training and read, not curves alone |
| Both axes measured | Response length **and** task accuracy, before and after |
| Capability check | Full stratified set re-run; regressions reported |

**Common failure:** fixing verbosity and not re-checking task accuracy. Preference tuning trades against other dimensions, and a model that is now concise and 4 points less accurate has not been improved — it has been changed.
