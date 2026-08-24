# Final Assessment — The Capstone Repository

**The graded artifact is the repository itself**, not a written exam. This is deliberate: the capstone is the thing you will show a hiring panel, and the rubric below is written to mirror what a reviewer actually checks.

---

## The Deliverable

A public repository containing the spine project — a **production agentic RAG service** — built across all twelve modules, plus its documentation.

**Two to three promoted standalone exercises** (Lesson 12.1) may accompany it as separate small repositories. They contribute to the *Range* section only.

---

## How This Is Graded

Six dimensions, 100 points. Weighting reflects hiring signal rather than engineering effort — which is why documentation and evaluation together are worth more than the retrieval and agent implementations combined.

| Dimension | Points |
|---|---|
| 1. Reproducibility and completeness | 15 |
| 2. Evaluation rigor | 25 |
| 3. Retrieval quality | 15 |
| 4. Agent robustness | 15 |
| 5. Production readiness | 15 |
| 6. Security posture | 15 |
| **Total** | **100** |

Plus **up to 10 bonus points** for range and honesty (see below). Maximum 110.

**Pass: 70. Strong: 85+.**

A submission scoring below 70 on *Reproducibility* alone fails regardless of total, because a system a reviewer cannot run cannot be assessed.

---

## 1. Reproducibility and Completeness — 15 points

| Criterion | Points |
|---|---|
| Clean clone → one config file → one command → working system | 5 |
| Under five minutes, one manual step (adding an API key) | 3 |
| CPU-only path exists (`--profile hosted` or equivalent) | 3 |
| Seeded demo corpus so first run does something | 2 |
| Someone other than the author has run it, and this is stated | 2 |

**Auto-fail condition:** setup instructions that do not work as written.

**Verification:** the grader clones into a fresh directory and follows the README literally, changing nothing. Any step requiring knowledge not in the README loses points on the first two criteria.

---

## 2. Evaluation Rigor — 25 points

The heaviest-weighted dimension, because it is the strongest hiring signal and the one most candidate portfolios lack entirely.

| Criterion | Points |
|---|---|
| Golden dataset: 40+ items, documented strata with proportions and rationale | 4 |
| Held-out split recorded by ID; leakage check documented (Modules 3, 8) | 3 |
| Metrics chosen against a written charter; reported **per stratum**, never aggregate-only | 4 |
| LLM judge **calibrated** against human labels, with agreement rate and a stated trust boundary | 5 |
| Two-tier CI gate on quality **and** cost, with thresholds derived from measured noise | 4 |
| `eval-report.md`: baseline plus every change with its measured effect on quality, cost, and latency | 3 |
| **At least one reverted change, with reasoning** | 2 |

**Notes on scoring:**

- The judge calibration line is worth 5 points on its own. An uncalibrated judge scores zero here regardless of how sophisticated the metrics are.
- "Reported per stratum" means the report shows strata. A single overall number scores 1 of 4, not 4.
- The reverted change is worth 2 points and is the single most credible entry in most repositories. If everything the author tried helped, they did not push hard enough.

---

## 3. Retrieval Quality — 15 points

| Criterion | Points |
|---|---|
| Embedding model chosen from a committed benchmark **on the project's own corpus** | 3 |
| Chunking strategy chosen from a measured comparison; contextual-header effect isolated | 3 |
| Vector store decision record with concrete switching conditions | 2 |
| Hybrid retrieval measured against dense-only and sparse-only, reported per query type | 3 |
| Reranking funnel with `retrieve_k` tuned to measured recall saturation | 2 |
| Grounded answers with **mechanically validated** citations and working refusal | 2 |

**Common deductions:** adopting a leaderboard-ranked embedding model without benchmarking (−3); reporting hybrid as an aggregate improvement without showing which query type produced it (−2); citations rendered but not validated against the retrieved set (−2).

---

## 4. Agent Robustness — 15 points

| Criterion | Points |
|---|---|
| Agent loop written by hand first, with the framework port proven by trajectory eval | 3 |
| All five budget dimensions, checked **before** each call | 3 |
| Repeated-action detection with a corrective message permitting refusal | 2 |
| Typed termination reasons; `REFUSED` distinguishable from `BUDGET_EXHAUSTED` | 2 |
| Chaos test results under injected tool failure: completion rate, mean and **p95** cost | 3 |
| Checkpointing demonstrated: approve-and-resume, resume-after-crash, time-travel debug | 2 |

**Notes:** the hand-rolled-first requirement is verified by git history or by the presence of both implementations. `docs/framework-tradeoff.md` naming what the framework hides is required for the first criterion's full marks.

---

## 5. Production Readiness — 15 points

| Criterion | Points |
|---|---|
| Streaming service with typed events and **cancellation propagated to the provider** | 3 |
| A stated SLO, met, with a per-stage latency budget before and after | 3 |
| Optimizations applied one at a time with the eval suite run after each | 2 |
| End-to-end tracing with LLM attributes, keyed on the correlation ID | 3 |
| Online evaluation sampling with an offline/online gap analysis | 2 |
| Docker Compose with health checks, externalized config, CI pipeline | 2 |

**Notes:** cancellation must be demonstrated as reaching the provider, not merely stopping the loop — the wasted-spend measurement (before and after) is the evidence. An SLO stated after the fact to match the achieved numbers scores 1 of 3.

---

## 6. Security Posture — 15 points

| Criterion | Points |
|---|---|
| Threat model naming untrusted input channels, reachable tools, and exfiltration paths | 3 |
| Tool permission matrix: credential scope, argument constraints, rate limit, approval | 3 |
| Egress allowlists and resolved-path checks on every outward-reaching tool | 2 |
| Data inventory across all stores; cache keys scoped by tenant; retention implemented | 3 |
| Red-team report: systematic grid, five categories, findings with status | 2 |
| **Every successful attack converted to a regression test** in CI | 2 |

**Notes:** a red-team report claiming zero findings scores 0 on the last two criteria unless the method demonstrates genuine coverage — a first red-team against a real system essentially always finds something. Accepted risks stated with reasoning score full marks; unstated ones score zero.

---

## Bonus — up to 10 points

| Criterion | Points |
|---|---|
| Two or three promoted standalone exercises, each documented with problem, approach, and measured result | 4 |
| A public write-up of one non-obvious measured finding | 3 |
| An honest negative result reported prominently (e.g. a fine-tune that lost, with the analysis) | 3 |

**On the negative result:** this is bonus rather than core because not every project produces one. Where it exists, it is among the most credible things in a portfolio — direct evidence that the evaluation is real and acted upon rather than decorative.

---

## Documentation Requirements

Documentation is not a separate dimension; it is how most of the above is evidenced. A repository can lose points across five dimensions for one missing artifact.

**Required:**

| Artifact | Evidences |
|---|---|
| `README.md` — seven-section structure, demo GIF, diagram, results table, limitations | Dimensions 1–6 |
| `eval-report.md` — baseline through final, with reverted entries | Dimension 2 |
| `spine/eval/charter.md`, `dataset.md`, `judge.md` | Dimension 2 |
| `docs/adr-*.md` — decision records with revisit conditions | Dimensions 3, 4 |
| Model cards — intended use, out-of-scope use, limitations | Dimensions 2, 4 |
| `docs/threat-model.md`, `security-report.md`, `data-inventory.md` | Dimension 6 |
| `docs/slo.md` | Dimension 5 |

**The governing rule:** every claim carries a number or gets deleted.

---

## Grader's Four-Minute Pass

Before detailed scoring, the grader spends four minutes as a hiring reviewer would and records:

1. **At what point could I tell what this does?**
2. **Did the setup work as written?**
3. **Which claim did I want to verify, and could I?**
4. **What specific question would I open the interview with?**

A submission that produces a *specific* question in item 4 is already succeeding at what the capstone is for. A submission where the honest answer to item 4 is "what does this actually do?" will score poorly on Dimension 1 regardless of what the code contains.

---

## Self-Assessment

Score yourself against this rubric before submitting, and commit the result as `docs/self-assessment.md` with your scores and one line of justification each.

Two reasons this is required. It usually surfaces two or three cheap fixes worth 5–10 points. And a candidate who has honestly assessed their own work against a rubric — including where it falls short — is rehearsing exactly the self-awareness the Lesson 12.3 deep-dive tests.
