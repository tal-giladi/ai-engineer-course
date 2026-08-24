# Module 4 Quiz — Retrieval I: Embeddings, Chunking, Vector Stores

**8 questions · ~30 min plus the practical**

---

## Q1 — Diagnosis

A team benchmarks three embedding models on their own corpus of API documentation (mean chunk length 780 tokens):

| Model | Max seq length | recall@5 |
|---|---|---|
| Leaderboard leader | 512 | 0.71 |
| Mid-ranked asymmetric | 8192 | 0.86 |
| Small fast model | 512 | 0.74 |

(a) What is likely wrong, and what one check confirms it?
(b) Predict what happens to each model's score after the fix.
(c) What does this say about using leaderboards to choose?

---

## Q2 — Short answer

An asymmetric embedding model is being used without its required `query:` and `passage:` prefixes. Describe what the team will observe, and why this is a particularly dangerous class of bug.

---

## Q3 — Applied judgment

Rank these chunking strategies for a corpus of markdown API documentation, and give the mechanism for the largest single gain:

- Fixed-size 1000 chars with 200 overlap
- Recursive character splitting
- Structure-aware splitting on headings
- Structure-aware plus contextual headers
- Semantic splitting on embedding similarity

---

## Q4 — Design

A chunk reads, in full:

> "This is not supported in v3."

(a) Why will this chunk retrieve poorly?
(b) Name two different techniques that fix it, and state the cost of each.

---

## Q5 — Diagnosis

A filtered query — restricted to `section == "v3 API"`, which is 5% of the corpus — returns 1.4 results on average when `k=10` is requested, and recall drops from 0.86 to 0.41.

(a) What is happening?
(b) Why is there no error?
(c) How do you verify the cause in pgvector specifically?
(d) Beyond quality, why does this matter in a multi-tenant system?

---

## Q6 — Multiple choice

Your corpus is 40k documents, your team already operates Postgres, and chunks need to join to document metadata for permission filtering. Which is the better default?

A. Qdrant — purpose-built stores are faster
B. pgvector — one system, transactional consistency, SQL joins to the metadata you need
C. Both, with a routing layer
D. Neither; 40k documents does not need a vector index

---

## Q7 — Diagnosis

A query for `"Why does the batch endpoint return 413?"` retrieves the correct chunk at rank 19 under dense retrieval. The team has tried three chunking strategies and two embedding models; the rank barely moves.

(a) Why will no further chunking or embedding work fix this?
(b) What is the actual fix, and what is the mechanism?
(c) What would you expect the correct chunk's rank to be after the fix?

---

## Q8 — Practical

Index a mid-sized open-source codebase and measure retrieval recall across three chunking strategies: fixed-size, function-boundary-aware, and function-boundary with a contextual header carrying file path and class name.

**Deliverable:** `exercises/m4-code-retrieval/` with the recall table and a paragraph on why code chunking differs from prose chunking.

---
---

# Answer Key

## Q1

**(a)** Two of the three models have a 512-token maximum sequence length while chunks average 780 tokens. Those models are **silently truncating roughly a third of every chunk** — no error, no warning, plausible-looking results.

**The check:** compute the chunk token-length distribution against each model's declared maximum. Sixty seconds.

**(b)** After re-chunking below 480 tokens:
- Leaderboard leader: rises substantially (roughly 0.71 → 0.83)
- Mid-ranked asymmetric: essentially unchanged — it was never truncating
- Small fast model: rises substantially (roughly 0.74 → 0.85)

**And the ranking changes materially** once the confound is removed.

**(c)** The leaderboard ranking did not predict performance on this corpus, and a configuration detail moved recall more than the model choice did. Leaderboards are a shortlist; the decision comes from benchmarking on your own golden set — which Module 3 exists to make a fifteen-minute job.

## Q2

**What they observe:** noticeably worse retrieval, with no error and entirely plausible results. Typical magnitude is 13–16 points of recall@5.

**Why it is dangerous:**
- It produces no signal at all — every query returns results, they are just worse
- It is invisible without a controlled comparison, because there is no "correct" baseline to notice a deviation from
- It leads teams to blame the embedding model, the chunking, or the corpus, and to spend days on the wrong thing

**The control:** read the model card before benchmarking, and measure with correct prefixes, without, and with the same prefix on both sides. Three runs, one of which will surprise you.

## Q3

**Ranking (typical, on markdown docs):**

1. Structure-aware + contextual headers — ~0.87
2. Semantic splitting — ~0.79
3. Structure-aware — ~0.78
4. Recursive character splitting — ~0.68
5. Fixed-size with overlap — ~0.61

**Largest single gain:** fixed-size → structure-aware, roughly +17 points.

**Mechanism:** documents already encode their semantic structure in headings, and fixed-size splitting throws that away — cutting tables in half, separating headings from the content they introduce, and mixing unrelated topics in one chunk. Respecting the document's own boundaries means each chunk covers one thing, so its single embedding vector is near that thing rather than being an average of several.

**Worth noting:** semantic splitting ties structure-aware at much higher index cost on this corpus, and would win on unstructured prose. That is why "best practice" answers are unhelpful and measurement is not.

## Q4

**(a)** The chunk has no subject. Embedded alone, its vector sits near generic "not supported" content and nowhere near batch endpoints, streaming, or whatever the section was actually about. A query naming the feature will not match it.

**(b) Two fixes:**

**Contextual headers** — prepend the document title and heading path before embedding: `[API Reference > Batch Endpoints > Limitations]`. **Cost:** ~25 tokens per chunk, one line of code. Typically the single best return-per-effort improvement in the retrieval stack.

**Small-to-big** — keep the small chunk for embedding and retrieval, but expand to its parent section or neighbours before generation, using `prev_id`/`next_id` metadata. **Cost:** the metadata must exist at index time, and generation context grows.

**Also creditable:** a model-generated situating summary per chunk (contextual retrieval) — effective, and costs a generation call per chunk, so measure whether it beats the free header version.

## Q5

**(a) Post-filtering.** The system retrieves the top 10 by similarity across the whole corpus, then discards those failing the filter. With a filter matching 5% of documents, roughly half a result survives on average.

**(b)** No error, because nothing failed. The vector search succeeded and the filter succeeded. The system returned fewer results than requested and reported success.

**(c)** Run `EXPLAIN ANALYZE` on the filtered query. The planner may choose to scan the vector index first and apply the WHERE clause afterward, which is post-filtering by another name. Assumption is not verification here.

**(d)** In a multi-tenant system the filter is a **permissions boundary**. A post-filter means the search ran across other tenants' documents before discarding them. Quality collapse is the visible symptom; the retrieval having touched data outside the tenant's scope is the more serious issue, and Lesson 11.3 treats it as a security concern.

## Q6 — **B**

40k documents is comfortably within pgvector's range. The permission-filtering requirement means chunks must join to document metadata — a SQL join in one query the planner optimizes, versus denormalizing that metadata into a payload and keeping it synchronized.

The decisive factor is not speed; it is **one fewer stateful system and no dual-write problem**. A document deleted from Postgres whose vectors still answer queries is a genuine bug class that pgvector eliminates by construction.

**A full answer also states the switching conditions:** above roughly 5M vectors, or if filtered p95 exceeds the latency budget, migrate — made cheap by the `VectorStore` interface.

**D** is wrong: 40k documents times several chunks each is well past the point where exact search is viable at interactive latency.

## Q7

**(a)** This is a **lexical gap**, not a chunking or embedding problem. `413` is a rare token; embedding models give rare tokens weak, unstable representations, and `413` sits near `412` and `414` in embedding space because to the model they are all small numbers in similar contexts. No amount of re-chunking changes that, and no general-purpose embedding model fixes it.

**(b) Hybrid retrieval — add BM25 alongside dense and fuse with RRF.** BM25 scores by exact term overlap weighted by rarity, so `413` is a strong signal rather than a weak one. RRF combines ranks rather than incomparable scores, requiring no tuning.

**(c)** BM25 alone would likely put it at **rank 1**; RRF typically lands it at **rank 1–3**, because fusion trades a little of each specialist's home-ground performance for much better mixed-query performance.

**The general finding:** hybrid loses slightly to each specialist on its own ground and wins substantially overall, because real traffic contains both query types and you cannot route in advance.

## Q8 — Practical rubric

| Criterion | Full credit |
|---|---|
| Controlled comparison | Embedding model and vector store held constant across the three strategies |
| Per-stratum reporting | Recall reported by query type, not only aggregate |
| Header isolation | The contextual-header effect measured on its own, as a single-variable change |
| Sequence limit respected | Chunk sizes checked against the embedding model's maximum, headers included |
| Reflection | Explains what differs about code |

**Target reflection:** code chunking differs because function and class boundaries are strong semantic boundaries that fixed-size splitting destroys; identifiers are exact tokens that favor BM25; and a function detached from its file path and class name is far less interpretable than a paragraph detached from its heading — so contextual headers matter more for code, not less.
