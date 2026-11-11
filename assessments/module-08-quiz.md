# Module 8 quiz — Fine-Tuning I: When, and SFT/LoRA

Ten scenario questions on supervised fine-tuning, weighted toward the decision because that is the part interviews probe: when to prompt, retrieve or fine-tune, what a fine-tune costs to own, how to build a clean dataset from production traces, how to read LoRA training curves, and how to prove or disprove improvement against the baseline. They test reasoning, not recall. Most describe a proposal, a training run or an evaluation result and ask what it shows or what to do next.

**Covers**

- [08.1 · The Fine-Tuning Decision: When Not To](../lessons/module-08/lesson-01.md) — prompting vs retrieval vs fine-tuning, anti-patterns, TCO, the six-question framework.
- [08.2 · Building a Dataset from Production Traces](../lessons/module-08/lesson-02.md) — sourcing, curation, the chat-template bug, the leakage check, sizing.
- [08.3 · LoRA and QLoRA Supervised Fine-Tuning on Colab](../lessons/module-08/lesson-03.md) — LoRA and QLoRA, the five hyperparameters, reading curves, model cards.
- [08.4 · Did It Work? Measuring Against the Baseline](../lessons/module-08/lesson-04.md) — off-task regression, the comparison table, ship / router / discard, negative results.

**Pass mark:** 70% (7 of 10). Unlimited retries; answers and explanations appear after you submit.

The hands-on part of this module is the [Module 8 practical](../projects/module-08-practical.md): a format-adherence fine-tune with a before/after evaluation that includes off-task strata.
