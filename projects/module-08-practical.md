# Module 8 practical — A format-adherence fine-tune

This practical runs the whole module end to end on one narrow task. You fine-tune a small model to reliably produce a strict output format that the base model breaks about 15% of the time, then prove the improvement against your baseline and check whether general capability regressed along the way. The goal is not only a working adapter but an honest evaluation of it, whichever way the result falls.

## Task

Fine-tune a small model to reliably produce a strict output format the base model violates ~15% of the time. Quantify the improvement **and** check whether general capability regressed.

## Deliverable

`exercises/m8-format-adherence/` with datasheet, notebook, adapter link, and before/after evaluation including off-task strata.

## How it is assessed

| Criterion | Full credit |
|---|---|
| Leakage check | Run and reported, exact-hash and near-duplicate, count stated even if zero |
| Chat template | Uses `apply_chat_template`; formatted examples inspected before training |
| Curve interpretation | Written interpretation — healthy, overfitting, or underfitting — with the stopping point identified |
| Output sampling | Outputs generated and read at checkpoints, not curves only |
| **Off-task evaluation** | Full stratified set plus a general capability probe, with out-of-scope specifically reported |
| Comparison | Quality, cost, and latency against the prompted baseline |
| Recommendation | Ship / router / discard, with reasoning tied to the charter |

**The off-task row is the one that separates submissions.** A submission reporting only the target metric has demonstrated it can run a training job, not that it can evaluate one.
