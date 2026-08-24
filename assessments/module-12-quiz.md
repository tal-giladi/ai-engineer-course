# Module 12 Quiz — Capstone, Portfolio & Interview Readiness

**7 questions · ~30 min**

This module's real assessment is the capstone repository, graded against `assessments/capstone-rubric.md`. These questions cover the judgment around it.

---

## Q1 — Applied judgment

Two capstone repositories, reviewed for four minutes each.

**A:** README is a feature list — multi-agent orchestration, voice input, web search, PDF parsing, Slack integration. Setup fails on an undocumented environment variable. No evaluation, no tests, no diagram.

**B:** README has a one-line description, a 20-second demo GIF, one architecture diagram, a results table, and four honest limitations. Setup: clone, one key, one command, working in three minutes. `eval-report.md` documents eleven changes with measured effects, two marked REVERTED.

A has substantially more features.

(a) Which gets the callback, and at what point in the four minutes is each decision made?
(b) What is the general principle, stated as a rule for triage?

---

## Q2 — Short answer

You have limited time before applying. Rank these four by expected return, and justify the top choice:

- Add a web UI to the capstone
- Write the eval report from your running log
- Add multi-modal document support
- Run the clean-clone test and fix what breaks

---

## Q3 — Diagnosis

An interviewer probes a claim:

> **Candidate:** "We used hybrid retrieval."
> **Interviewer:** "Why?"
> **Candidate:** "Dense retrieval was missing queries with error codes and function names."
> **Interviewer:** "Why not just use a better embedding model?"
> **Candidate:** "...that's a good question, I think hybrid is just the standard approach."

(a) Which rung of the depth ladder failed, and why is it the most common failure point?
(b) What artifact from this course was written specifically to answer that question?
(c) Write the answer the candidate should have given.

---

## Q4 — Applied judgment

Which of these makes the strongest interview story, and why?

A. "We built a RAG system with hybrid retrieval, reranking, and an agent loop."
B. "Context recall was 0.94 and precision was 0.38, which told me retrieval was finding the answer and ranking was burying it — so I added a reranker rather than re-chunking."
C. "We got faithfulness from 0.71 to 0.89."
D. "I used LangGraph because it has checkpointing and interrupts."

---

## Q5 — Design

You are given: *"Design an AI assistant for our internal engineering documentation."*

(a) Name eight clarifying questions. Star the two that most signal production experience.
(b) The answer includes *"some documents are restricted to specific teams."* Name three separate parts of your design this affects.
(c) The answer includes *"confident wrong answers about security procedures are dangerous."* What does this change about your evaluation plan specifically?

---

## Q6 — Short answer

In a system design round you have 40 minutes. A candidate spends 30 on architecture, 8 on requirements, 2 on failure modes, and none on evaluation or cost.

(a) What are the two most costly omissions, and why?
(b) Suggest a corrected time budget.

---

## Q7 — Design

Mid-round, the interviewer adds: *"Actually, we'd also like it to file tickets automatically."*

Give the answer. Name the risk chain, the architectural response, and three specific controls — including one that is not a security control.

---
---

# Answer Key

## Q1

**(a) B, by a wide margin.**

- **A is decided at roughly 90 seconds** — the setup fails, the reviewer cannot run it, no claim in the README can be verified, and there is no reason to keep reading.
- **B is decided at roughly 4 minutes** — the reviewer ran it, read the eval report, and arrived at a *specific question* about one of the trade-offs. That is the best possible outcome, because the deep-dive now starts from something concrete rather than "tell me about your project."

**(b) The triage rule: completeness and evidence beat capability.** A reviewer cannot evaluate a feature they cannot run, and cannot verify a claim with no number attached. Every hour spent making the system runnable and its results legible returns more than an hour spent on a new feature.

## Q2 — Ranked

1. **Run the clean-clone test and fix what breaks.** If a reviewer cannot start the system, everything else in the repository is invisible. Highest return by a wide margin, and it typically takes under an hour.
2. **Write the eval report.** The differentiating artifact — almost no candidate has one, and it is the evidence that every claim in the README is real. The work is already done; this is transcription.
3. **Add a web UI.** Some value — it improves the demo — but a GIF of the CLI achieves most of it for a fraction of the effort.
4. **Add multi-modal support.** Lowest return. It is scope creep at exactly the stage where scope creep kills projects, and it adds a capability nobody asked to see.

**Justification for the top choice:** the failure it prevents is total. A broken setup means the reviewer forms no opinion of the system at all — not a poor opinion, no opinion — and moves on at 90 seconds.

## Q3

**(a) Rung 3 — "why not the alternative."** It is the most common failure point because most candidates adopted a default without ever evaluating an alternative. There is nothing to say, because the comparison was never made.

**(b) The architecture decision record** — `docs/adr-embedding-model.md` from Lesson 4.1, which recorded the three-model benchmark, and `adr-vector-store.md`, `adr-agent-architecture.md`, and the rest. Every decision record in this course exists to answer rung 3.

**(c) The answer:**

> "We tried that first. We benchmarked three embedding models on our own corpus rather than trusting the leaderboard — the leaderboard leader actually placed third. Even the best one got exact-token queries to about 0.44 recall@10, versus 0.94 for BM25. It's a representation limitation rather than a quality one: rare tokens like `413` sit near `412` in embedding space because the model saw them in similar contexts. No embedding model was going to close that, so we fused instead."

That answer contains: the alternative was tried, the numbers, the mechanism, and why the mechanism makes the alternative structurally unable to work.

## Q4 — **B**

**B** demonstrates diagnostic reasoning: reading a metric vector, localizing a failure to a stage, and choosing the fix that matches the diagnosis rather than the intuitive fix. It also implies, without stating, that the candidate has an evaluation harness, understands RAG-specific metrics, and knows the difference between a retrieval miss and a ranking failure. One sentence, four capabilities.

**A** is a technology tour with no problem, reasoning, or numbers. **C** has a number but no mechanism — the immediate follow-up "how?" has nowhere to go. **D** is better than A but stops at rung 2, and names features rather than the problem they solved.

## Q5

**(a) Eight clarifying questions:**

1. How many users, and how many queries per day?
2. Interactive or batch? What p95 latency is acceptable?
3. ★ **What accuracy bar do you need, and what happens when it's wrong?**
4. ★ **What's the cost ceiling — per request or per month?**
5. How large is the corpus, in what format, and how often does it change?
6. Who can write to the corpus?
7. Are there access restrictions on documents?
8. Can the data leave your infrastructure?

**The two starred questions signal production experience** because they are the ones people who have only built demos do not think to ask. Question 6 is also a strong signal — it is the injection-surface question.

**(b) Three parts affected by restricted documents:**
- **Retrieval:** permission filters must be **pre-filters**, not post-filters — otherwise recall collapses and the search touched documents outside the user's scope
- **Caching:** cache keys must include permission scope, or one team's answer is served to another (Lesson 11.3)
- **Credentials and evaluation:** the retrieval credential must be scoped, and the golden set needs a permission-boundary stratum

**(c)** Security procedures become **their own stratum with an absolute hard floor in CI** — refusal accuracy on security topics can never regress regardless of what the aggregate says. The stated danger is *confident wrong answers*, which means faithfulness and refusal on that stratum are non-negotiable, not band-gated.

## Q6

**(a) Evaluation and cost.**

**Evaluation**, because "how would you know it works?" is the question that distinguishes someone who has shipped from someone who has designed on a whiteboard. It is also the section most candidates skip entirely, which makes two minutes spent there disproportionately differentiating.

**Cost**, because in this field cost is a first-class design constraint, not an afterthought. A design that would cost $57k/month against a $5k budget is not a design. Doing the arithmetic out loud and turning the number into an architectural decision — routing, a smaller model, self-hosting — is the sequence the round is testing.

**(b) Corrected budget for 40 minutes:**

| Section | Minutes |
|---|---|
| Requirements | 6 |
| Approach + retrieval + agent boundary | 15 |
| **Evaluation plan** | **5** |
| **Cost model** | **3** |
| Failure modes | 5 |
| Buffer for probes | 6 |

Architecture at 30 minutes is over-indexing on the part that is most comfortable to talk about.

## Q7

**The answer:**

> "That changes the risk profile substantially. Filing a ticket is a write action, and the corpus is a wiki — so anyone who can edit a page can potentially cause tickets to be filed. That's the three-part chain: untrusted content, a tool that acts, and now an effect that leaves the system.
>
> Architecturally I'd split the contexts. A retrieval agent reads documents with no write tools and returns a structured summary; a separate action agent holds the ticket tool and only ever sees that summary. Injected text in a document never reaches a context that can act on it.
>
> Then three controls: an approval gate above a severity or volume threshold — conditional, so it stays rare enough that people actually read it; a scoped credential that can only create tickets in one project, so a compromise is bounded; and an **idempotency key** on ticket creation, because retries and resumes would otherwise file duplicates.
>
> That last one isn't a security control — it's a correctness control that resumability introduced. It's worth mentioning because it's the kind of thing that gets missed when a system gains durability."

**Full credit requires:**
- Naming the three-part risk chain explicitly
- Context isolation as the architectural response (and recognizing it as the legitimate multi-agent case)
- Two security controls: conditional approval gate and scoped credential
- **One non-security control: the idempotency key** — this is the part that shows the candidate is reasoning about the whole system rather than reciting a security checklist

**Also creditable:** per-session rate limits on ticket creation, and noting that ticket creation should be logged with the agent run's correlation ID for audit.
