# Lesson 5.3 — Reranking and Context Assembly

**Module 5: Retrieval II — Production RAG** · ~13 min
**Objective:** M5-LO2 (part 2) — Improve precision with a cross-encoder reranker, and assemble context deliberately.

---

## Introduction

Two jobs get conflated in most RAG implementations, and separating them is worth doing explicitly.

**Reranking** decides *which* chunks the model sees. **Context assembly** decides *how* it sees them — in what order, with what deduplication, at what total length.

The first is well known. The second is barely discussed and is free quality that most systems leave on the table: the same five chunks, assembled two different ways, can differ by ten points in answer quality without a single retrieval change.

---

## Key Concept 1: Bi-encoders and cross-encoders

Your retriever from Module 4 is a **bi-encoder**: it embeds the query and each document *independently*, then compares vectors. Independence is what makes it fast — every document was embedded once at index time, and query time is a nearest-neighbour lookup over millions of precomputed vectors.

Independence is also what limits it. The document's vector was computed without knowing the query. It is a general-purpose summary, not an assessment of relevance to *this* question.

A **cross-encoder** takes query and document **together** as one input and outputs a relevance score. It can attend across both — noticing that the document mentions *this specific version*, or that it discusses the topic but explicitly excludes the case being asked about.

It is far more accurate and far too slow to run over a corpus: it requires one forward pass **per query-document pair**. A million documents means a million forward passes per query.

---

## Key Concept 2: The funnel

The resolution is obvious once stated: **retrieve wide and cheap, rerank narrow and accurate.**

```
1,000,000 chunks
    ↓  bi-encoder ANN search (~10 ms)
   50 candidates
    ↓  cross-encoder rerank (~80 ms for 50 pairs)
    5 chunks
    ↓  assemble
  context → model
```

The tunables are `retrieve_k` and `rerank_k`:

- **`retrieve_k` too low** and the reranker cannot fix what retrieval missed. Its quality ceiling is whatever recall@retrieve_k is.
- **`retrieve_k` too high** and latency grows linearly with no gain past the point where recall saturates.
- **`rerank_k`** is your context budget decision, and Key Concept 4 explains why bigger is not better.

Tune `retrieve_k` by finding where **recall@k saturates** on your golden set — the point where retrieving more stops finding more. That is usually somewhere between 30 and 100, and it is a measurement, not a convention.

**Options:** open cross-encoder rerankers from Hugging Face run locally on CPU at acceptable latency for tens of candidates. Hosted reranking APIs are stronger and add a network round trip. Late-interaction models sit between bi- and cross-encoders on the cost/quality curve. Measure on your corpus; the ranking of these varies by domain.

---

## Key Concept 3: Assembly — deduplicate first

Retrieval returns near-duplicates constantly. Overlapping chunks from Lesson 4.2, the same paragraph in a changelog and a reference page, three versions of one section.

Five chunks that are variations of the same paragraph give the model one fact and consume five slots. Deduplicate on embedding similarity above a threshold (0.95-ish, measured on your corpus), keeping the highest-ranked of each cluster.

This is nearly free and routinely recovers two or three usable slots.

---

## Key Concept 4: Order for attention, and use less than you think

**Lost in the middle.** Models attend most reliably to the beginning and end of a long context, and least reliably to the middle. This is well-documented and it means position is a quality decision.

The practical response: put the **most relevant chunk first** and the **second most relevant last**, filling the middle with the weaker ones.

```python
def order_for_attention(chunks):          # chunks pre-sorted by relevance
    if len(chunks) <= 2:
        return chunks
    return [chunks[0]] + chunks[2:] + [chunks[1]]
```

Six lines. Measurable gain. Almost nobody does it.

**More context is often worse.** The instinct is that if 5 chunks help, 20 must help more. Measured, quality frequently *drops*: the relevant material is diluted, the model has more opportunities to latch onto something tangential, and the middle positions receive less attention. It also costs proportionally more (Lesson 1.3) and raises TTFT.

Find your own optimum by sweeping `rerank_k` against answer quality. For most documentation QA it lands between 3 and 8, which surprises people expecting 20.

**Small-to-big expansion.** The technique promised in Lesson 4.2. You embedded and retrieved small precise chunks; now expand each survivor to its parent section or its neighbours using the `prev_id` / `next_id` metadata before assembling. Sharp retrieval, rich context — the tension from Lesson 4.2 resolved rather than compromised.

**Budget explicitly.** Compute the token cost of your assembled context, subtract the system prompt and reserved output space, and trim deliberately from the *least* relevant chunk rather than letting the model silently truncate the last one.

---

## Worked Example 1: The reranker

Same 60 golden-set queries, hybrid retrieval plus Lesson 5.2 rewriting as the baseline.

| Configuration | recall@5 | context precision | multi-hop | p95 latency |
|---|---|---|---|---|
| Retrieve 5 directly | 0.71 | 0.44 | 0.62 | 340 ms |
| Retrieve 20 → rerank 5 | 0.84 | 0.71 | 0.79 | 410 ms |
| **Retrieve 50 → rerank 5** | **0.89** | **0.83** | **0.86** | 490 ms |
| Retrieve 100 → rerank 5 | 0.89 | 0.83 | 0.86 | 680 ms |

Recall@k saturates at 50 — retrieving 100 costs 190 ms and finds nothing new. **Retrieve 50, rerank 5.**

The pagination query from Lesson 5.1, sitting at rank 23:

| | Rank |
|---|---|
| Bi-encoder | 23 |
| After cross-encoder rerank | **2** |

Fixed, with no re-chunking and no new embedding model — exactly as the failure taxonomy predicted.

## Worked Example 2: The assembly ablation

Same reranked top-5, four assembly strategies, answer quality by the calibrated judge from Lesson 3.4:

| Assembly | Answer quality | Context tokens |
|---|---|---|
| As-ranked, no dedup, no expansion | 0.74 | 2,900 |
| + deduplication | 0.79 | 2,100 |
| + order for attention | 0.82 | 2,100 |
| + small-to-big expansion | **0.85** | 3,400 |

Eleven points, and **not one retrieval change** — the same five chunks in all four rows. Deduplication also cut tokens by 28%, so two of these steps improved quality *and* cost.

And the `rerank_k` sweep, with everything else held at the winning configuration:

| rerank_k | Answer quality | Context tokens | Cost/request |
|---|---|---|---|
| 3 | 0.81 | 2,200 | $0.0024 |
| **5** | **0.85** | 3,400 | $0.0031 |
| 8 | 0.84 | 5,100 | $0.0044 |
| 12 | 0.79 | 7,400 | $0.0061 |
| 20 | 0.71 | 11,800 | $0.0094 |

Quality peaks at 5 and **falls off a cliff by 20** — worse than 3 chunks, at four times the cost. This is the "more context is worse" effect measured on a real system, and it is the row that changes how people build.

---

## Practice

**1. Add the reranker.** A cross-encoder from Hugging Face is the local-first option; a hosted reranking API is fine if latency permits. Wrap it behind an interface so it can be swapped.

**2. Find where recall saturates.** Sweep `retrieve_k` at 10, 20, 50, 100 and plot recall@retrieve_k. Pick the saturation point, not a round number.

**3. Sweep `rerank_k`** against answer quality, context tokens, and cost. **You are looking for the peak and the decline** — if quality rises monotonically across your whole sweep, extend the sweep until it does not.

**4. Run the assembly ablation.** Four configurations as above, one change at a time, using your Module 3 judge. Report quality and token count for each.

**5. Implement small-to-big** using the `prev_id` / `next_id` metadata you added in Lesson 4.2. Compare expanding to neighbours versus expanding to the parent section.

**6. Budget the context explicitly.** Compute assembled token count, reserve output space, and trim from the least-relevant end. Log when trimming occurs — silent truncation is a Lesson 5.1 failure mode and you should be able to see it happening.

**7. Append both deltas to `eval-report.md`** — the reranker and the assembly ablation separately. The assembly entry is the interesting one: measurable quality gain, no retrieval change, lower token cost.

---

## Visual Aid

*A funnel diagram, wide to narrow, in four stages. Top: "1,000,000 chunks" with a wide mouth. Then "bi-encoder ANN — 10 ms, $0" narrowing to "50 candidates." Then "cross-encoder rerank — 80 ms, $0" narrowing to "5 chunks." Then "assemble — dedup, order, expand — 0 ms, $0" to "context, 3,400 tokens." Finally "generate — 2,400 ms, $0.0031." Each stage is annotated with its latency and cost, making visible that all the quality work happens in the cheap stages and all the cost is in the last one. To the right, a small line chart of answer quality against rerank_k, peaking at 5 and declining sharply, labeled "more context is not more quality."*

---

## Key Takeaways

- **Bi-encoders are fast because they embed independently; cross-encoders are accurate because they do not.** The funnel — retrieve 50 cheaply, rerank to 5 accurately — uses each where it belongs.
- **Tune `retrieve_k` to where recall saturates**, measured on your golden set. Beyond that point you are buying latency.
- **Deduplicate, then order most-relevant first and second-most last.** Six lines of code for a measurable gain, because models attend poorly to the middle of long contexts.
- **More context is frequently worse.** Sweep `rerank_k` and expect a peak somewhere between 3 and 8, with a real decline past it — at four times the cost.
- **Small-to-big resolves the chunking tension** — embed small, generate from big — using metadata you added in Module 4.

---

## Next

You have the right chunks, ranked well, assembled deliberately. The model can still say something the context does not support, cite a document that was never retrieved, and answer confidently about things your documentation has never covered.

**Lesson 5.4** builds the trust layer: grounding that is enforced rather than requested, citations validated mechanically against the retrieved set, and the hardest behavior of all — a system that says "the documentation doesn't cover this."
