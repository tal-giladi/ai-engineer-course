# Module 8 Plan — Fine-Tuning I: When, and SFT/LoRA

**Duration:** 55 min · **Lessons:** 4 · **Depends on:** Module 3
**Outline reference:** `curriculum/course-outline.md` §8, Module 8

---

## Prerequisites

**From Module 3:** the golden dataset, metrics, calibrated judge, and committed baseline. Fine-tuning without a baseline is indistinguishable from superstition, and this module refuses to proceed without one.
**From Modules 1, 5, 6:** production telemetry and trajectories, which become the raw material for the training set in Lesson 8.2.
**From the student:** no PyTorch or GPU experience assumed. Enough is taught inline to run the job and read the loss curve; transformer internals are explicitly out of scope.

**Environment:** paid Google Colab with GPU (T4 sufficient for QLoRA on a 7–8B model, A100 preferable), Hugging Face account with a write token.

---

## Framing

Fine-tuning is the part of the job most likely to be misapplied. It appears in roughly a quarter of postings, so it cannot be skipped, but the professionally valuable skill is the *decision* — and a candidate who can explain why they chose retrieval over fine-tuning for a specific problem interviews far better than one who fine-tuned reflexively. The module therefore opens with the decision and closes with measurement, with the technique in between.

---

## Module Learning Objectives

**M8-LO1** — Decide between prompting, retrieval, and fine-tuning for a given problem, with a cost and maintenance argument for the choice.
**M8-LO2** — Build a supervised dataset from production traces, including deduplication, filtering, and held-out splits.
**M8-LO3** — Run LoRA/QLoRA supervised fine-tuning on Colab GPU and push the adapter to Hugging Face.
**M8-LO4** — Prove — or disprove — improvement against the Module 3 baseline rather than asserting it.

---

## Lesson 8.1 — The Fine-Tuning Decision: When *Not* To

*13 min · serves M8-LO1*

**Concept.** Match the technique to the problem. **Prompting** changes instructions and is free to iterate — the right answer for most format, tone, and task-framing problems. **Retrieval** injects knowledge at request time and is the right answer for facts, because facts change and a fine-tuned model's knowledge is frozen at training time. **Fine-tuning** changes the model's behavioral priors and is the right answer for consistent format adherence the base model keeps violating, domain style and vocabulary, latency and cost reduction by moving a task to a smaller model, and tasks where the instruction is easier to demonstrate than to describe. State the anti-patterns bluntly: fine-tuning to teach facts (they go stale and the model confabulates around them), fine-tuning before exhausting prompting (you have made iteration a training run), and fine-tuning without a baseline (you cannot tell). Then the part practitioners underweight — **the total cost of ownership**: a fine-tuned model is an artifact that must be versioned, re-evaluated on every base-model upgrade, retrained as data drifts, and served on infrastructure you now operate. The question is never "would this help?" but "is this worth owning?"

**Example.** Three real problems worked through a decision matrix — a customer-support bot that keeps inventing policy details (retrieval, not fine-tuning), a system whose JSON structure fails 8% of the time despite constrained decoding (fine-tuning is a legitimate candidate), and an expensive classification task running at scale on a frontier model (fine-tune a small model — this is a cost problem with a training answer). Then the TCO worked in numbers for the second case: training cost, evaluation cost, serving cost, and the re-training bill when the base model is deprecated.

**Practice.** Students complete a decision matrix for four supplied scenarios plus their own spine project, writing a recommendation and TCO estimate for each. At least one recommendation must be "do not fine-tune," with the reasoning stated.

**Summary.** Prompt for behavior, retrieve for facts, fine-tune for priors and for cost. And before any of it, ask whether you want to own the artifact for the next two years.

**Key terms:** prompting vs retrieval vs fine-tuning, behavioral priors, knowledge cutoff, catastrophic forgetting, total cost of ownership, model versioning.
**Visual aid:** Decision tree — problem type → recommended technique → the TCO question that can still veto it.
**Hook:** "You've decided it's justified. Everything now depends on the data, and you already have it — it's sitting in the telemetry you've been writing since Module 1."

---

## Lesson 8.2 — Building a Dataset from Production Traces

*13 min · serves M8-LO2*

**Concept.** Data quality dominates every other variable in fine-tuning, and this audience has an advantage: they have been logging structured traces since Lesson 1.4. Cover **sourcing** — production traces filtered to successful outcomes, human-corrected outputs (the highest-value source, because a correction encodes exactly the gap), and synthetic generation from a stronger model, which is legitimate, extremely common, and must be labeled as such and license-checked. Cover **curation**, which is most of the work: deduplication including near-duplicates that silently overweight one pattern, filtering out examples with the very defect being trained away, balancing so a majority pattern does not swamp the rest, and length distribution matching the target use. Cover **formatting** — chat templates are model-specific and a mismatched template is the single most common cause of a training run that appears to work and produces garbage. Cover **splitting** with the leakage discipline from Lesson 3.2, and the non-negotiable rule: the Module 3 held-out golden set must never appear in training, or the final comparison is worthless. Cover sizing honestly — 500–1,000 excellent examples routinely beat 50,000 scraped ones for LoRA.

**Example.** A 4,000-trace log reduced to 800 training examples, walked filter by filter with the count after each step and one example shown at each stage. Then a chat-template mismatch demonstrated: a model trained with the wrong template, its loss curve looking perfectly healthy, and its outputs unusable.

**Practice.** Students build a training set from their own spine telemetry — filter, deduplicate, balance, format with the correct chat template, split — and produce a datasheet documenting every decision, the count after each filter, and an explicit leakage check against the Module 3 held-out set.

**Summary.** Curation is the job. Verify the chat template before you spend a GPU hour, and prove the held-out set never leaked.

**Key terms:** supervised fine-tuning data, instruction/response pair, near-duplicate detection, class balance, chat template, synthetic data, distillation data, datasheet, leakage check.
**Visual aid:** Funnel diagram — 4,000 traces → filters → 800 examples — with the count and rejection reason at each stage.
**Hook:** "Clean data, correct template, held-out set intact. Now the part everyone thinks is the hard part, and isn't."

---

## Lesson 8.3 — LoRA and QLoRA Supervised Fine-Tuning on Colab

*14 min · serves M8-LO3*

**Concept.** Enough theory to make informed choices, no more. **Full fine-tuning** updates every weight and needs many times the model size in GPU memory — out of reach on Colab and rarely necessary. **LoRA** freezes the base model and trains small low-rank adapter matrices injected into attention layers, typically under 1% of parameters, which cuts memory dramatically and produces an adapter measured in megabytes rather than gigabytes. **QLoRA** additionally quantizes the frozen base to 4-bit, making a 7–8B model trainable on a free-tier-class GPU. Cover the hyperparameters that actually matter and what each does: rank (capacity — higher fits more, overfits sooner), alpha (scaling, conventionally 2× rank), target modules (attention projections by default, all-linear for harder tasks), learning rate (LoRA tolerates roughly 10× higher than full fine-tuning), and epochs (2–3 is usually right; more memorizes). Cover reading the training curve — training loss falling while validation loss rises is overfitting, and the fix is fewer epochs or more data, not a bigger rank. Cover the practical workflow end to end in `transformers` + `peft` + `trl`, then pushing the adapter to Hugging Face with a model card.

**Example.** A complete Colab notebook fine-tuning a 7–8B model with QLoRA on the Lesson 8.2 dataset — 800 examples, roughly 25 minutes, adapter under 100 MB — with the loss curves shown and interpreted, including a deliberate overfitting run at 8 epochs for contrast.

**Practice.** Students run the fine-tune on their own dataset, read their loss curves and state whether they overfit, push the adapter to Hugging Face with a model card recording base model, data, hyperparameters, and intended use. They then run one variation (different rank or epoch count) and compare curves.

**Summary.** LoRA makes fine-tuning accessible and adapters portable. Rank, learning rate, and epochs are the levers; the validation curve tells you when to stop.

**Key terms:** full fine-tuning, LoRA, QLoRA, adapter, rank, alpha, target modules, 4-bit quantization, epochs, overfitting, `peft`, `trl`, model card.
**Visual aid:** Two loss-curve charts side by side — healthy convergence and clear overfitting — with the stopping point marked on each.
**Hook:** "The adapter exists and the loss went down. Neither of those facts tells you it's better."

---

## Lesson 8.4 — Did It Work? Measuring Against the Baseline

*13 min · serves M8-LO4*

**Concept.** The lesson that makes the previous three worth anything, and the one that most tutorials omit. Falling loss means the model fits the training distribution — it says nothing about task performance, and this is where practitioners deceive themselves. Run the Module 3 harness against the fine-tuned model on the untouched held-out set, and compare on the full metric vector plus cost and latency. Cover **regression checking**: fine-tuning on a narrow task frequently degrades everything else, so evaluate off-task capabilities too — a model that formats perfectly and can no longer refuse out-of-scope questions is worse overall, and only the stratified golden set will reveal it. Cover the **honest comparison table**: fine-tuned small model versus prompted frontier model, on quality, cost per thousand requests, p95 latency, and operational burden — because the fine-tune usually wins on cost and latency while losing some quality, and the decision belongs to whoever owns the budget. Cover the outcome practitioners are reluctant to report: sometimes the fine-tune loses, and reporting that is a professional strength. Close with the artifact — a model card and eval report that a hiring panel can read.

**Example.** A fine-tuned model beating the frontier baseline on format adherence (0.92 to 0.99) at one-twentieth the cost, while dropping four points on answer relevance and — caught only by the stratified set — losing half its refusal accuracy. Then the decision discussion: ship, ship with a router, or discard.

**Practice.** Students evaluate their fine-tuned model on the held-out golden set across all metrics plus cost and latency, produce the comparison table including off-task regression, and write a ship/no-ship recommendation with reasoning. `eval-report.md` is updated whichever way the result falls.

**Summary.** Loss is not quality. Evaluate on the untouched held-out set, check for off-task regression, compare on cost and latency as well as accuracy, and report a negative result honestly.

**Key terms:** held-out evaluation, off-task regression, capability degradation, cost per thousand requests, model card, ship/no-ship decision, negative result.
**Visual aid:** Comparison table — baseline vs fine-tuned across quality metrics, cost, latency, and operational burden — with regressions highlighted in red.
**Hook:** "Supervised fine-tuning teaches the model to imitate examples. It cannot teach it that one good answer is better than another good answer. That needs preferences — Module 9."

---

## Spine Increment

**Deliverable:** `spine/finetune/` — a datasheet, a Colab training notebook, a LoRA adapter published to Hugging Face with a model card, and a committed evaluation comparing it against the Module 3 baseline.

**Acceptance criteria:**
1. A written decision record justifying fine-tuning for this task over prompting and retrieval, with a TCO estimate.
2. A datasheet with per-filter counts and an explicit leakage check against the held-out golden set.
3. Adapter trained, loss curves interpreted in writing, model card complete.
4. Evaluation on the untouched held-out set across the full metric vector, plus cost and latency.
5. Off-task regression explicitly checked and reported.
6. A ship/no-ship recommendation — negative results accepted and expected in some cases.

---

## Standalone Exercise

Fine-tune a small model to reliably produce a strict output format the base model violates roughly 15% of the time, and quantify the improvement — including whether general capability regressed. Deliverable: `exercises/m8-format-adherence/` with dataset, notebook, adapter link, and before/after evaluation.

---

## Dependencies

| Lesson | Requires |
|---|---|
| 8.1 | Module 3 baseline |
| 8.2 | 8.1, plus Module 1/5/6 telemetry |
| 8.3 | 8.2 |
| 8.4 | 8.3, plus the Module 3 harness |

**Downstream:** Module 9 applies preference tuning to this checkpoint and distills from it. Module 10 serves the adapter with vLLM. Module 12 uses the model card and eval report as portfolio artifacts.

---

## Assessment Focus (Phase 4)

Decision-weighted. Given four scenarios, choose the technique and defend it — with at least one where fine-tuning is wrong. Given a loss curve, diagnose overfitting and prescribe the fix. Given a training set built from production logs, identify the leakage. Given a fine-tuned model that beat the baseline on the target metric and lost refusal accuracy, make a ship recommendation. One practical: given a dataset and a failing evaluation, determine whether the problem is the data, the hyperparameters, or the chat template.
