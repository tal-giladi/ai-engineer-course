# Module 5 Plan — Retrieval II: Production RAG

**Duration:** 60 min · **Lessons:** 5 · **Depends on:** Modules 3, 4
**Outline reference:** `curriculum/course-outline.md` §8, Module 5

---

## Prerequisites

**From Module 3:** eval harness, calibrated judge, CI gate.
**From Module 4:** hybrid retrieval over a chunked, metadata-carrying index.
**From Module 2:** the typed answer contract, which this module extends with citations and a refusal flag.

---

## Module Learning Objectives

**M5-LO1** — Classify a wrong RAG answer by failure mode — retrieval miss, ranking failure, context overflow, or generation drift — and fix the layer that is actually broken.
**M5-LO2** — Improve recall with query rewriting and multi-query retrieval, and precision with a cross-encoder reranker, measuring each independently.
**M5-LO3** — Produce grounded, cited answers and make the system refuse when the context does not support an answer.
**M5-LO4** — Evaluate a RAG pipeline on faithfulness, context precision, and context recall, and read those numbers as a diagnosis.

---

## Lesson 5.1 — The RAG Failure Taxonomy

*12 min · serves M5-LO1*

**Concept.** "The RAG is wrong" is not a bug report. A RAG pipeline has four sequential stages and each fails differently, so the first professional skill is localization. **Retrieval miss** — the answer is not in the retrieved set at any rank, so nothing downstream can save it; caused by chunking, embedding fit, or the lexical gap. **Ranking failure** — the answer is in the candidate set at rank 23, below the cut; caused by weak ranking rather than weak retrieval, and fixed by reranking rather than by re-chunking. **Context assembly failure** — the right chunk was retrieved and ranked but got truncated, buried in the middle of a long context where models attend poorly, or crowded out by nine near-duplicates. **Generation drift** — everything needed was in the context and the model still answered from parametric memory, hedged, or fabricated a citation. Each has a distinct fix, and each has a distinct metric signature — which is the whole reason Lesson 5.5 exists.

**Example.** Four wrong answers from the spine system, one per failure mode, each traced back through the pipeline with the retrieved set printed at each stage. The critical demonstration: two of them look identical in the output and require completely different fixes.

**Practice.** Students build `spine/eval/diagnose.py`, a script that takes a failing query and dumps the retrieved candidates, their ranks, the assembled context, and the final answer — then run it over every failure in their Module 4 eval and tag each with its failure mode. They produce a failure distribution table.

**Summary.** Localize before fixing. Four stages, four failure modes, four different repairs — and the distribution tells you which lesson in this module matters most for your system.

**Key terms:** retrieval miss, ranking failure, context assembly failure, generation drift, lost-in-the-middle, failure localization.
**Visual aid:** Pipeline diagram with the four stages and a failure-mode label under each, plus a worked trace of one query through all four.
**Hook:** "Start with the one that nothing downstream can rescue: the answer never showed up at all."

---

## Lesson 5.2 — Query Rewriting and Multi-Query Retrieval

*12 min · serves M5-LO2*

**Concept.** Users write bad queries — terse, ambiguous, riddled with pronouns that refer to three turns ago, and phrased in words the corpus never uses. The vocabulary mismatch between question and document is the root cause of most retrieval misses. Cover the techniques in order of cost: **query expansion** (add synonyms and likely corpus terms), **query decomposition** (split a compound question into retrievable sub-questions), **multi-query generation** (write three paraphrases, retrieve for each, fuse the results with RRF from Lesson 4.4), **HyDE** (generate a hypothetical answer and embed *that*, because answers live closer to documents than questions do), and **conversational rewriting** (resolve "does it support that?" into a standalone query using dialogue history — non-optional for any chat interface). Cover the cost discipline: each of these adds at least one model call before retrieval even starts, so each must be justified per stratum.

**Example.** The query "does it support that?" in turn four of a conversation, retrieving nothing useful, then rewritten to "does the v3 batch API support streaming responses?" and retrieving correctly at rank 1. Followed by multi-query on an ambiguous question, with the three generated paraphrases shown and the fused result table.

**Practice.** Students implement conversational rewriting and multi-query retrieval, evaluating each independently against the Module 3 golden set. They report recall gain and added latency and cost per technique, then decide which to keep — with at least one technique expected to be rejected on cost.

**Summary.** Rewrite the query before you blame the retriever. Measure each technique separately, because they do not compose additively and they all cost a round trip.

**Key terms:** vocabulary mismatch, query expansion, query decomposition, multi-query retrieval, HyDE, conversational query rewriting, standalone query.
**Visual aid:** Before/after retrieval results for one conversational query, with the rewrite step drawn in between.
**Hook:** "The right document is now in your candidate set — at rank 23, with a cut-off of 5."

---

## Lesson 5.3 — Reranking and Context Assembly

*13 min · serves M5-LO2*

**Concept.** Two distinct jobs that get conflated. **Reranking**: a bi-encoder embeds query and document independently and is fast enough to search millions; a cross-encoder reads query and document *together* and is far more accurate but far too slow to run over the corpus. The production pattern is the funnel — retrieve 50 cheaply, rerank to 5 accurately. Cover cross-encoder rerankers from Hugging Face and hosted reranking APIs, and the latency budget the funnel consumes. **Context assembly**: what you do with the surviving 5. Cover deduplication of near-identical chunks, ordering for the lost-in-the-middle effect (most relevant first and last, weakest in the middle), the small-to-big expansion promised in Lesson 4.2, token budgeting against the context window, and the counterintuitive result that more context frequently makes answers worse.

**Example.** The rank-23 document from Lesson 5.2 promoted to rank 2 by a cross-encoder, with a table of recall@5 before and after and the added p95 latency. Then a context-assembly ablation: the same reranked top-5 assembled four ways, with answer quality varying by 11 points on nothing but ordering and deduplication.

**Practice.** Students add a cross-encoder reranker to the funnel and tune the retrieve-k, rerank-k pair against quality and latency. They then run a context-assembly ablation of their own — ordering, deduplication, small-to-big — and adopt the winner. Both deltas go into `eval-report.md`.

**Summary.** Retrieve wide and cheap, rerank narrow and accurate, then assemble deliberately. Ordering and deduplication are free quality that most systems leave on the table.

**Key terms:** bi-encoder, cross-encoder, reranking funnel, retrieve-k/rerank-k, deduplication, lost-in-the-middle, small-to-big expansion, context budget.
**Visual aid:** Funnel diagram — 10,000 chunks → 50 retrieved → 5 reranked → assembled context — with per-stage latency and cost annotated.
**Hook:** "Perfect context, and the model can still make something up. Now you make that impossible to hide."

---

## Lesson 5.4 — Grounding, Citations, and Knowing When to Refuse

*12 min · serves M5-LO3*

**Concept.** The trust layer, and the feature most visibly missing from demo-grade RAG. **Grounding**: instructing and structuring the generation so every claim traces to retrieved context — and enforcing it through the Module 2 answer contract rather than through a polite request in the prompt. **Citations**: span-level attribution that a user can verify, which requires the chunk metadata added in Lesson 4.2 and a contract where the citation identifiers are validated against the actual retrieved set, because models will happily cite a document that was never retrieved. **Refusal**: the hardest and most valuable behavior — saying "the documentation doesn't cover this" instead of confabulating. Cover why this is genuinely hard (models are trained toward helpfulness), how to make it measurable (the out-of-scope stratum deliberately built into the Module 3 golden set), and the calibration trade-off: an over-refusing system is useless in a different way, so refusal rate is tracked in both directions.

**Example.** An ungrounded answer and a grounded one for the same question, side by side with their citations. Then a fabricated citation caught by validating identifiers against the retrieved set, and the code that catches it. Then two systems at the same accuracy, one refusing 3% and one refusing 22%, and a discussion of which is preferable and for what audience.

**Practice.** Students extend the Module 2 answer contract with `citations: list[ChunkId]`, `refused: bool`, and `refusal_reason`, enforce that every citation exists in the retrieved set, and implement refusal. They evaluate on the out-of-scope stratum and report refusal rate in both directions — missed refusals and false refusals.

**Summary.** Ground it, cite it verifiably, and let it say no. Validate citations mechanically, and treat refusal as a calibrated behavior with a two-sided error rate.

**Key terms:** grounding, attribution, span-level citation, citation validation, hallucination, confabulation, refusal, over-refusal, abstention calibration.
**Visual aid:** Annotated answer showing each sentence linked to its source chunk, with one unsupported sentence flagged.
**Hook:** "Everything in this module was a change you claimed made it better. Time to prove it with metrics built for exactly this."

---

## Lesson 5.5 — Measuring RAG: Faithfulness, Context Precision, Context Recall

*13 min · serves M5-LO4*

**Concept.** RAG-specific metrics, taught as **diagnostics** rather than a scorecard — each one isolates a different stage of the Lesson 5.1 taxonomy, so the metric vector localizes the failure automatically. **Context recall**: did retrieval find the information needed? Low recall means a retrieval miss — go back to Module 4. **Context precision**: is the needed information ranked highly within what was retrieved? Low precision with adequate recall is a ranking failure — reranking is the fix. **Faithfulness**: is every claim in the answer supported by the provided context? Low faithfulness with good context is generation drift — a grounding and prompt problem. **Answer relevance**: does the answer address the question that was asked? Cover how these are computed with the LLM-as-judge machinery from Lesson 3.4, which means the calibration discipline applies here too — RAGAS-style metrics are judge outputs and inherit judge bias. Close with the diagnostic table that maps every metric combination to a failure mode and a fix.

**Example.** Four metric vectors from four broken pipelines, each read as a diagnosis before the cause is revealed — then the cause revealed, matching. This is the exact skill an AI system design interview probes.

**Practice.** Students implement the four metrics in `spine/eval/rag_metrics.py`, add them to the Module 3 CI suite with per-stratum reporting, and produce the full before/after report for Module 5 — showing which of their changes moved which metric. They also re-run their Lesson 3.4 judge calibration on the new metrics, since these are judge-based.

**Summary.** Context recall points at retrieval, context precision at ranking, faithfulness at generation. The metric vector is a diagnosis, and reading it out loud is an interview skill.

**Key terms:** faithfulness, context precision, context recall, answer relevance, RAGAS, judge-based metric, metric vector, diagnostic reading.
**Visual aid:** The diagnostic matrix — recall high/low × precision high/low × faithfulness high/low → named failure mode → the lesson that fixes it.
**Hook:** "Your RAG system is production-grade and it can only do one thing: look something up. Module 6 gives it the ability to act — and everything gets harder."

---

## Spine Increment

**Deliverable:** `spine/rag/` — conversational query rewriting, multi-query retrieval, cross-encoder reranking funnel, deliberate context assembly, grounded answers with validated citations and refusal, and a four-metric RAG eval suite wired into CI.

**Acceptance criteria:**
1. `diagnose.py` localizes any failing query to one of the four failure modes.
2. Each retrieval technique is measured independently with its recall gain, latency, and cost recorded; at least one is rejected on cost.
3. Citations are validated against the retrieved set — a fabricated citation cannot pass the contract.
4. Refusal works on the out-of-scope stratum, with both missed-refusal and false-refusal rates reported.
5. Faithfulness, context precision, context recall, and answer relevance run per stratum in CI; `eval-report.md` shows the full Module 5 before/after.

---

## Standalone Exercise

A supplied RAG pipeline with three seeded defects — one per failure mode. Students diagnose each **from the metrics alone**, before reading any code, write down their diagnosis, then fix and verify. Deliverable: `exercises/m5-rag-triage/` with the predicted diagnoses, the actual causes, and an honest note on any they got wrong.

---

## Dependencies

| Lesson | Requires |
|---|---|
| 5.1 | Modules 3, 4 |
| 5.2 | 5.1 (fix retrieval misses first) |
| 5.3 | 5.2 (rerank what rewriting retrieved) |
| 5.4 | 5.3, plus Module 2 contract |
| 5.5 | 5.1–5.4 (measures all of them), plus Lesson 3.4 judge |

**Downstream:** Module 6 makes retrieval one tool among several. Module 10 caches and optimizes this pipeline. Module 11 treats retrieved chunks as an untrusted injection surface — a document in the corpus can carry instructions.

---

## Assessment Focus (Phase 4)

Almost entirely diagnostic, mirroring the AI system design interview round. Given a metric vector, name the failure mode and the fix. Given a conversational query that retrieves nothing, identify the missing rewrite step. Given a reranker that improved precision but blew the latency budget, propose three responses. Given an answer citing a document that was not retrieved, name the missing validation. One practical: raise faithfulness on a supplied pipeline without lowering answer relevance.
