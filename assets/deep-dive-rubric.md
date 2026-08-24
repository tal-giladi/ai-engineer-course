# Project Deep-Dive Rubric

**Used in:** Lesson 12.3, practice step 5.

Score a recorded or live mock deep-dive. 50 points. Self-assessment works if you record yourself and score a day later.

**Interviewer instruction: push every claim three levels deep. Do not accept the first answer.**

---

## 1. Structure — 8 points

| Criterion | Points |
|---|---|
| Opened with the **problem**, not the technology | 2 |
| Architecture presented with **reasons**, not just component names | 2 |
| Trade-offs stated as trade-offs, including what each cost | 2 |
| Covered evaluation without being asked | 1 |
| Ended with specific next improvements | 1 |

**Common failure:** opening with "I used LangChain, Qdrant, and FastAPI." That is a technology tour and it invites a probe the candidate cannot survive.

---

## 2. Depth Under Probing — 15 points

The heaviest section. For each of the three most-probed claims, mark the highest rung reached:

| Rung | Points each |
|---|---|
| 1 — stated the claim | 1 |
| 2 — explained why | 2 |
| 3 — explained **why not the alternative**, with evidence | 3 |
| 4 — named what it cost or what broke | 4 |
| 5 — explained how they know it worked, with numbers | 5 |

Score the three claims and sum (max 15).

**Rung 3 is where most candidates stop.** Reaching rung 5 on even one claim is a strong signal.

---

## 3. Quantification — 8 points

| Criterion | Points |
|---|---|
| Used specific numbers rather than "much better" / "significantly" | 3 |
| Numbers were consistent across the conversation | 2 |
| Stated the measurement conditions (dataset size, held-out, judge calibration) | 2 |
| Distinguished a real difference from noise | 1 |

**Full marks on the last item** requires something like "that difference was inside our judge's noise band, so we didn't act on it."

---

## 4. Stories — 9 points

Award 3 points each for up to three distinct stories told with situation, action, numbers, and lesson:

☐ A diagnosed failure — localized from metrics, fixed at the right layer
☐ A change reverted because evaluation said so
☐ A negative result reported honestly
☐ An attack found against their own system
☐ A framework limitation debugged through the abstraction

**Deduct 2** if every story is a success story. A candidate with no failures either has not shipped or is not telling you about it.

---

## 5. Honesty and Calibration — 10 points

| Criterion | Points |
|---|---|
| Said "I don't know" at least once, cleanly | 3 |
| Followed it immediately with how they would find out | 2 |
| Volunteered a weakness without being asked | 3 |
| No claim collapsed under probing | 2 |

**The third item is the highest-value behavior in the round.** Volunteering "our out-of-scope stratum was all technical questions, so we missed that real users ask about pricing — online eval caught it two weeks in at 25 points below offline" demonstrates more than any success story can.

**Deduct up to 4** if any claim was walked back or contradicted under probing. That is the fastest way to lose the round.

---

## Scoring

| Total | Reading |
|---|---|
| 42–50 | Strong. Reaches rung 4–5, quantifies naturally, volunteers weaknesses. |
| 33–41 | Solid. Likely stopping at rung 3 on some claims, or thin on stories. |
| 24–32 | Needs work. Usually: no evaluation section, or every claim stops at rung 2. |
| Below 24 | Rebuild the answer from the five-part structure and re-run. |

---

## After the Mock

Write out **the ten questions you handled worst**, with the answers you wish you had given.

Then verify each answer against your artifacts. If an answer requires a number you do not have, that is not an interview problem — go and measure it. Several of the best answers in this course's worked examples came from exactly that loop.
