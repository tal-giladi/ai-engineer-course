---
id: "04.1"
module: 4
minutes: 13
practice_minutes: 90
prerequisites: ["03.5"]
objectives:
  - "Choose an embedding model on evidence, benchmarking candidates on their own corpus rather than trusting a leaderboard."
volatility: implementation
sources: []
last_verified: "2026-08-24"
---

# 04.1 · Embeddings: Choosing and Benchmarking a Model

Retrieval-augmented generation is the most in-demand skill in AI engineering — it appears in a large majority of applied LLM job postings, and it is the pattern most interview questions circle back to. Everything in the next two modules builds toward a retrieval system you can defend in a system design round.

It starts with the embedding model, which is the component people choose the most casually and revisit the least. The usual process is: open a leaderboard, take the top model that fits, move on. That process is wrong in a specific and measurable way, and this lesson is about the fifteen minutes of work that replaces it.

---

## Key Concept 1: What an embedding actually gives you

An embedding model maps text to a fixed-length vector such that semantically similar texts land near each other. You retrieve by embedding the query, then finding the nearest document vectors by cosine similarity.

The only question that matters is narrower than "is this a good model": **do the queries your users write land near the documents that answer them, in your corpus?** A model can be excellent in general and mediocre at that, and general excellence is what leaderboards measure.

---

## Key Concept 2: The axes that actually move retrieval quality

**Training domain.** A model trained predominantly on general web text carries weaker representations for API documentation, legal text, medical notes, or code. If your corpus is specialized, general-purpose leaderboard rank predicts your results poorly.

**Asymmetric vs symmetric, and the prefix trap.** Symmetric models assume both sides are the same kind of text (sentence similarity). Asymmetric models are trained for short-query-to-long-document retrieval, which is what you are doing. Many asymmetric models require **instruction prefixes** — something like `query: ` on queries and `passage: ` on documents. Omitting them, or applying the same prefix to both, degrades retrieval substantially while producing no error and perfectly plausible-looking results. This is the single most common silent misconfiguration in the whole retrieval stack. Read the model card. Every time.

**Sequence length.** Every embedding model has a maximum input length and **silently truncates** beyond it. A 512-token limit against 800-token chunks means a third of every chunk is invisible to retrieval — and the system works, returns results, and quietly misses anything whose answer sits in the tail. Check this before blaming anything else.

**Dimensionality.** More dimensions means more storage and slower search, and is not a quality proxy — a well-trained 384-dimension model routinely beats a poorly-matched 1536-dimension one. Some newer models support Matryoshka truncation, letting you trade dimensions for quality along a measured curve.

**Multilingual coverage.** If your corpus or your users are not English-only, this dominates everything else on this list.

**Where it runs.** Local models (via `sentence-transformers` or your Hugging Face account) cost nothing per call, keep data in-house, and add GPU or CPU load. Hosted embedding APIs cost per token and add network latency. At the volumes in this course, local is usually right; at production scale, model the cost both ways.

---

## Key Concept 3: Leaderboards are a shortlist, not an answer

Public benchmarks like MTEB are genuinely useful for one thing: reducing hundreds of candidates to three or four worth testing.

They should not decide for you, for three reasons:

- **They average over tasks that are not yours.** A model excelling at clustering and sentence similarity may be unremarkable at asymmetric retrieval over technical documentation.
- **Contamination is a real and growing problem.** Benchmark data leaks into training sets, inflating scores in ways that do not transfer.
- **Your corpus is idiosyncratic.** Your vocabulary, your document lengths, your query style.

The professional move is straightforward: shortlist three from the leaderboard, benchmark them on your own golden set from Module 3, and pick with evidence. You already built everything needed to do this — this is the first payoff of putting evaluation before retrieval.

---

## Key Concept 4: How to benchmark, in fifteen minutes

You need retrieval-specific labels: for each query, which chunk IDs are relevant. Add a `relevant_chunks` field to your golden set items. For 40–60 items this is a manageable pass through your corpus, and it is reusable for the rest of the course.

Then, per candidate model:

1. Embed the corpus (same chunking for all candidates — isolate the variable)
2. Embed the queries with the correct prefix
3. Retrieve top-k
4. Compute **recall@5**, **recall@10**, **MRR** — per stratum

And record the operational numbers too: index size on disk, embedding throughput, and query latency. A model that wins by two points and is four times slower to embed may lose once you are re-indexing a large corpus regularly.

---

## Worked Example: The leaderboard leader places third

Corpus: 4,200 chunks of API documentation. Golden set: 60 queries with relevance labels. Chunking held constant.

| Model | Dim | Max seq | recall@5 | MRR | Index size | Embed throughput |
|---|---|---|---|---|---|---|
| `leaderboard-top-general` | 1024 | 512 | 0.71 | 0.63 | 17 MB | 210 chunks/s |
| `mid-rank-asymmetric` | 768 | 8192 | **0.86** | **0.79** | 13 MB | 340 chunks/s |
| `small-fast` | 384 | 512 | 0.74 | 0.66 | 6 MB | 1,100 chunks/s |

The leaderboard leader placed third on recall. The mid-ranked asymmetric model won by fifteen points, and was also smaller and faster to embed.

**Then the sequence-length bug.** Chunks averaged 780 tokens. Two of the three models cap at 512 — roughly a third of every chunk was silently truncated. Re-run with chunks capped at 480 tokens:

| Model | recall@5 (780-token chunks) | recall@5 (480-token chunks) |
|---|---|---|
| `leaderboard-top-general` | 0.71 | **0.83** (+0.12) |
| `mid-rank-asymmetric` | 0.86 | 0.85 (−0.01) |
| `small-fast` | 0.74 | **0.85** (+0.11) |

Twelve points were being lost to a configuration detail, and the ranking changed materially once it was fixed. Note also that the 8192-token model was unaffected — it was never truncating.

**And the prefix check.** The asymmetric model requires `query: ` and `passage: ` prefixes. Run without them:

| Configuration | recall@5 |
|---|---|
| Correct prefixes | 0.85 |
| No prefixes | 0.72 |
| Same prefix both sides | 0.69 |

Thirteen to sixteen points, from three words of configuration. No error, no warning, entirely plausible results in every case.

**The decision:** `small-fast` at 480-token chunks. It ties the winner on recall, uses less than half the storage, and embeds three times faster — which matters because the corpus re-indexes nightly. That reasoning goes in the decision record.

---

## Practice

**1. Add relevance labels.** Extend your golden set items with `relevant_chunks: [chunk_id, ...]`. Do this for all items — you will use these labels for the rest of the course.

**2. Shortlist three models** from a leaderboard, deliberately spanning the space: one high-ranked general model, one asymmetric retrieval-specialized model, one small fast model. Note each one's max sequence length and required prefixes **before** running anything.

> **If your corpus or your users are not English-only,** add multilingual coverage as a fourth benchmark axis and include at least one multilingual model in the shortlist. This dominates every other axis on this list when it applies, and a model that wins on English recall can be unusable on the rest of your traffic.

**3. Benchmark.** Same chunking across all three. Report recall@5, recall@10, and MRR **per stratum**, plus index size, embedding throughput, and query latency.

**4. Run the truncation check.** Compute your chunk token-length distribution against each model's limit. If any chunks exceed it, re-run that model with chunks inside the limit and report the difference. This is the highest-value fifteen minutes in the lesson.

**5. Run the prefix check.** For any asymmetric model, measure with correct prefixes, without, and with the same prefix on both sides. Report all three.

**6. Decide and record.** Write `docs/adr-embedding-model.md`: candidates, results table, the chosen model, the reasoning (including operational factors, not just recall), and the conditions under which you would revisit. This is your first architecture decision record, and Module 12 reuses it verbatim in the interview deep-dive.

---

## Visual Aid

*A grouped bar chart, recall@5 on the y-axis, three models grouped on the x-axis. For each model, two bars: "chunks exceed max_seq" (hatched) and "chunks within max_seq" (solid). Two of the three models show a large jump between bars; one shows none. Below the chart, a small inset table listing each model's max sequence length beside the corpus's mean chunk length, with the two mismatches circled. Caption: "Two configuration details — sequence limits and instruction prefixes — moved recall more than the choice of model did."*

---

## Key Takeaways

- **The only question is whether your queries land near your documents, in your corpus.** Leaderboards measure something else.
- **Shortlist from the leaderboard, decide on your own golden set.** You built the harness in Module 3 specifically so this is a fifteen-minute job.
- **Check max sequence length first.** Silent truncation of long chunks costs double-digit recall and produces no error.
- **Check instruction prefixes for asymmetric models.** Missing or mismatched prefixes cost another double-digit chunk of recall, equally silently.
- **Decide on operational factors too** — index size, embedding throughput, re-indexing frequency — not on recall alone, and write the decision record while the reasoning is fresh.

---

## Next

You have the best embedding model for your corpus. There is a decision upstream of it that will affect your recall more than that choice did, that almost nobody measures, and that quietly determines the ceiling on everything in Module 5.

**Lesson 4.2** is about chunking: the tension between chunks small enough to embed precisely and large enough to answer from, the strategies that resolve it, and the two techniques that sidestep the tension entirely.
