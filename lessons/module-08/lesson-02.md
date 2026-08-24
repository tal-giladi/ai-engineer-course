# Lesson 8.2 — Building a Dataset from Production Traces

**Module 8: Fine-Tuning I** · ~13 min
**Objective:** M8-LO2 — Build a supervised dataset from production traces, including deduplication, filtering, and held-out splits.

---

## Introduction

Data quality dominates every other variable in fine-tuning. Not by a little — a clean 800-example set will beat a noisy 50,000-example set on the same base model with the same hyperparameters, reliably.

Which means this lesson is where the real work is, and Lesson 8.3 — the actual training — is the easy part. Budget accordingly: most of a fine-tuning project is data curation.

You have an advantage most people building their first dataset do not. Since Lesson 1.4 you have been writing structured `CallRecord`s with inputs, outputs, prompt versions, and outcomes. Since Lesson 2.3 you have been writing dead-letter records of every failure. That is a dataset waiting to be curated, which is precisely why the instrumentation went in first.

---

## Key Concept 1: Sourcing

**Production traces** are the best source, because they carry your real input distribution — the phrasings, the edge cases, the malformed inputs you would never have invented.

Filter to successful outcomes. From your telemetry: calls that validated on the first attempt, produced a usable result, and were not flagged downstream.

**Human-corrected outputs are the highest-value source per example.** A correction encodes exactly the gap between what the model did and what it should have done. If your system has any review step, those corrections are gold — weight them heavily.

**Dead letters** from Lesson 2.3 are your hard cases. A model that persistently failed a format is exactly the situation you are training away, and these examples are pre-vetted as difficult.

**Synthetic generation** — using a stronger model to generate examples — is legitimate, extremely common, and used in most modern fine-tuning. Three conditions:

- **Label it.** A `source: synthetic` field. You will want to ablate later.
- **Verify it.** Model-generated training data has a characteristic failure: it is suspiciously easy for models, because it was written by one.
- **Check the license.** Provider terms differ on whether outputs may be used to train other models. This is a real professional constraint and checking it is part of the job (Lesson 9.3 returns to this).

---

## Key Concept 2: Curation is most of the work

**Deduplication, including near-duplicates.** Exact duplicates are easy. Near-duplicates are the problem: production traffic is repetitive, and "reset my password" arriving 400 times in slightly different wording will teach the model that this pattern is 40% of the world. Deduplicate on embedding similarity above ~0.95 (your Module 4 pipeline does this) and keep one representative per cluster.

**Filter out the defect you are training away.** This one is easy to get backwards. If you are fine-tuning to fix a semantic field error, examples *containing* that error must not be in the training set as targets. Filter by outcome, not just by completion.

**Balance.** If 70% of your traces are one category, the model will over-predict it. Cap per category, or at least know the distribution and check whether it matches your intent.

**Length distribution.** Training on short examples and serving long ones — or the reverse — degrades quality. Match the distribution you expect at inference.

**Quality filtering.** Run your Module 3 judge over candidate examples and drop the bottom decile. You built a calibrated instrument; use it.

---

## Key Concept 3: The chat template, and the failure it causes

This is the single most common cause of a fine-tuning run that appears successful and produces unusable output. It deserves its own concept.

Instruction-tuned models expect a specific format — special tokens delimiting system, user, and assistant turns. Every model family differs. If you train with the wrong template:

- **The loss curve looks completely healthy.** It descends smoothly. Nothing warns you.
- The model learns to produce text in a structure the inference-time template does not match.
- Outputs at inference are truncated, run on, ignore the system prompt, or emit template tokens as literal text.

**Always use the tokenizer's own template:**

```python
text = tokenizer.apply_chat_template(
    [{"role": "system", "content": sys},
     {"role": "user", "content": inp},
     {"role": "assistant", "content": out}],
    tokenize=False,
)
```

Never hand-write the format. And **verify before training**: print three formatted examples and read them. Sixty seconds against a wasted GPU hour and a confusing debugging session.

Related and equally important: **mask the prompt tokens in the loss.** You want the model to learn to produce the *response*, not to reproduce the input. Most `trl` training paths handle this via a completion-only collator — confirm yours does.

---

## Key Concept 4: Splitting, and the leakage check

Split into train and validation, with validation used only for early stopping.

Then the rule that matters most, which was set up back in Lesson 3.2: **your Module 3 held-out golden set must never appear in training data.**

This is not theoretical. Your training data comes from production traces. Your golden set was partly sourced from real user queries. There is a genuine, likely overlap.

If it leaks, your Lesson 8.4 evaluation compares a model that memorized the test set against a baseline that did not. It will look **excellent** — which is why this failure is dangerous rather than merely wrong.

```python
holdout_hashes = load_holdout_hashes()      # from Lesson 3.2
before = len(examples)
examples = [e for e in examples
            if normalized_hash(e.input) not in holdout_hashes]
print(f"leakage check: removed {before - len(examples)} examples")
```

Report the number in your datasheet, even when it is zero. Also check near-duplicates against the held-out set by embedding similarity — an exact-hash check misses a paraphrase of a test question, and a paraphrase is nearly as damaging.

---

## Key Concept 5: Sizing

For LoRA on a well-chosen base model:

- **200–500 examples** — viable for a narrow format or style task
- **500–2,000** — the sweet spot for most tasks in this course
- **2,000–10,000** — broader tasks, or where you need robustness across variation
- **Beyond that** — usually diminishing returns for LoRA; you are into different territory

**800 excellent examples beat 50,000 scraped ones.** Consistently. If you have 50,000 candidates, your job is to select the best 1,000, not to use all of them.

---

## Worked Example: 4,000 traces → 800 examples

Extraction task, funnel with counts at every stage:

| Stage | Remaining | Removed | Reason |
|---|---|---|---|
| Raw traces (90 days) | 4,102 | — | |
| Successful outcomes only | 3,318 | 784 | failed validation or flagged downstream |
| **Held-out leakage check** | **3,301** | **17** | **matched golden-set hashes** |
| Near-duplicate removal (>0.95) | 1,842 | 1,459 | repetitive production traffic |
| Contains the target defect | 1,690 | 152 | would teach the error |
| Judge quality filter (bottom decile) | 1,521 | 169 | Module 3 judge |
| Category balance cap (max 300/category) | 1,088 | 433 | one category was 41% |
| Length distribution match | 942 | 146 | outliers |
| **+ human corrections (weighted 2×)** | **1,032** | — | highest-value source |
| **Final** | **1,032** | | 800 train / 232 validation |

Read two rows carefully.

**Seventeen leaked examples.** Small, and enough to inflate the Lesson 8.4 comparison misleadingly. They were removed because the check ran; without it, nobody would have known.

**1,459 near-duplicates — 36% of the corpus.** Real production traffic is far more repetitive than it feels. Without this step, one common phrasing would have dominated training.

**Then the template bug, demonstrated.** The first training run used a hand-written format string:

```
### Instruction:
{input}
### Response:
{output}
```

Loss curve: smooth descent, 1.84 → 0.41 over three epochs. Textbook.

Inference output:

```
### Response: The extracted fields are... ### Instruction: ###
Response: ### Instruction:
```

The model learned to generate the delimiter structure it was trained on, which the inference-time template does not produce or terminate. Twenty-five minutes of GPU and an hour of confusion.

Re-run with `apply_chat_template`: identical hyperparameters, near-identical loss curve, and outputs that work.

**Nothing in the loss curve distinguishes these two runs.** That is the lesson.

---

## Practice

**1. Extract candidates** from your telemetry and dead-letter store. Filter to successful outcomes and pull the failures separately as hard cases.

**2. Run the leakage check first.** Exact hashes against your Lesson 3.2 held-out IDs, plus an embedding near-duplicate check at >0.92. **Report the count in your datasheet even if it is zero.**

**3. Deduplicate near-duplicates** at >0.95 using your Module 4 embedding pipeline. Expect to remove a lot and record the percentage.

**4. Filter out the defect** you are training away. Verify by sampling 20 examples and confirming none exhibits it.

**5. Balance and match length distribution.** Cap dominant categories; drop length outliers.

**6. Format with `apply_chat_template`.** Then **print three formatted examples and read them character by character**, including special tokens. Confirm prompt-token masking is active.

**7. Split** 80/20 train/validation and save as JSONL.

**8. Write the datasheet** — `spine/finetune/datasheet.md`: sources with counts, the full funnel table with per-stage removals and reasons, the leakage check result, the final size and split, category and length distributions, synthetic proportion if any, license note, and known limitations. This is a portfolio artifact for Module 12 and it is what a hiring manager reads to judge whether you understand data.

---

## Visual Aid

*A vertical funnel, wide at top narrowing to bottom, with each stage labeled and the remaining count printed inside. To the right of each stage, a horizontal bar showing the number removed and the reason. The "held-out leakage check" stage is outlined in red with a callout: "17 examples — enough to make the Module 8.4 comparison meaningless, and invisible without the check." The "near-duplicate removal" stage is outlined in amber: "36% of the corpus." At the bottom, two boxes: "800 train / 232 validation." Off to the side, a small paired panel: two identical-looking loss curves side by side, one labeled "hand-written template — outputs unusable," the other "apply_chat_template — works," with the caption "these curves cannot tell you which is which."*

---

## Key Takeaways

- **Data quality dominates.** 800 curated examples beat 50,000 scraped ones on the same base model. Most of a fine-tuning project is curation.
- **Your telemetry is the dataset.** Production traces carry the real input distribution; human corrections are the highest-value source per example; dead letters are pre-vetted hard cases.
- **Near-duplicates are a third or more of real traffic.** Without deduplication one phrasing dominates training.
- **Verify the chat template before spending a GPU hour.** A wrong template produces a perfect-looking loss curve and unusable outputs, and nothing in training warns you.
- **Run the held-out leakage check and report the number.** A leak makes your Lesson 8.4 comparison look excellent and mean nothing.
- **The datasheet is a portfolio artifact.** Per-stage counts and reasons show you understand data; a bare example count does not.

---

## Next

Clean, deduplicated, correctly-formatted data with a verified leakage check. Now the part everyone assumes is the hard part, which takes about twenty-five minutes.

**Lesson 8.3** covers LoRA and QLoRA — enough theory to make informed hyperparameter choices, no transformer internals — and how to read a loss curve well enough to know when to stop.
