# Lesson 4.3 — Vector Stores in Docker: pgvector vs Qdrant

**Module 4: Retrieval I** · ~13 min
**Objective:** M4-LO3 — Stand up pgvector and Qdrant in Docker, index the same corpus in both, and articulate when each is the right call.

---

## Introduction

"Which vector database?" is a question you will be asked in an interview, and the answer that impresses is not a product name. It is the reasoning: what the workload looks like, what the filtering requirements are, what the team already operates, and what the migration costs if you are wrong.

This lesson gives you that reasoning by running both options side by side on your own corpus. It also covers one filtering mistake that will cut your recall in half without producing a single error — the kind of thing that separates people who have run these systems from people who have read about them.

---

## Key Concept 1: What a vector store actually does

Finding the nearest vectors to a query vector by exact comparison is O(n) and fine for ten thousand chunks. At a million, it is not.

Vector stores use **approximate nearest neighbour (ANN)** search: index structures that find *almost* the true nearest neighbours, much faster. The dominant one is **HNSW** — a layered graph you traverse from a coarse top layer down to a fine bottom one. **IVF** is the other common family, partitioning the space into clusters and searching only the nearest few.

The parameters most teams never touch, and should know:

- **`ef_construction`** (build time) — higher builds a better graph, more slowly
- **`ef_search`** / `hnsw.ef_search` (query time) — higher searches more thoroughly, more slowly. **This is your recall/latency dial**, tunable per query without re-indexing.
- **`m`** — connections per node; higher means better recall and more memory

The important realization: **ANN recall is a tunable you control, not a fixed property.** If your retrieval evaluation looks weak, check `ef_search` before you blame your embedding model. Measure ANN recall against exact search on a sample — if it is below ~0.95, turn the dial up.

---

## Key Concept 2: pgvector — extend what you run

`pgvector` adds vector types and indexes to Postgres.

**What you get:**
- **One system.** No new stateful service to deploy, back up, monitor, secure, or explain to whoever is on call.
- **Transactional consistency.** Chunks and their business data update atomically. No dual-write problem, no reconciliation job, no eventually-consistent window where a document is deleted but its vectors are not.
- **SQL joins.** Filter vectors by anything in your schema — tenant, permissions, date, document status — in one query the planner optimizes.
- **Familiar operations.** Your existing backup, replication, and monitoring apply unchanged.

**What you give up:**
- A real performance ceiling. Somewhere in the tens of millions of vectors, depending on hardware and filter complexity, it becomes the wrong tool.
- Fewer specialized features — quantization options, sophisticated payload indexing, sparse-vector support.
- Vector search competes for the same resources as your transactional workload.

---

## Key Concept 3: Qdrant — purpose-built

Qdrant stands in here for the category (Pinecone, Weaviate, Milvus differ in deployment model more than in fundamentals).

**What you get:**
- **Filtering as a first-class feature.** Payload indexes make filtered vector search fast, which is the operation that most often falls over elsewhere.
- **Quantization** — scalar and binary — cutting memory dramatically with a measurable, tunable quality cost.
- **Horizontal scale** designed in.
- **Sparse vectors alongside dense**, which makes the hybrid search in Lesson 4.4 native rather than assembled.

**What you give up:**
- A second stateful system: deployment, backup, monitoring, security, on-call.
- **The dual-write problem.** Your source of truth is Postgres; your vectors are elsewhere. Deletes, updates, and re-indexing all need reconciliation, and this is where real bugs live — a document deleted from Postgres whose vectors still answer queries is a genuine data-leak class of bug.
- No joins to relational data. Filtering data must be denormalized into the payload and kept in sync.

---

## Key Concept 4: Pre-filter vs post-filter — the trap

Most real queries are filtered: only this tenant, only the v3 docs, only documents this user may read.

There are two ways to combine a filter with vector search, and one of them is quietly broken.

**Post-filtering:** retrieve the top k by vector similarity, then discard those failing the filter.

Take k=10 with a filter matching 5% of the corpus. The top 10 by similarity are drawn from everything; on average, half a result survives. You asked for 10 and got 0 or 1, **with no error**. Recall collapses and the system looks like it has a retrieval quality problem.

**Pre-filtering:** restrict the candidate set to matching documents, then search within it. You get 10 relevant results because you searched 10 relevant results' worth of space.

Pre-filtering is harder to implement — the ANN index is built over everything, so the engine must intersect the filter with graph traversal — which is exactly why it is a differentiating feature. Qdrant's payload indexes do this. pgvector with an indexed WHERE clause can do this, but **query plan matters**: the planner may choose to scan the vector index first and filter after, which is post-filtering by another name. Check with `EXPLAIN ANALYZE`.

**Diagnostic:** if filtered queries return fewer results than requested, or filtered recall is far below unfiltered recall, you are post-filtering. This will come up again in Lesson 11.3, where a tenant filter that silently post-filters is both a quality bug and a security-adjacent one.

---

## Key Concept 5: Hide the choice behind an interface

Same seam argument as Lesson 1.2:

```python
class VectorStore(Protocol):
    def upsert(self, chunks: list[Chunk]) -> None: ...
    def search(self, vector: list[float], k: int,
               filters: dict | None = None) -> list[SearchResult]: ...
    def delete(self, ids: list[str]) -> None: ...
```

`PgVectorStore` and `QdrantStore` implement it. Your retrieval code depends on the protocol. Switching is configuration, benchmarking both is instantiating both, and the "we can migrate if we outgrow it" claim in your decision record becomes true rather than aspirational.

---

## Worked Example: Both, measured

4,200 chunks, 768 dimensions, both from one `docker-compose.yml`, HNSW in both, `ef_search` tuned so ANN recall vs exact ≥ 0.98 in both.

| | pgvector | Qdrant |
|---|---|---|
| Index build | 34 s | 21 s |
| p95 query latency (k=10, unfiltered) | 11 ms | 6 ms |
| p95 query latency (k=10, filtered to 5%) | 38 ms | 8 ms |
| Recall vs exact search | 0.98 | 0.99 |
| Memory | 340 MB (shared with PG) | 410 MB (dedicated) |
| New operational surface | none | one stateful service |

At this scale both are comfortably fast enough. The filtered-query column is where they separate, and that gap widens with corpus size.

**Now the post-filter demonstration.** Query filtered to `section_path[0] == "v3 API"` (5% of the corpus):

| Approach | Results returned (k=10) | recall@10 |
|---|---|---|
| Pre-filter | 10 | 0.86 |
| Post-filter | 1.4 (mean) | **0.41** |

Forty-five points, silently. The post-filtered system returns fewer results than requested and no error anywhere says so. A team without this diagnostic spends a week on embedding models.

**The decision, for this project:** pgvector. The corpus is small, Postgres is already in the stack, chunks join to document metadata that lives there, and one fewer stateful system is worth more than 30 ms at p95. The record also states the switching conditions: **above ~5 million vectors, or if filtered p95 exceeds 100 ms, migrate to Qdrant** — a migration made cheap by the `VectorStore` interface.

That is the answer that lands in an interview. Not "Qdrant is faster."

---

## Practice

**1. Compose both.** Add pgvector and Qdrant to `docker-compose.yml` with health checks. Both must come up with one command.

**2. Implement the interface.** `VectorStore` protocol plus both implementations. Retrieval code depends only on the protocol.

**3. Index the same corpus in both,** using your Lesson 4.2 winning chunking strategy and Lesson 4.1 embedding model.

**4. Benchmark.** Reproduce the comparison table on your own corpus: build time, p95 unfiltered, p95 filtered, ANN recall vs exact, memory. Tune `ef_search` in both until ANN recall ≥ 0.95, and note what you had to set.

**5. Demonstrate the post-filter collapse.** Implement both orderings deliberately, run a filtered query set, and report results-returned and recall for each. For pgvector, run `EXPLAIN ANALYZE` on your filtered query and confirm what the planner actually did — this is the part that surprises people.

**6. Write the decision record.** `docs/adr-vector-store.md`: requirements, both benchmarks, the choice, the reasoning, the switching conditions with concrete thresholds, and the estimated migration cost. Module 12 uses this document directly.

---

## Visual Aid

*Two panels. Left, "post-filter": a wide field of dots representing the corpus, with a small blue subset marked "matches filter." A circle labeled "top-10 by similarity" is drawn over the densest region and contains 9 grey dots and 1 blue; an arrow points right to a nearly-empty results box labeled "1 result." Right, "pre-filter": the same field, but everything except the blue subset is greyed out first, and the top-10 circle is drawn within the blue subset only, containing 10 blue dots, with an arrow to a full results box labeled "10 results." Caption: "Same k, same filter, same index. Forty-five points of recall, and no error either way."*

---

## Key Takeaways

- **ANN recall is a tunable, not a fixed property.** Check `ef_search` before blaming your embedding model; verify against exact search on a sample.
- **pgvector wins on operational simplicity, transactional consistency, and joins to relational data.** It is the honest default when the corpus fits and Postgres is already there.
- **Qdrant-class stores win on filtered search, quantization, sparse vectors, and scale** — at the cost of a second stateful system and the dual-write problem.
- **Post-filtering silently destroys recall on selective filters.** Fewer results than requested, or filtered recall far below unfiltered, is the diagnostic. Verify the query plan, do not assume.
- **Hide the store behind an interface** so "we can migrate later" is a fact rather than a hope — and write the decision record with concrete switching thresholds.

---

## Next

Both stores are indexed and fast, and both just failed the same query: someone searched for an exact error code, and the answer sat at rank 19 while semantically-similar-but-wrong chunks filled the top 5.

Dense vectors are good at meaning and bad at exactness. There is a fifty-year-old technique that is the reverse, and combining them is what production retrieval actually looks like.

**Lesson 4.4** adds BM25 alongside your dense retriever and fuses the two — with an honest accounting of the cost, because hybrid search roughly doubles your query work and must earn it on the numbers.
