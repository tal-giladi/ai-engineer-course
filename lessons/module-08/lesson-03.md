---
id: "08.3"
module: 8
minutes: 14
practice_minutes: 90
prerequisites: ["08.2"]
objectives:
  - "Run LoRA/QLoRA supervised fine-tuning on Colab GPU and push the adapter to Hugging Face."
volatility: implementation
sources: []
last_verified: "2026-08-24"
---

# 08.3 · LoRA and QLoRA Supervised Fine-Tuning on Colab

This is the lesson people expect to be hard. It is not. With a clean dataset from Lesson 8.2, a QLoRA run on a 7–8B model takes about twenty-five minutes on a Colab GPU and roughly sixty lines of code.

The engineering is in knowing which knobs matter, what each one does, and — most importantly — **when to stop**. You need no transformer internals for any of that, and this lesson deliberately teaches none.

---

## Key Concept 1: Why full fine-tuning is out of reach, and why that is fine

**Full fine-tuning** updates every weight. Memory required is roughly the model size for weights, again for gradients, and twice more for optimizer state — around 16× the parameter count in bytes for a standard setup. A 7B model needs well over 100 GB. Not happening on Colab, and rarely necessary.

**LoRA** (Low-Rank Adaptation) freezes the base model entirely and injects small trainable matrices into attention layers. You train typically well under 1% of the parameters. Memory drops enormously, and the output is an **adapter** measured in tens of megabytes rather than gigabytes.

The intuition, without the mathematics: the update a task requires is far simpler than the full weight matrix. Representing it as the product of two thin matrices captures most of it at a fraction of the parameters.

**QLoRA** adds one thing: quantize the *frozen* base to 4-bit. The base is not being trained, so lower precision costs little, and a 7–8B model becomes trainable on a modest GPU. This is what you will use.

**Why adapters are operationally excellent** (and this pays off in Lesson 9.4): they are small enough to version in a registry, one base model can serve many adapters, and swapping is cheap.

---

## Key Concept 2: The hyperparameters that matter

Five. Ignore the rest at first.

**`r` (rank)** — adapter capacity. Higher fits more complex behavior and overfits sooner. Start at **16**. Use 8 for narrow format tasks; go to 32 or 64 only when 16 demonstrably underfits, which is rarer than people assume.

**`lora_alpha`** — a scaling factor. Convention is **2×r**. Set it and stop thinking about it.

**`target_modules`** — which layers get adapters. Attention projections (`q_proj`, `k_proj`, `v_proj`, `o_proj`) are the standard default. Targeting all linear layers, including the MLP projections, helps on harder tasks at higher memory cost. Start with attention.

**Learning rate** — LoRA tolerates far higher rates than full fine-tuning. **1e-4 to 2e-4** is the usual range. Too low and nothing moves; too high and the loss becomes unstable.

**Epochs** — **2 to 3**. This is where most mistakes happen. More epochs means memorization, not learning. Watch validation loss and stop when it turns.

**`lora_dropout`** at 0.05 is a reasonable regularizer for small datasets.

---

## Key Concept 3: The workflow

```python
from transformers import AutoModelForCausalLM, AutoTokenizer, BitsAndBytesConfig
from peft import LoraConfig
from trl import SFTTrainer, SFTConfig

bnb = BitsAndBytesConfig(
    load_in_4bit=True,
    bnb_4bit_quant_type="nf4",
    bnb_4bit_compute_dtype=torch.bfloat16,
    bnb_4bit_use_double_quant=True,
)

model = AutoModelForCausalLM.from_pretrained(BASE, quantization_config=bnb,
                                             device_map="auto")
tokenizer = AutoTokenizer.from_pretrained(BASE)

peft_config = LoraConfig(
    r=16, lora_alpha=32, lora_dropout=0.05,
    target_modules=["q_proj", "k_proj", "v_proj", "o_proj"],
    task_type="CAUSAL_LM",
)

trainer = SFTTrainer(
    model=model,
    train_dataset=train_ds,
    eval_dataset=val_ds,
    peft_config=peft_config,
    args=SFTConfig(
        num_train_epochs=3,
        per_device_train_batch_size=4,
        gradient_accumulation_steps=4,      # effective batch 16
        learning_rate=2e-4,
        lr_scheduler_type="cosine",
        warmup_ratio=0.03,
        bf16=True,
        eval_strategy="steps",
        eval_steps=25,                       # frequent enough to see the turn
        logging_steps=5,
        output_dir="./out",
    ),
)
trainer.train()
```

**Choosing a base model.** Prefer an instruction-tuned model over a base completion model — it already follows instructions, so you are adjusting rather than teaching from scratch. Match the size to the task: a 7–8B model handles most narrow tasks and fits comfortably in 4-bit on a T4-class GPU.

**Evaluate frequently.** `eval_steps=25` rather than once per epoch. You cannot see the overfitting turn at one point per epoch.

---

## Key Concept 4: Reading the curves

Two lines: training loss and validation loss.

**Healthy.** Both descend, validation flattening slightly above training. Stop at the flattening point.

**Overfitting.** Training keeps falling; validation bottoms out and rises. Everything after that turn is memorization. **The fix is fewer epochs or more data — not a bigger rank.** Increasing `r` when overfitting makes it worse, and this is a common wrong instinct.

**Underfitting.** Both descend and stop high, output barely changed. Raise the learning rate, add epochs, or raise rank — in that order.

**Unstable.** Loss spikes or diverges. Learning rate too high, or a data problem — one enormous example, or a formatting error.

**Suspiciously perfect.** Training loss near zero within one epoch on a small dataset usually means duplicates survived deduplication (Lesson 8.2) or the task is trivial.

**And the warning worth repeating:** a healthy curve does not mean a working model. The chat-template bug from Lesson 8.2 produces a textbook curve and unusable output. **Always sample outputs at checkpoints**, not just curves. This is the single most important habit in this lesson.

---

## Key Concept 5: Publish with a model card

```python
model.push_to_hub("username/docs-extract-lora")
tokenizer.push_to_hub("username/docs-extract-lora")
```

The adapter is tens of megabytes. The **model card** is what makes it usable by anyone including future you:

```markdown
# docs-extract-lora
Base: <base-model-id> (commit abc123f)
Method: QLoRA, r=16, alpha=32, attention projections only
Data: 800 train / 232 val — see datasheet.md
Hyperparameters: lr 2e-4, 3 epochs, effective batch 16
Evaluation: see eval-report.md (Module 3 held-out set)

## Intended use
Extracting structured fields from support tickets in <domain>.

## Out of scope
General conversation, other document types, other languages.
Not evaluated for factual accuracy — this model formats, it does not know.

## Limitations
Trained on 90 days of traffic; degrades on ticket formats introduced
after that window.
```

The "out of scope" and "limitations" sections are what make this read as professional. Their absence reads as unawareness — the same principle as stated limitations in Lesson 3.2's datasheet.

---

## Worked Example: Two runs

**Run A — 800 examples, r=16, lr 2e-4, 3 epochs, T4 GPU.**

| Step | Train loss | Val loss |
|---|---|---|
| 25 | 1.84 | 1.79 |
| 50 | 1.21 | 1.18 |
| 100 | 0.79 | 0.81 |
| 150 | 0.61 | 0.72 |
| 200 | 0.52 | 0.69 |
| 250 | 0.47 | **0.68** ← flattens |
| 300 | 0.44 | 0.68 |

Healthy. Validation flattens around step 250 and does not rise. Three epochs was right. Wall clock 24 minutes; adapter 84 MB.

Sampled outputs at step 250: correct format, correct field semantics.

**Run B — identical, 8 epochs.**

| Step | Train loss | Val loss |
|---|---|---|
| 250 | 0.47 | 0.68 |
| 400 | 0.29 | 0.71 |
| 550 | 0.16 | 0.79 |
| 700 | 0.08 | 0.88 |
| 800 | **0.04** | **0.94** ← rising |

Training loss looks fantastic. Validation has been rising since step 250.

Sampled outputs at step 800: it reproduces training examples nearly verbatim, including a specific customer name from the training data appearing in an unrelated extraction. Classic memorization.

**Run A held-out accuracy: 0.94. Run B: 0.81** — worse than the Module 3 baseline.

The wrong response would have been "increase rank to fit better." The correct response is fewer epochs.

---

## Practice

**1. Set up the notebook.** Colab with GPU, `transformers`, `peft`, `trl`, `bitsandbytes`, `accelerate`. Load your Lesson 8.2 dataset from Hugging Face or Drive.

**2. Choose a base model** — instruction-tuned, 7–8B, that fits in 4-bit on your GPU. Note the exact model ID and commit for provenance.

**3. Verify the formatting one more time.** Print three fully-formatted training examples including special tokens. Sixty seconds; it has already saved someone a GPU hour.

**4. Run the baseline configuration:** r=16, alpha=32, lr 2e-4, 3 epochs, attention projections, eval every 25 steps.

**5. Sample outputs at checkpoints.** Generate on five held-out validation inputs at three points during training. **Read them.** This is where the template bug and memorization become visible and where the curves lie.

**6. Read your curves and write the interpretation** — one paragraph in your notebook: healthy, overfitting, or underfitting, and what you would change.

**7. Run one variation.** Either 8 epochs (to see overfitting yourself — recommended, it is the most instructive twenty-five minutes in the module) or a different rank. Compare curves and sampled outputs.

**8. Push the winner** with a full model card including intended use, out-of-scope use, and limitations.

**9. Record the cost.** GPU minutes and dollar cost for all runs including failures. This feeds the TCO model you built in Lesson 8.1 — check your estimate against reality.

---

## Visual Aid

*Two loss-curve charts side by side, same axes. Left, "Run A — 3 epochs": two lines descending, training slightly below validation, validation flattening around step 250 with a green marker labeled "stop here." Right, "Run B — 8 epochs": training loss diving toward zero while validation bottoms at step 250 and climbs steadily; the divergence region is shaded red and labeled "memorization." Beneath each chart, a boxed sample output: Run A shows a correct extraction; Run B shows an extraction containing a customer name from the training set, circled. Caption in bold beneath both: "The curves for a chat-template bug look exactly like Run A. Sample the outputs."*

---

## Key Takeaways

- **LoRA freezes the base and trains small adapter matrices; QLoRA quantizes the frozen base to 4-bit.** A 7–8B model becomes trainable on a modest Colab GPU, producing an adapter of tens of megabytes.
- **Five hyperparameters matter:** rank (start 16), alpha (2×rank), target modules (attention first), learning rate (1e-4–2e-4), epochs (2–3). Ignore the rest initially.
- **Validation loss tells you when to stop.** When it turns upward, everything after is memorization — and the fix is fewer epochs or more data, never a bigger rank.
- **Sample outputs at checkpoints.** A healthy curve is compatible with a chat-template bug and with memorization; the outputs are not.
- **The model card's out-of-scope and limitations sections** are what make the artifact professional, and they are read in interviews.

---

## Next

You have an adapter and a descending loss curve. Neither of those facts tells you the model is better at your task.

Loss measures fit to the training distribution. It says nothing about held-out task performance, and it says nothing at all about whether you broke something else in the process.

**Lesson 8.4** runs your Module 3 harness against the fine-tuned model on the untouched held-out set, checks for the capability regression that narrow training so often causes, and produces the honest comparison — including the possibility that the answer is "do not ship this."
