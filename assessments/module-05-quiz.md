# Module 5 Quiz — Retrieval II: Production RAG

**9 questions · ~35 min plus the practical**

This module is examined almost entirely diagnostically, mirroring the AI system design round.

---

## Q1 — Diagnosis (metric vector reading)

For each metric vector, name the failure mode and the fix. Do not read ahead to the answers first — this is the drill.

**(a)**
```
context recall 0.42   context precision 0.88   faithfulness 0.91
```

**(b)**
```
context recall 0.94   context precision 0.38   faithfulness 0.62
```

**(c)**
```
context recall 0.91   context precision 0.86   faithfulness 0.44
```

**(d)**
```
context recall 0.93   context precision 0.84   faithfulness 0.90   answer relevance 0.51
```

---

## Q2 — Short answer

In vector (a) above, faithfulness is 0.91 — the highest of the four. Explain why that number is not good news, and state the general rule it illustrates.

---

## Q3 — Diagnosis

Two failing queries produce output that looks identical to a user: a plausible, confidently wrong answer.

**Query A:** the correct chunk is not in the top 100 candidates.
**Query B:** the correct chunk is at rank 23; the cutoff is 5.

(a) Why is distinguishing these worth doing?
(b) What single diagnostic field separates them?
(c) What is the wrong action a team commonly takes on B?

---

## Q4 — Applied judgment

You measure four query-rewriting techniques:

| Technique | recall@10 | multi-hop | conversational | Added TTFT |
|---|---|---|---|---|
| Baseline | 0.88 | 0.71 | 0.52 | — |
| + conversational rewrite | 0.91 | 0.72 | 0.89 | +180 ms |
| + decomposition | 0.93 | 0.88 | 0.89 | +340 ms |
| + multi-query (3×) | 0.94 | 0.89 | 0.90 | +520 ms |
| + HyDE | 0.92 | 0.86 | 0.88 | +910 ms |

Which do you adopt and which do you reject? Give the reasoning for each decision.

---

## Q5 — Multiple choice

You sweep `rerank_k` and observe:

| rerank_k | Answer quality | Cost |
|---|---|---|
| 3 | 0.81 | $0.0024 |
| 5 | 0.85 | $0.0031 |
| 8 | 0.84 | $0.0044 |
| 12 | 0.79 | $0.0061 |
| 20 | 0.71 | $0.0094 |

What explains the decline past 5?

A. The reranker degrades on larger candidate sets
B. Relevant material is diluted, and models attend poorly to the middle of long contexts
C. The evaluation is noisy at these sample sizes
D. Token limits are being exceeded

---

## Q6 — Design

The same reranked top-5 chunks, assembled four different ways, produce answer quality of 0.74, 0.79, 0.82, and 0.85.

Name the three assembly changes that account for the improvement, in the order they were most likely applied, and give the mechanism for each. Note which of them also *reduced* cost.

---

## Q7 — Short answer

A model returns an answer citing `sdk/async.md`, a file that does not exist and was never retrieved.

(a) Write the check that catches this, in four lines or fewer.
(b) What should happen when it fires?
(c) What harder citation failure does this check **not** catch?

---

## Q8 — Applied judgment

Two refusal configurations:

| | Config A | Config B |
|---|---|---|
| Answer accuracy (in-scope) | 0.87 | 0.84 |
| Missed refusal rate | 0.58 | 0.06 |
| False refusal rate | 0.02 | 0.19 |

(a) Which for a customer-facing documentation assistant? Why?
(b) B's false refusals concentrate in one stratum — ambiguous queries. What is the targeted fix, and why is it better than loosening refusal globally?

---

## Q9 — Practical

`exercises/m5-rag-triage/` contains a RAG pipeline with three seeded defects, one per failure mode.

**Diagnose each from the metrics alone, before reading any code.** Write down your predictions, then investigate, fix, and verify.

**Deliverable:** predicted diagnoses, actual causes, fixes with before/after numbers, and an honest note on any you got wrong.

---
---

# Answer Key

## Q1

**(a) Retrieval miss.** Context recall 0.42 means the needed information is not in the retrieved context. Stop reading the other metrics. **Fix:** Module 4 — chunking, embedding model, hybrid search — plus query rewriting (5.2).

**(b) Ranking failure.** Recall 0.94 means retrieval found it; precision 0.38 means it is buried. Faithfulness suffers downstream because the model is working from the wrong material in the top positions. **Fix:** reranking (5.3).

**(c) Generation drift.** Retrieval and ranking are both good; the model has what it needs and is not using it — usually answering from training memory. **Fix:** grounding structure and citation validation (5.4).

**(d) Answering the wrong question.** Everything is sound except relevance. The answers are well-supported and about something else. **Fix:** check query rewriting (5.2) — over-aggressive multi-query or paraphrase drift is the usual cause.

## Q2

**Why it is not good news:** faithfulness measures whether claims are supported by the *provided context*. If the context does not contain the answer, the model is being faithful to a context that lacks it — producing an incomplete answer or a refusal, faithfully. A high number here describes correct behavior in a broken pipeline.

**The general rule:** **read the metric vector top-down and stop at the first low value.** Downstream metrics are uninterpretable when an upstream stage failed. Reporting "faithfulness 0.91" from pipeline (a) as a success is one of the more common ways to misread these metrics.

## Q3

**(a)** Because the fixes share nothing. A retrieval miss requires re-chunking, a different embedding model, hybrid search, or query rewriting — hours of work including a re-index. A ranking failure is fixed by adding a reranker. Choosing wrong costs a week.

**(b)** **The rank of the expected chunk in the full candidate list** (retrieved wide, e.g. k=100). Present at rank 23 means ranking; absent at k=100 means retrieval.

**(c)** Teams commonly re-chunk the corpus and swap embedding models — because from the *output* both failures look identical, and re-chunking is the intuitive response to "it didn't find the right thing." The diagnostic tool exists precisely because intuition cannot distinguish these.

## Q4

**Adopt: conversational rewrite.** 37 points on the conversational stratum for 180 ms. The largest single gain in the table and the cheapest. Non-negotiable for any multi-turn system.

**Adopt: decomposition.** 16 points on multi-hop — the stratum it was designed for — for 340 ms. Clear win on its target.

**Reject: multi-query.** One point of aggregate recall for another 180 ms of TTFT, and it roughly doubles the added cost. One point is inside the noise band of a 60-item set. Not worth it.

**Reject: HyDE.** It made things *worse* than decomposition alone (0.92 vs 0.93) and costs the most by a wide margin. On this corpus, documentation is already written in a Q&A-adjacent style, so hypothetical answers add noise rather than signal.

**Full credit** requires rejecting at least two and citing the noise band for multi-query. Adopting everything that shows a positive number is the failure mode this question tests.

**Note:** HyDE might win on a corpus of dense prose. Rejecting it here is a measurement result, not a general verdict.

## Q5 — **B**

Two mechanisms compound: relevant material is diluted among more chunks, giving the model more opportunities to latch onto something tangential, and models attend least reliably to the middle of long contexts, so chunks in the middle positions contribute little while still consuming budget.

**A** is wrong — the reranker's job ends before assembly. **C** is wrong; the decline is monotonic across five points and 14 points in magnitude, well outside noise. **D** is wrong; 20 chunks is well within the window.

**The practical consequence:** "retrieve more" is not a quality strategy. Sweep and find your peak, which is usually between 3 and 8.

## Q6

**In likely order:**

**1. Deduplication (+0.05).** Retrieval returns near-duplicates constantly — overlapping chunks, the same paragraph in a changelog and a reference page. Five variations of one paragraph give the model one fact and consume five slots. **This also reduced cost**, cutting context tokens by roughly 28%.

**2. Ordering for attention (+0.03).** Most relevant chunk first, second-most last, weaker material in the middle — because models attend least reliably to the middle of a long context. Six lines of code, no cost change.

**3. Small-to-big expansion (+0.03).** Expanding each surviving chunk to its parent section or neighbours before generation. Sharp retrieval, rich context. This one *increases* context tokens, so it is the only step that costs more.

**The headline:** eleven points of quality with the same five chunks and no retrieval change — and two of the three steps improved cost as well.

## Q7

**(a)**
```python
retrieved_ids = {c.id for c in retrieved}
invalid = [c for c in answer.citations if c not in retrieved_ids]
if invalid:
    raise CitationError(f"cited but not retrieved: {invalid}")
```

**(b)** Route into the Module 2 repair loop with the specific error. The model corrects on the first repair in the large majority of cases. Record the pre-repair fabrication rate — it is a useful number.

**(c)** It does not catch a **real citation attached to a claim the document does not support** — the chunk was retrieved, the ID is valid, and the document says something different. That requires support verification (a second model call), which is what the faithfulness metric measures at evaluation time. Running it on every production request is a cost decision; sampling is often the right compromise.

## Q8

**(a) Config B.** For a customer-facing assistant, a confident wrong answer about pricing, security, or policy creates real problems — potentially commitments the company did not make. An unnecessary refusal creates mild friction and a follow-up question. A missed refusal rate of 0.58 means the system answers more than half of out-of-scope questions from general knowledge, which is the failure mode most likely to get a system withdrawn.

The charter's priority ordering — "a wrong answer is worse than a refusal" — decides this, which is why that document is written before any metric.

**(b) Targeted fix:** for ambiguous queries, answer the most likely interpretation **and state which case the answer covers**, rather than refusing. This converts a refusal into a useful hedged answer.

**Why better than global loosening:** loosening refusal globally would raise the missed-refusal rate on genuinely out-of-scope questions — the failure that actually matters. The targeted fix moved false refusal from 0.19 to 0.09 with missed refusal unchanged, producing a system better than either A or B.

**The general principle:** cluster your false refusals before adjusting anything. They usually concentrate, and concentrated failures have targeted fixes.

## Q9 — Practical rubric

| Criterion | Full credit |
|---|---|
| Predictions written first | Diagnoses recorded before reading code — the whole point |
| Correct localization | Each defect mapped to the right stage |
| Fix matches diagnosis | Reranking for ranking failure, not re-chunking |
| Verification | Before/after metric vectors, per stratum |
| Honesty | Wrong predictions reported, with what misled them |

**The honesty section is graded.** A submission with three correct predictions and no reflection is weaker than one with two correct, one wrong, and a clear analysis of why the metric vector was misread — because the second demonstrates the diagnostic skill actually improving.
