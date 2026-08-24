# Lesson 9.1 — Preference Data: Defining What "Better" Means

**Module 9: Fine-Tuning II** · ~13 min
**Objective:** M9-LO1 — Construct preference data that encodes a real quality dimension, and explain what SFT cannot teach that preferences can.

---

## Introduction

Your Lesson 8.4 evaluation left a specific residue: the fine-tuned model was five points below baseline on ambiguous and adversarial strata, and its refusal behavior had to be patched with a router rather than fixed.

Those are not format problems. The outputs were well-formed. They were *worse* in ways that supervised fine-tuning has no vocabulary for — too hedged, too confident, too long, answering when it should have declined.

SFT can only say "produce this." It cannot say "this is better than that." Preference tuning can, and that comparative signal is where most remaining quality lives.

---

## Key Concept 1: What SFT structurally cannot do

An SFT example is one input and one target output. The model learns to imitate the target.

The problem: for most tasks there are many acceptable outputs, and your training target is only one of them. The model learns "produce something like this," which is a destination. It learns nothing about the *direction* from a worse acceptable answer to a better one.

Consider two responses to the same question:

**A:** *"Based on the documentation provided, it appears that batch endpoints may not currently support streaming responses, though you may wish to verify this with the latest version of the API reference as behavior can vary between releases."*

**B:** *"Batch endpoints do not support streaming responses."*

Both are correct. Both are well-formed. B is better for a documentation assistant — it is direct, unhedged, and does not manufacture doubt where the documentation is clear.

As SFT data, using B as the target teaches "produce sentences like B" — which is a weak signal, because A is also a sentence like the kinds of sentences the model produces. As a **preference pair**, (chosen: B, rejected: A) teaches directness itself, as a direction, generalizing to inputs neither example covers.

That is the whole difference, and it is why preference tuning fixes qualities like verbosity, hedging, and calibration that SFT cannot touch.

---

## Key Concept 2: The lineage, briefly

Worth knowing well enough to discuss in an interview, not worth implementing.

**RLHF** — collect human comparisons, train a separate *reward model* to predict which output a human prefers, then optimize the policy against that reward with reinforcement learning. It works, it is how the major instruction-tuned models were shaped, and it is heavy: two models, an RL loop, and considerable instability.

**DPO** (Direct Preference Optimization) — derives an objective mathematically equivalent to that RLHF setup but computed **directly on preference pairs**, with no reward model and no RL loop. Same signal, ordinary supervised training. This is why it became the practical default and why you can run it on Colab.

**ORPO** folds preference optimization into the SFT objective itself — no reference model, no separate SFT stage. Cheaper and simpler; less control. Lesson 9.2.

**RFT / RLVR** (reinforcement fine-tuning with verifiable rewards) — for tasks where correctness can be checked programmatically: code that passes tests, mathematics with a checkable answer. The reward is the verifier, not a model. Increasingly prominent, appears in senior postings, and out of scope here — but know the shape and when it applies.

---

## Key Concept 3: Where pairs come from

**Human comparison.** Show two outputs, ask which is better. The gold standard, and expensive. Even 100 human-labeled pairs are valuable — as calibration for the scalable sources below.

**Your calibrated judge (Lesson 3.4).** Generate two responses, have the judge pick. Scalable and cheap. Its ceiling is its measured agreement rate: a judge that agrees with humans 86% of the time produces preference data that is roughly 86% right, and the model will faithfully learn the 14% too. **This is why the Lesson 3.4 calibration was load-bearing rather than academic.**

**Production signal.** Thumbs up/down, edits, regenerations, abandonment. Genuine and noisy — a thumbs-down may be about latency, not quality.

**Constructed pairs.** Generate the rejected response deliberately with the defect you want to remove. To train against verbosity, prompt one generation for a padded answer and one for a direct one. This is the most controlled source and the one you will use most, because it isolates the dimension by construction.

---

## Key Concept 4: One dimension per pair, and the length confound

The two rules that decide whether preference data works.

**Vary one dimension.** If chosen and rejected differ in verbosity *and* accuracy *and* formatting, the model cannot tell which difference you meant. It will latch onto whichever is easiest to detect — usually a surface feature. Pairs that differ on one axis produce a clean gradient; pairs that differ on three produce noise.

**Control for length.** This is the specific, well-documented failure and it deserves its own attention.

If your chosen responses average twice the length of rejected ones — which happens easily, because judges exhibit verbosity bias (Lesson 3.4) and thorough answers *are* often longer — the model will learn **"longer is better"** as the dominant signal. You will have trained verbosity into the weights while believing you trained quality.

The audit is simple and mandatory:

```python
chosen_len = mean(len(p.chosen.split()) for p in pairs)
rejected_len = mean(len(p.rejected.split()) for p in pairs)
ratio = chosen_len / rejected_len
# want roughly 0.9–1.1, unless brevity IS the dimension
```

If the ratio is off and length is *not* your target dimension, fix the data: rebalance, or construct length-matched pairs. Do not fix it with a prompt instruction during training — the data is the signal.

Audit formatting and position too. If chosen responses are systematically more bulleted, you are training bullets.

---

## Worked Example 1: A pair that teaches nothing, and one that does

**Bad pair — three dimensions at once:**

> **Chosen:** *"Batch endpoints do not support streaming. See api/batch.md#limitations."* (short, cited, direct)
> **Rejected:** *"I think streaming might work with batch endpoints in some configurations, though I'm not certain."* (long, uncited, hedged, **and factually wrong**)

Correctness, citation, and hedging all differ. The model will most likely learn the easiest signal — brevity — and you wanted calibration.

**Good pair — one dimension:**

> **Chosen:** *"Batch endpoints do not support streaming responses. See api/batch.md#limitations."*
> **Rejected:** *"Based on the documentation, it appears batch endpoints may not support streaming responses, though behavior can vary. See api/batch.md#limitations."*

Both correct. Both cited. Both similar length. The only difference is hedging. **That** teaches calibration.

## Worked Example 2: The confound, and what it produced

A team builds 400 preference pairs using their judge, no audit.

```
mean chosen length:   142 words
mean rejected length:  71 words
ratio: 2.00
```

They run DPO. The target dimension — answer completeness — improves by three points.

Response length increases by 89%.

Sampled outputs after training:

> **Before:** *"Rate limits are 10 requests per minute for batch endpoints."*
> **After:** *"Regarding rate limiting for batch endpoints, the documentation specifies a limit of 10 requests per minute. It's worth noting that this applies specifically to batch operations, and you may wish to consider this constraint when designing your integration. Rate limits are an important consideration in API design generally..."*

The model learned "longer is better," precisely as predicted, and the cost is measurable: p95 latency up 60%, cost per request up 71%, and — measured by the judge on the held-out set — quality *down* two points, because padding dilutes.

**The fix, in the data:** regenerate rejected responses at matched length so the pairs differ only on completeness.

```
mean chosen length:   118 words
mean rejected length: 112 words
ratio: 1.05
```

Re-run: completeness +4 points, length +6%, latency and cost flat.

**Same algorithm, same hyperparameters, same number of pairs.** The difference was entirely in the data audit.

---

## Practice

**1. Choose one dimension** from your Lesson 3.1 charter and your Lesson 8.4 residual limitations. Good candidates: hedging and confidence calibration, verbosity, refusal calibration (the regression you routed around), or citation discipline. **One dimension. Write it as a sentence** describing what "better" means on this axis.

**2. Build 200–400 pairs**, preferring constructed pairs where the rejected response is generated deliberately with the defect. Use your calibrated judge for the rest, and record its agreement rate as the data's quality ceiling.

**3. Human spot-check 30 pairs.** Read them and confirm you agree with the labels. Where you disagree, ask whether the pair varies on more than one dimension — that is usually the cause.

**4. Run the confound audit.** Report:
   - Length ratio (chosen / rejected) — target 0.9–1.1 unless brevity is the dimension
   - Formatting differences — bullets, headers, code blocks
   - Position bias, if pairs were judged — swap and re-check a sample
   - Any systematic difference other than the target dimension

**5. Fix what you find, in the data.** Regenerate at matched length or rebalance. Re-audit and report before and after.

**6. Document it** in `spine/finetune/preference/datasheet.md`: the dimension as a sentence, the sourcing method, the judge's agreement rate, the audit results before and after, the final count, and known limitations. This sits alongside your Lesson 8.2 datasheet.

---

## Visual Aid

*Two preference pairs shown as side-by-side cards. Top pair labeled "BAD — 3 dimensions vary," with three highlight colours marking length, correctness, and hedging differences, and a red annotation: "the model will learn the easiest signal, not the one you meant." Bottom pair labeled "GOOD — 1 dimension," with a single highlight on the hedging language and green annotations confirming matched length, matched citation, matched correctness. To the right, a small paired histogram of chosen vs rejected word counts: the top one shows two clearly separated distributions labeled "ratio 2.00 — you are training length"; the bottom shows heavily overlapping distributions labeled "ratio 1.05."*

---

## Key Takeaways

- **SFT teaches a destination; preferences teach a direction.** That is why preference tuning fixes verbosity, hedging, and calibration — qualities SFT structurally cannot express.
- **DPO removed the reward model and the RL loop from RLHF**, which is why preference tuning is now ordinary supervised training you can run on Colab. Know RFT/RLVR by shape for tasks with verifiable answers.
- **Judge-labeled preference data inherits the judge's agreement rate as a ceiling** — Lesson 3.4's calibration is load-bearing here.
- **Vary one dimension per pair.** Multi-dimensional pairs train the easiest-to-detect surface feature, not the quality you intended.
- **Audit the length confound before training.** A 2:1 chosen/rejected length ratio trains verbosity into the weights, and the cost shows up as latency, spend, and — measurably — worse quality.

---

## Next

You have clean preference pairs varying on one audited dimension. Now the algorithm that turns "this one is better" into weights — and the two ways it fails while its training metrics look excellent.

**Lesson 9.2** runs DPO and ORPO, explains the leash parameter that keeps the model from drifting, and covers reward hacking: the case where the margin climbs beautifully and the outputs get worse.
