---
id: "09.2"
module: 9
minutes: 13
practice_minutes: 90
prerequisites: ["09.1"]
objectives:
  - "Run DPO or ORPO on top of an SFT checkpoint and evaluate for both gains and regressions."
volatility: implementation
sources: []
last_verified: "2026-08-24"
---

# 09.2 · DPO and ORPO in Practice

You have preference pairs that vary on one audited dimension. This lesson turns them into weights.

The mechanics are familiar — it is a training loop with a different loss function, and everything you learned in Lesson 8.3 about curves, epochs, and sampling outputs applies. What is new is the failure mode: preference tuning can produce a training run whose metrics look *excellent* while the model gets measurably worse, in a way that supervised fine-tuning does not.

That failure has a name, and catching it is the point of this lesson.

---

## Key Concept 1: DPO, without the mathematics

DPO adjusts the model so that, for each pair, the chosen response becomes more likely and the rejected response less likely — **while staying close to a frozen reference model**.

That second clause is the whole design. Without it, the optimization has a trivially degenerate solution: make the chosen responses arbitrarily more likely, which destroys the model. The reference model is a leash.

Three things follow practically:

**You need an SFT checkpoint first.** DPO refines behavior; it does not teach a task. Run it on top of Lesson 8.3's adapter, not on a base model.

**Two model copies live in memory** — the policy being trained and the frozen reference. This is why QLoRA and a small base model are assumed. (With LoRA there is a convenient shortcut: the reference is the base model with adapters disabled, so you do not pay for a second full copy.)

**`beta` controls the leash:**

- **Low `beta` (0.01–0.05)** — loose leash. The model can move far, learns the preference strongly, and risks drifting into degradation.
- **High `beta` (0.3–0.5)** — tight leash. Safe, and often nothing changes.
- **Start at 0.1.** Sweep 0.05 and 0.3 to see the shape.

```python
from trl import DPOTrainer, DPOConfig

trainer = DPOTrainer(
    model=sft_model,                  # your Lesson 8.3 adapter
    ref_model=None,                   # LoRA: reference = adapters disabled
    train_dataset=pref_ds,            # prompt / chosen / rejected
    peft_config=peft_config,
    args=DPOConfig(
        beta=0.1,
        num_train_epochs=1,           # usually 1; 2 at most
        learning_rate=5e-6,           # ~40× lower than SFT
        per_device_train_batch_size=2,
        gradient_accumulation_steps=8,
        eval_steps=20,
        logging_steps=5,
    ),
)
```

Two settings to note. **Learning rate is far lower than SFT** — 5e-6 rather than 2e-4. Preference tuning nudges; SFT teaches. Using an SFT learning rate here will wreck the model. And **one epoch is usually right**; preference data overfits fast.

---

## Key Concept 2: ORPO, the cheaper path

ORPO combines the SFT objective and the preference objective into one loss, trained in a single stage.

**Advantages:** no reference model, so roughly half the memory. No separate SFT stage — train from a base model directly on preference data. Simpler pipeline, fewer artifacts to version.

**Trade-offs:** less separation of concerns, so less control over the two effects independently. Fewer worked examples and community recipes available.

**Choose ORPO when** memory is tight, you are starting from scratch, or you want one stage instead of two. **Choose DPO when** you already have an SFT checkpoint you are happy with — which, having just completed Module 8, you do.

The rest of this lesson uses DPO; everything about reading runs applies to both.

---

## Key Concept 3: Reading a preference run

The metrics differ from SFT and the important one is not loss.

**Reward margin** — the gap between the implicit reward assigned to chosen versus rejected responses. It should rise. This is your primary signal.

**Reward accuracy** — the fraction of pairs where the model now prefers the chosen response. Rises toward some plateau; 100% means memorization.

**KL divergence from the reference** — how far the policy has drifted. Should rise slowly and steadily. A sharp climb means the leash is too loose and degradation is likely.

**Loss** — descends, and tells you much less than in SFT. Do not lead with it.

Three failure shapes:

**Under-training.** Margin barely moves, KL near zero, outputs unchanged. `beta` too high or learning rate too low.

**Drift.** KL climbs sharply, outputs become erratic — incoherent, repetitive, or bizarrely formatted. `beta` too low or too many epochs.

**Reward hacking.** The important one. The margin climbs beautifully, reward accuracy approaches 1.0, KL looks reasonable — and sampled outputs are worse. The model found a shortcut: a surface feature correlated with "chosen" in your data that is not the quality you meant. **This is what the Lesson 9.1 confound audit was preventing**, and it is why the audit is mandatory rather than advisory.

---

## Key Concept 4: Sample outputs, or you will not see it

Every failure above except under-training is invisible in the training metrics.

**Generate on the same five held-out prompts at three or four checkpoints during the run, and read them.** Not skim — read. Compare against the pre-DPO outputs.

This is the same habit from Lesson 8.3, and it matters more here, because reward hacking specifically produces *good-looking metrics*. The curve is not merely uninformative; it is actively reassuring while the model degrades.

---

## Key Concept 5: Check for capability collapse

Same discipline as Lesson 8.4. Preference tuning on one dimension can degrade others — often the ones it trades against.

Evaluate after DPO on:

- The target dimension (should improve)
- Your full stratified golden set (watch refusal again)
- Task accuracy from Module 8 (preference tuning can erode it)
- The general capability probe
- **Response length and cost** — always, because verbosity is the most common uninvited passenger

---

## Worked Example 1: A healthy run

400 length-matched pairs targeting hedging calibration. `beta=0.1`, 1 epoch, lr 5e-6.

| Step | Reward margin | Reward accuracy | KL |
|---|---|---|---|
| 20 | 0.14 | 0.61 | 0.02 |
| 40 | 0.51 | 0.74 | 0.09 |
| 60 | 0.88 | 0.83 | 0.18 |
| 80 | 1.12 | 0.87 | 0.26 |
| 100 | 1.24 | 0.89 | 0.31 |

Margin rising, accuracy plateauing sensibly below 1.0, KL climbing gently.

Sampled outputs:

> **Before:** *"Based on the documentation, it appears that batch endpoints may not support streaming, though this could vary."*
> **After:** *"Batch endpoints do not support streaming responses."*

Exactly the target dimension. Held-out evaluation: hedging score +0.21, task accuracy −0.01 (noise), length +3%, refusal unchanged. Ship.

## Worked Example 2: Reward hacking

Same task, but with pairs built before the Lesson 9.1 audit — chosen responses averaged twice the length of rejected.

| Step | Reward margin | Reward accuracy | KL |
|---|---|---|---|
| 20 | 0.31 | 0.72 | 0.03 |
| 40 | 0.94 | 0.88 | 0.11 |
| 60 | 1.67 | 0.94 | 0.19 |
| 80 | 2.41 | **0.98** | 0.24 |
| 100 | 2.98 | **0.99** | 0.29 |

Better than the healthy run on every number. Margin more than double, accuracy at 0.99, KL well controlled. By the metrics, an excellent run.

Sampled outputs:

> *"Regarding your question about batch endpoint streaming support, it's worth examining the documentation carefully. The documentation indicates that batch endpoints do not currently support streaming responses. This is an important architectural consideration, and it's worth noting that this constraint applies to all batch operations..."*

Longer, more padded, *and still hedged*. The model did not learn calibration at all. It learned that length predicts "chosen," which was true in the data.

Held-out: hedging score +0.02 (nothing), length +94%, cost +71%, task accuracy −0.03, overall quality −0.02.

**The run with better training metrics produced the worse model.**

## Worked Example 3: The `beta` sweep

Same audited data, three values:

| `beta` | Final margin | KL | Hedging Δ | Coherence check |
|---|---|---|---|---|
| 0.03 | 3.41 | 0.94 | +0.26 | **degraded** — repetition, occasional truncation |
| 0.10 | 1.24 | 0.31 | +0.21 | fine |
| 0.30 | 0.38 | 0.08 | +0.06 | fine, barely changed |

`beta=0.03` moved the target dimension most and broke the model — visible only in sampled outputs, since its margin was the best of the three. `beta=0.3` was safe and nearly useless. **0.1 chosen.**

---

## Practice

**1. Run DPO on your Lesson 8.3 SFT adapter** with your audited pairs. `beta=0.1`, 1 epoch, lr 5e-6, eval every 20 steps.

**2. Sweep `beta`** at 0.03, 0.1, and 0.3. Three short runs.

**3. Sample outputs at checkpoints** for every run — the same five held-out prompts, at three points, compared against pre-DPO. **Read them.** Record what you see.

**4. Record the three metrics** — reward margin, reward accuracy, KL — for each run, and note which run had the best metrics versus which had the best outputs. If those differ, you have found reward hacking, which is the most valuable result available in this lesson.

**5. Evaluate the winner** on the full held-out set: target dimension, all strata, task accuracy from Module 8, general capability probe, and **response length and cost**.

**6. Report both gain and regression.** If the target improved and something else fell, state the trade and make a ship recommendation, as in Lesson 8.4.

**7. Optional: run ORPO** on the same data from the base model and compare — quality, memory, wall clock, and pipeline complexity. A one-paragraph comparison is a good interview answer.

**8. Append to `eval-report.md`** with the full SFT → DPO progression.

---

## Visual Aid

*Two training dashboards side by side, each with three stacked mini-charts (reward margin, reward accuracy, KL divergence) and a text panel beneath showing a sampled output. Left, "Run A — audited data": margin rising to 1.24, accuracy plateauing at 0.89, KL gentle, and a short direct sampled answer in green. Right, "Run B — length-confounded data": every chart visibly better — margin to 2.98, accuracy 0.99, KL controlled — with a green tint on the charts, and beneath, a long padded hedged sampled answer boxed in red. A large annotation spans both: "Run B wins on every training metric and is the worse model. The charts cannot tell you this. The samples can."*

---

## Key Takeaways

- **DPO makes chosen responses more likely and rejected less, on a leash to a frozen reference.** `beta` is the leash: 0.1 to start, low drifts, high does nothing.
- **Run DPO on an SFT checkpoint, at a much lower learning rate (~5e-6) and usually one epoch.** Preference tuning nudges; it does not teach.
- **ORPO merges SFT and preference into one cheaper stage** with less independent control — a good choice when memory is tight or you are starting fresh.
- **Reward margin, reward accuracy, and KL are the metrics; loss is not.** Rising margin, plateauing accuracy below 1.0, gently rising KL is healthy.
- **Reward hacking produces better training metrics and a worse model.** Sampling outputs at checkpoints is the only way to see it — and the Lesson 9.1 confound audit is what prevents it.
- **Check for capability collapse afterward,** including response length and cost, which is where verbosity hides.

---

## Next

You have improved a small model using your own preferences. The other way to improve a small model is to let a much larger one teach it — and that is where the biggest cost reductions in this field come from.

**Lesson 9.3** covers distillation as an economics problem: taking a narrow high-volume task off a frontier model, what makes the transfer work, the licensing constraint that is a genuine professional concern, and the routing pattern that beats choosing between teacher and student.
