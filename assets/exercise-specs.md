# Supplied Exercise Specifications

**Status: specifications only. The code artifacts described here are not yet built.**

Eight lessons hand students a pre-built artifact to work against — a broken pipeline to diagnose, a vulnerable agent to attack, a dataset to curate. Those artifacts are course infrastructure, not student deliverables, and they must exist before the course runs. This document specifies each one precisely enough to build.

The other five `exercises/` paths referenced in the lessons (`m2-invoice-extractor`, `m4-code-retrieval`, `m6-research-agent`, `m7-graph-research-agent`, `m8-format-adherence`) are **student output directories** and require nothing supplied.

---

## Shared conventions

- Python 3.11+, `uv`-installable, no GPU required unless stated
- Each ships with `README.md` (task only — never the answer), `pyproject.toml`, and a `make verify` target the student runs to check their fix
- Ground truth lives in a `.solution/` directory excluded from the student distribution
- Seeded randomness throughout, so failures are reproducible
- No real credentials, no real external endpoints

---

## `m2-prose-extractor` — Lesson 2.1

**Purpose:** let students feel the surface area of prose parsing before they are allowed to fix it.

**Contents:**
- A working extractor pulling three fields (priority, category, customer tier) from support tickets using regex and string splitting
- 200 input records with known-correct labels in `labels.jsonl`
- A runner producing predictions

**Required failure profile:** roughly 8% mismatch, of which **zero raise exceptions**. All failures must parse cleanly into wrong values. Include specifically:
- 3 negation cases (`"not high — this is routine"` parsing as `high`)
- 2 near-miss enum cases (`Billing/Payments` vs allowlist `billing`)
- 2 context-bleed cases (a sentence mentioning enterprise customers generally, read as tier)
- 1 total-refusal case where every field defaults

**Student deliverable:** a catalogue of distinct failure shapes and the ratio of silent-wrong to raised. No fixing.

---

## `m2-adversarial-inputs` — Lesson 2.3

**Purpose:** produce a repair-attempt recovery table with a realistic shape, so the budget decision is data-driven.

**Contents:** 200 records seeded to produce, under a generous budget of 5:
- ~15% first-attempt validation failure
- Attempt-2 recovery around 85% of those
- Attempt-3 recovery around 60% of the remainder
- Attempts 4–5 recovering essentially nothing
- 8 permanently-failing records: 5 genuine refusals (unreadable OCR), 2 that trip `finish_reason == "length"`, 1 requiring a field the input genuinely lacks

**The last group is the point** — all 8 are terminal from attempt 1, so enabling classification should cut tokens with no change to final success rate.

---

## `m3-summarizer-eval` — Lesson 3.2 practice; Module 3 quiz Q11

**Purpose:** a system with a *specific* weakness that a naive metric hides.

**Contents:**
- A summarizer that silently drops the final third of documents over ~2,000 tokens
- A document set where only ~10% are long enough to trigger it
- A tempting-but-wrong default metric path (ROUGE against references) under which the defect is nearly invisible, because the summary overlaps heavily with the first two-thirds

**Student deliverable:** an evaluation that exposes it, the numbers, a fix, and a written explanation of why their first metric failed.

---

## `m5-rag-triage` — Module 5 quiz Q9

**Purpose:** the module's central skill — diagnosing from metrics before reading code.

**Contents:** a working RAG pipeline with a corpus, a golden set, and **three seeded defects, one per failure mode**:

| Defect | Implementation | Expected metric signature |
|---|---|---|
| Retrieval miss | Dense-only retrieval; corpus rich in error codes | recall ↓, precision ok |
| Ranking failure | Reranker disabled; `k=12` straight from bi-encoder | recall ok, precision ↓ |
| Generation drift | Grounding structure removed; no citation validation | recall + precision ok, faithfulness ↓ |

The defects must be **independently diagnosable** — fixing one must not mask another. Ship the metric harness working, so students diagnose from output rather than building tooling.

**Student deliverable:** predicted diagnoses recorded *before* reading code, then actual causes, fixes, and an honest note on misreads.

---

## `m8-decision` — Lesson 8.1 practice

**Purpose:** exercise the decision, including a case where the answer is "do not fine-tune."

**Contents:** four scenario briefs, each with volume, cost, error profile, change frequency, and prior prompting attempts. Designed to resolve as:
1. Retrieval (knowledge problem, quarterly-changing facts)
2. Fine-tuning (semantic errors, four measured prompt versions plateaued)
3. Distillation (high-volume classification, cost problem)
4. **Prompting** — a case that looks like fine-tuning and is not, because prompting was never genuinely attempted

Include a TCO worksheet template.

---

## `m9-verbosity-fix` — Module 9 quiz Q8

**Purpose:** preference tuning against a known, measurable dimension, with the confound trap live.

**Contents:**
- An SFT checkpoint (small model, LoRA adapter) with a pronounced verbosity problem — mean response roughly 2.5× longer than necessary
- A task-accuracy evaluation set, so students can verify accuracy did not drop
- A response generator producing candidate pairs

**Critical design note:** the naive path — generating pairs and judging them — must **naturally produce a length-confounded set** (chosen/rejected ratio near 2.0), so students who skip the audit reproduce the reward-hacking result from Lesson 9.2 rather than being told about it.

---

## `m10-latency-halving` — Module 10 quiz Q10

**Purpose:** trace-driven optimization with a mandatory revert.

**Contents:** a working agent endpoint at p95 ≈ 8.4 s, instrumented with tracing, containing deliberately:
- Two independent reranker calls running sequentially (free win, ~17%)
- Citation validation on 100% of requests (sampling win, ~26%)
- No embedding cache (small free win)
- A retrieval configuration where trimming to 3 chunks **cuts latency and costs 5 points of quality** — the change that must be reverted

Ship with the eval suite wired, so students can gate every change.

---

## `m11-vulnerable-agent` — Module 11 quiz Q9

**Purpose:** systematic red-teaming with real successes.

**Contents:** an agent in an isolated local deployment — synthetic data, fake credentials, a local logging endpoint standing in for exfiltration — with tools `search_docs`, `fetch_url`, `send_email`, `run_python`, and a deliberately weak posture:

| Category | Seeded vulnerability |
|---|---|
| Instruction override | System prompt extractable; contains no secrets (intended to be *accepted*, not fixed) |
| Context manipulation | Corpus writable; no grounding on refusal |
| Tool abuse | Path check on the string, not the resolved path — `%2e%2e%2f` bypasses it |
| Data exfiltration | **No egress allowlist**, and no output-side URL validation — the citation-URL channel must work |
| Resource exhaustion | Budget checked *after* the call rather than before |

At least one vulnerability must be **unfixable without removing a feature**, so students practise recommending a trade-off rather than only patching.

**Ship an attack-log template and a regression-test skeleton**, since the graded deliverable is the tests, not the attacks.

---

## Build order

If building incrementally, this order unblocks the most course time per unit effort:

1. `m5-rag-triage` — the highest-value exercise in the course
2. `m11-vulnerable-agent` — the most distinctive portfolio artifact
3. `m2-prose-extractor` and `m2-adversarial-inputs` — early, and both are small
4. `m10-latency-halving`
5. `m3-summarizer-eval`
6. `m9-verbosity-fix` — requires a GPU to prepare the checkpoint
7. `m8-decision` — documents only, buildable in an hour

**Estimate:** roughly 6–9 engineering days total, with `m9-verbosity-fix` and `m11-vulnerable-agent` accounting for about half.
