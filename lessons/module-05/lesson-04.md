---
id: "05.4"
module: 5
minutes: 12
practice_minutes: 75
prerequisites: ["05.3"]
objectives:
  - "Produce grounded, cited answers and make the system refuse when the context does not support an answer."
volatility: implementation
sources: []
last_verified: "2026-08-24"
---

# 05.4 · Grounding, Citations, and Knowing When to Refuse

This is the layer that separates a RAG demo from something an organization will put in front of its customers, and it is the layer most tutorials skip entirely.

A demo answers questions. A production system answers questions **it can support**, shows you **where the answer came from**, and says **"I don't know"** when the documentation does not cover it. All three of those are engineering problems with mechanical solutions, not prompt-engineering aspirations — which is the main thing this lesson has to teach.

---

## Key Concept 1: Grounding is enforced, not requested

The naive approach is a line in the prompt: *"Only use the provided context. Do not use your own knowledge."*

That helps. It does not work reliably, for a structural reason: the model has no mechanism to separate what it read in the context from what it knows from training. Both are just activations. When its training knowledge is confident and the context is ambiguous, training tends to win — and it wins *most often* precisely where it is most dangerous, such as an API version it saw thousands of times during training and that you have since changed.

So instruct, but do not rely on instruction. Enforce with structure, using the Module 2 contract:

```python
class GroundedAnswer(BaseModel):
    reasoning: str
    claims: list[Claim]           # each claim carries its supporting chunk
    answer: str
    citations: list[str]          # chunk IDs
    refused: bool
    refusal_reason: str | None
```

Asking the model to emit **claims with their supporting chunks** — before writing the prose answer — changes the task from "write an answer" to "find support, then write." The generation order matters here for exactly the reason it mattered in Lesson 2.2: the model conditions the answer on the claims it just committed to.

Then verify what came back, which is Key Concept 2.

---

## Key Concept 2: Citations must be validated mechanically

Models fabricate citations. Not occasionally — routinely, and in the most convincing possible form: a plausible filename, a plausible section, a plausible anchor. `sdk/async.md#usage` looks exactly like a real citation whether or not that file exists.

This is trivially catchable, and the fact that it is trivially catchable is the point:

```python
def validate_citations(answer: GroundedAnswer, retrieved: list[Chunk]):
    retrieved_ids = {c.id for c in retrieved}
    invalid = [c for c in answer.citations if c not in retrieved_ids]
    if invalid:
        raise CitationError(f"cited but not retrieved: {invalid}")
```

Four lines. It catches every fabricated citation of the "file does not exist" and "was not retrieved for this query" varieties — which is most of them. Route the failure into your Lesson 2.3 repair loop with the specific error, and the model corrects the great majority on the first repair.

**Two further levels, in increasing cost:**

**Span-level attribution** — the citation points to a specific span within a chunk, not just the chunk. Better for users, who can see the sentence rather than hunting a page.

**Support verification** — a second call checks whether the cited chunk actually says what the claim asserts. This catches the harder case: a *real* citation attached to a claim the document does not support. It costs a model call and is what the faithfulness metric in Lesson 5.5 measures at evaluation time. Whether to run it in production is a cost decision; running it on a sample is often the right compromise.

**Design the user-visible citation for verifiability.** A citation the user cannot click through and check is decoration. Source, section path, and ideally a quoted span.

---

## Key Concept 3: Refusal, and why it is hard

Saying "the documentation does not cover this" is the highest-value behavior in the system and the hardest to obtain.

It is hard because models are trained toward helpfulness. Refusing feels, to a model shaped by that training, like failing the task. Given a question and a context that does not answer it, the pull toward producing *something* is strong — and what it produces is fluent, confident, and wrong, which is the worst possible failure mode for a documentation assistant.

**What helps:**

**Make refusal a first-class output**, not an absence of one. The `refused: bool` field means refusing is a valid, structured completion rather than a failure to produce the expected shape.

**Give it a decision procedure, not an instruction.** "Refuse if unsure" is not actionable. This is:

```
Before answering, determine: does the provided context contain the
specific information needed to answer this question?
  - If yes: answer, citing the chunks used.
  - If it contains related but not sufficient information: set
    refused=true, and state what the documentation does cover.
  - If it contains nothing relevant: set refused=true.
Answering from general knowledge is a failure, even if correct.
```

That last sentence does real work. It reframes the objective the model is optimizing for.

**Make partial refusal available.** "The docs cover X and Y but not Z" is more useful than a flat no, and gives the model an honest option that still feels helpful.

**Test it explicitly.** This is why your Lesson 3.2 golden set has an out-of-scope stratum at 13% — refusal is invisible in ordinary traffic and degrades silently.

---

## Key Concept 4: Refusal is two-sided

A system that refuses everything is perfectly faithful and completely useless. Track both directions:

- **Missed refusal rate** — answered a question it should have refused. Produces confident wrong answers. The dangerous one.
- **False refusal rate** — refused a question it could have answered. Produces a useless assistant. The annoying one.

They trade against each other, and the correct operating point is a product decision that your Lesson 3.1 charter should already have decided. For a documentation assistant serving customers, missed refusals are worse. For an internal tool used by engineers who can verify answers themselves, over-refusal is more costly.

Report both, always. A single "refusal rate" hides the trade entirely.

---

## Worked Example 1: Ungrounded vs grounded

**Question:** "Does the SDK support async operations?"
**Retrieved chunk (`sdk/roadmap.md#planned`):** *"Async support is planned for a future release. Currently all SDK operations are synchronous."*

**Ungrounded:**

> Yes, the SDK provides full async support. Use `AsyncClient` for non-blocking operations, and all methods have `async` variants. See `sdk/async.md`.
> — *cites `sdk/async.md`*

Fluent, specific, entirely wrong, and citing a file that does not exist. This is Q4 from Lesson 5.1.

**Grounded, with validation:**

> The SDK does not currently support async operations — all operations are synchronous. Async support is listed as planned for a future release.
> — *cites `sdk/roadmap.md#planned`*

The four-line citation check would have caught the first answer before it reached a user.

## Worked Example 2: Two refusal calibrations

Same system, two prompt configurations, evaluated on the full golden set.

| | Config A (permissive) | Config B (strict) |
|---|---|---|
| Answer accuracy (in-scope) | 0.87 | 0.84 |
| **Missed refusal rate** | **0.58** | **0.06** |
| **False refusal rate** | **0.02** | **0.19** |
| Overall quality (judge) | 0.71 | 0.83 |

Config A answers 58% of out-of-scope questions from general knowledge. Config B refuses 19% of answerable ones.

For a customer-facing documentation assistant, **B is clearly correct** — a confident wrong answer about pricing or security creates real problems, while an unnecessary refusal creates mild friction. The charter said so in Lesson 3.1, which is why that document exists.

The interesting refinement: Config B's false refusals were concentrated in one stratum — ambiguous queries, where the model refused rather than answering the most likely interpretation. The fix was not loosening refusal globally but adding a clarification path: answer the likely interpretation *and* state which case it covers. That moved false refusal to 0.09 with missed refusal unchanged, which is a better system than either A or B.

---

## Practice

**1. Extend the answer contract** with `claims`, `citations`, `refused`, and `refusal_reason`, generated in that order.

**2. Implement citation validation.** Reject any citation not in the retrieved set, route to the Lesson 2.3 repair loop with a specific error, and record the pre-repair fabrication rate. That number is worth knowing.

**3. Implement refusal** with a decision procedure, not an instruction. Include the partial-refusal option.

**4. Measure both directions.** On the out-of-scope stratum, missed refusal rate. On the in-scope strata, false refusal rate. Report separately.

**5. Calibrate against your charter.** Produce a permissive and a strict configuration, measure both, and choose deliberately with the charter's priority ordering as your justification. Write down the reasoning.

**6. Look at where false refusals cluster.** If they concentrate in one stratum — ambiguous queries usually — the fix is targeted, not a global loosening. Try it.

**7. Add a fabrication regression test.** A golden-set case that previously produced a fabricated citation, asserted to fail if it returns. This is your first security-adjacent regression test and the pattern generalizes in Module 11.

---

## Visual Aid

*An answer rendered as it would appear to a user, with each sentence connected by a thin line to a card on the right representing its source chunk, showing source path and the supporting quoted span. Three sentences connect cleanly to green cards. One sentence connects to a red-outlined card marked "CITED BUT NOT RETRIEVED — `sdk/async.md`" with an X over it, and the sentence itself is struck through. Below, a small two-bar gauge labeled "refusal calibration" showing missed-refusal and false-refusal rates with the chosen operating point marked. Caption: "Four lines of validation catch the red card. Everything else is prompt work."*

---

## Key Takeaways

- **Grounding is enforced through structure, not requested in a prompt.** Ask for claims-with-support before the prose answer; training knowledge otherwise wins where it is most confidently wrong.
- **Validate every citation against the retrieved set** — four lines, catches most fabrication, and feeds the repair loop you already built.
- **Refusal needs a decision procedure and a first-class output field**, plus an explicit reframe that answering from general knowledge is a failure even when correct.
- **Track both missed refusals and false refusals.** They trade against each other and a single number hides the trade; your charter decides the operating point.
- **Cluster your false refusals before loosening anything.** They usually concentrate in one stratum and have a targeted fix.

---

## Next

Every change in this module was a claim: rewriting improved recall, reranking improved precision, grounding improved faithfulness. Your Module 3 harness can score a general notion of quality, but it cannot yet tell you *which stage* a wrong answer came from.

**Lesson 5.5** adds the four RAG-specific metrics — and teaches them as a diagnostic instrument rather than a scorecard, because the combination of the four localizes the failure automatically. That reading skill is the one that gets tested in an AI system design round.
