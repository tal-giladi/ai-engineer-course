# Module 12 Plan — Capstone, Portfolio & Interview Readiness

**Duration:** 55 min · **Lessons:** 4 · **Depends on:** all previous modules
**Outline reference:** `curriculum/course-outline.md` §8, Module 12

---

## Prerequisites

All eleven previous modules. The spine project exists; this module finishes it, documents it, and rehearses defending it.

**Carried forward for reuse:** the `job-map.md` from Lesson 1.1, the eval report from Modules 3–10, the framework trade-off document from Lesson 7.1, the fine-tuning decision record and model cards from Modules 8–9, the SLO and latency analysis from Module 10, and the threat model and security report from Module 11. Nothing in this module is written from scratch — it is assembly, and students should discover that the artifacts a hiring panel wants have been accumulating since lesson one.

---

## Framing

The brief's actual goal is employability, and the research is specific about what that requires: hiring managers look for production instincts — error handling, evaluation, deployment, structured thinking — and a well-documented project with a clear write-up beats five undocumented notebooks. Interview processes center on a project deep-dive with follow-up probes, and an AI system design round. Three projects show range; documentation is what makes any of them count. This module converts eleven modules of work into those artifacts and rehearses the two rounds.

---

## Module Learning Objectives

**M12-LO1** — Finish and harden the spine system against a definition of done drawn from real job requirements.
**M12-LO2** — Document the work the way hiring panels read it: model card, eval report, architecture decision record, and a README that earns a second look.
**M12-LO3** — Deliver a project deep-dive: problem, architecture and why, trade-offs, evaluation metrics, what they would improve next.
**M12-LO4** — Work an AI system design round out loud: requirements, retrieval strategy, agent boundaries, eval plan, cost model, failure modes.

---

## Lesson 12.1 — Shipping the Capstone

*13 min · serves M12-LO1*

**Concept.** Finishing, treated as a skill with a checklist rather than a feeling. Introduce a **definition of done** assembled from job requirements rather than from personal satisfaction: the system runs from a clean clone with one command; every claim in the README is backed by a number in the eval report; the eval suite is green and its results are committed; the security regression suite passes; cost and latency SLOs are stated and met; the failure modes are documented; and someone else has run it. Cover the two failure patterns that kill portfolio projects at this stage — **scope creep**, where a student adds a feature instead of finishing, and **the undocumented 90%**, where the system works and nobody can tell. Cover triage: with limited time, completeness and evidence beat capability every time, because a reviewer cannot evaluate a feature they cannot run. Cover the portfolio-range question raised by the research — three projects show range — and resolve it honestly: the spine system is the deep project, and the eleven standalone exercises are the range, with two or three worth promoting into small documented repositories of their own.

**Example.** Two versions of the same capstone reviewed side by side. One has more features, a broken setup path, and no evaluation. The other does less, runs in ninety seconds from a clean clone, and reports its own limitations. The second is the one that gets a callback, and the reviewer's reasoning is walked line by line.

**Practice.** Students run their capstone against the definition-of-done checklist, triage the gaps into must-fix and accept-and-document, and close the must-fix list. They then perform a clean-clone test — fresh directory, README only, no prior knowledge — and fix every step that fails, which reliably surfaces two or three assumptions.

**Summary.** Done is a checklist, not a feeling. Completeness and evidence beat features, and the clean-clone test finds what you cannot see.

**Key terms:** definition of done, scope creep, clean-clone test, triage, must-fix vs accept-and-document, portfolio depth vs range.
**Visual aid:** Definition-of-done checklist with each item traced to the module that produced it and the job requirement it answers.
**Hook:** "It works and it's finished. A reviewer will spend four minutes deciding whether to keep reading — and the code isn't what they read first."

---

## Lesson 12.2 — Documenting Like a Hire: Model Card, Eval Report, Architecture Doc

*13 min · serves M12-LO2*

**Concept.** Documentation as evidence of engineering judgment rather than as a chore, taught through the four artifacts hiring panels actually read. The **README** as a four-minute pitch: what it does, a screenshot or short demo, one-command setup, architecture in one diagram, results in one table, and known limitations — with limitations argued for explicitly, because stated limits read as judgment while their absence reads as unawareness. The **eval report**, which is the differentiating artifact almost nobody has: baseline, each change, its measured effect, the metric definitions, the judge's calibration and trust boundary, and the changes that were tried and reverted — a reverted change is strong evidence of discipline. The **model card** for the fine-tuned artifacts from Modules 8–9: base model, data provenance, hyperparameters, evaluation, intended use, and out-of-scope use. The **architecture decision records** already written in Modules 4, 7, and 8, which answer the "why" questions before they are asked and can be reused verbatim in the deep-dive. Cover the write-up beyond the repository — the research is explicit that a public write-up multiplies a project's reach — and the rule that governs all of it: **every claim carries a number or gets deleted.**

**Example.** A README rewritten from a feature list into the four-minute structure, with a before/after and a reviewer's reaction to each. Then an eval report excerpt showing a reverted optimization, annotated with why that section is the strongest page in the repository.

**Practice.** Students write the README to the four-minute structure, assemble the eval report from the running `eval-report.md`, finalize model cards, and collect their existing decision records into `docs/`. They then swap repositories with a peer (or use the supplied review rubric solo) and get a four-minute cold review, fixing what the reviewer could not find.

**Summary.** Four artifacts: README, eval report, model card, decision records. Every claim carries a number, limitations are stated deliberately, and the eval report is the thing most candidates do not have.

**Key terms:** README structure, eval report, model card, architecture decision record, data provenance, intended use, stated limitations, public write-up, cold review.
**Visual aid:** Annotated README wireframe with each section labeled by the reviewer question it answers and the time budget it consumes.
**Hook:** "The repository speaks for you until someone asks you to speak for it. That happens in round two, and it lasts forty-five minutes."

---

## Lesson 12.3 — The Project Deep-Dive Round

*14 min · serves M12-LO3*

**Concept.** The interview round the research identifies as central, and the one this entire course has been generating material for. Cover the **five-part structure** that reliably works: the problem and why it was worth solving; the architecture and the reasons behind each significant choice; the trade-offs, stated as trade-offs rather than as wins; the evaluation — what was measured, how, and what the numbers were; and what you would improve next. Cover the **depth ladder** interviewers use — a claim, then "why," then "why not the alternative," then "what broke," then "how did you know" — and the reality that most candidates fail at rung three, because they adopted a default without considering an alternative. Every decision record written during this course exists to answer rung three, which is worth making explicit. Cover the **high-value stories** this course produced, which map directly onto the probes panels use: a diagnosed failure with numbers (the Module 5 triage), a change that was reverted because evaluation said so (Module 10), a negative result reported honestly (Module 8's ship/no-ship), an attack found against your own system (Module 11), and a framework limitation you hit and worked around (Module 7). Cover the meta-skills: say "I don't know" cleanly and follow it with how you would find out; quantify everything; and never oversell — the fastest way to fail this round is a claim that collapses on the second probe.

**Example.** Two transcribed answers to "tell me about your RAG system." The first is a tour of the technology stack. The second states the problem, names one measured failure mode, explains the fix and its cost, and reports the number — then survives three follow-up probes. The probes are shown, with the exact point where the first answer runs out of material.

**Practice.** Students prepare a five-minute deep-dive plus a one-page brief, then run a mock round — recorded and self-assessed against a rubric, or with a peer — where the interviewer must push every claim three levels deep. They then write answers to the ten follow-up questions they handled worst.

**Summary.** Problem, architecture, trade-offs, evaluation, next steps. Expect to be pushed three levels on every claim, bring numbers, and let the decision records do the work they were written for.

**Key terms:** project deep-dive, depth ladder, trade-off framing, quantified claim, negative result, failure story, calibrated uncertainty.
**Visual aid:** The depth-ladder diagram — one claim probed through four levels — with the course artifact that answers each level named beside it.
**Hook:** "That round is about what you built. The next one is about what you'd build, for a problem you've never seen, on a whiteboard."

---

## Lesson 12.4 — The AI System Design Round

*13 min · serves M12-LO4*

**Concept.** The round that is new relative to classic system design, and where this course's structure converts directly into a framework. Give students a repeatable seven-step approach: **clarify requirements** (users, volume, latency tolerance, accuracy bar, cost ceiling, data sensitivity — and note that asking about the accuracy bar and the cost ceiling immediately signals production experience); **choose the approach** (prompting, retrieval, fine-tuning, agent — with the Module 8 decision matrix as the reasoning); **design retrieval** (sources, chunking, index, hybrid, reranking); **draw agent boundaries** (which decisions are the model's, which are code's, what tools exist, what their blast radius is); **plan evaluation** (dataset, metrics, judge, CI gate, online sampling — the step most candidates omit entirely, and therefore the cheapest way to stand out); **model cost and latency** (calls per request, tokens per call, cache hit rate, and an actual arithmetic estimate); and **name failure modes and defenses** (hallucination, injection, provider outage, cost runaway). Cover the classic traps: jumping to a framework before requirements, no evaluation plan, no cost model, treating the model as deterministic, and ignoring the data pipeline that feeds retrieval. Cover the common prompts they will actually get — a documentation assistant, a support agent with actions, a document-processing pipeline, an internal search system — and note that each maps onto modules of this course.

**Example.** A full worked round on "design an AI assistant for our internal engineering documentation," from clarifying questions to a drawn architecture, with the evaluation plan and the cost arithmetic done out loud — and a deliberate mid-round curveball ("the corpus includes customer-specific documents") handled by reaching for tenant-scoped filtering and cache scoping from Modules 4 and 11.

**Practice.** Students work three timed design rounds — one documentation assistant, one action-taking agent, one high-volume processing pipeline — producing for each a clarifying-question list, an architecture sketch, an evaluation plan, a cost model, and a failure-mode table. They self-assess against a rubric weighted toward evaluation and cost, since those are the differentiators.

**Summary.** Clarify, choose, retrieve, bound the agent, plan the evaluation, model the cost, name the failures. The evaluation plan and the cost model are where most candidates say nothing, and where this course has given you the most to say.

**Key terms:** AI system design round, requirements clarification, accuracy bar, cost ceiling, agent boundary, evaluation plan, cost model, failure-mode analysis, tenant isolation.
**Visual aid:** The seven-step framework as a single-page reference card, with each step annotated by the module that supplies its content.
**Hook:** "Twelve modules, one production system, and the artifacts to prove all of it. Go back to `job-map.md` from Lesson 1.1 — the gaps you wrote down on day one are the checklist you just finished."

---

## Spine Increment

**Deliverable:** the finished, documented, public portfolio repository — the capstone.

**Acceptance criteria:**
1. Definition-of-done checklist complete; clean-clone test passes with no undocumented step.
2. README follows the four-minute structure with a diagram, a results table, and stated limitations.
3. `eval-report.md` covers baseline through Module 11, including at least one reverted change and the judge's trust boundary.
4. Model cards complete for every fine-tuned artifact; decision records collected in `docs/`.
5. Security report and threat model included, with open issues stated.
6. A five-minute deep-dive brief and answers to the ten hardest follow-up questions, committed to the repository (or kept private, at the student's choice).

---

## Standalone Exercise

Take three real job postings for roles you would actually apply to, map every requirement against the course's artifacts, and produce resume bullets citing measured results plus an honest gap list with a plan for the top two gaps. Deliverable: `exercises/m12-job-mapping.md` — a direct continuation of the `job-map.md` started in Lesson 1.1.

---

## Dependencies

| Lesson | Requires |
|---|---|
| 12.1 | Modules 1–11 |
| 12.2 | 12.1 |
| 12.3 | 12.2, plus decision records from 4, 7, 8 |
| 12.4 | All modules (synthesis) |

**Downstream:** none — this is the course terminus. The final assessment is the capstone repository against the published rubric.

---

## Assessment Focus (Phase 4)

Performance-based rather than knowledge-based. The graded artifact is the capstone repository against the rubric — retrieval quality, agent robustness, eval rigor, deployment, security posture, documentation. Written questions cover: given a README, list what a reviewer cannot determine; given a claim, write the three follow-up probes it invites; given a system design prompt, produce the clarifying questions and the cost model; given a candidate answer with no evaluation plan, identify what is missing and why it matters.
