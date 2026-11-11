---
id: "04.2"
module: 4
minutes: 14
practice_minutes: 90
prerequisites: ["04.1"]
objectives:
  - "Design and defend a chunking strategy, and demonstrate its effect on retrieval recall."
volatility: implementation
sources: []
last_verified: "2026-08-24"
---

# 04.2 · Chunking: The Decision That Quietly Determines Your Recall

Chunking gets one paragraph in most RAG tutorials: split at 1,000 characters with 200 overlap, move on. It deserves a great deal more attention, because it sets a hard ceiling on everything downstream. No reranker, no query rewriting, and no better model can retrieve information that your chunking destroyed.

It is also the cheapest thing to fix. Re-chunking and re-indexing a corpus takes minutes. Changing your embedding model, your vector store, or your generation prompt is a bigger commitment with a smaller payoff.

---

## Key Concept 1: The central tension

You are optimizing two things that pull against each other.

**Chunks must be small enough to embed precisely.** An embedding is a single fixed-length vector — one point in space for the entire chunk. A 3,000-token chunk covering five topics produces a vector that is the average of five things and near none of them. Retrieval degrades.

**Chunks must be large enough to answer from.** A single sentence embeds sharply but often lacks the context that makes it useful. "This is not supported in v3" is precise and useless — what is not supported?

Small chunks: good retrieval, poor generation. Large chunks: poor retrieval, good generation. Every naive strategy is a compromise on this axis, and the two techniques in Key Concept 4 dissolve it entirely — which is why they are the important part of this lesson.

---

## Key Concept 2: The strategy ladder

**Fixed-size with overlap.** Split every N characters or tokens, with an overlap so that content spanning a boundary appears in both. Simple, predictable, and content-blind — it will happily split a table in half or separate a heading from the paragraph it introduces. This is your honest baseline, not your answer.

**Recursive character splitting.** Try to split on paragraph breaks; if a chunk is still too large, split on sentences; then on words. Respects natural boundaries opportunistically. A clear improvement for prose, still blind to document structure.

**Structure-aware splitting.** Use the document's own boundaries. Markdown headings, HTML sections, function and class definitions in code, rows in a table, articles in a contract. The insight is that documents already encode their semantic structure and most chunkers ignore it. This is the single largest ordinary win available.

**Semantic splitting.** Embed sentences, walk through the document, and split where consecutive-sentence similarity drops sharply — a topic boundary. Elegant, and expensive: it costs an embedding pass over the whole corpus at index time. Worth it for unstructured text with no reliable markup; usually unnecessary where structure exists.

---

## Key Concept 3: Metadata is not optional

Every chunk carries a payload, decided now, used everywhere later:

```python
@dataclass
class Chunk:
    id: str
    text: str
    source: str            # "api/batch.md"
    section_path: list[str] # ["API Reference", "Batch", "Limitations"]
    position: int          # ordinal within the document
    prev_id: str | None    # for small-to-big expansion
    next_id: str | None
    token_count: int
    indexed_at: datetime   # for staleness and re-indexing
```

Each field earns its place downstream:

- `section_path` → contextual headers (below), citations (Lesson 5.4), metadata filtering (Lesson 4.3)
- `prev_id`/`next_id` → small-to-big expansion (below and Lesson 5.3)
- `source` → citations, and per-tenant filtering in Module 11
- `token_count` → context budgeting in Lesson 5.3
- `indexed_at` → incremental re-indexing, staleness checks

Adding these later means re-indexing the corpus. Adding them now costs nothing.

---

## Key Concept 4: The two techniques that dissolve the tension

**Contextual headers.** Prepend each chunk's document title and heading path to its text *before embedding*:

```
[API Reference > Batch Endpoints > Limitations]

Streaming responses are not available for this endpoint.
Requests exceeding 100 items are rejected.
```

Now the chunk embeds as being about batch endpoint limitations, not as an orphaned sentence about streaming. A query like "can batch endpoints stream?" matches strongly. Without the header, that chunk's vector sits near a cluster of unrelated streaming content.

This costs a handful of tokens per chunk and is routinely the largest single-line improvement in the whole retrieval stack.

A more expensive variant, sometimes called contextual retrieval, uses a model at index time to write a one-sentence situating summary per chunk. It works well and costs a generation call per chunk — measure whether it beats the free header version on your corpus before paying for it.

**Small-to-big.** Decouple the two requirements entirely: **embed small, retrieve small, generate from big.** Index precise 200–300 token chunks, and when one is retrieved, expand to its parent section or its neighbours (using `prev_id`/`next_id`) before passing context to the model.

You get sharp retrieval and rich generation context, which is the tension resolved rather than compromised. The cost is bookkeeping — the parent/neighbour relationships must exist, which is why they are in the metadata above. Lesson 5.3 implements the expansion side.

---

## Worked Example: One page, five strategies

Corpus: 4,200 chunks of API documentation. Embedding model and vector store held constant. Golden set: 60 queries with relevance labels.

| Strategy | Mean chunk tokens | Chunks | recall@5 | multi-hop recall@5 |
|---|---|---|---|---|
| Fixed 1000 chars / 200 overlap | 240 | 5,100 | 0.61 | 0.44 |
| Recursive character split | 290 | 4,600 | 0.68 | 0.51 |
| Structure-aware (markdown headings) | 380 | 4,200 | 0.78 | 0.66 |
| Structure-aware + contextual headers | 405 | 4,200 | **0.87** | **0.79** |
| Semantic splitting | 350 | 4,450 | 0.79 | 0.68 |

Three things to read out of this.

**Structure-aware over fixed-size: +17 points.** The largest single jump, from respecting boundaries the document already had.

**Contextual headers: +9 points on top of that, for 25 extra tokens per chunk.** Best return per unit effort in this course.

**Semantic splitting ≈ structure-aware, at much higher index cost.** For a corpus with reliable markdown structure, it buys nothing. On unstructured text it would be the winner. This is why "best practice" answers are unhelpful and measurement is not.

**The query all five miss.** One golden-set query fails under every strategy:

> "Why does the batch endpoint return 413?"

The relevant chunk exists and *is retrieved* — at rank 19. Retrieval found it; ranking buried it. Re-chunking will not fix this, and no further chunking experiment should be run against it.

That distinction — retrieval miss versus ranking failure — is the taxonomy in Lesson 5.1, and the fix is the reranker in Lesson 5.3. Recognizing which one you have is the difference between a productive afternoon and a wasted week.

---

## Practice

**1. Implement three strategies:** fixed-size with overlap (baseline), structure-aware for your corpus format, and structure-aware plus contextual headers. Emit the full `Chunk` metadata for all three.

**2. Respect the sequence limit.** Cap chunk size below your embedding model's max sequence length from Lesson 4.1 — including the header tokens, which are easy to forget.

**3. Evaluate all three** against your golden set: recall@5, recall@10, MRR, **per stratum**. Expect multi-hop and long-context strata to separate the strategies most.

**4. Isolate the header effect.** Run structure-aware with and without contextual headers as the only difference, and report the delta on its own. This is a one-line change with a large effect and you should see its size directly.

**5. Add small-to-big scaffolding.** Populate `prev_id`, `next_id`, and a `parent_section_id`. You will not use them until Lesson 5.3, but adding them now avoids a re-index later.

**6. Adopt the winner and record it.** Append to `eval-report.md` with the before/after table — this is your **first measured improvement over the Module 3 baseline**. Note the cost side too: more chunks means more storage and slower indexing.

**7. Classify the failures.** For every golden-set query still failing after your best strategy, check whether the relevant chunk was retrieved at *any* rank. Split them into "never retrieved" and "retrieved but ranked low." Save the list — Module 5 opens by working on exactly this split.

---

## Visual Aid

*One documentation page rendered five times in a row, each with chunk boundaries drawn as coloured horizontal bands. Under the fixed-size version, a band visibly cuts a code block and a table in half, annotated in red. Under the structure-aware version, bands align with headings. Under the contextual-headers version, each band has a small grey prefix box showing `[API Reference > Batch > Limitations]`. Each rendering has its recall@5 printed beneath in large type: 0.61, 0.68, 0.78, 0.87, 0.79. Caption: "Same corpus, same embedding model, same vector store. Twenty-six points."*

---

## Key Takeaways

- **Chunking sets a hard ceiling on retrieval** and is the cheapest part of the stack to change. Nothing downstream can recover information the chunker destroyed.
- **The tension is precision versus sufficiency:** small chunks embed sharply but answer poorly; large chunks answer well but embed vaguely.
- **Structure-aware splitting is the largest ordinary win** — documents already encode their semantics and most chunkers throw that away.
- **Contextual headers are the best return per unit effort in the course:** a few tokens per chunk, about nine points of recall in the worked example, one line of code.
- **Small-to-big dissolves the tension** — embed small, generate from big — and depends on the parent/neighbour metadata you should add now rather than after a re-index.
- **Distinguish "never retrieved" from "retrieved and ranked low."** Only the first is a chunking problem.

---

## Next

You have good chunks with good vectors and nowhere to put them. Storing and searching a few thousand vectors is easy; doing it with metadata filtering, at scale, with an operational story is a genuine architectural decision — and one you will be asked to defend.

**Lesson 4.3** compares the two live options: extend the Postgres you already run, or add a purpose-built vector database. Both run in Docker, both get benchmarked, and one filtering mistake in either will silently cut your recall in half.
