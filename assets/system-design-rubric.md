# AI System Design Round Rubric

**Used in:** Lesson 12.4, practice step 5.

Score a 40-minute timed round. 50 points, **deliberately weighted toward evaluation and cost** — the two sections most candidates omit entirely and therefore the cheapest places to differentiate.

---

## 1. Requirements Clarification — 8 points

| Criterion | Points |
|---|---|
| Asked before designing anything | 2 |
| Covered users, volume, and arrival pattern | 1 |
| Covered latency tolerance | 1 |
| ★ **Asked about the accuracy bar and what happens when it's wrong** | 2 |
| ★ **Asked about the cost ceiling** | 1 |
| Covered data sensitivity, access restrictions, or multi-tenancy | 1 |

**The two starred items are the strongest early signal in the round.** People who have only built demos do not think to ask them.

**Deduct 2** for spending more than 8 minutes here. Clarification is not the deliverable.

---

## 2. Approach and Architecture — 12 points

| Criterion | Points |
|---|---|
| Chose prompting / retrieval / fine-tuning / agent with explicit reasoning | 3 |
| Retrieval design: chunking, embedding selection, index choice, hybrid where warranted | 3 |
| Addressed the **data pipeline** — ingestion, updates, deletions, re-indexing | 2 |
| Drew a clear agent boundary, or argued convincingly for no agent | 2 |
| Named the pre-filter requirement where permissions or tenancy apply | 2 |

**Bonus consideration:** arguing *against* complexity — a fixed pipeline over an agent, one agent over five — reads as judgment, not as low ambition.

**Deduct 3** for naming a framework or vector database before requirements were established.

---

## 3. Evaluation Plan — 10 points

| Criterion | Points |
|---|---|
| Raised it unprompted | 2 |
| Golden dataset with **stratification by failure mode**, over-sampling what degrades silently | 2 |
| Metrics chosen for the task shape, with a diagnostic reading | 2 |
| LLM judge with **calibration against human labels** and a stated trust boundary | 2 |
| CI gating and online production sampling | 2 |

**Most candidates score 0–2 here.** Two minutes of coherent coverage is often the most substantive thing said in the round.

---

## 4. Cost and Latency Model — 8 points

| Criterion | Points |
|---|---|
| Raised it unprompted | 2 |
| **Did the arithmetic out loud** — calls × tokens × price | 3 |
| Produced a per-stage latency budget | 1 |
| **Turned the number into an architectural decision** (routing, smaller model, self-hosting) | 2 |

**The last item is what the section is testing.** A cost estimate that does not change the design is a calculation; one that produces "so I'd route the common case to a smaller model" is engineering.

---

## 5. Failure Modes — 7 points

| Criterion | Points |
|---|---|
| Hallucination and grounding | 1 |
| Prompt injection, framed as the three-part risk chain | 2 |
| Provider outage with a fallback plan | 1 |
| Cost runaway with caps and a kill switch | 2 |
| Named what degraded mode looks like to a user | 1 |

---

## 6. Handling the Curveball — 5 points

The interviewer injects a mid-round change ("the corpus includes customer-specific documents", "we also want it to file tickets", "volume is 50× what I said").

| Criterion | Points |
|---|---|
| Identified what the change actually alters about the risk or cost profile | 2 |
| Adapted the design rather than defending the original | 2 |
| Named a specific control or mechanism, not a general intention | 1 |

---

## Scoring

| Total | Reading |
|---|---|
| 42–50 | Strong. Evaluation and cost covered without prompting, arithmetic done aloud. |
| 33–41 | Solid architecture, likely thin on evaluation or cost. |
| 24–32 | Classic system-design answer applied to an AI problem — missing the three concerns that make it different. |
| Below 24 | Re-run using the seven-step reference card with a timer per section. |

---

## Time Budget for 40 Minutes

| Section | Minutes |
|---|---|
| Requirements | 6 |
| Approach, retrieval, agent boundary | 15 |
| **Evaluation plan** | **5** |
| **Cost model** | **3** |
| Failure modes | 5 |
| Buffer for probes and curveball | 6 |

**If evaluation and cost are the sections getting squeezed when you run long, that is the habit to fix** — they are worth 18 of 50 points here for the same reason they differentiate in a real round.
