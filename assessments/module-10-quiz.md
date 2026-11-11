# Module 10 Quiz — Serving, Deployment & Performance

**10 questions · ~37 min plus the practical**

This module maps most directly onto the interview deep-dive, so the questions are framed the way that round frames them.

---

## Q1 — Diagnosis

A service handles 5 concurrent users at p95 9.1 s, and collapses at 20 concurrent (p95 61 s, 12% errors). CPU utilization is 14%.

(a) What is the most likely cause, and why does low CPU point to it?
(b) Name three specific places to look.

---

## Q2 — Short answer

Twenty percent of clients disconnect roughly three seconds into a nine-second agent run.

(a) What does this cost you, and why is it invisible in a standard load-test report?
(b) Cancellation must propagate through three layers. Name them and state what each stops.

---

## Q3 — Diagnosis

Latency budget from traces:

```
Total p95: 8,400 ms
  query rewrite            420 ms    5%
  embedding                180 ms    2%
  vector search             40 ms    0.5%
  rerank (2 sequential)  2,900 ms   35%
  generation             2,400 ms   29%
  citation validation    2,420 ms   29%
```

(a) Rank the three highest-return optimizations and state the risk of each.
(b) A colleague proposes switching to a faster model to cut generation time by 30%. What is the resulting improvement to total p95, and what does that illustrate?

---

## Q4 — Applied judgment

Semantic cache threshold sweep:

| Threshold | Hit rate | p95 | Cost | Quality |
|---|---|---|---|---|
| 0.90 | 41% | 2,240 ms | $0.0031 | 0.79 |
| 0.94 | 28% | 2,760 ms | $0.0042 | 0.84 |
| 0.97 | 19% | 3,180 ms | $0.0051 | 0.87 |
| 0.99 | 6% | 4,600 ms | $0.0066 | 0.87 |

Baseline quality is 0.87.

(a) Which threshold, and why?
(b) At 0.90, what is happening mechanically? Give a concrete two-query example.
(c) Why is this failure mode more dangerous than a cache miss?

---

## Q5 — Design

You are sizing a vLLM deployment. Your measured context distribution is:

```
p50: 2,100 tokens   p95: 5,800   p99: 7,900   max: 12,400
```

Capacity measurements:

| `--max-model-len` | Max concurrent | Throughput |
|---|---|---|
| 4,096 | 96 | 61 req/s |
| 8,192 | 48 | 41 req/s |
| 16,384 | 23 | 26 req/s |
| 32,768 | 11 | 14 req/s |

(a) What do you set, and why not 32,768 "to be safe"?
(b) What do you do with the requests above your setting?
(c) State the general relationship this table demonstrates, and one architectural consequence beyond serving.

---

## Q6 — Diagnosis

A trace shows this inside one tool call:

```
tool: search_docs                         2,900 ms
  ├─ embed                                   60 ms
  ├─ vector_search                           38 ms
  ├─ vector_search                           41 ms
  ├─ vector_search  (×8 total)
  └─ rerank                               2,100 ms
```

(a) What is wrong?
(b) Why would logs not have surfaced this?
(c) Name two plausible causes.

---

## Q7 — Applied judgment

Offline versus online evaluation:

| | Offline | Online (10% sample) | Gap |
|---|---|---|---|
| Faithfulness | 0.87 | 0.79 | −0.08 |
| Refusal accuracy | 0.86 | 0.61 | **−0.25** |

(a) What does the refusal gap most likely mean?
(b) Name the two fixes, and say which is the more important one.
(c) What does this say about the relationship between offline and online evaluation generally?

---

## Q8 — Design

A CI run reports:

```
EVAL GATE: PASSED    Overall 0.871 → 0.869 (within band)
COST GATE: FAILED    $0.0026 → $0.0071 (+173%)
  By call_site:  validate  $0.0006 → $0.0051  (+750%)
```

(a) Why does gating on quality alone fail here?
(b) Which telemetry field located the problem, and from which module?
(c) What would this look like without the cost gate?

---

## Q9 — Multiple choice

A 25-minute full eval of experiment branch `exp/rerank-k8` is running in your only checkout when a production bug on `main` needs a fix now. You also want a coding agent to try a second experiment at the same time.

What is the best setup?

A. Stash the experiment, switch to `main` for the fix, then switch back and restart the eval from the beginning
B. Clone the repository twice more so each task has a separate copy with its own history and remotes
C. Add a worktree per branch: fix `main` in one, give the agent its own; recreate `.env` and separate the stacks
D. Check out `main` in a second worktree while the first one also stays on `main`, then cherry-pick the fix

---

## Q10 — Practical

Take the supplied slow agent endpoint in `exercises/m10-latency-halving/` and cut p95 latency by half without losing eval score.

**Deliverable:** trace-based budget decomposition, each optimization with its measured effect on latency, cost, and quality, and a note on any change you had to revert.

---
---

# Answer Key

## Q1

**(a) A blocking call in the request path.** Low CPU with collapsing latency is the signature: the workers are not doing work, they are *waiting* while holding an execution slot. One synchronous call serializes an entire worker regardless of how many requests arrive.

**(b) Three places:**
- **A synchronous database driver** in the retrieval path — the most common culprit
- **A tool from Module 6 using a synchronous HTTP library** — easy to miss because tools were written independently
- **CPU-bound work on the event loop** — a local cross-encoder reranker or tokenization of a large document, which should be on a thread pool

**Note:** the model call is usually *not* the culprit, which is why teams look in the wrong place first.

## Q2

**(a)** You continue paying for tokens for the remaining six seconds of a run nobody will read — roughly 20% of spend producing nothing.

**It is invisible in standard load-test reports** because those measure latency, throughput, and error rate. A cancelled request is not an error and does not appear in latency percentiles. You have to instrument spend-on-cancelled-requests deliberately, which almost nobody does.

**(b) Three layers:**
1. **The agent loop** — stops taking further steps (`CANCELLED` termination reason from Lesson 6.4)
2. **The in-flight model call** — closing the HTTP connection to the provider is what actually stops the meter
3. **Telemetry** — record the partial spend, or cost-per-successful-request is wrong

Stopping at layer 1 is the common half-fix: the agent stops, and the model call already in flight generates to completion and bills you.

## Q3

**(a) Three highest-return:**

1. **Parallelize the two sequential rerank calls** — they are independent. **~1,450 ms saved, zero quality risk, one line.** Always start with free wins.
2. **Sample citation validation at 10%** instead of every request — **~2,180 ms and 23% cost saved.** Low risk: the four-line mechanical citation check (Lesson 5.4) still runs on every request; only the expensive model-based support verification is sampled.
3. **Semantic caching** — large win, **real quality risk**, requires a threshold sweep (Q4).

**(b)** Generation is 29% of the budget; a 30% improvement to it yields **8.7% of total p95** — roughly 730 ms, taking 8,400 to ~7,670.

**This illustrates Amdahl's law.** Optimizing the most *visible* stage rather than the largest one is the most common way to waste a week in this field, and it is why budget decomposition precedes optimization.

## Q4

**(a) 0.97.** It preserves baseline quality exactly (0.87) with a 19% hit rate, meaningful cost and latency savings, and no correctness risk. 0.90 buys speed by returning wrong answers — 8 points below the floor. 0.99 gives up most of the benefit for no additional quality.

**(b)** At 0.90, semantically-similar-but-materially-different queries hit the same cache entry:

> *"What's the rate limit for batch endpoints?"* → cached
> *"What's the rate limit for streaming endpoints?"* → similarity ~0.94 → **cache hit**

The second user receives the batch endpoint's limits, presented confidently, with citations.

**(c)** A cache *miss* costs latency and money and produces a correct answer. A **false cache hit** produces a confidently wrong answer with valid-looking citations, no error, no log line, and nothing in the trace distinguishing it from a correct response — because the cache hit *is* the response. It is undetectable without deliberately validating a sample of hits against fresh generations.

## Q5

**(a) 8,192.** It covers p99 (7,900) with headroom and gives 48 concurrent sequences at 41 req/s.

**Not 32,768,** because capacity is KV-cache-bound and roughly inversely proportional to maximum context. Setting 32,768 gives 11 concurrent instead of 48 — **four times less capacity** — to accommodate the 0.1% of requests between 12,400 and 32,768 tokens. That is paying a 4× capacity penalty for a rounding error's worth of traffic.

**(b)** Route them to a hosted profile through the Lesson 1.2 seam. One branch of routing logic, and the tail is handled without distorting the whole deployment.

**(c) The relationship: doubling maximum context roughly halves concurrency**, because KV-cache memory per sequence scales with context length and total KV-cache memory is fixed.

**Architectural consequence beyond serving:** trimming retrieved context (Lesson 5.3) is not only a quality and cost decision — it **multiplies your serving capacity**. A change from 5 chunks to 3 that holds quality is simultaneously a latency win, a cost win, and a throughput win.

## Q6

**(a)** Eight vector searches inside a single tool call, followed by a rerank over the union of all eight. The tool is executing its retrieval path eight times.

**(b)** Because every individual operation **succeeded, and succeeded quickly** — 38–41 ms each. Logs record events, not structure. There is nothing anomalous about a fast successful vector search; the anomaly is entirely in the *relationship* between eight of them, which only a span tree represents.

**(c) Two plausible causes:**
- **A cache keyed on the wrong value** — e.g. keyed on the original query while the code searches with rewritten queries, so every request is a miss and re-runs the full multi-query expansion
- **Layered retries** — a retry wrapper re-invoking the whole tool, compounding with the multi-query expansion inside it (the Lesson 7.1 retry-layering hazard)

## Q7

**(a)** The golden set's out-of-scope stratum does not represent what production users actually ask out-of-scope. Typically every out-of-scope example is a *technical* question, while real users ask about pricing, billing, contracts, and account specifics — categories the corpus does not cover and the eval set never tested.

**(b) Two fixes:**
- **Immediate:** a scope classifier routing the missing topic area to a refusal
- **Structural:** add real production out-of-scope questions to the golden set as a new sub-stratum

**The structural fix is more important**, because without it the gap silently reappears the next time the input distribution shifts, and nothing will catch it.

**(c)** Your offline set is a **guess about reality**, and the offline/online gap is a measurement of how good the guess is. The gap is not a nuisance to be minimized — it is a finding that tells you specifically what your evaluation is missing. Production failures flowing back into the golden set is what keeps an evaluation practice honest over years.

## Q8

**(a)** The change did not harm quality — it harmed **cost**, tripling it. A quality-only gate has no visibility into this and passes it. The regression then surfaces weeks later as an unexplained bill, with no obvious change to attribute it to.

**(b) `call_site`**, added in **Lesson 1.4**. The breakdown localizes the regression to the `validate` stage in about ten seconds, which converts an investigation into a lookup.

**(c)** Without it: the PR merges green, spend rises 173%, and someone notices at the end of the billing period. The investigation then has to reconstruct which of several weeks of merges caused it — the exact "week-eight surprise" scenario Lesson 1.4 opens with.

## Q9 — **C**

Worktrees give one directory per branch from a single repository, so the running eval is never disturbed, the fix happens in parallel, and the agent's edits cannot collide with yours. Each worktree is a fresh checkout, so gitignored files such as `.env` must be recreated and each Compose stack needs its own project name and ports.

**A** throws away the running eval and changes files under a running process. **B** works but duplicates history and remotes and drifts out of sync, which worktrees avoid. **D** fails because Git refuses to check out the same branch in two worktrees.

## Q10 — Practical rubric

| Criterion | Full credit |
|---|---|
| SLO stated first | Explicit p95, cost, and quality-floor targets before any optimization |
| Budget decomposed | Per-stage latency and cost from traces, not guesses |
| Free wins first | Parallelization and sampling attempted before risky changes |
| One change at a time | Full eval suite run after each, with all three axes recorded |
| Threshold swept | Any cache or routing threshold chosen from a swept curve, per stratum |
| **At least one revert** | A change measured, found to cost quality, and reverted with reasoning |
| Target met | p95 halved, eval score within the noise band |

**The revert is required for full credit.** If every change helped, the student did not push hard enough — and the reverted entry is the single most credible line in an eval report.
