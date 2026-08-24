# Lesson 9.3 — Distilling a Frontier Model into a Small One

**Module 9: Fine-Tuning II** · ~14 min
**Objective:** M9-LO3 — Distill a frontier model's behavior on a narrow task into a small deployable model, and quantify the cost and latency win.

---

## Introduction

This is the most immediately valuable technique in the fine-tuning track, because it addresses a problem every team with an LLM in production eventually has: **one narrow, high-volume task is the largest line on the AI budget, and it does not need a frontier model.**

Classifying support tickets, extracting fields from documents, routing queries, tagging content — these are tasks where you are paying for a model capable of writing code and reasoning about philosophy, to assign one of four labels, two million times a month.

Distillation moves that work to a small model that does the one job nearly as well, at a fraction of the price. And the framing that matters — the one that gets you a good answer in a system design interview — is that this is an **economics decision with a training implementation**, not a research technique.

---

## Key Concept 1: What distillation actually is here

Classical distillation matches a student model's output *distribution* to a teacher's, using the full probability vector. That requires logit access, which hosted APIs generally do not provide.

**Behavioral distillation** — what you will do — is simpler and works well:

1. Take real inputs from production
2. Generate high-quality outputs with the teacher (a frontier model, prompted well)
3. Filter out the teacher's mistakes
4. SFT a small student on those pairs (Module 8, unchanged)
5. Optionally DPO the student (Lesson 9.2)
6. Evaluate against the teacher as baseline (Lesson 8.4, unchanged)

Every step is machinery you already have. The new thinking is in steps 1 and 3.

---

## Key Concept 2: The four conditions

Distillation succeeds under specific conditions and fails predictably outside them.

**1. The task must be narrow.** "Classify tickets into four categories" distils well. "Be a helpful assistant" does not. A small model has limited capacity — spend it on one job.

**2. The input distribution must match production.** This is the condition most often violated. If you distil on inputs you *imagined*, the student performs well on those and poorly on the real traffic, which is messier — typos, mixed languages, truncated pastes, angry customers writing in fragments. **Sample real inputs from your telemetry.** You have been recording them since Lesson 1.4 precisely so this is possible.

**3. Filter the teacher's mistakes.** Teachers are wrong sometimes. Distilling unfiltered means training the student to reproduce those errors, and the student will do so confidently. Use your Module 3 judge, or a verification step, or human review on a sample. Expect to discard 5–15%.

**4. Volume.** Distillation usually needs more data than task fine-tuning — several thousand examples rather than several hundred — because the student must *generalize* to a task rather than adapt an existing capability. The teacher generates it, so volume is a cost question rather than a labor question.

**Reasoning distillation.** If the teacher produces intermediate reasoning before its answer, training the student on the reasoning *and* the answer often transfers more than the answer alone — the student learns the procedure, not just the mapping. Worth measuring; it costs more tokens per example and frequently pays for itself.

---

## Key Concept 3: The licensing constraint

This is a genuine professional obligation, and checking it is part of the job.

Model provider terms differ on whether outputs may be used to train other models. Some prohibit training competing models on their outputs; some restrict it to non-commercial use; some permit it. Terms change.

**Read the terms for the specific provider and model you are using as teacher, at the time you use it.** If they prohibit it, options include using an open-weights model with a permissive license as teacher, using a provider whose terms allow it, or not distilling.

Being the engineer who raises this before the project rather than after is a small thing that marks you as someone who has worked in a real organization.

---

## Key Concept 4: Routing beats choosing

The framing "teacher or student" is a false binary, and the third option is usually best.

**Route.** The student handles the common case; a confidence or complexity signal escalates the rest to the teacher.

Escalation signals, in rough order of usefulness:

- **Student confidence** — where your output schema includes a confidence field (Lesson 2.2) or you can read token probabilities
- **Input characteristics** — unusual length, unfamiliar vocabulary, multiple languages
- **Disagreement** — run both on a sample and escalate where they differ, useful for tuning the threshold
- **Downstream stakes** — escalate high-value cases regardless of confidence

The economics are compelling because the distribution is usually skewed: escalating 10% of traffic recovers most of the quality gap while retaining ~90% of the cost saving.

**Set the threshold by measurement**, not intuition: sweep it, plot quality against cost, and pick the knee. And **monitor the escalation rate in production** — a rising rate means the input distribution has drifted, which is an early warning that the student needs retraining (Module 10's drift detection).

---

## Worked Example: 2M tickets a month

**The situation.** Ticket classification into four categories, ~2M/month, on a frontier model. Accuracy 0.94. Cost $4,100/month. p95 latency 1,900 ms.

**Step 1 — sample real inputs.** 8,000 tickets from 90 days of telemetry, stratified across categories, deliberately including the messy ones: 6% non-English, 11% under 20 words, 3% pasted stack traces.

**Step 2 — check the license.** Teacher's terms permit training. Recorded in the datasheet with the date and version of the terms.

**Step 3 — generate.** Teacher labels all 8,000, with reasoning before the label. Cost: $31. (Note how small that is against the monthly bill — teacher generation is rarely the expensive part.)

**Step 4 — filter.** Judge review plus 200 human spot-checks: 640 discarded (8%) as teacher errors or ambiguous. **7,360 remain.**

**Step 5 — train.** QLoRA on an 8B instruct model, r=16, 3 epochs, reasoning included in the target. 41 minutes on an A100.

**Step 6 — evaluate** on the untouched held-out set:

| | Teacher | Student | Student + routing (12%) |
|---|---|---|---|
| Accuracy | 0.94 | 0.92 | **0.94** |
| Cost / month | $4,100 | **$180** | $672 |
| p95 latency | 1,900 ms | **310 ms** | 480 ms |
| Operational burden | none | GPU serving | GPU + routing logic |

Read the three columns as three real options:

- **Student alone:** 23× cheaper, 6× faster, 2 points of accuracy given up
- **Routed at 12%:** matches teacher accuracy exactly, still 6× cheaper, and most of the latency win retained

**The threshold sweep** that produced 12%:

| Escalation rate | Accuracy | Monthly cost |
|---|---|---|
| 0% | 0.92 | $180 |
| 5% | 0.930 | $385 |
| **12%** | **0.940** | **$672** |
| 25% | 0.942 | $1,160 |
| 50% | 0.943 | $2,140 |

The knee is clear: 12% recovers the full gap; beyond that you are buying 0.003 accuracy for hundreds of dollars a month.

**The ablation worth reporting:** trained without the teacher's reasoning, on labels alone, the student reached 0.87 rather than 0.92. Reasoning distillation was worth five points — enough that it would have changed the routing threshold materially.

**Annual saving: $41,100.** Against a Lesson 8.1 TCO of roughly $12,000/year for owning the artifact. That is the argument, and it is the kind of number that belongs in a Module 12 interview answer.

---

## Practice

**1. Pick a narrow, high-volume component** of your spine system. Query classification, scope detection (the router from Lesson 8.4 is an excellent candidate), reranking, or extraction all work.

**2. Sample real inputs from telemetry.** Stratified, and **deliberately including the messy ones** — short, non-English, malformed. This is condition 2 and it is where distillations fail.

**3. Check the teacher's terms of service** and record the finding with the date. Even if permitted, write it down.

**4. Generate teacher outputs, with reasoning** if your task benefits from it. Record the generation cost.

**5. Filter.** Judge review plus human spot-check on a sample. Report the discard rate and what the discarded cases had in common — that pattern is informative.

**6. Train the student** using Module 8's pipeline unchanged.

**7. Build the three-way comparison:** teacher, student, routed. Quality, cost per million requests, p95 latency, operational burden.

**8. Sweep the escalation threshold** and find the knee. Plot quality against cost; do not pick a round number.

**9. Run the reasoning ablation** — train once with reasoning in the target and once without. Report the difference. It is usually larger than people expect.

**10. Write `docs/adr-distillation.md`:** the task, the cost model with real volumes, the three options, the chosen configuration, the threshold and how you found it, the license note, and the drift-monitoring plan for the escalation rate.

---

## Visual Aid

*A two-axis chart. X-axis: monthly cost (log scale, $100 to $10,000). Y-axis: accuracy (0.85 to 0.96). Three labeled points: "Student" bottom-left ($180, 0.92), "Teacher" far right ($4,100, 0.94), and a curve of routed configurations arcing between them with markers at 5%, 12%, 25%, 50% escalation. The 12% marker is circled and annotated "knee — matches teacher accuracy at 1/6 the cost." A dashed horizontal line at 0.94 shows the teacher's accuracy level, and the curve visibly flattens against it after 12%. Inset in the corner: a small bar pair labeled "reasoning distillation ablation — with: 0.92, without: 0.87."*

---

## Key Takeaways

- **Distillation is an economics decision with a training implementation.** The target is a narrow, high-volume task where you are paying frontier prices for a fraction of a frontier model's capability.
- **Four conditions:** narrow task, input distribution sampled from real production telemetry, teacher outputs filtered for errors, and enough volume (thousands, not hundreds).
- **Check the teacher's terms of service.** Providers differ on training use, and raising it before the project marks professional judgment.
- **Routing beats choosing.** Escalating a small fraction on a confidence signal typically recovers the full quality gap while keeping most of the saving — and the threshold should come from a swept curve, not intuition.
- **Distil the teacher's reasoning, not just its answers,** where the task benefits. It routinely transfers several points more.
- **Monitor the escalation rate in production** — a rise means input drift and that the student needs retraining.

---

## Next

You now have up to three model artifacts: an SFT adapter, a preference-tuned adapter, and a distilled student. None of them is serving traffic, and getting a trained adapter into production has its own set of traps — merging decisions, quantization quality costs that must be measured rather than quoted, and the versioning discipline that Lesson 8.1's TCO argument promised would arrive.

**Lesson 9.4** ships them.
