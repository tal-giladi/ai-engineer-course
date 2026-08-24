# Module 9 Plan — Fine-Tuning II: Preference Tuning & Distillation

**Duration:** 55 min · **Lessons:** 4 · **Depends on:** Module 8
**Outline reference:** `curriculum/course-outline.md` §8, Module 9

---

## Prerequisites

**From Module 8:** an SFT checkpoint, a datasheet discipline, and the habit of evaluating against an untouched held-out set. Preference tuning is applied *on top of* SFT and is not a substitute for it.
**From Module 3:** the calibrated judge — which becomes a preference labeler here, making Lesson 3.4's calibration work load-bearing rather than academic.

**Environment:** Colab GPU. DPO holds two model copies in memory, so QLoRA and a small base model are assumed; ORPO is offered as the lower-memory path.

---

## Framing

SFT teaches the model to imitate examples. It has no way to express that one acceptable answer is better than another acceptable answer, and that comparative signal is where most remaining quality lives — tone, verbosity, hedging, refusal calibration, format preference under ambiguity. Preference tuning is how that signal gets in. Distillation is the same machinery pointed at a cost problem: use a strong model to teach a small one a narrow task. Both appear in senior postings; together they complete the training track.

---

## Module Learning Objectives

**M9-LO1** — Construct preference data that encodes a real quality dimension, and explain what SFT cannot teach that preferences can.
**M9-LO2** — Run DPO or ORPO on top of an SFT checkpoint and evaluate for both gains and regressions.
**M9-LO3** — Distill a frontier model's behavior on a narrow task into a small deployable model, and quantify the cost and latency win.
**M9-LO4** — Ship the result: merge or serve adapters, quantize, and account for the quality cost of quantization.

---

## Lesson 9.1 — Preference Data: Defining What "Better" Means

*13 min · serves M9-LO1*

**Concept.** Start with the gap: an SFT set contains one target output per input, so anything the model learns about *relative* quality is accidental. A preference set contains a prompt with a chosen and a rejected response, which expresses a direction rather than a destination — and direction is what fixes verbosity, hedging, unwarranted confidence, and miscalibrated refusal. Place the technique in its lineage briefly: RLHF trains a reward model then optimizes against it with reinforcement learning; DPO derives an equivalent objective that skips the reward model entirely, which is why it became the practical default. Mention RFT/RLVR as the frontier for tasks with a verifiable answer, so students can speak to it, without teaching it hands-on. Then the real content — **sourcing pairs**: human comparisons (best, expensive), the Lesson 3.4 calibrated judge (scalable, and only as trustworthy as its measured agreement), production A/B signal, and constructed pairs where the rejected response is deliberately generated with the defect being trained away. Cover the failure modes: pairs differing on more than one dimension teach nothing clean, and length-correlated preferences produce a model that learns "longer is better" — the same verbosity bias from Lesson 3.4, now baked into weights.

**Example.** Two responses to one question, both factually correct, one hedging through three sentences of throat-clearing. As SFT data this teaches nothing; as a preference pair it teaches directness. Then a contaminated set where chosen responses average twice the length of rejected ones, and the resulting model's verbosity — the concrete cost of not controlling the confound.

**Practice.** Students define one quality dimension for the spine project from their Lesson 3.1 charter, build 200–400 preference pairs using their calibrated judge with human spot-checking, and run a confound audit — length, formatting, and position — documenting what they found and corrected.

**Summary.** Preferences encode direction. Vary one dimension per pair, audit for length confounds, and remember the judge's measured agreement is the ceiling on the data's quality.

**Key terms:** preference pair, chosen/rejected, RLHF, reward model, DPO, ORPO, RFT/RLVR, judge-labeled preferences, length confound, confound audit.
**Visual aid:** Side-by-side pair with the varying dimension highlighted and the controlled dimensions listed underneath.
**Hook:** "You have pairs. Now the algorithm that turns 'this one is better' into weights."

---

## Lesson 9.2 — DPO and ORPO in Practice

*13 min · serves M9-LO2*

**Concept.** **DPO** optimizes the policy to raise the likelihood of chosen responses and lower rejected ones, held near a frozen reference model by a KL term — the intuition being "shift toward the chosen response, but don't wander far from where SFT left you." The `beta` parameter controls that leash: too low and the model drifts and degrades, too high and nothing changes. Cover the memory implication — a reference copy alongside the policy — and why LoRA plus a small base is assumed. Cover **ORPO** as the practical alternative: it folds preference optimization into the SFT objective itself, needs no reference model and no separate SFT stage, and is therefore cheaper and simpler, at some cost in control. Cover reading a preference-tuning run, which differs from SFT: watch the implicit reward margin between chosen and rejected, and watch for **reward hacking** — the margin climbing beautifully while outputs get worse, typically by exploiting a confound in the data. Cover the two characteristic failure modes: over-optimization (fluent, confident, degraded) and capability collapse (the target dimension improves and something unrelated falls apart).

**Example.** A DPO run on the Lesson 9.1 dataset, with reward margin and KL divergence plotted, producing measurably more direct answers. Then a deliberate high-`beta`/low-`beta` comparison showing under-training and drift. Then a reward-hacked run where the margin looks excellent and sampled outputs are visibly worse — the reason sampling outputs during training is mandatory, not optional.

**Practice.** Students run DPO (or ORPO) on their SFT checkpoint, sweep two `beta` values, sample outputs at checkpoints rather than trusting the curves, and select a run with justification. They report both the target-dimension gain and any capability regression.

**Summary.** DPO shifts toward preferred behavior on a leash; ORPO does it in one cheaper stage. Watch the margin, sample the outputs, and assume reward hacking until you have looked.

**Key terms:** DPO, ORPO, reference model, KL divergence, `beta`, implicit reward margin, reward hacking, over-optimization, capability collapse.
**Visual aid:** Training-run dashboard — reward margin, KL divergence, and sampled outputs at three checkpoints — with the reward-hacking run beside the healthy one.
**Hook:** "You've improved a small model with your own preferences. Now use a much bigger model as the teacher instead."

---

## Lesson 9.3 — Distilling a Frontier Model into a Small One

*14 min · serves M9-LO3*

**Concept.** Distillation reframed as an economics problem this audience will meet immediately: a frontier model doing a narrow, high-volume task is usually the largest line on an AI budget, and most of that capability is unused. The practical form is **behavioral distillation** — generate high-quality outputs from the teacher on your real input distribution, then SFT (and optionally DPO) a small student on them, using everything from Module 8. Cover what makes it work: a *narrow* task, an input distribution that matches production (sampled from real telemetry rather than imagined), teacher outputs filtered by the Module 3 judge so errors are not distilled, and enough volume — usually thousands, not hundreds, since the student must generalize rather than adapt. Cover reasoning distillation briefly: training on the teacher's intermediate reasoning as well as its answer often transfers more than the answer alone. Cover the licensing reality plainly, since it is a genuine professional constraint: provider terms differ on whether outputs may be used to train competing models, and checking is part of the job. Close with the routing pattern that beats a binary choice — student handles the common case, a confidence or complexity signal escalates the rest to the teacher, capturing most of the saving while protecting the tail.

**Example.** A classification task at 2 million requests a month costed on the frontier model, then distilled into a small model at a fraction of the price with a two-point quality drop — and then the routed version recovering the two points while keeping most of the saving, with all three configurations in one table.

**Practice.** Students pick a narrow high-volume component of the spine system, generate a filtered teacher dataset from real inputs, train a student, and produce the three-way comparison: teacher, student, routed. They include the cost model and check the teacher's terms of service.

**Summary.** Distill narrow, high-volume tasks. Match the real input distribution, filter the teacher's mistakes, check the license, and consider routing before choosing between teacher and student.

**Key terms:** knowledge distillation, teacher/student, behavioral distillation, reasoning distillation, input distribution matching, teacher output filtering, model routing, escalation, terms of service.
**Visual aid:** Cost-per-million-requests chart with quality on a second axis, plotting teacher, student, and routed configurations.
**Hook:** "Three model artifacts, none of them served. Getting them into production has its own set of traps."

---

## Lesson 9.4 — Shipping Adapters: Merging, Quantizing, Serving

*13 min · serves M9-LO4*

**Concept.** The operational half, and the part that turns a notebook result into something deployable. **Merging**: a LoRA adapter can be merged into base weights for a single self-contained model — simpler to serve and slightly faster — or kept separate, which allows one base model in memory serving many adapters and swapping them per request. Cover the trade honestly: merged is simpler, unmerged is far more efficient when serving several fine-tunes. **Quantization** for inference: 8-bit and 4-bit weights, the memory and throughput win, and the quality cost, which is real, task-dependent, and must be measured rather than assumed — several points on a hard reasoning task, near zero on a narrow classification. Cover formats and where each belongs (GPTQ/AWQ for GPU serving, GGUF for CPU and local). Cover **versioning and provenance** as a first-class concern: adapters need semantic versions, a recorded base model and commit, the datasheet, the eval report, and a rollback path — because a fine-tuned model is a deployed artifact with a lifecycle, which was the TCO argument from Lesson 8.1 arriving on schedule. Close with the operational trigger nobody plans for: the base model gets deprecated, and the whole chain must be re-run.

**Example.** One adapter served three ways — merged and full-precision, merged and 4-bit quantized, and unmerged with hot-swapping — compared on memory, p95 latency, throughput, and held-out quality, with the quantization quality cost measured rather than quoted.

**Practice.** Students merge or mount their adapter for vLLM serving, quantize one variant, and evaluate all configurations on the held-out set for quality, latency, and memory. They write the model's version record — base model and commit, adapter version, datasheet link, eval report link, rollback instruction — and a one-paragraph plan for base-model deprecation.

**Summary.** Merge for simplicity, keep separate for multi-adapter efficiency, and measure what quantization costs on *your* task. Version the artifact with its data and evaluation, and plan for the base model going away.

**Key terms:** adapter merging, multi-adapter serving, hot-swapping, post-training quantization, GPTQ, AWQ, GGUF, quality/throughput trade-off, model versioning, provenance, rollback, deprecation.
**Visual aid:** Serving-configuration matrix — memory, p95 latency, throughput, quality delta — across the three configurations.
**Hook:** "You have models worth serving. Module 10 is where the whole system becomes a service someone else can call — and where every number you've collected since Module 1 finally gets a dashboard."

---

## Spine Increment

**Deliverable:** `spine/finetune/preference/` and `spine/finetune/distill/` — a preference dataset with a confound audit, a DPO or ORPO checkpoint, a distilled student with a routing configuration, and a serving-ready versioned adapter.

**Acceptance criteria:**
1. Preference pairs vary one dimension; the confound audit is documented with what was found and corrected.
2. The preference run reports the target-dimension gain *and* an explicit capability-regression check.
3. Outputs were sampled at checkpoints during training, not just curves read.
4. Distillation compares teacher, student, and routed on quality and cost per million requests, with the license checked.
5. The adapter is served under vLLM in at least two configurations with measured quality, latency, and memory.
6. A version record exists with base model, datasheet, eval report, and rollback path.

---

## Standalone Exercise

Take a supplied SFT model with a known verbosity problem and correct it with preference tuning without losing task accuracy. Deliverable: `exercises/m9-verbosity-fix/` with the preference set, the confound audit, the training run, and a before/after evaluation covering both response length and task accuracy.

---

## Dependencies

| Lesson | Requires |
|---|---|
| 9.1 | Module 8 SFT checkpoint, Lesson 3.4 judge |
| 9.2 | 9.1 |
| 9.3 | Module 8 (SFT machinery); independent of 9.1–9.2 |
| 9.4 | 9.2 or 9.3 (needs an artifact to ship) |

**Downstream:** Module 10 serves these artifacts and routes between them. Module 11 covers the safety implications of a self-hosted model with no provider-side filtering. Module 12 uses the model cards and cost analysis as portfolio material.

---

## Assessment Focus (Phase 4)

Judgment plus operational reasoning. Given a quality problem, decide whether SFT, DPO, or neither applies. Given a preference set where chosen responses are systematically longer, predict the trained model's behavior. Given a training run with a rising reward margin and worsening samples, name the failure. Given a distillation proposal for a broad open-ended task, argue against it. Given a quantization decision, state what must be measured before shipping. One practical: given a task and a volume, model the cost of teacher, student, and routed configurations and recommend one.
