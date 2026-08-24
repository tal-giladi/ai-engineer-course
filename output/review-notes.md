# Course Review Notes — AI Engineer

**Phase 5 review.** Reviewed: 1 outline, 12 module plans, 53 lessons, 12 quizzes, 1 capstone rubric, 4 supporting assets.
**Reviewer:** solo build, self-review against `CLAUDE.md` rules and the brief.

---

## 1. Verification Performed

These were checked mechanically, not by impression.

| Check | Method | Result |
|---|---|---|
| Lesson count matches plans | Directory count per module vs plan | **PASS** — 4/4/5/4/5/5/4/4/4/5/5/4 = 53 |
| Every module LO referenced by a lesson | Diff of `M#-LO#` tokens between plans and lessons | **PASS** — all 51 objectives covered, no orphans |
| Every lesson maps to ≥1 objective (`CLAUDE.md` Rule 1) | `**Objective:**` header present in all 53 | **PASS** |
| Lesson word count 1,000–2,000 (Rule: Phase 3.4) | `wc -w` per file | **1 exception** — see §3 |
| Required lesson sections | Grep for Key Takeaways, Visual Aid | **PASS** — all 53 have both |
| Module duration 30–60 min (Rule 4) | Plan headers | **PASS** — 50–60 min, 12 modules ≈ 11 h |
| Referenced files exist | Grep for `assets/` and `exercises/` paths | **1 gap resolved, 1 open** — see §2 |

---

## 2. Open Gap — Supplied Exercise Code

**This is the one substantive thing not done, and it is deliberate rather than overlooked.**

Eight lessons and quiz practicals hand the student a pre-built artifact — a broken RAG pipeline to diagnose, a vulnerable agent to attack, an SFT checkpoint with a verbosity problem. These are course *infrastructure*, and none of them exist as code.

| Path | Needed by | Nature |
|---|---|---|
| `m2-prose-extractor` | Lesson 2.1 | Python, small |
| `m2-adversarial-inputs` | Lesson 2.3 | Data + harness, small |
| `m3-summarizer-eval` | Lesson 3.2, quiz Q10 | Python, small |
| `m5-rag-triage` | Module 5 quiz Q9 | Full pipeline, **large** |
| `m8-decision` | Lesson 8.1 | Documents only, trivial |
| `m9-verbosity-fix` | Module 9 quiz Q8 | Requires GPU to prepare, **large** |
| `m10-latency-halving` | Module 10 quiz Q9 | Instrumented service, medium |
| `m11-vulnerable-agent` | Module 11 quiz Q9 | Full agent, **large** |

**Mitigation shipped:** `assets/exercise-specs.md` specifies each one precisely — required failure profiles, expected metric signatures, seeded defects, and the design constraints that make each exercise teach what it is supposed to teach. It also gives a build order and an estimate (~6–9 engineering days, half of it in the three large ones).

**Impact if not built:** the course is fully teachable — every lesson's Concept, Example, and Summary stand alone, and students can substitute their own systems for most practicals. But five practicals lose their diagnostic bite, because "diagnose three defects you introduced yourself" is a much weaker exercise than "diagnose three defects someone else hid."

**Recommendation:** build `m5-rag-triage` and `m11-vulnerable-agent` before first delivery. They carry the two most distinctive skills in the course and the two most distinctive portfolio artifacts. The other six can follow.

The five remaining `exercises/` paths are student *output* directories and correctly require nothing supplied.

---

## 3. Findings and Fixes

### Fixed during review

**F1 — Missing referenced assets.** Three rubrics were referenced in Module 12 lessons and did not exist: `assets/review-rubric.md`, `deep-dive-rubric.md`, `system-design-rubric.md`. **All three written.** They are now scored instruments rather than prose suggestions, which also makes Module 12's practice steps gradeable.

**F2 — Wrong cross-references in my own spec file.** `exercise-specs.md` cited "Lesson 3.10", "Lesson 5.9", "Lesson 9.8", "Lesson 10.9", "Lesson 11.9" — these are quiz question numbers, not lessons, and no such lessons exist. **Corrected to "Module N quiz Q#".**

**F3 — Brief rendered incorrectly.** `brief/course-brief.md` had every markdown character backslash-escaped (`\#`, `\-`, `\~`), so it rendered as literal text. **Unescaped and blank lines collapsed.** Content unchanged.

### Accepted, with reasoning

**A1 — Lesson 12.4 is 2,252 words, over the 2,000 target.** It carries both the seven-step system design framework and the course conclusion. Splitting it would create a 400-word closing lesson, which is worse. **Accepted.** All other 52 lessons fall within 1,434–1,996.

**A2 — 12 modules exceeds `CLAUDE.md`'s "5–8 recommended".** The brief explicitly specifies 8–12 modules and the user approved a two-module fine-tuning track. The brief's explicit duration governs a general recommendation. **Accepted and documented** in the outline §7.

**A3 — Module 12 quiz is 7 questions, below the 5–10 midpoint.** Its real assessment is the capstone rubric (100 points, six dimensions). The quiz covers only the judgment around the artifact. **Accepted.**

---

## 4. Consistency Across Modules

**Verified as coherent:**

- **The correlation ID thread.** `request_id` is introduced in Lesson 1.4 with an explicit promise ("this will matter later"), used to group agent calls in 6.2, becomes the checkpoint thread ID in 7.2, and becomes the trace ID in 10.4. Every reference is consistent and each one calls back to the original.
- **The seam argument.** Introduced in 1.2 for model providers, reused for vector stores in 4.3, for tools via MCP in 7.4, for serving in 10.2, and for fallback routing in 11.5. Each later use explicitly names the earlier one.
- **The eval harness accretion.** Built in Module 3, extended with RAG metrics in 5.5, trajectory scoring in 6.4, used to prove the framework port in 7.1, the fine-tune in 8.4, every optimization in 10.3, and as the security regression suite in 11.4.
- **Numbers.** Worked-example figures are internally consistent where they recur — the 8,400 ms budget in 10.3 matches the trace in 10.4; the 0.61 → 0.87 chunking progression in 4.2 matches the README results table in 12.2; the refusal collapse of −0.45 in 8.4 matches its citation in the Module 8 quiz.

**One deliberate repetition:** the "more context is worse" finding appears in 1.3 (as context-window mechanics), 5.3 (measured, with a sweep), and 10.2 (as a serving-capacity consequence). This is reinforcement across three different framings, not redundancy — each adds a distinct consequence.

---

## 5. Difficulty Progression

Progression was checked against the audience — experienced developers, new to this stack.

| Modules | Cognitive load | Notes |
|---|---|---|
| 1–2 | Low | Familiar patterns (API clients, validation, config) applied to a new dependency. Deliberate on-ramp. |
| 3 | **Medium-high** | The hardest conceptual module, placed third on purpose. Nothing here is technically difficult; the discipline is. |
| 4–5 | Medium | New domain knowledge (embeddings, chunking, reranking) but concrete and measurable. |
| 6 | **High** | Peak difficulty. Five lessons, agent loop from scratch, state management, failure handling, sandboxing. |
| 7 | Medium | Deliberate relief after 6 — the port is a mapping onto something already understood. |
| 8–9 | Medium-high | New domain (training), but the decision framing keeps it grounded. |
| 10 | **Low-medium for this audience** | Their home ground. Converts existing seniority into new capability. |
| 11 | Medium-high | New threat model, familiar controls. |
| 12 | Low technical, high reflective | Synthesis and rehearsal. |

**Assessment: the curve is sound.** The 6 → 7 relief after the peak is intentional and works. Module 3's early placement is the course's most consequential structural decision and is defended in the lesson itself.

**One risk flagged:** Module 6 is genuinely demanding (five lessons, ~60 min of instruction, considerably more of practice). Students who skipped Lesson 6.3's state/conversation separation will find Module 7 confusing, because LangGraph *requires* that separation. This dependency is stated in both 6.3's closing and 7.1's opening, but it is the most likely place for a student to get stuck.

---

## 6. Gaps in Coverage — Accepted

Documented in the outline §9 and re-verified here:

- **Kubernetes and cloud-managed AI services.** Named in many postings, out of scope. This audience already deploys containers; the time is invested in AI-specific depth. Module 10.5 notes the porting path.
- **Voice and browser agents, graph databases, RLHF/RFT internals.** Mentioned as landscape where relevant (9.1 covers RFT/RLVR by shape), not taught hands-on.
- **Multi-modal.** Not covered. A growing requirement; the strongest candidate for a future module 13.

**One gap identified during review that was not previously documented:** the course teaches evaluation of *English* systems throughout. Multilingual retrieval is mentioned in 4.1 as an embedding-selection axis and nowhere else. For a course aimed at an international audience this is a real limitation and is now noted in the summary's limitations section.

---

## 7. Alignment With the Brief

| Brief requirement | Status |
|---|---|
| Experienced developers from other backgrounds | Every module assumes systems fluency and no ML background. Module 10 explicitly leverages existing seniority. |
| Prerequisites honored (Docker, coding agents, architecture) | Docker used from Module 4; architecture reasoning throughout; coding agents referenced in 7.4 as an MCP host. |
| Local Docker, HF paid, Colab paid | All three used. Colab GPU in 8.3 and 9.2–9.3; HF in 4.1, 8.3, 9.4; Docker throughout. |
| **"Do a little research job before you start"** | Job-market research conducted before the outline; six findings drive the module order; sources cited in outline §11; coverage matrix in §9. |
| **"Students can apply for professional AI engineer jobs"** | Module 12 is dedicated to this. Every artifact from Modules 3–11 is reused as portfolio or interview material. |
| 8–12 modules, ~8–12 hours | 12 modules, ~11 hours. |
| "Going solo, minor changes allowed" | Duration and module count adjusted within the stated range; two-module fine-tuning track per user decision. |

---

## 8. Recommendations, Prioritized

1. **Build `m5-rag-triage` and `m11-vulnerable-agent`** before first delivery (§2). Highest teaching value per engineering hour.
2. **Pilot Module 6 with two or three students and time it.** It is the peak-difficulty module and the most likely stall point.
3. **Add a multilingual note to Lesson 4.1's practice** — one line asking students whose corpus is not English-only to include it as a benchmark axis. Cheap, closes a real gap.
4. **Consider a Module 13 on multi-modal** if the field's hiring requirements continue in that direction. Do not add it speculatively.
5. **Re-verify the job-market claims annually.** The research underpinning §4 of the outline is dated August 2026; hiring patterns in this field move fast, and stale percentages in a course about evidence-based decisions would be a poor look.
