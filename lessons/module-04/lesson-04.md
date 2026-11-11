---
id: "04.4"
module: 4
minutes: 13
practice_minutes: 75
prerequisites: ["04.3"]
objectives:
  - "Implement hybrid retrieval combining dense vectors and BM25 with rank fusion, and measure the gain over either alone."
volatility: implementation
sources: []
last_verified: "2026-08-24"
---

# 04.4 · Hybrid Retrieval: Dense + BM25 + Fusion

Your dense retriever understands meaning. Ask it about "authentication failures" and it will find documents about login errors, credential rejection, and token expiry, even though none of them use your words. That is genuinely impressive and it is why embeddings took over.

Now ask it about error code `413`. Or the function `batch_upsert_v2`. Or version `3.1.4`. These are the queries where dense retrieval quietly falls apart, and they are extremely common in exactly the technical domains where AI engineering jobs cluster.

The fix is not a better embedding model. It is a technique that predates embeddings by decades, running alongside them.

---

## Key Concept 1: Two failure modes, pointing opposite ways

**Dense retrieval** embeds text into a semantic space. Its strength is paraphrase — the query and document need share no vocabulary. Its weakness is that the embedding is lossy by design: it compresses meaning and discards surface form. Rare tokens the model saw infrequently in training get weak, unstable representations. `413` sits near `412` and `414` in embedding space, because to the model they are all just small numbers in similar contexts.

**Sparse retrieval (BM25)** scores documents by exact term overlap, weighted by how rare each term is in the corpus and normalized for document length. A document containing `413` scores highly for a query containing `413`, and rarity means `batch_upsert_v2` is a far stronger signal than `the`.

Its weakness is the **lexical gap**: a query for "login problems" will not match a document that only says "authentication failures." Zero shared terms, zero score.

They fail in opposite directions. That is the whole argument for running both.

**Queries where BM25 wins decisively:**
- Error codes, status codes, exit codes
- Function, class, and parameter names
- Version numbers and product SKUs
- Proper nouns and acronyms
- Exact phrases from a document
- Anything the embedding model never saw during training — including your own domain's jargon

That list describes a large fraction of real queries against technical documentation, internal wikis, and codebases.

---

## Key Concept 2: Fusion — combining ranks, not scores

You have two ranked lists. Combining them is less obvious than it looks, because **the scores are not comparable**. Cosine similarity lives in [-1, 1] and clusters tightly around 0.7–0.9 in practice; BM25 is unbounded and depends on corpus statistics. Averaging them directly is meaningless.

**Reciprocal Rank Fusion (RRF)** avoids the problem by ignoring scores entirely and using only rank position:

```python
def rrf(rankings: list[list[str]], k: int = 60) -> list[str]:
    scores = defaultdict(float)
    for ranking in rankings:
        for rank, doc_id in enumerate(ranking, start=1):
            scores[doc_id] += 1.0 / (k + rank)
    return sorted(scores, key=scores.get, reverse=True)
```

Why it works well in practice:

- **Scale-free.** Rank 1 is rank 1 in both lists; no normalization needed, no drift when score distributions shift.
- **No tuning.** The constant `k=60` is a widely-used default that performs robustly; `k` damps the influence of top ranks so a single system cannot dominate.
- **Extensible.** Add a third retriever and the formula is unchanged — which matters in Lesson 5.2 when multi-query retrieval produces several lists.
- **Rewards agreement.** A document ranked moderately by both beats one ranked first by one and absent from the other, which is usually the right instinct.

**Weighted score fusion** — normalize both score sets and take a weighted sum — can beat RRF when carefully tuned on a specific corpus. It is also fragile: the normalization depends on score distributions that drift as your corpus grows, so a weight tuned in March degrades quietly by September. Start with RRF. Move to weighted fusion only if you have measured a gain and are willing to re-tune.

---

## Key Concept 3: Implementation, and its cost

**With pgvector:** Postgres has full-text search built in. `tsvector` columns, GIN indexes, and `ts_rank_cd` give you BM25-like scoring in the same database, in the same transaction, filtered by the same WHERE clause. This is a genuine advantage of the pgvector path and worth noting in your Lesson 4.3 decision record.

**With Qdrant:** sparse vectors are natively supported alongside dense, with server-side fusion. Cleaner, and one of the reasons the purpose-built stores exist.

**Standalone:** a BM25 library over an in-memory index works fine at course scale and poorly at production scale, where you want the index next to the data.

**The cost, stated honestly:** hybrid roughly doubles retrieval work — two index lookups instead of one, plus fusion. In the benchmark below that is single-digit milliseconds, invisible next to a multi-second generation call. But it is not free, and the correct posture is that it must earn its place on your numbers, per stratum. If your corpus is prose with no identifiers and your users write natural questions, hybrid may buy you almost nothing.

---

## Worked Example: Where the win actually lives

Same corpus, same chunking, same embedding model, same store. 60 golden-set queries split into three groups.

**Paraphrase queries** (the query shares no vocabulary with the document):

| | recall@10 |
|---|---|
| Dense only | **0.91** |
| BM25 only | 0.38 |
| RRF hybrid | 0.88 |

**Exact-token queries** (error codes, function names, versions):

| | recall@10 |
|---|---|
| Dense only | 0.44 |
| BM25 only | **0.94** |
| RRF hybrid | 0.91 |

**Mixed / natural queries:**

| | recall@10 |
|---|---|
| Dense only | 0.79 |
| BM25 only | 0.61 |
| RRF hybrid | **0.85** |

**Overall, all 60:**

| | recall@10 | p95 latency | Index size |
|---|---|---|---|
| Dense only | 0.72 | 11 ms | 13 MB |
| BM25 only | 0.64 | 7 ms | 9 MB |
| **RRF hybrid** | **0.88** | 19 ms | 22 MB |

Read this carefully, because it contains the real lesson:

**Hybrid loses to each specialist on its home ground** — 0.88 vs 0.91 on paraphrase, 0.91 vs 0.94 on exact-token. Fusion costs you a couple of points where one retriever was already dominant.

**Hybrid wins overall by sixteen points**, because real query traffic contains both kinds and you cannot route in advance.

**The gain is concentrated.** Aggregate recall moved from 0.72 to 0.88, and almost all of that came from the exact-token group moving 0.44 → 0.91. Reporting only the aggregate would have told you "hybrid is better" without telling you *why* or *for whom* — which is exactly the per-stratum discipline from Lesson 3.3.

**The concrete case:** the `413` query from Lesson 4.2, which sat at rank 19 under dense retrieval and could not be fixed by any chunking change:

| | Rank of correct chunk |
|---|---|
| Dense | 19 |
| BM25 | 1 |
| RRF | 2 |

Fixed. Not by a better model, by a different kind of index.

---

## Practice

**1. Build the sparse index.** Postgres full-text if you chose pgvector, Qdrant sparse vectors if you chose Qdrant. Index the same chunks, with the same metadata and the same filters available.

**2. Implement RRF.** Keep it generic over N ranked lists — Lesson 5.2 will pass it four.

**3. Split your golden set by query type.** Tag each item as paraphrase, exact-token, or mixed. If you have almost no exact-token queries, **add some** — real users search for error codes and function names, and a golden set without them will tell you hybrid is worthless when it is not.

**4. Evaluate three configurations** — dense only, sparse only, hybrid — reporting recall@5, recall@10, and MRR per query type *and* per stratum, plus p95 latency and index size.

**5. Decide, with the cost visible.** Adopt hybrid only if your numbers justify it. If your corpus genuinely has no exact-token queries and hybrid buys two points for double the latency, **declining it is the correct answer** and a good decision record. Write down which way you went and why.

**6. Append to `eval-report.md`.** This closes Module 4. Your entry should show the cumulative journey from the Module 3 baseline: baseline → chunking strategy → contextual headers → hybrid, with the delta at each step and the cost alongside.

---

## Visual Aid

*Three vertical result lists side by side for the single query "Why does the batch endpoint return 413?". Left, "Dense (top 5)": five plausible-looking chunks about streaming, rate limits, and error handling — none correct — with a note beneath: "correct chunk at rank 19." Middle, "BM25 (top 5)": the correct chunk highlighted green at rank 1, followed by four less relevant exact-match hits. Right, "RRF fused (top 5)": correct chunk green at rank 2, with the surrounding results a visibly better mix than either list alone. Caption: "Dense understands the question. BM25 found the answer. Fusion keeps both."*

---

## Key Takeaways

- **Dense and sparse fail in opposite directions.** Dense handles paraphrase and blurs exact tokens; BM25 nails exact tokens and cannot cross the lexical gap.
- **Exact-token queries are common in technical domains** — error codes, function names, versions, jargon — and dense retrieval is genuinely weak on all of them.
- **Fuse ranks, not scores.** RRF needs no tuning, no normalization, survives distribution drift, and extends to N retrievers unchanged.
- **Hybrid loses slightly to each specialist on its home ground and wins substantially overall**, because real traffic is mixed and you cannot route in advance.
- **Report per query type.** The aggregate hides where the gain came from, and hybrid costs roughly double the retrieval work — it must earn its place on your numbers.

---

## Next

Your retrieval is measurably good: the right chunks, indexed well, findable by meaning and by exact term. And your system still gets answers wrong.

Sometimes the right document is retrieved but ranked below the cutoff. Sometimes it makes the cutoff and gets buried mid-context where the model does not attend to it. Sometimes everything needed is right there and the model answers from memory anyway, with a citation it invented.

Those are four different failures with four different fixes, and telling them apart from the metrics is the single most useful diagnostic skill in RAG.

**Module 5** builds that skill, then the production layer on top of it: query rewriting, reranking, deliberate context assembly, grounded answers with verifiable citations, and a system that knows how to say "the documentation doesn't cover this."
