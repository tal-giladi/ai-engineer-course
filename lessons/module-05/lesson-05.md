---
id: "05.5"
module: 5
minutes: 13
practice_minutes: 90
prerequisites: ["05.4"]
objectives:
  - "Evaluate a RAG pipeline on faithfulness, context precision, and context recall, and read those numbers as a diagnosis."
volatility: implementation
sources: []
last_verified: "2026-08-24"
---

# 05.5 · Measuring RAG: Faithfulness, Context Precision, Context Recall

RAGAS and the metric names in this lesson appear by name in job postings, which is a reason to know them. It is not the reason they matter.

They matter because **together they localize a failure automatically**. Lesson 5.1 taught you to classify a wrong answer by tracing it through the pipeline, one query at a time. These four metrics do the same classification across your whole evaluation set, in aggregate, without you looking at a single trace.

That turns "the RAG is wrong" into "context recall is 0.94, precision is 0.41 — this is a ranking problem" in about ten seconds. Being able to say that sentence out loud, from a table, is one of the highest-value interview skills in this course.

---

## Key Concept 1: The four metrics

Each isolates one stage of the Lesson 5.1 pipeline.

**Context recall — did retrieval find what was needed?**

Of the information required to answer correctly, how much appears somewhere in the retrieved context? Computed by decomposing the reference answer into claims and checking each against the retrieved chunks.

This is the **ceiling on everything downstream**. What retrieval misses, generation cannot use. Low context recall means stop reading the other three metrics and go back to Module 4.

**Context precision — is the needed information ranked highly?**

Of the retrieved chunks, how many are actually relevant, and are the relevant ones near the top? Position-aware: a relevant chunk at rank 1 counts for more than the same chunk at rank 9, because of the attention effects from Lesson 5.3.

Low precision with adequate recall is the signature of a **ranking failure**. You retrieved it; you buried it.

**Faithfulness — is every claim supported by the context?**

Decompose the generated answer into individual claims and check each against the retrieved context. A claim that is *true in the world* but *unsupported by the context* scores as unfaithful — and that is deliberate. An unsupported claim is a hallucination that happened to be right, and it will not be right next time.

Low faithfulness with good context is **generation drift**.

**Answer relevance — does the answer address the question?**

Independent of correctness. An answer can be perfectly faithful and answer a different question, which is a common failure with over-aggressive query rewriting from Lesson 5.2.

---

## Key Concept 2: These are judge outputs, and inherit judge problems

This is where most implementations go wrong, and it is worth being blunt about.

Every one of these metrics is computed by a model: decomposing an answer into claims is a model call, deciding whether a chunk supports a claim is a model call. RAGAS and its equivalents are convenience wrappers around LLM-as-judge.

Which means **everything from Lesson 3.4 applies unchanged**:

- **They need calibration.** An uncalibrated faithfulness score is a number with unknown meaning. Hand-label 30 answers for faithfulness, run the metric, compute agreement, and state a trust boundary.
- **They inherit bias.** Verbosity bias affects claim decomposition — longer answers produce more claims, which changes the denominator in ways that are not obviously neutral.
- **They cost money and time.** Four metrics × several judge calls each × 60 items is a real per-run bill, which is why your Lesson 3.5 smoke tier exists.
- **Rubric drift applies.** Change the claim-decomposition prompt and all previous scores are on a different scale. Version the prompts.

Importing a library does not exempt you from calibration. If anything it makes calibration more important, because the library's defaults were tuned on somebody else's data.

---

## Key Concept 3: The diagnostic matrix

Here is the payoff. Read the vector, get the diagnosis:

| Context recall | Context precision | Faithfulness | Diagnosis | Fix |
|---|---|---|---|---|
| **Low** | any | any | Retrieval miss | Chunking, embeddings, hybrid (Module 4), query rewriting (5.2) |
| High | **Low** | any | Ranking failure | Reranking (5.3) |
| High | High | **Low** | Generation drift | Grounding, citation validation (5.4) |
| High | High | High + **low answer relevance** | Answering the wrong question | Check query rewriting (5.2) |
| High | High | High | Working. Look at cost and latency. | — |

Two rules for reading it:

**Read top-down, and stop at the first low value.** Downstream metrics are uninterpretable when an upstream stage failed. Faithfulness on a context that never contained the answer is meaningless — the model had nothing to be faithful to.

**Read per stratum, never in aggregate.** A system averaging 0.85 faithfulness might be 0.95 on simple factual and 0.40 on out-of-scope, and the out-of-scope number is the one that matters. This is the Lesson 3.3 discipline, and it applies with full force here.

---

## Worked Example: Four vectors, four diagnoses

Four broken pipelines. Read each vector *before* reading the cause — this is exactly the exercise an interviewer runs.

**Pipeline A**

```
context recall     0.42     faithfulness      0.91
context precision  0.88     answer relevance  0.86
```

Stop at recall. Retrieval is not finding the material. Note the trap: faithfulness is 0.91, which looks excellent — the model is being faithful to a context that lacks the answer, so it is faithfully producing an incomplete or refusing response. **Diagnosis: retrieval miss.**
*Actual cause: dense-only retrieval on a corpus full of error codes and function names. Fix: hybrid search. Recall 0.42 → 0.89.*

**Pipeline B**

```
context recall     0.94     faithfulness      0.62
context precision  0.38     answer relevance  0.71
```

Recall is high — the material is being retrieved. Precision is low — it is buried. Faithfulness suffers downstream because the relevant chunk is at rank 9 of 12 and the model is working from the wrong material. **Diagnosis: ranking failure.**
*Actual cause: no reranker, k=12 taken straight from the bi-encoder. Fix: retrieve 50 → rerank 5. Precision 0.38 → 0.81, faithfulness 0.62 → 0.88.*

**Pipeline C**

```
context recall     0.91     faithfulness      0.44
context precision  0.86     answer relevance  0.89
```

Retrieval and ranking both good. The model has what it needs and is not using it. **Diagnosis: generation drift.**
*Actual cause: no grounding structure and no citation validation. The model was answering from training knowledge about a previous API version. Fix: claims-with-support contract plus mechanical citation validation. Faithfulness 0.44 → 0.87.*

**Pipeline D**

```
context recall     0.93     faithfulness      0.90
context precision  0.84     answer relevance  0.51
```

Everything sound except relevance. The answers are well-supported and about something else. **Diagnosis: the query being retrieved for is not the query being asked.**
*Actual cause: over-aggressive multi-query rewriting. Paraphrase 3 drifted to an adjacent topic, RRF ranked its results highly, and the answer addressed the paraphrase. Fix: constrain the rewriter and validate that rewrites preserve intent. Relevance 0.51 → 0.88.*

**Now per stratum**, on the fixed pipeline:

| Stratum | Recall | Precision | Faithfulness | Relevance |
|---|---|---|---|---|
| simple_factual | 0.96 | 0.89 | 0.94 | 0.93 |
| multi_hop | 0.88 | 0.79 | 0.86 | 0.88 |
| ambiguous | 0.84 | 0.76 | 0.85 | 0.79 |
| **out_of_scope** | n/a | n/a | **0.61** | 0.90 |
| long_context | 0.87 | 0.71 | 0.82 | 0.86 |
| adversarial | 0.81 | 0.74 | 0.88 | 0.84 |
| **Overall** | **0.89** | **0.80** | **0.86** | **0.87** |

The aggregate looks healthy. Out-of-scope faithfulness at 0.61 says the system is still occasionally answering questions it should refuse — the Lesson 5.4 problem, still not fully solved. Note also that recall and precision are `n/a` for out-of-scope: there is no relevant context to retrieve, so only faithfulness and refusal behavior are meaningful there. Metrics that do not apply should be reported as not applicable, never as zero.

---

## Practice

**1. Implement the four metrics** in `spine/eval/rag_metrics.py`. Use a RAGAS-style library or hand-roll them — hand-rolling claim decomposition once is genuinely instructive and takes about thirty minutes. Version the prompts either way.

**2. Calibrate.** Hand-label 30 answers for faithfulness against your own definition. Run the metric. Compute agreement, read the disagreements, revise, and record the trust boundary in `spine/eval/judge.md` alongside your Lesson 3.4 judge.

**3. Handle non-applicable strata.** Out-of-scope items have no relevant context; report recall and precision as `n/a`, not `0.0`. A zero will drag your aggregate and hide a healthy system.

**4. Wire into CI.** Add to your Lesson 3.5 full tier with per-stratum reporting. Watch the cost — cache judge verdicts keyed on (answer, rubric version).

**5. Build the diagnostic table** as a runnable command: `make diagnose` prints the four-metric vector per stratum and the matrix reading for each. You want this to be a reflex.

**6. Write the Module 5 report.** Append to `eval-report.md` with each Module 5 change and which metric it moved:

| Change | Recall | Precision | Faithfulness | Relevance | Cost | p95 |
|---|---|---|---|---|---|---|
| Module 4 baseline | | | | | | |
| + conversational rewrite | | | | | | |
| + decomposition | | | | | | |
| + reranking | | | | | | |
| + assembly (dedup/order/expand) | | | | | | |
| + grounding & citation validation | | | | | | |

Each row should move the metric its lesson predicted. **Where it does not, that is the interesting finding** — investigate it and write down what you learn.

**7. Practise reading vectors aloud.** Take four rows from your own table, cover the change column, and state the diagnosis from the numbers. This is the interview drill.

---

## Visual Aid

*A decision-tree flowchart used as a diagnostic. Root: "context recall < threshold?" → YES branch to a red box "RETRIEVAL MISS — Module 4, Lesson 5.2" with a note "stop here; downstream metrics uninterpretable." NO branch → "context precision < threshold?" → YES to "RANKING FAILURE — Lesson 5.3." NO → "faithfulness < threshold?" → YES to "GENERATION DRIFT — Lesson 5.4." NO → "answer relevance < threshold?" → YES to "WRONG QUESTION — check rewriting, Lesson 5.2." NO → green box "WORKING — optimize cost and latency, Module 10." Down the right-hand side, a vertical reminder bar: "Read per stratum. The aggregate hides the stratum that matters."*

---

## Key Takeaways

- **The four metrics are a diagnostic instrument, not a scorecard.** Recall points at retrieval, precision at ranking, faithfulness at generation, relevance at the query.
- **Read the vector top-down and stop at the first low value.** Downstream metrics are uninterpretable when an upstream stage failed — high faithfulness on an empty context means nothing.
- **These are judge outputs.** They need calibration, they inherit verbosity bias, they cost money, and importing a library does not exempt you from any of that.
- **Report per stratum, and report non-applicable as `n/a`.** Out-of-scope items have no relevant context; scoring them zero hides a healthy system.
- **Reading a metric vector aloud as a diagnosis is an interview skill.** Practise it on your own numbers until it is automatic.

---

## Next

Your RAG system is production-grade: measured, diagnosable, grounded, cited, and capable of refusing. It can also do exactly one thing — look something up and answer.

It cannot check whether a service is currently down. It cannot run a calculation. It cannot decide that this question needs three lookups instead of one, or notice that its first search returned nothing useful and try a different angle.

**Module 6** gives it the ability to act: tool calling, a control loop you write by hand, state and memory across long tasks, recovery when tools fail, and sandboxing for the moment its output becomes an action. Everything gets harder, and it is the module interviewers probe hardest.
