# Module 5 practical — Triage a RAG pipeline from its metrics

This practical runs the diagnostic drill from the module on a real pipeline. You get a RAG system with three seeded defects, one per failure mode, and you have to localize each one from the metric vectors before you look at any code. Then you investigate, fix, and verify. What matters most is not the fixes but how honestly you compare your predictions with what you actually found.

## Task

`exercises/m5-rag-triage/` contains a RAG pipeline with three seeded defects, one per failure mode.

**Diagnose each from the metrics alone, before reading any code.** Write down your predictions, then investigate, fix, and verify.

## Deliverable

Predicted diagnoses, actual causes, fixes with before/after numbers, and an honest note on any you got wrong.

## How it is assessed

| Criterion | Full credit |
|---|---|
| Predictions written first | Diagnoses recorded before reading code — the whole point |
| Correct localization | Each defect mapped to the right stage |
| Fix matches diagnosis | Reranking for ranking failure, not re-chunking |
| Verification | Before/after metric vectors, per stratum |
| Honesty | Wrong predictions reported, with what misled them |

**The honesty section is graded.** A submission with three correct predictions and no reflection is weaker than one with two correct, one wrong, and a clear analysis of why the metric vector was misread — because the second demonstrates the diagnostic skill actually improving.
