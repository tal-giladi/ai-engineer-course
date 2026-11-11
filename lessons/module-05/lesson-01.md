---
id: "05.1"
module: 5
minutes: 12
practice_minutes: 60
prerequisites: ["04.4"]
objectives:
  - "Classify a wrong RAG answer by failure mode and fix the layer that is actually broken."
volatility: concept
sources: []
last_verified: "2026-08-24"
---

# 05.1 · The RAG Failure Taxonomy

"The RAG is wrong" is not a bug report. It is the beginning of a week of guessing, and the guessing usually starts at the wrong end — someone tries a different embedding model when the problem was context ordering, or rewrites the prompt when the answer was never retrieved.

A RAG pipeline has four sequential stages. Each fails in a distinct way, each failure has a distinct fix, and — crucially — **each has a distinct metric signature**. Once you can localize a failure, the fix is usually obvious. Localization is the skill.

This is also the exact skill probed in an AI system design interview. "Your RAG system is returning wrong answers. Walk me through how you'd diagnose it" is a standard question, and the difference between a good and a weak answer is whether you enumerate stages or start listing technologies.

---

## Key Concept 1: The four stages

```
query → [1 RETRIEVE] → candidates → [2 RANK] → top-k
      → [3 ASSEMBLE] → context → [4 GENERATE] → answer
```

Each stage can only work with what the previous one handed it. That ordering is why localization matters: a stage-1 failure cannot be fixed at stage 3, and effort spent there is wasted.

---

## Key Concept 2: Failure mode 1 — retrieval miss

**Symptom:** the chunk containing the answer is not in the candidate set at *any* rank.

**Nothing downstream can recover this.** No reranker can promote a document it never received; no prompt can make the model cite text it never saw. This is the failure to check for first, always.

**Causes:**
- Chunking destroyed or diluted the information (Lesson 4.2)
- Embedding model is a poor fit, or is silently truncating (Lesson 4.1)
- Lexical gap — an exact-token query with no sparse retriever (Lesson 4.4)
- The query is phrased so differently from the corpus that nothing matches (fixed in Lesson 5.2)
- A filter excluded it — including a silent post-filter (Lesson 4.3)
- **The information genuinely is not in the corpus.** This is not a bug; the correct behavior is refusal (Lesson 5.4)

**Metric signature:** **low context recall.**

**Diagnostic:** search for the answer text directly in your corpus. If it exists, retrieve with k=100 and check whether it appears anywhere. Present at rank 60 means retrieval works and ranking is broken — a different problem. Absent at k=100 is a true retrieval miss.

---

## Key Concept 3: Failure mode 2 — ranking failure

**Symptom:** the right chunk *is* in the candidate set, at rank 23, and your cutoff is 5.

This is the most commonly misdiagnosed mode, because from the outside it looks identical to a retrieval miss — the model did not see the chunk either way. The distinction is entirely visible in the pipeline and invisible in the output, which is why you need a diagnostic tool rather than intuition.

Getting this wrong is expensive: teams re-chunk their corpus, swap embedding models, and re-index — hours of work — when the fix was adding a reranker.

**Causes:**
- Dense similarity is a coarse ranking signal; near-misses often outscore the exact answer
- Several near-duplicate chunks crowd the top positions
- The query matches the *topic* of many chunks and the *answer* in only one

**Metric signature:** **context recall is fine, context precision is low.**

**Fix:** reranking (Lesson 5.3). Retrieve wide, rank narrow.

---

## Key Concept 4: Failure mode 3 — context assembly failure

**Symptom:** the right chunk was retrieved *and* ranked into the top-k, and the model still did not use it.

**Causes:**
- **Truncation.** The context exceeded the budget and the chunk was cut, often silently.
- **Lost in the middle.** Models attend less reliably to material in the middle of a long context. A chunk at position 7 of 12 may as well be absent (Lesson 1.3).
- **Duplicate crowding.** Five near-identical chunks consume the budget and add nothing.
- **Too much context.** Counterintuitively, more retrieved material frequently produces *worse* answers — the signal is diluted.
- **Fragmentation.** The chunk is present but its surrounding context is not, so it is uninterpretable (the case small-to-big solves).

**Metric signature:** **context recall and precision both fine, faithfulness low or answer relevance low.**

**Fix:** deliberate context assembly (Lesson 5.3) — deduplicate, order for attention, expand small-to-big, budget explicitly.

---

## Key Concept 5: Failure mode 4 — generation drift

**Symptom:** everything needed was in the context, well-placed, and the model still got it wrong.

**Causes:**
- **Parametric override** — the model answers from training memory rather than the provided context. Especially common where its memory is confidently wrong, such as an older version of the API you documented.
- **Fabricated citations** — a citation to a document that was not retrieved, or was retrieved and does not say that.
- **Hedging** when the context is clear, or **confidence** when it is not.
- **Failure to refuse** — answering an out-of-scope question from general knowledge (the failure from Lesson 3.1).

**Metric signature:** **context recall and precision good, faithfulness low.**

**Fix:** grounding, citation validation, refusal (Lesson 5.4).

---

## Key Concept 6: The diagnostic tool

You cannot classify by looking at the answer. Build the tool that shows you the pipeline:

```python
def diagnose(query: str) -> Diagnosis:
    """Dump every stage for one query."""
    rewritten = rewrite(query)                 # Lesson 5.2
    candidates = retrieve(rewritten, k=100)    # wide, for diagnosis
    reranked = rerank(query, candidates, k=10)
    context = assemble(reranked)
    answer = generate(query, context)

    return Diagnosis(
        candidate_ranks={c.id: i for i, c in enumerate(candidates)},
        rank_of_expected=...,      # from golden set relevant_chunks
        made_cutoff=...,
        position_in_context=...,   # where in the assembled context
        context_tokens=len(context),
        truncated=...,
        answer=answer,
        cited_ids=answer.citations,
        cited_but_not_retrieved=...,
    )
```

Four fields do most of the work: **rank of the expected chunk in the full candidate list**, **whether it made the cutoff**, **its position in the assembled context**, and **whether any citation refers to something not retrieved**. Those four numbers classify almost every failure in seconds.

---

## Worked Example: Four wrong answers

Same system, four failing queries, all superficially similar.

**Q1: "What's the max payload size for batch uploads?"** → *"The documentation does not specify a maximum payload size."*

```
rank_of_expected: not in top 100
context_recall: 0.0
```
The chunk mentions "413 Payload Too Large" and never says "max payload size." Lexical gap, dense-only retrieval. **Retrieval miss → hybrid search (Lesson 4.4).**

**Q2: "How do I paginate the results endpoint?"** → *a plausible answer describing the wrong pagination scheme*

```
rank_of_expected: 23      made_cutoff: False
context_recall: 0.85      context_precision: 0.31
```
Seven near-duplicate chunks about pagination in other endpoints outranked it. **Ranking failure → reranking (Lesson 5.3).**

**Q3: "What are the rate limits for the v3 API?"** → *correct limits for v2*

```
rank_of_expected: 3       made_cutoff: True
position_in_context: 6 of 12     context_tokens: 9,400
context_recall: 0.92      context_precision: 0.78     faithfulness: 0.41
```
Retrieved and ranked well, then buried mid-context among eleven other chunks, several of them v2 rate limits. **Context assembly failure → deduplicate, reorder, trim (Lesson 5.3).**

**Q4: "Does the SDK support async?"** → *"Yes, the SDK provides full async support via `AsyncClient`." Cites `sdk/async.md`.*

```
rank_of_expected: 1       made_cutoff: True
position_in_context: 1 of 4      context_tokens: 2,100
context_recall: 1.0       context_precision: 0.95     faithfulness: 0.20
cited_but_not_retrieved: ["sdk/async.md"]
```
Perfect retrieval. The retrieved chunk says async support is planned but not available. The model answered from training memory about a *different* SDK and invented a citation to a file that does not exist. **Generation drift → grounding and citation validation (Lesson 5.4).**

**The point:** Q2 and Q3 produce output that looks the same to a user — a plausible, confidently wrong answer. Their fixes share nothing. And Q4's fabricated citation was caught by one line of code checking cited IDs against retrieved IDs.

---

## Practice

**1. Build `spine/eval/diagnose.py`** with the fields above. Make it runnable on a single query from the command line, printing a readable report.

**2. Run it over every failure.** Take every golden-set query your Module 4 pipeline gets wrong and produce a diagnosis for each.

**3. Classify each** into the four modes. Where you are unsure, note which field would have resolved it and add that field to the tool.

**4. Build the failure distribution:**

| Mode | Count | % of failures |
|---|---|---|
| Retrieval miss | | |
| Ranking failure | | |
| Context assembly | | |
| Generation drift | | |

**5. Use it to prioritize.** This table tells you which of the next four lessons matters most for *your* system. If 70% of your failures are ranking failures, Lesson 5.3 is your highest-value hour. Note your prediction before you start, and check it at the end of the module.

**6. Cross-check against Lesson 4.2.** In Module 4 you split failures into "never retrieved" and "retrieved but ranked low." Confirm the classifications agree, and investigate any that do not.

---

## Visual Aid

*A four-stage horizontal pipeline: RETRIEVE → RANK → ASSEMBLE → GENERATE, each as a box. Below each box, a red failure label ("retrieval miss," "ranking failure," "context assembly failure," "generation drift"), and below that, the metric signature in monospace ("context recall ↓", "precision ↓ recall ok", "both ok, faithfulness ↓", "all ok, faithfulness ↓"). A dashed grey arrow runs backwards from GENERATE to RETRIEVE labeled "where teams look first." A solid green arrow runs forward labeled "where you should look first." Beneath, four small trace strips show Q1–Q4 from the worked example, with the failing stage highlighted in each.*

---

## Key Takeaways

- **Localize before fixing.** Four stages, four failure modes, four different repairs — and effort spent on the wrong stage is entirely wasted.
- **Retrieval miss is unrecoverable downstream.** Always check first whether the chunk was retrieved at any rank.
- **Ranking failure is the most misdiagnosed mode.** It looks identical to a retrieval miss from the output and is trivially distinguishable from the pipeline.
- **The four decisive fields:** rank of the expected chunk in the full candidate list, whether it made the cutoff, its position in the assembled context, and whether any citation was never retrieved.
- **Your failure distribution tells you where to spend the rest of this module.** Build it before you optimize anything.

---

## Next

Start where nothing downstream can help you: the queries where the answer never showed up at all. Often the problem is not your index — it is that the user wrote "does it support that?" in turn four of a conversation, or asked in words your corpus never uses.

**Lesson 5.2** fixes the query before it reaches the retriever: conversational rewriting, multi-query generation, decomposition, and HyDE — each measured separately, because each costs a round trip.
