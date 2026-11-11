# Module 3 practical — Exposing a summarizer's hidden weakness

This practical puts the whole module to work on a system that looks fine at first glance. You get a mediocre summarizer with one specific weakness, and your job is to build an evaluation that makes the weakness visible, prove it with numbers, fix it, and prove the fix. The point is less the fix than the write-up: explaining why the first metric you reached for did not show the problem.

## Task

Take the supplied mediocre summarizer in `exercises/m3-summarizer-eval/`. It has a specific weakness.

Build an evaluation that exposes it, prove the weakness with numbers, fix it, and prove the fix.

## Deliverable

Dataset, metrics, before/after report, and a paragraph on why the first metric you tried failed to reveal the problem.

## How it is assessed

| Criterion | Full credit |
|---|---|
| Weakness identified | Correctly names the summarizer's specific weakness |
| Stratification | Dataset includes a stratum in which the weakness becomes visible; it is invisible without it |
| Metric choice | First metric attempted plausibly failed (e.g. ROUGE, or an aggregate) and the write-up explains why |
| Proof | Before/after numbers, per stratum |
| Reflection | Explains the mechanism by which the first metric hid the problem |

**The reflection paragraph is the graded part.**

<details><summary>Reviewer notes (spoilers)</summary>

The planted weakness: the summarizer drops the final third of long documents. Full credit on stratification requires a long-document stratum.

Target reflection answer: "ROUGE looked fine because the summary overlapped heavily with the first two-thirds of the document, and the aggregate hid it because only 6 of 60 items were long."

</details>
