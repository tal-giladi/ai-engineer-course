# Lesson 10.3 — Caching, Batching, and the Latency Budget

**Module 10: Serving, Deployment & Performance** · ~12 min
**Objective:** M10-LO3 — Meet a stated latency and cost SLO using caching, batching, routing, and context trimming — and show the measurement.

---

## Introduction

Optimization without measurement is the most common way to waste a week in this field. Someone decides the model is slow, switches to a faster one, loses two points of quality, and discovers the p95 barely moved — because 60% of the latency was in two sequential reranker calls nobody had looked at.

This lesson treats optimization as **budget allocation**: decompose where the time and money actually go, then spend deliberately, and re-run the eval suite after every single change. That last part is not optional. Several of the techniques here quietly cost quality, and one of them can return a confidently wrong answer.

---

## Key Concept 1: Decompose the budget first

Before optimizing anything, break down a real request from traces (Lesson 10.4 builds this properly; a stopwatch around each stage works today):

```
Total p95: 8,400 ms
  query rewrite (1 model call)          420 ms    5%
  embedding (3 queries)                 180 ms    2%
  vector search                          40 ms    0.5%
  BM25 search                            25 ms    0.3%
  rerank (2 sequential calls)         2,900 ms   35%   ←
  context assembly                       15 ms    0.2%
  generation                          2,400 ms   29%
  citation validation (1 model call)  2,420 ms   29%   ←
```

Two stages nobody suspected account for 64%: the reranker running twice sequentially instead of once, and a citation-validation call on every request that could run on a sample.

**Amdahl's law applies with full force.** Making generation 30% faster improves total latency by 9%. Fixing the double rerank improves it by 17%, for free, by deleting a line.

Do the same for cost, because the distributions differ — cheap-but-slow and fast-but-expensive stages are both common.

---

## Key Concept 2: Caching at four levels

**Exact-match response cache.** Hash the full input, cache the output. Trivially correct, low hit rate on natural language — people rarely phrase things identically. Worth having; do not expect much.

**Semantic cache.** Embed the query, return a cached answer if a previous query is within a similarity threshold. High hit rates — 20–40% on real traffic — and **the one genuinely dangerous optimization in this lesson.**

The danger: two queries can be semantically close and require different answers.

> *"What's the rate limit for batch endpoints?"*
> *"What's the rate limit for streaming endpoints?"*

Cosine similarity around 0.94. At a threshold of 0.92, the second query gets the first one's answer — confidently, with citations, and wrong. There is no error, no log line, and nothing distinguishes it from a correct response.

Mitigations: set the threshold from measurement rather than a default (see the worked example), scope cache keys by any filter that affects the answer (**and by tenant** — Lesson 11.3 covers the cross-tenant leak this prevents), set a TTL matched to how often your corpus changes, and validate a sample of cache hits against fresh generations continuously.

**Embedding cache.** Text-to-vector is deterministic per model. Cache it. Free, no correctness risk, always worth doing.

**Provider prefix caching.** From Lesson 2.4 — order prompts stable-first. Already done, and it applies to tool definitions and few-shot blocks too.

---

## Key Concept 3: Batching, routing, trimming

**Batching** is a large win for offline and bulk paths — document ingestion, backfills, evaluation runs — where you can submit many requests together. For interactive requests it *adds* latency, because you wait to fill a batch. Batch the bulk paths; leave the interactive path alone.

**Routing** sends the common case to a small or fine-tuned model and escalates the rest. This is Lesson 9.3's pattern at the system level, and it is usually the largest cost lever available. Route on query complexity, on the student model's confidence, or on stakes.

**Context trimming** reduces prefill directly. Fewer retrieved chunks, shorter conversation history, tighter tool results. Lesson 5.3 already proved more context is not better, so this frequently improves quality *and* cost *and* latency simultaneously — and Lesson 10.2 added a fourth benefit: it multiplies your serving capacity.

**Parallelizing independent stages.** The double rerank above is a special case: stages that do not depend on each other should not be sequential. Free latency, no quality risk.

---

## Key Concept 4: Gate every change on the eval suite

Each technique here can cost quality:

| Technique | How it can cost quality |
|---|---|
| Semantic cache | Wrong answer from a near-miss |
| Routing | Small model handles a case it should have escalated |
| Context trimming | Drops a chunk that was needed |
| Model swap | Straightforwardly worse |
| Reduced reranking | Precision drops |

So: **one change at a time, full eval suite after each, revert what costs more than it saves.** This is the Lesson 3.1 discipline applied to performance work, and it is what makes the resulting eval report credible.

Expect to revert at least one change. That entry in your report — "we tried X, it cut latency 18% and cost 3 points of faithfulness, we reverted" — is one of the strongest things you can show a hiring panel.

---

## Worked Example: 8.4 s to 2.9 s

Starting point: p95 8,400 ms, $0.0094/request, quality 0.87. **SLO: p95 under 3,000 ms, cost under $0.004, quality not below 0.85.**

**Change 1 — parallelize the double rerank.** The two reranker calls were independent and running sequentially.

| | p95 | Cost | Quality |
|---|---|---|---|
| Before | 8,400 ms | $0.0094 | 0.87 |
| After | 6,950 ms | $0.0094 | 0.87 |

−17% latency, zero risk, one line. Always start here.

**Change 2 — citation validation on a 10% sample** instead of every request.

| | 5,150 ms | $0.0072 | 0.87 |

−26%, −23% cost. Fabricated citations are still *caught* by the four-line mechanical check from Lesson 5.4; only the expensive model-based support verification moved to sampling.

**Change 3 — embedding cache.**

| | 4,980 ms | $0.0070 | 0.87 |

Small, free, permanent.

**Change 4 — semantic cache at threshold 0.90.**

| | 2,240 ms | $0.0031 | **0.79** ← |

Enormous latency and cost win. **Quality dropped 8 points**, below the SLO floor. The eval report shows the failures concentrated in one stratum: near-miss queries, exactly as predicted.

**Change 4b — threshold swept:**

| Threshold | Hit rate | p95 | Cost | Quality |
|---|---|---|---|---|
| 0.90 | 41% | 2,240 ms | $0.0031 | 0.79 |
| 0.94 | 28% | 2,760 ms | $0.0042 | 0.84 |
| **0.97** | **19%** | **3,180 ms** | **$0.0051** | **0.87** |
| 0.99 | 6% | 4,600 ms | $0.0066 | 0.87 |

At 0.97, quality is fully preserved with a 19% hit rate. Chosen — and this sweep is the difference between a cache that works and one that ships wrong answers.

**Change 5 — route simple factual queries to the fine-tuned 8B model** (Module 9), escalating on low confidence.

| | **2,890 ms** | **$0.0026** | **0.86** |

**SLO met.** p95 2,890 ms, cost $0.0026, quality 0.86.

**Change 6 — trim retrieval from 5 chunks to 3.** Tried, and reverted:

| | 2,510 ms | $0.0021 | **0.81** |

−13% latency and −19% cost for 5 points of quality, concentrated in the multi-hop stratum. Below the floor. **Reverted.**

**Final:** 8,400 → 2,890 ms (−66%), $0.0094 → $0.0026 (−72%), quality 0.87 → 0.86 (within the noise band from Lesson 3.4).

The two cheapest wins — parallelizing and sampling — delivered 40% of the latency reduction for essentially no risk, and neither was visible without the budget decomposition.

---

## Practice

**1. State an SLO.** Explicit numbers for p95 latency, cost per request, and a quality floor. Derive them from your Lesson 3.1 charter, and write them in `docs/slo.md`. **Without a stated target, optimization has no stopping condition.**

**2. Decompose the budget.** Per-stage latency and cost for a representative request set. Find your equivalent of the double rerank — there is usually one.

**3. Start with the free wins.** Parallelize independent stages, sample expensive verification, cache embeddings. No quality risk; measure anyway.

**4. Add a semantic cache and sweep the threshold.** Four values minimum, reporting hit rate, latency, cost, **and quality per stratum**. Pick from the curve. Scope cache keys by tenant and by any filter that affects the answer.

**5. Add routing** using your Module 9 model and an escalation signal. Sweep the threshold as in Lesson 9.3.

**6. Try context trimming** and measure honestly. It may improve quality; it may not.

**7. Run the full eval suite after every single change,** and record latency, cost, and per-stratum quality each time.

**8. Revert at least one change** and record why. If everything you tried helped, try something more aggressive until something does not.

**9. Append the full table to `eval-report.md`** — every change, its effect on all three axes, and the reverted ones. This section is a portfolio centerpiece.

---

## Visual Aid

*A stacked horizontal bar chart, one bar per optimization step, running top to bottom. Each bar is segmented by pipeline stage (rewrite, embed, search, rerank, generate, validate) in consistent colours, so segments visibly shrink or vanish down the sequence. To the right of each bar: cost and quality score. The "semantic cache @ 0.90" bar is dramatically shorter but its quality figure is boxed in red at 0.79; the next bar, "@ 0.97," is slightly longer with quality restored to 0.87 in green. The final "trim to 3 chunks" bar is shortest of all, greyed out and struck through, annotated "REVERTED — quality 0.81, below floor." A vertical dashed line marks the 3,000 ms SLO, with the last non-reverted bar just inside it.*

---

## Key Takeaways

- **Decompose the budget before optimizing.** The largest stage is frequently one nobody suspected, and Amdahl's law means optimizing the wrong stage is nearly wasted effort.
- **Start with the free wins:** parallelize independent stages, sample expensive verification, cache embeddings. In the worked example these delivered 40% of the improvement at zero quality risk.
- **The semantic cache is the one dangerous optimization** — near-miss queries get confidently wrong cached answers with no error. Sweep the threshold against per-stratum quality, scope keys by tenant and filter, and validate a sample of hits continuously.
- **Batch the bulk paths, route the interactive ones, trim context everywhere.** Trimming often improves quality, cost, latency, and serving capacity at once.
- **State an SLO, change one thing at a time, run the full eval after each, and revert what costs more than it saves.** The reverted entry is the most credible line in your eval report.

---

## Next

Every optimization in this lesson depended on knowing where the time and money went. You got that from a stopwatch, which does not scale past a handful of requests and tells you nothing about what happened inside a fifteen-step agent run in production.

**Lesson 10.4** builds proper tracing — and the correlation ID you added back in Lesson 1.4, which has been threading quietly through every module since, finally becomes the thing that makes an entire agent run legible in one view.
