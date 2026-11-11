---
id: "05.2"
module: 5
minutes: 12
practice_minutes: 90
prerequisites: ["05.1"]
objectives:
  - "Improve recall with query rewriting and multi-query retrieval, measured independently."
volatility: implementation
sources: []
last_verified: "2026-08-24"
---

# 05.2 · Query Rewriting and Multi-Query Retrieval

Every retrieval technique in Module 4 assumed the query was a reasonable description of what the user wanted. Real queries are not.

Real queries are three words long. They contain pronouns referring to something said two turns ago. They use the words the user knows rather than the words your documentation uses. They bundle three questions into one sentence. And your beautifully-tuned hybrid retriever, handed "does it support that?", will return nothing useful — not because retrieval is broken, but because that string genuinely does not describe anything.

The cheapest fix in RAG is often upstream of the retriever entirely.

---

## Key Concept 1: The vocabulary mismatch

Your corpus was written by people who know the system. Your queries come from people who do not. That asymmetry produces predictable misses:

| User asks | Corpus says |
|---|---|
| "login problems" | "authentication failures" |
| "it's too slow" | "latency optimization", "throughput tuning" |
| "can I undo it?" | "rollback", "revert", "transaction abort" |
| "the thing that sends emails" | "notification dispatch service" |

Dense retrieval closes some of this — that is what embeddings are for — but not all of it, especially where your domain's jargon is rare enough that the embedding model has weak representations for it. BM25 closes none of it.

Query rewriting closes the gap from the other side: transform the query into something that looks like the corpus before you search.

---

## Key Concept 2: Conversational rewriting — not optional

If your system has more than one turn, this is mandatory, and it is the first thing to build.

```
Turn 1: "What are the batch endpoint limits?"
Turn 2: "Does it support streaming?"
```

Turn 2 embedded alone is a query about streaming in general. It retrieves streaming documentation for every endpoint and probably not the batch page. The user's actual question — "does the batch endpoint support streaming responses?" — is nowhere in the string you are searching with.

The fix is a small model call that produces a **standalone query**:

```
Given the conversation history, rewrite the latest user message as a
self-contained search query. Resolve all pronouns and implicit
references. Output only the rewritten query.

History:
  User: What are the batch endpoint limits?
  Assistant: Batch endpoints accept up to 100 items per request...
  User: Does it support streaming?

Rewritten: Does the batch endpoint support streaming responses?
```

Two implementation notes that matter:

- **Rewrite for retrieval, generate from the original.** The rewritten query goes to the retriever; the model answering the user should still see the real conversation, or the answer will read oddly.
- **Cap the history.** Feeding twenty turns into the rewriter makes it expensive and makes rewrites drift toward earlier topics. The last three to five turns, plus a running summary, is usually right.

---

## Key Concept 3: The rest of the toolkit

**Query expansion.** Add likely corpus terms — synonyms, expanded acronyms, related jargon — to the query before searching. Cheap, and particularly effective for BM25, which has no semantic flexibility of its own.

**Query decomposition.** Split a compound question into retrievable parts:

> "How do I set up authentication and what are the rate limits for authenticated requests?"
> → "How do I set up authentication?"
> → "What are the rate limits for authenticated requests?"

Retrieve for each, union the results. This is the main fix for the multi-hop stratum, where a single query cannot match two documents that each hold half the answer.

**Multi-query generation.** Write three paraphrases of the query, retrieve for each, fuse with RRF from Lesson 4.4:

```
Original: "Why is my upload failing?"
  1. "upload error troubleshooting"
  2. "file upload failure causes"
  3. "413 payload too large upload"
```

Each paraphrase covers different vocabulary. RRF handles the merge with no tuning — this is precisely why you built it generic over N lists.

**HyDE** (Hypothetical Document Embeddings). Ask the model to *write an answer* to the query, then embed that hypothetical answer and retrieve with it. The insight is that answers are structurally and lexically closer to documents than questions are — a question and its answer often share few words, while an answer and the document containing it share many. HyDE can work well on corpora where questions and documents differ sharply in style. It also fails distinctively: when the model's hypothetical answer is confidently wrong, you retrieve confidently wrong neighbourhoods. Measure it; do not adopt it on reputation.

---

## Key Concept 4: Every one of these costs a round trip

This is the discipline that separates this lesson from a list of tricks.

Each technique adds at least one model call **before retrieval even starts**, which means it lands directly on your time-to-first-token — the latency the user feels most (Lesson 1.3). Multi-query with three paraphrases means one generation call plus three retrievals plus fusion.

So:

- **Measure each technique independently.** They do not compose additively, and two of them together can be worse than either alone.
- **Report per stratum.** Decomposition helps multi-hop and does nothing for simple factual queries. The aggregate will understate it and the stratum will show it.
- **Report cost and latency alongside recall**, every time.
- **Expect to reject at least one.** A technique that buys two points of recall for 600 ms of TTFT is a bad trade for an interactive assistant, and saying so with numbers is a stronger result than adopting it.

Small models are usually sufficient for rewriting. It is a mechanical transformation, not a reasoning task — use your cheapest backend and note the saving.

---

## Worked Example 1: The conversational rewrite

Turn 4 of a real conversation. Query: *"does it support that?"*

**Without rewriting:**

| Rank | Chunk |
|---|---|
| 1 | Streaming responses — overview |
| 2 | Server-sent events in the SDK |
| 3 | Webhook delivery guarantees |
| 4 | Streaming for chat endpoints |
| 5 | Long-polling alternatives |

The correct chunk — batch endpoint limitations — is at rank 41.

**With rewriting** → *"Does the batch endpoint support streaming responses?"*

| Rank | Chunk |
|---|---|
| 1 | **Batch endpoints — limitations** ✓ |
| 2 | Batch endpoints — overview |
| 3 | Streaming responses — overview |

Rank 41 → rank 1, from one cheap model call.

## Worked Example 2: Each technique, measured

60 golden-set queries, hybrid retrieval from Module 4 as the baseline.

| Configuration | recall@10 | multi-hop | conversational | Added TTFT | Added cost |
|---|---|---|---|---|---|
| Baseline (hybrid) | 0.88 | 0.71 | 0.52 | — | — |
| + conversational rewrite | 0.91 | 0.72 | **0.89** | +180 ms | +$0.0002 |
| + decomposition | 0.93 | **0.88** | 0.89 | +340 ms | +$0.0005 |
| + multi-query (3×) | 0.94 | 0.89 | 0.90 | +520 ms | +$0.0011 |
| + HyDE | 0.92 | 0.86 | 0.88 | +910 ms | +$0.0019 |

Reading the table:

**Conversational rewriting is the clear win** — 37 points on the conversational stratum for 180 ms. Adopt without hesitation.

**Decomposition earns its place on multi-hop** — 16 points on the stratum that motivated it, for 340 ms. Adopt.

**Multi-query adds one point for another 180 ms and doubles the added cost.** Inside the noise band from Lesson 3.4. **Reject** — and note in the eval report that it was measured and rejected, which is exactly the kind of entry that reads as discipline.

**HyDE made things worse** and cost the most. On this corpus, documentation is written in a Q&A-adjacent style already, so hypothetical answers added noise rather than signal. **Reject.** On a corpus of dense prose it might have won — which is the argument for measuring rather than adopting on reputation.

**Final configuration:** conversational rewriting + decomposition. +0.05 recall, +520 ms TTFT, +$0.0007 per request. Recorded with the trade-off stated.

---

## Practice

**1. Implement conversational rewriting.** Prompt in `spine/prompts/`, versioned. Cap history at three to five turns. Route it to your cheapest backend and confirm quality holds.

**2. Implement decomposition and multi-query.** Both output a list of queries; both feed your existing RRF from Lesson 4.4.

**3. Implement HyDE** if your corpus style suggests it might help. Optional, but running it and rejecting it is a legitimate outcome worth documenting.

**4. Measure each in isolation**, then in combination. Report recall@10 per stratum, added TTFT, and added cost per technique. Build the table above with your own numbers.

**5. Reject at least one on cost.** If everything genuinely helps, report the marginal return of the weakest and state why it still clears the bar. The point is that the decision is made on evidence.

**6. Append to `eval-report.md`,** including the rejected techniques and the reasoning. Rejections are more informative than adoptions.

---

## Visual Aid

*Two horizontal pipelines. Top: `"does it support that?"` → retriever → a results list of five grey chunks, with a small far-right marker at position 41 showing the correct chunk in green, labeled "rank 41." Bottom: the same query → a small box labeled "rewrite (cheap model, 180 ms)" → `"Does the batch endpoint support streaming responses?"` → retriever → results list with the correct chunk green at position 1. A cost annotation beneath the rewrite box reads "+$0.0002." Caption: "The retriever was never the problem."*

---

## Key Takeaways

- **The query is often the bug.** Users write terse, pronoun-laden, jargon-mismatched queries; no index tuning fixes a string that does not describe anything.
- **Conversational rewriting is mandatory for multi-turn systems** and usually the single largest retrieval gain available. Rewrite for retrieval, generate from the original.
- **Decomposition targets multi-hop; multi-query and expansion target vocabulary; HyDE targets question/document style mismatch.** Each addresses a different failure and shows up in a different stratum.
- **Every technique costs a round trip before retrieval starts**, landing directly on TTFT. Measure each independently, per stratum, with cost and latency.
- **Expect to reject one.** A measured rejection is a stronger portfolio entry than an unmeasured adoption.

---

## Next

The right document is now in your candidate set for almost every query. For a meaningful share of them it is sitting at rank 23, and your cutoff is 5 — the ranking failure from Lesson 5.1, which no amount of query rewriting will fix.

**Lesson 5.3** adds the reranking funnel that promotes it, and then deals with what happens to the five survivors: deduplication, ordering for attention, and the counterintuitive finding that more context frequently makes answers worse.
