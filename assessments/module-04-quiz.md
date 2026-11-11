# Module 4 quiz — Retrieval I: Embeddings, Chunking, Vector Stores

Ten scenario questions on the first half of the retrieval stack: picking an embedding model on your own evidence, chunking so the information survives, choosing and filtering a vector store without silently losing recall, and fusing dense and BM25 retrieval. They test reasoning, not recall. Most describe a benchmark table, a failing query or a design choice and ask what it shows or what to do next.

**Covers**

- [04.1 · Embeddings: Choosing and Benchmarking a Model](../lessons/module-04/lesson-01.md) — leaderboards as a shortlist, silent truncation, instruction prefixes, operational factors.
- [04.2 · Chunking: The Decision That Quietly Determines Your Recall](../lessons/module-04/lesson-02.md) — precision vs sufficiency, the strategy ladder, contextual headers, small-to-big, retrieval miss vs ranking failure.
- [04.3 · Vector Stores in Docker: pgvector vs Qdrant](../lessons/module-04/lesson-03.md) — ANN recall as a dial, pgvector vs Qdrant trade-offs, pre-filter vs post-filter, switching thresholds.
- [04.4 · Hybrid Retrieval: Dense + BM25 + Fusion](../lessons/module-04/lesson-04.md) — opposite failure modes, reciprocal rank fusion, per-query-type reporting, when hybrid earns its cost.

**Pass mark:** 70% (7 of 10). Unlimited retries; answers and explanations appear after you submit.

The hands-on part of this module is the [Module 4 practical](../projects/module-04-practical.md): chunking a codebase for retrieval and measuring the difference.
