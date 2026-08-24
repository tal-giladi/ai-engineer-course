# Cold Review Rubric — Four Minutes on a Repository

**Used in:** Lesson 12.2, practice step 8.

Give this to a peer, or use it on yourself after a day away. **Set a four-minute timer.** Do not let the author explain anything — the whole point is that the repository has to work without them.

Answer every question from the repository alone. Where you cannot, that is the finding.

---

## Minute 1 — Orientation

**1. Within 15 seconds, can you say what this system does?**
☐ Yes — write it in one sentence: ____________________
☐ No — you had to read past the first paragraph
☐ No — still unclear after a minute

**2. Is there visual evidence it works** (GIF, screenshot, recorded demo)?
☐ Yes ☐ No

**3. Who is it for, and what problem does it solve?**
☐ Stated ☐ Inferable ☐ Absent

---

## Minute 2 — Can you run it?

**4. Follow the setup instructions literally. Change nothing. Add nothing from your own knowledge.**

Record every step that failed or required something not in the README:

| Step | What was missing |
|---|---|
| | |

**5. Time from `git clone` to first working query:** ______

**6. Does it require a GPU?** ☐ No — CPU path documented ☐ Yes — no alternative offered

**7. Was there data to query on first run?** ☐ Yes, seeded ☐ No — empty index

---

## Minute 3 — Is any of it real?

**8. Pick the boldest claim in the README. Write it here:** ____________________

Can you find a number backing it?
☐ Yes — where: ____________________
☐ No

**9. Is there an evaluation report?**
☐ Yes, with per-stratum results ☐ Yes, aggregate only ☐ No

**10. Does the evaluation state how the judge was calibrated?**
☐ Yes, with agreement rate ☐ Mentioned, no number ☐ No judge, or not stated

**11. Is there any entry showing a change that was tried and reverted?**
☐ Yes ☐ No

**12. Can you tell why the significant architectural choices were made?**
☐ Yes — decision records ☐ Partially ☐ No

---

## Minute 4 — Judgment

**13. Does the author state limitations?**
☐ Yes, specific and credible ☐ Vague ☐ None

**14. Is there evidence they tested their own system adversarially** (security report, red-team findings)?
☐ Yes ☐ No

**15. Anything that reads as overclaiming** — a capability asserted without evidence?
Note it: ____________________

---

## The Two Questions That Matter

**16. What specific question would you open an interview with?**

____________________________________________

*If your honest answer is "what does this actually do?", the repository has failed its primary job regardless of the code inside it. If it is a specific question about a trade-off, the repository has succeeded.*

**17. Would you keep reading past four minutes?**
☐ Yes ☐ No — and at what point did you decide: ______

---

## Feedback to the Author

**Three things to fix first**, in priority order. Anything under "can you run it" outranks everything else.

1.
2.
3.

**One thing that was notably good:**

---

## For the Author Receiving This

Do not explain. Do not defend. Every point of confusion is a defect in the documentation, not in the reviewer.

The most valuable items are 4, 8, and 16. If setup failed, fix that before anything else. If your boldest claim has no number, either measure it or delete the claim. If item 16 came back as "what does this do?", rewrite the first three lines of the README.
