# Module 4 Plan — Retrieval I: Embeddings, Chunking, Vector Stores

**Duration:** 55 min · **Lessons:** 4 · **Depends on:** Modules 1, 3
**Outline reference:** `curriculum/course-outline.md` §8, Module 4

---

## Prerequisites

**From Module 1:** `LLMClient` and telemetry.
**From Module 3:** the golden dataset and eval harness — this module's improvements must be *measured*, and retrieval metrics (recall@k, MRR, nDCG) were introduced in Lesson 3.3 specifically so they could be used here.
**From the student:** Docker, SQL and Postgres basics, and enough search intuition to know what an inverted index is.

**Environment:** Docker with ~4 GB free for pgvector and Qdrant containers; a Hugging Face account for embedding model downloads.

---

## Module Learning Objectives

**M4-LO1** — Choose an embedding model on evidence, benchmarking candidates from Hugging Face on their own corpus rather than trusting a leaderboard.
**M4-LO2** — Design and defend a chunking strategy, and demonstrate its effect on retrieval recall.
**M4-LO3** — Stand up pgvector and Qdrant in Docker, index the same corpus in both, and articulate when each is the right call.
**M4-LO4** — Implement hybrid retrieval combining dense vectors and BM25 with rank fusion, and measure the gain over either alone.

---

## Lesson 4.1 — Embeddings: Choosing and Benchmarking a Model

*13 min · serves M4-LO1*

**Concept.** An embedding model is a compression of meaning into a fixed vector, and the only question that matters is whether *your* queries land near *your* documents. Cover the selection axes that actually move retrieval quality: training domain (a model trained on web text underperforms on API documentation or legal text), asymmetric vs symmetric models and why query/document prefixes matter for the asymmetric ones, dimensionality as a storage and latency trade rather than a quality proxy, sequence length as a hard truncation limit that silently discards the tail of a long chunk, and multilingual coverage. Then the core discipline: **public leaderboards are a starting shortlist, not an answer** — they are averaged over benchmarks that are not your corpus, and are increasingly contaminated. The professional move is to benchmark three candidates on your own golden set, which Module 3 already provided.

**Example.** Three Hugging Face embedding models benchmarked on the docs corpus with recall@5 and MRR, where the leaderboard leader places third and a smaller, faster model wins — plus a case where the winner's 512-token limit was silently truncating half of every large chunk, and what the numbers looked like once that was fixed.

**Practice.** Students shortlist three embedding models, embed the spine corpus with each, and evaluate against the Module 3 golden set on recall@5, MRR, index size, and embedding throughput. They commit the comparison table and a one-paragraph defense of their choice.

**Summary.** Shortlist from the leaderboard, decide on your own data. Check sequence limits and query/document prefixes before blaming the model.

**Key terms:** embedding model, dense vector, cosine similarity, asymmetric retrieval, query/document prefix, dimensionality, sequence length truncation, MTEB.
**Visual aid:** Benchmark table — model, dimensions, max sequence length, recall@5, MRR, index size, throughput — with the leaderboard rank in the final column for contrast.
**Hook:** "You picked the best model for your corpus. Now the thing that will hurt your recall more than that choice ever did: how you cut the documents up."

---

## Lesson 4.2 — Chunking: The Decision That Quietly Determines Your Recall

*14 min · serves M4-LO2*

**Concept.** Chunking is the highest-leverage and least-discussed decision in RAG. Frame the fundamental tension: chunks small enough to embed precisely are too small to answer from, and chunks large enough to answer from dilute their own embedding. Walk the strategy ladder — fixed-size with overlap (the honest baseline), recursive character splitting on structural separators, **structure-aware** splitting that respects the document's own boundaries (markdown headings, function definitions, table rows), semantic splitting on embedding-similarity shifts, and the two techniques that decouple the tension entirely: **contextual headers** (prepend the document title and heading path to every chunk so an orphaned paragraph knows what it is about) and **small-to-big** (embed small precise chunks, retrieve, then expand to the surrounding parent chunk for generation). Cover the metadata that must ride along with every chunk — source, section path, position, timestamp — because Module 5's filtering and citations depend on it existing now.

**Example.** One API reference page chunked five ways, with recall@5 for each strategy on the same query set. Fixed-size at 0.61; structure-aware at 0.78; structure-aware plus contextual headers at 0.87. Then the failing query that all five miss, and why it is a ranking problem rather than a chunking problem — foreshadowing Lesson 4.4 and Module 5.

**Practice.** Students implement three chunking strategies over the spine corpus, evaluate each against the Module 3 golden set, and adopt the winner. They must add contextual headers to at least one strategy and report the isolated effect. Result goes into `eval-report.md` as the first measured improvement over baseline.

**Summary.** Respect the document's own structure, give every chunk enough context to stand alone, carry metadata from the start, and prove the choice with recall numbers rather than intuition.

**Key terms:** fixed-size chunking, overlap, recursive splitting, structure-aware chunking, semantic chunking, contextual headers, small-to-big retrieval, chunk metadata.
**Visual aid:** One document rendered five times with chunk boundaries drawn as coloured overlays, recall@5 printed under each.
**Hook:** "Good chunks, good vectors, and nowhere to put them. Two containers, two philosophies."

---

## Lesson 4.3 — Vector Stores in Docker: pgvector vs Qdrant

*13 min · serves M4-LO3*

**Concept.** The architectural choice this audience will be asked to justify in an interview: extend the database you already run, or add a purpose-built one. Cover what a vector store actually does — approximate nearest neighbour search via HNSW or IVF, with recall traded against latency by tuning parameters most teams never touch. Cover **pgvector**: transactional consistency with your relational data, one system to operate and back up, SQL joins between vectors and business data, and a genuine performance ceiling at scale. Cover **Qdrant** (as representative of Pinecone, Weaviate, and Milvus): purpose-built filtering, payload indexing, quantization, horizontal scale, and the cost of a second stateful system in the architecture. Cover metadata filtering as the feature that decides most real cases — pre-filter versus post-filter and why post-filtering silently destroys recall. Name the honest default: if the corpus fits comfortably and Postgres is already in the stack, pgvector until it hurts.

**Example.** The same corpus indexed in both, from a single `docker-compose.yml`, with a table comparing index build time, p95 query latency at k=10, recall against exact search, memory footprint, and lines of operational surface. Then the filtered query — "only documents from the v3 API section" — run as pre-filter and post-filter, showing recall collapsing from 0.86 to 0.41 under post-filtering.

**Practice.** Students index the spine corpus in both stores behind one `VectorStore` interface (the same seam pattern as Lesson 1.2), run the comparison, and write a decision record naming their choice, the conditions under which they would switch, and the migration cost.

**Summary.** Both work. Choose on operational surface and filtering needs, hide the choice behind an interface, and pre-filter — always.

**Key terms:** ANN search, HNSW, IVF, recall/latency trade-off, pgvector, Qdrant, payload/metadata filtering, pre-filter vs post-filter, quantization, decision record.
**Visual aid:** Side-by-side comparison table plus a diagram of pre-filter vs post-filter showing where candidates are discarded in each.
**Hook:** "Both stores just failed the same query — someone searched for an exact error code, and vectors are bad at exact. There's a fifty-year-old answer for that."

---

## Lesson 4.4 — Hybrid Retrieval: Dense + BM25 + Fusion

*13 min · serves M4-LO4*

**Concept.** Dense and sparse retrieval fail in opposite directions, which is why production systems run both. Dense embeddings capture meaning and handle paraphrase, but blur exact tokens — error codes, function names, version numbers, rare proper nouns, anything the embedding model never saw. BM25 is exact-term matching with term-frequency weighting: excellent on those, useless on paraphrase. Cover fusion: **Reciprocal Rank Fusion** as the robust default because it combines ranks rather than incomparable scores and needs no tuning, versus weighted score fusion which is stronger when tuned and fragile when the score distributions drift. Cover the practical implementations — Postgres full-text search alongside pgvector, or Qdrant's sparse vectors — and the honest caveat that hybrid roughly doubles query cost and latency, so it must earn its place on the numbers.

**Example.** A query set split into three groups — paraphrase queries, exact-token queries, and mixed — with recall@10 for dense alone, BM25 alone, and RRF. Dense wins group one, BM25 wins group two by a wide margin, and RRF wins overall while losing slightly to each specialist on its home ground. Then the latency and cost column that makes the trade-off explicit.

**Practice.** Students implement BM25 alongside their dense retriever, fuse with RRF, and evaluate all three configurations against the Module 3 golden set — reporting per-stratum results, since the whole point is that the win is concentrated in one stratum. They record the retrieval quality and cost delta in `eval-report.md`.

**Summary.** Dense for meaning, sparse for exactness, RRF to combine without tuning. Report per stratum, because the aggregate hides where hybrid actually earns its cost.

**Key terms:** sparse retrieval, BM25, TF-IDF, lexical gap, hybrid search, Reciprocal Rank Fusion, weighted score fusion, rank vs score combination.
**Visual aid:** Three-column results diagram for one exact-token query — dense top-5, BM25 top-5, fused top-5 — with the correct document highlighted at rank 14, rank 1, and rank 2 respectively.
**Hook:** "Your retrieval is genuinely good now, and the system still gets answers wrong. Module 5 is about the four different reasons why — and how to tell them apart from the numbers."

---

## Spine Increment

**Deliverable:** `spine/retrieval/` — ingestion pipeline, benchmarked embedding model, structure-aware chunking with contextual headers and metadata, a `VectorStore` interface with pgvector and Qdrant implementations, and hybrid dense+BM25 retrieval with RRF.

**Acceptance criteria:**
1. Embedding model chosen from a committed three-way benchmark on the project's own golden set.
2. Chunking strategy chosen from a measured three-way comparison, with metadata sufficient for Module 5 citations.
3. Both vector stores run from `docker-compose.yml` behind one interface; a decision record names the choice and the switching conditions.
4. Hybrid retrieval measured against dense-only and BM25-only, reported per stratum with cost and latency.
5. `eval-report.md` shows the retrieval recall improvement over the Module 3 baseline.

---

## Standalone Exercise

Index a mid-sized open-source codebase and measure retrieval recall across three chunking strategies — fixed-size, function-boundary-aware, and function-boundary with a contextual header carrying the file path and class name. Deliverable: `exercises/m4-code-retrieval/` with the recall table and a paragraph on why code chunking differs from prose chunking.

---

## Dependencies

| Lesson | Requires |
|---|---|
| 4.1 | Modules 1, 3 |
| 4.2 | 4.1 (needs an embedding model to measure against) |
| 4.3 | 4.2 (needs chunks to index) |
| 4.4 | 4.3 (needs a working dense retriever to fuse with) |

**Downstream:** Module 5 builds reranking, query rewriting, and citations directly on this pipeline and depends on the chunk metadata added in 4.2. Module 10 optimizes and caches these queries. Module 11 treats retrieved documents as untrusted input — the indirect prompt injection surface.

---

## Assessment Focus (Phase 4)

Applied judgment: given a corpus description and query profile, choose an embedding model and chunking strategy with reasons. Given a query that fails under dense retrieval, predict whether BM25 recovers it. Given a filtered query with collapsed recall, diagnose post-filtering. Given pgvector-vs-Qdrant requirements, write the decision record. One practical: improve recall@5 on a supplied corpus by a stated margin and show the measurement.
