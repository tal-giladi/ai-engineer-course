---
id: "12.1"
module: 12
minutes: 13
practice_minutes: 90
prerequisites: ["11.5"]
objectives:
  - "Finish and harden the spine system against a definition of done drawn from real job requirements."
volatility: concept
sources: []
last_verified: "2026-08-24"
---

# 12.1 · Shipping the Capstone

You have built a production AI system across eleven modules. The remaining risk is not technical — it is that the project stalls at 90% forever, because "finished" is a feeling and the feeling never arrives.

This lesson replaces the feeling with a checklist, drawn from what hiring reviewers actually check rather than from what feels satisfying to build. It also names the two patterns that kill portfolio projects at exactly this stage, both of which feel productive while they happen.

---

## Key Concept 1: Two ways this fails

**Scope creep disguised as ambition.** You are one feature from impressive, so you add multi-modal input. Then streaming voice. Then a web UI. Each is genuinely interesting, none is finished, and the project drifts.

The reviewer's perspective is the correction: **they cannot evaluate a feature they cannot run.** A working system with four capabilities beats a broken one with nine, every time, and it is not close.

**The undocumented 90%.** The system works. It is genuinely good. The README says "AI agent with RAG." There is no architecture diagram, no results, no setup path that works.

This is worse than scope creep because the work is *already done* and invisible. Two hours of documentation converts a project a reviewer skips into one they read.

**The triage rule:** with limited time, **completeness and evidence beat capability.** Every hour spent making the system runnable and its results legible returns more than an hour spent on a new feature.

---

## Key Concept 2: The definition of done

Drawn from what hiring reviewers check.

**Runs**
- [ ] Clean clone → one config file → one command → working system, under five minutes
- [ ] A CPU-only path exists (Lesson 10.5) so reviewers without a GPU can run it
- [ ] A seeded demo corpus, so first run does something
- [ ] Someone other than you has run it

**Evidence**
- [ ] Every claim in the README backed by a number in `eval-report.md`
- [ ] Eval suite green, results committed
- [ ] Security regression suite green (Lesson 11.4)
- [ ] SLO stated and met, with the measurement shown (Lesson 10.3)

**Honesty**
- [ ] Known limitations documented
- [ ] At least one reverted change recorded with reasoning
- [ ] Accepted security risks stated (Lesson 11.4)
- [ ] Failure modes documented

**Legible**
- [ ] Architecture diagram
- [ ] Decision records for the significant choices (Modules 4, 7, 8, 9, 10)
- [ ] Model cards for every fine-tuned artifact
- [ ] Trace or screenshot showing the system working

That is the standard. Not "has every feature I imagined."

---

## Key Concept 3: The clean-clone test

The highest-value fifteen minutes in this lesson, and you should have done it once in Lesson 10.5. Do it again, because things have changed since.

```bash
cd $(mktemp -d)
git clone <repo> && cd <repo>
# follow your own README literally — no shortcuts, no prior knowledge
```

You will find things. There is always something: a tool installed months ago, an environment variable in your shell profile, a model already in your Hugging Face cache, a migration you ran by hand, a Docker network that exists on your machine.

Every one of those is a step where a reviewer stops.

**Do it twice.** The first pass reveals fixes that introduce new assumptions.

**Then have another person run it.** Watch without helping and write down every point of confusion. This is uncomfortable and it is the single most informative twenty minutes available.

---

## Key Concept 4: Depth and range

Hiring guidance suggests roughly three projects show range while staying focused. You have a natural answer.

**The spine system is the deep project** — the one that survives a forty-five-minute deep-dive. It gets full documentation.

**Two or three standalone exercises become the range.** You built eleven. Promote the ones that demonstrate a *different* skill:

- **Lesson 5.5's RAG triage** — diagnosing three seeded defects from metrics alone. Demonstrates diagnostic skill, which is rare in portfolios.
- **Lesson 11.4's vulnerable-agent exercise** — attack, fix, regression-test. Demonstrates security thinking.
- **Lesson 8.4 or 10.3** — a fine-tune with an honest negative result, or a latency halving with a reverted change. Demonstrates measurement discipline.

Each needs only a short README: problem, approach, result with numbers, what you learned. Thirty minutes each, and they add breadth without diluting the deep project.

---

## Worked Example: Two capstones reviewed

A hiring manager, four minutes each.

**Project A — "AI Research Assistant"**

README: a feature list. Multi-agent orchestration, voice input, web search, PDF parsing, a Slack integration, a Chrome extension.

Setup: `pip install -r requirements.txt` then `python main.py`. It fails on a missing environment variable that is not documented. Six variables are required; two are documented.

No evaluation. No tests. No architecture diagram. Last commit says "wip".

**Reviewer, at 90 seconds:** cannot run it, cannot verify any claim. Moves on.

**Project B — "Documentation QA Agent"**

README: what it does, a 20-second demo GIF, one architecture diagram, a results table, a "known limitations" section with four honest entries.

Setup: clone, `cp .env.example .env`, add one key, `docker compose --profile hosted up`. Working in three minutes against a seeded corpus.

`eval-report.md`: baseline through eleven changes, each with its measured effect on quality, cost, and latency. Two entries marked REVERTED with reasoning.

`docs/`: five decision records. `security-report.md`: 40 attacks, 5 fixed with regression tests, 2 accepted with reasoning.

**Reviewer, at four minutes:** ran it, read the eval report, formed a specific question about the reranking trade-off. Callback.

Project A has more features. Project B is the one that gets the interview — and the reviewer arrived with a question, which is the best possible outcome, because the deep-dive now starts from something specific rather than from "tell me about your project."

---

## Practice

**1. Run the definition-of-done checklist** honestly. Every box, yes or no. No partial credit.

**2. Triage the gaps** into must-fix and accept-and-document:
   - **Must-fix:** anything under "Runs." A system a reviewer cannot start is invisible.
   - **Usually must-fix:** anything under "Evidence." It is the differentiator.
   - **Accept-and-document:** missing features, known weaknesses, untested areas.

**3. Close the must-fix list.** Resist adding anything not on it.

**4. Run the clean-clone test twice.** Fix every failing step. Time it from clone to first working query.

**5. Get someone else to run it.** Watch silently. Write down every hesitation. Fix all of them.

**6. Write the limitations section** — four to six honest entries. From your eval report's weakest strata, your security report's accepted risks, your model card's out-of-scope notes, and your known scaling ceilings.

**7. Promote two standalone exercises** into small documented repositories. Pick ones demonstrating *different* skills from the spine project.

**8. Freeze scope.** Write down the three things you would build next, put them in the README under "future work," and stop building.

---

## Visual Aid

*A checklist rendered as four columns — Runs, Evidence, Honesty, Legible — with items as checkboxes. Each item carries two small tags: the module that produced it (e.g. "10.5", "3.5", "11.4") and the reviewer question it answers ("can I start it?", "is this real?", "do they know their own weaknesses?", "can I understand it in four minutes?"). To the right, a stopwatch graphic marked at 90 seconds ("Project A: reviewer leaves") and 4 minutes ("Project B: reviewer has a specific question"). Beneath: "Completeness and evidence beat capability."*

---

## Key Takeaways

- **Done is a checklist, not a feeling** — four categories: it runs, there is evidence, it is honest about limits, and it is legible in four minutes.
- **Two failure patterns at this stage:** scope creep, and the undocumented 90%. The second is worse, because the work is already done and invisible.
- **Completeness and evidence beat capability.** A reviewer cannot evaluate what they cannot run.
- **Run the clean-clone test twice, then watch someone else do it.** There is always something, and the second pass finds what the first pass introduced.
- **The spine system is depth; two promoted exercises are range** — pick ones showing different skills, especially diagnosis and security.

---

## Next

Your system is finished, runnable, and honest. A reviewer will now spend four minutes deciding whether to keep reading — and the code is not what they read first.

**Lesson 12.2** covers the four artifacts hiring panels actually read: a README structured for a four-minute skim, the eval report that almost no candidate has, model cards, and the decision records you have already written without realizing you were writing interview material.
