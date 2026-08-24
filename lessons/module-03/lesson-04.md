# Lesson 3.4 — LLM-as-Judge and Judge Calibration

**Module 3: Evaluation Foundations** · ~13 min
**Objective:** M3-LO3 — Implement LLM-as-judge, calibrate the judge against human labels, and quantify how far to trust it.

---

## Introduction

Using a model to score another model's output is standard practice — it appears in most senior job descriptions, it is what RAGAS-style RAG metrics are built on, and it is the only affordable way to score open-ended text at volume.

It is also, done carelessly, a random number generator with good manners. A judge produces confident, well-formatted, plausible-sounding scores whether or not it is measuring anything. There is no error state. A broken judge and a working judge look identical.

The thing that separates the two is **calibration**: proving your judge agrees with a human, and stating how far that agreement extends. That step is the difference between an evaluation practice and evaluation theater, and it is what an interviewer is checking for when they ask how you evaluate.

---

## Key Concept 1: Judges do comparison better than scoring

Ask a model "rate this answer 1–10" and you will get a number. Ask it again on an identical input and you may get a different one. Ask it across a hundred answers and the scores cluster tightly around 7 and 8, because absolute quality on an unanchored scale is not something a model has a stable sense of.

Ask instead "which of these two answers is better, A or B?" and reliability jumps substantially. Relative judgment is a much easier task.

So, in order of preference:

**1. Pairwise comparison.** "Here are two answers. Which better satisfies the rubric?" Best reliability. Ideal for comparing a candidate change against your baseline — which is exactly the question you ask in every module from here.

**2. Anchored rubric scoring.** If you need an absolute number, use a short scale where every point is *defined by an example*:

```
3 — Fully supported by context, directly answers the question, cites correctly.
2 — Answers the question but has an unsupported claim OR a missing citation.
1 — Partially answers, or hedges when the context is clear.
0 — Unsupported, contradicts the context, or answers an out-of-scope question.
```

Four anchored points beat a 1–10 vibe scale every time.

**3. Binary checks.** "Is every claim in this answer supported by the context? Yes/no." Highest reliability of all. Several binary checks often beat one composite score.

---

## Key Concept 2: One dimension per call, reasoning first

Two design rules that cost nothing and matter a lot.

**One dimension per judge call.** Do not ask a single call to score faithfulness, relevance, tone, and completeness. The scores collapse toward each other — a judge that thinks an answer is good rates it well on everything. Separate calls give you separable signals, which is what lets a metric vector localize a problem.

**Reasoning before the verdict.** Structurally enforce it, using your Lesson 2.2 schema ordering:

```python
class Verdict(BaseModel):
    reasoning: str = Field(description="Cite the specific text you are judging.")
    winner: Literal["A", "B", "tie"]
```

Because generation is sequential, a `reasoning` field before `winner` genuinely conditions the verdict. Placed after, it is post-hoc rationalization that costs tokens and buys nothing.

**Use a different model family as judge than as generator.** Models exhibit self-preference — a measurable tendency to prefer their own outputs. Judging a model with itself introduces a bias in exactly the direction that makes your system look good.

---

## Key Concept 3: The three biases, and their fixes

These are well-documented and each has a cheap mitigation. Skipping the mitigations is how judges end up measuring the wrong thing.

**Position bias.** In pairwise comparison, judges systematically favor one position — often the first. *Fix:* run every comparison twice with the order swapped. If the verdict flips, it is a tie. This doubles judge cost and is worth it; the flip rate is also a free diagnostic of judge reliability.

**Verbosity bias.** Judges prefer longer answers, largely independent of quality. This is the most dangerous one, because if you use judge scores to build preference data in Module 9, you will train a model to be verbose. *Fix:* control for length — compare the score/length correlation, and if it is high, either add an explicit rubric instruction that length is not quality, or compare only length-matched pairs.

**Self-preference.** Covered above. *Fix:* cross-family judging.

There is a fourth worth naming: **rubric drift**. If you edit the rubric, all previous scores are on a different scale. Version your judge prompt exactly like any other prompt (Lesson 2.4), and when the rubric changes, re-score the baseline or your comparison is meaningless.

---

## Key Concept 4: Calibration — the step everyone skips

Here is the procedure. It takes about an hour and it is what makes every number downstream mean something.

**1. Sample 30–50 items** spanning your strata, including hard cases.

**2. Label them yourself,** applying your rubric. This is the ground truth. It is tedious. Do it anyway — you will also discover rubric ambiguities you did not know were there, which is a second benefit.

**3. Run the judge** on the same items, blind to your labels.

**4. Compute agreement.** For binary or pairwise: percentage agreement, and Cohen's kappa if you want to account for chance. For scored: correlation, plus mean absolute difference.

**5. Look at the disagreements individually.** This is the most informative half hour in the module. Disagreements fall into three buckets:
- *The judge was wrong* → fix the rubric.
- *You were wrong* → good, the judge caught something.
- *The rubric permitted both* → fix the rubric (this is usually the biggest bucket).

**6. State the trust boundary explicitly**, in writing:

> "This judge agrees with human labels on 86% of items (κ = 0.71, n = 40). Mean absolute difference on the 0–3 scale is 0.31. **We do not act on score differences smaller than 0.4.**"

That last sentence is the whole point. It converts your judge from a source of numbers into a measuring instrument with a stated resolution, and it feeds directly into the CI threshold policy in Lesson 3.5.

**Recalibrate** when you change the rubric, change the judge model, or when a stratum's failure profile shifts.

---

## Worked Example: A judge that measured length

A team builds a faithfulness judge on a 1–5 scale. It runs, produces scores, everyone is satisfied. Then someone calibrates it.

**Initial calibration, n = 40:**

```
Agreement with human labels: 61%
Correlation with human score:      0.42
Correlation with answer word count: 0.79   ←
```

The judge correlates more strongly with **length** than with human judgment. It is a verbosity detector wearing a faithfulness label. Every optimization the team had made on the basis of this judge had been selecting for longer answers.

Diagnosis, from reading the disagreements: the rubric said "rate the faithfulness of this answer from 1 to 5," with no anchors. With nothing to anchor on, the judge fell back on a proxy — thoroughness, which reads as length.

**The revision:**

```diff
- Rate the faithfulness of this answer from 1 to 5.
+ For each factual claim in the answer, determine whether it is
+ supported by the provided context. Then assign:
+   3 — every claim supported
+   2 — one unsupported claim, not central to the answer
+   1 — one unsupported claim central to the answer
+   0 — multiple unsupported claims, or contradicts the context
+ Length is not a factor. A one-sentence fully-supported answer
+ scores 3.
```

**After recalibration, same 40 items:**

```
Agreement with human labels: 86%
Correlation with human score:      0.84
Correlation with answer word count: 0.11   ←
Position-swap flip rate:            4%
```

Trust boundary recorded: *do not act on differences below 0.4 points.*

Note what the fix was: anchors, a decomposed procedure ("for each claim"), and one explicit instruction about length. No model change, no framework, no additional cost.

---

## Practice

**1. Build a pairwise judge** for your primary quality dimension. Schema with `reasoning` before `winner`. Use a different model family than your generator. Prompt lives in `spine/prompts/judge_pairwise_v1.md` with a version header.

**2. Handle position bias.** Run every comparison in both orders; flip means tie. Record the flip rate — it is your reliability diagnostic.

**3. Label 30–50 items by hand** from your dev set, spanning strata. Save as `spine/eval/human_labels.yaml`.

**4. Calibrate.** Run the judge blind, compute agreement, and **read every disagreement**, categorizing each into the three buckets. Revise the rubric based on what you find, then recalibrate on the same items.

**5. Run a bias probe.** Compute the correlation between judge score and answer length on your dev set. If it is above roughly 0.4, your rubric needs a length instruction. Report what you found either way — a clean probe is a result.

**6. Write `spine/eval/judge.md`:** the rubric, the model used, agreement rate, kappa, flip rate, length correlation, and the trust boundary as an explicit sentence. Lesson 3.5 reads the threshold straight out of this file.

---

## Visual Aid

*Two scatter plots side by side, x-axis "human score," y-axis "judge score," both 0–3. Left, "before calibration": a diffuse cloud with weak upward trend, and each point sized by answer word count — the large points cluster visibly at the top regardless of human score, making the verbosity bias visible at a glance. Right, "after calibration": points hug the diagonal, and point size is now scattered evenly. On the right plot, a shaded horizontal band 0.4 wide straddles the diagonal, labeled "noise band — do not act on differences inside this."*

---

## Key Takeaways

- **An uncalibrated judge is decoration.** It produces confident numbers with no error state; broken and working judges look identical until you check.
- **Prefer pairwise over scoring, anchored scales over vibe scales, and binary checks where possible.** One dimension per call, reasoning before verdict, and a different model family than the generator.
- **Three biases with cheap fixes:** position (swap and re-run), verbosity (control for length — this one poisons Module 9 preference data if unfixed), self-preference (cross-family judging).
- **Calibrate against 30–50 hand-labeled items,** read every disagreement, and expect the biggest bucket to be rubric ambiguity rather than judge error.
- **State the trust boundary as a sentence** — "we do not act on differences below X." That sentence is what makes the judge an instrument instead of an opinion.

---

## Next

You can measure quality on open-ended output, and you know exactly how far to trust the measurement. What you cannot yet do is stop a regression from being merged, which means all of this still depends on someone remembering to run it.

**Lesson 3.5** wires the harness into CI with two tiers, sets thresholds from the noise band you just measured, and makes the failure report name the stratum that broke — so that from Module 4 onward, nothing ships without a number.
