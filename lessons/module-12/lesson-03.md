---
id: "12.3"
module: 12
minutes: 14
practice_minutes: 90
prerequisites: ["12.2"]
objectives:
  - "Deliver a project deep-dive: problem, architecture and why, trade-offs, evaluation metrics, what you would improve next."
volatility: concept
sources: []
last_verified: "2026-08-24"
---

# 12.3 · The Project Deep-Dive Round

Hiring research identifies the project deep-dive with follow-up probes as a central round in AI engineering interviews. Forty-five minutes, one project, and an interviewer whose job is to find out whether you understand what you built or assembled it from tutorials.

The good news is that this entire course has been generating material for this round. Every decision record, every reverted change, every measured failure exists to answer a specific kind of question you are about to be asked.

The bad news is that most candidates fail this round in a predictable place, and knowing where it is will not save you unless you have prepared for it.

---

## Key Concept 1: The five-part structure

Open with five minutes of structure, then let them drive.

**1. The problem.** What needed solving and why it was worth solving. Thirty seconds. Not the technology — the problem.

**2. The architecture, and the reasons.** The pipeline, and *why* each significant piece is what it is. Reasons, not names. "Hybrid retrieval, because a third of our queries contained error codes and function names that dense retrieval ranked at position 19."

**3. The trade-offs.** Stated as trade-offs, not wins. "Reranking gained 15 points on multi-hop and added 1.3 s at p95 (2.1 s → 3.4 s). We accepted it because multi-hop was our second-priority failure mode; Module 10's caching later recovered the latency."

Candidates who present only wins sound like they have not operated anything. Every real decision cost something.

**4. The evaluation.** What you measured, how, and the numbers. **Most candidates have nothing here**, which makes it the cheapest place to stand out. Mention the judge calibration — "our judge agreed with human labels 86% of the time, so we didn't act on differences under 0.04" — because almost nobody does, and it is immediately recognizable as real practice.

**5. What you would improve next.** Two or three specific things with reasons. This demonstrates you know your system's edges, and it is a much better answer than "nothing, it's finished."

---

## Key Concept 2: The depth ladder

Interviewers probe by repeatedly asking "why." The rungs are predictable:

**Rung 1 — the claim.** *"We used hybrid retrieval."*
Everyone reaches this.

**Rung 2 — why.** *"Dense retrieval missed exact-token queries — error codes, function names."*
Most candidates reach this.

**Rung 3 — why not the alternative.** *"Why not just a better embedding model?"*
**This is where most candidates fail**, because they adopted a default without considering an alternative. There is nothing to say.

**Rung 4 — what broke.** *"What did hybrid cost you? What went wrong with it?"*
Requires having actually run it.

**Rung 5 — how do you know.** *"How do you know it helped?"*
Requires measurement.

Every decision record you wrote in this course exists to answer rung 3. Every eval report entry answers rungs 4 and 5. That is not a coincidence — it is why those exercises were structured that way.

**Prepare by laddering your own claims.** Take your ten main architectural claims and write out four levels of "why" for each. Where you run out before rung 4, you have found a gap: either go and measure it, or stop making the claim.

---

## Key Concept 3: The five stories

Specific narratives beat general descriptions, and interviewers probe for exactly these shapes. You have all five.

**A diagnosed failure with numbers.** The Lesson 5.5 metric-vector diagnosis. *"Context recall was 0.94 and precision was 0.38, which told me retrieval was finding it and ranking was burying it — so a reranker rather than re-chunking."* This one story demonstrates systematic debugging better than any amount of description.

**A change you reverted because evaluation said so.** Lesson 10.3's semantic cache, or your trimming experiment. Demonstrates that your evaluation is real and that you act on it against your own preference.

**A negative result reported honestly.** Lesson 8.4's ship/no-ship. *"The fine-tune won on format and cost and halved refusal accuracy, so we routed around it instead of shipping it as-is."*

**An attack you found against your own system.** Lesson 11.4. *"Every tool-level control was in place and the model exfiltrated through a citation URL the user's browser would fetch — no tool involved."* Very few candidates have anything here.

**A framework limitation you hit.** Lesson 7.1. *"Answers started refusing after the port; the cause was the framework's message trimming cutting part of the system prompt. Found it by dumping the literal model input."* Demonstrates you can debug through an abstraction.

Prepare each as sixty to ninety seconds: situation, what you did, what the numbers were, what you learned.

---

## Key Concept 4: Meta-skills

**Quantify everything.** "Much better" is noise. "0.71 to 0.87 recall@5" is signal. You have the numbers; use them.

**Say "I don't know" cleanly, then say how you would find out.** *"I didn't measure that. I'd run it against the out-of-scope stratum and compare refusal accuracy, which would answer it in about ten minutes."* This is a strong answer. Guessing confidently is the weakest possible move, because it invites a probe you cannot survive.

**Never oversell.** The fastest way to fail is a claim that collapses on the second probe. If you used one framework feature lightly, say so.

**Own the whole system, including its weaknesses.** "The multi-hop stratum is our weakest at 0.86, and I know why — chunking splits across document boundaries" is stronger than pretending it is uniform.

**Bring the artifacts.** Have the eval report and architecture diagram open. Pointing at a real table is more convincing than describing one.

---

## Worked Example: Two answers, three probes

**Question:** *"Tell me about your RAG system."*

**Candidate A:**

> "I built a RAG system using LangChain and LangGraph with a Qdrant vector database. It uses OpenAI embeddings and I added a reranker to improve results. It's deployed with Docker and has observability with Langfuse."

A technology tour. No problem, no reasoning, no numbers.

*Probe 1: "Why Qdrant?"* → "It's fast and it's what's commonly used for vector search." **(Rung 2 failed.)**
*Probe 2: "How do you know the reranker improved results?"* → "The answers seemed better." **(Rung 5 failed.)**

Two probes in, the round is effectively over.

**Candidate B:**

> "It answers questions about product documentation with citations, and refuses when the docs don't cover the question — that refusal behavior turned out to be the hardest part.
>
> The architecture is hybrid retrieval into a cross-encoder reranker into an agent loop. Hybrid because about a third of our queries were error codes and function names, and dense-only ranked those around position 19 — BM25 put them at rank 1. The reranker came later, when the metrics showed context recall at 0.94 and precision at 0.38: retrieval was finding the answer and ranking was burying it.
>
> Overall faithfulness went 0.71 to 0.89 on a 60-item stratified held-out set, with the judge calibrated against human labels at 86% agreement, so we ignored differences under 0.04.
>
> The biggest surprise was that more context made things worse — quality peaked at five chunks and fell off sharply by twenty."

*Probe 1: "Why not just a better embedding model instead of hybrid?"*

> "We tried that first. Benchmarked three on our own corpus — the leaderboard leader placed third. The best one got exact-token queries to about 0.44 recall@10 versus 0.94 for BM25. It's a representation limitation: rare tokens like `413` sit near `412` in embedding space. No embedding model was going to fix it." **(Rung 3, with evidence.)**

*Probe 2: "What did hybrid cost you?"*

> "Roughly double the retrieval work — 11 ms to 19 ms at p95, negligible next to generation — and about two points on pure-paraphrase queries, where dense alone was better. Net positive because real traffic is mixed and you can't route in advance." **(Rung 4.)**

*Probe 3: "Anything you'd do differently?"*

> "The chunking. Multi-hop is our weakest stratum at 0.86 because structure-aware splitting cuts across document boundaries and multi-hop questions need two documents. I'd try a graph-style link between related sections. I'd also fix the eval set — our out-of-scope stratum was all technical questions, so we missed that production users ask about pricing, which online evaluation caught two weeks in at 25 points below offline."

The last answer volunteers a weakness in the candidate's own evaluation methodology, with the number, and how it was found. That is the answer of someone who has operated a system.

---

## Practice

**1. Write the five-part brief.** One page: problem, architecture with reasons, trade-offs, evaluation with numbers, next improvements. Rehearse until it is five minutes without notes.

**2. Ladder your ten main claims.** For each, write four levels of "why." **Where you run out before rung 4, either go measure it or drop the claim.** This is the highest-value exercise in the lesson.

**3. Prepare the five stories.** Sixty to ninety seconds each: situation, action, numbers, lesson.

**4. Run a mock round.** Record it, or find a partner. The interviewer's instruction: **push every claim three levels deep and do not accept the first answer.** Forty-five minutes.

**5. Self-assess** against `assets/deep-dive-rubric.md`: structure, quantification, depth under probing, honesty about limits, and quality of the "what's next" answer.

**6. Write out the ten questions you handled worst,** with the answers you wish you had given. Verify each against your artifacts — if the answer requires a number you do not have, measure it.

**7. Practise "I don't know."** Deliberately, three times in the mock. It should feel routine, followed immediately by how you would find out.

**8. Commit the brief** to your repository as `docs/project-brief.md`, or keep it private — your choice.

---

## Visual Aid

*A ladder diagram, five rungs, drawn vertically. Rung 1 "the claim — everyone reaches this." Rung 2 "why — most candidates." Rung 3 "why not the alternative" — drawn as a broken rung with a red marker labeled "most candidates fall here." Rung 4 "what did it cost / what broke." Rung 5 "how do you know." Beside each rung, the course artifact that answers it: rung 3 ↔ "decision records (Modules 4, 7, 8, 9, 10)", rung 4 ↔ "eval report — trade-off column", rung 5 ↔ "eval report — measurement + judge calibration". Caption: "Every decision record in this course was written to answer rung 3."*

---

## Key Takeaways

- **Five-part structure:** problem, architecture with reasons, trade-offs stated as trade-offs, evaluation with numbers, and what you would improve next.
- **The depth ladder breaks most candidates at rung three** — "why not the alternative" — because they adopted a default without considering one. Your decision records exist to answer exactly that.
- **Five stories beat any general description:** a diagnosed failure with numbers, a reverted change, an honest negative result, an attack against your own system, and a framework limitation you debugged through.
- **The evaluation section is where you stand out**, because most candidates have nothing there. Mention judge calibration.
- **Quantify everything, say "I don't know" cleanly with a plan, and never oversell.** A claim that collapses on the second probe ends the round.
- **Volunteering a weakness in your own methodology, with the number and how you found it,** is the strongest signal available.

---

## Next

That round was about what you built. The final round is about what you *would* build — a problem you have never seen, on a whiteboard, in forty-five minutes.

**Lesson 12.4** gives you a seven-step framework for the AI system design round, and shows that its two hardest steps — the evaluation plan and the cost model — are the two most candidates skip entirely, and the two this course has given you the most to say about.
