---
id: "06.3"
module: 6
minutes: 12
practice_minutes: 90
prerequisites: ["06.2"]
objectives:
  - "Manage state, memory, and a shrinking context window across a long-running task."
volatility: implementation
sources: []
last_verified: "2026-08-24"
---

# 06.3 · State, Memory, and Context Management

Your agent's conversation grows every step. Its context window does not. That is not a performance concern — it is a correctness ceiling, and it arrives sooner than you expect.

Worse, failure is gradual before it is total. Long before you hit the window limit, the goal has drifted to the top of a long context where the model attends to it poorly (Lesson 1.3), early tool results are crowding out recent ones, and the agent starts making decisions that would look obviously wrong to someone who could see the whole picture.

The fix starts with a distinction that most implementations never make.

---

## Key Concept 1: State is not conversation

In the Lesson 6.2 loop there is exactly one data structure: `messages`. It is simultaneously the agent's memory, its working notes, its audit log, and the thing sent to the model. Because it is all four, none of them can be managed independently.

Separate them:

**State** is the structured truth of the task, and it lives in *your* code:

```python
@dataclass
class AgentState:
    goal: str
    findings: list[Finding]          # what has been learned, with sources
    open_questions: list[str]        # what is still unknown
    completed_steps: list[StepSummary]
    artifacts: dict[str, Any]        # large results, held by handle
    budget_used: Budget
```

**Conversation** is what the model sees, and it is a **rendering of state**, regenerated each turn rather than accumulated:

```python
def render_messages(state: AgentState, recent: list[Message]) -> list[Message]:
    return [
        {"role": "system", "content": SYSTEM_PROMPT},
        {"role": "user", "content": render_goal_and_findings(state)},
        *recent[-6:],                # the last few raw exchanges
    ]
```

Once you make this split, context management stops being an unbounded problem and becomes a rendering decision. You choose what the model sees, every turn, from a complete structured record you still hold.

This is also the shape LangGraph enforces in Module 7 — a typed state object threaded through nodes. Building it by hand first means the framework will look like a familiar idea with better infrastructure, rather than a new concept.

---

## Key Concept 2: The techniques

**Rolling summarization.** Compress older turns into a running brief. Keep the last few exchanges raw, summarize everything before them into structured findings. Costs a model call periodically; typically buys back most of the window.

**Tool-result externalization.** A tool returns 8,000 tokens. Store the payload, put a handle in the conversation:

```
search_results(id="r_8891", count=47, top_3_summarized="...")
```

Add a `fetch_result(id, filter)` tool so the model can pull specific parts on demand. The full data is still available; it is just not resident. This generalizes the capping from Lesson 6.1.

**A durable scratchpad.** A file or state field the agent explicitly reads and writes: notes, intermediate conclusions, a checklist. Unlike conversation, it is compact and organized, and it survives summarization intact — which is exactly why it is worth having.

**Goal re-injection.** In a long context the original goal is at the top, in the region models attend to least. Re-render it near the end of the context on every turn. This is one line and measurably reduces drift.

**Selective retention.** Not all history is equal. A tool result that produced a finding matters; five failed searches do not. Prune deliberately: keep results that changed the state, summarize the rest to a single line.

---

## Key Concept 3: Three tiers of memory

**Working memory** — this run. The state object above. Dies when the run ends.

**Session memory** — this conversation, across runs. What the user asked earlier, preferences they stated, corrections they made. Usually a summary plus the last N turns, persisted per conversation.

**Long-term memory** — across sessions. Facts about the user, past decisions, accumulated context. This is Module 4's retrieval pipeline pointed at the agent's own history: store interactions, embed them, retrieve relevant ones at the start of a new session.

Two cautions worth stating now, because they become Module 11 problems:

- **Long-term memory is a security surface.** Content written into memory in one session is retrieved into context in another — indirect prompt injection with persistence (Lesson 11.1).
- **Memory is PII.** It is a store of user content with a retention policy you have to decide (Lesson 11.3).

---

## Key Concept 4: Summarization is lossy — decide what you lose

Every technique above discards information. Summarization discards the most, and it discards *unpredictably* — the model summarizing turn 4 does not know that a version number mentioned in passing will matter at turn 17.

Be deliberate rather than hopeful:

- **Summarize with a schema.** Instead of "summarize the conversation so far," extract into structured findings with sources. Structure resists drift better than prose.
- **Never summarize identifiers.** IDs, versions, error codes, filenames, exact figures: carry them forward verbatim in a dedicated state field. These are precisely what summaries lose and what later steps need.
- **Keep the raw trajectory.** Summarize what the *model* sees; keep everything for debugging. You will need it when a run goes wrong.
- **Measure it.** Run a long task with and without summarization and compare outcomes, not just token counts.

---

## Worked Example: A 20-step task, before and after

**Task:** *"Compare the rate limits, auth methods, and error handling across all three API versions, and summarize what changed."* Genuinely needs ~20 tool calls.

**Before — naive accumulation:**

| Step | Context tokens | Note |
|---|---|---|
| 1 | 1,200 | |
| 5 | 18,400 | three large search results resident |
| 8 | 34,900 | |
| 11 | 51,200 | |
| 12 | — | **context window exceeded, run failed** |

Eleven steps of correct work, discarded. Cost: $0.11 for nothing.

**After — state separation, summarization, externalization, goal re-injection:**

| Step | Context tokens | Note |
|---|---|---|
| 1 | 1,200 | |
| 5 | 8,900 | results externalized to handles |
| 8 | 11,200 | |
| 11 | 12,400 | summarization triggered at step 10 |
| 15 | 13,100 | |
| 19 | 14,800 | **completed** |

Peak context 14,800 against a 128k window — 12%. Nineteen steps, $0.09, correct answer.

**And the loss, shown rather than hidden.** The step-10 summarization compressed six early findings into three lines. One detail was dropped: v1's rate limit was per-key, while v2's is per-account. At step 17 the agent stated the v1 limit without the qualifier.

The fix was not "summarize better." It was a schema change — a `qualifiers: list[str]` field on `Finding`, carried verbatim through summarization:

```python
@dataclass
class Finding:
    claim: str
    source: str
    identifiers: list[str]   # never summarized
    qualifiers: list[str]    # never summarized
```

Re-run: correct, at 15,100 peak tokens. Three hundred extra tokens bought back the detail.

That is the lesson in miniature — **summarization loss is a schema problem, not a prompting problem.**

---

## Practice

**1. Split state from conversation.** Define `AgentState`, and make `render_messages(state, recent)` the only thing that constructs what the model sees. Your loop should never append to a growing message list again.

**2. Externalize large tool results.** Store payloads, return handles with a summary and a count. Add a `fetch_result` tool for retrieving parts on demand.

**3. Add rolling summarization** with a **structured** output — findings with claims, sources, identifiers, and qualifiers. Trigger on a context threshold (e.g. 30% of the window), not on a step count.

**4. Re-inject the goal** near the end of the rendered context every turn.

**5. Instrument context composition.** Log per step: total tokens, and the breakdown across system, goal/findings, recent messages, and tool results. This is what the visual aid plots.

**6. Run the 20-step task before and after.** Report peak context, step count, cost, and outcome for both.

**7. Find your summarization loss.** Diff the state before and after a summarization pass and identify at least one thing that was dropped. Then fix it with a schema field rather than a prompt change, and document it in `docs/agent-design.md`. If you find nothing lost, your task is not long enough — make it harder.

---

## Visual Aid

*Two stacked-area charts side by side, x-axis "step 1–20," y-axis "context tokens," with a horizontal red line at the window limit. Left, "naive": four bands (system, goal, conversation history, tool results) growing steadily, with tool results dominating; the stack crosses the red line at step 12 and the chart stops with an X. Right, "managed": the same four bands staying roughly flat around 12–15k, with a visible notch at step 10 where the history band drops sharply (annotated "summarization") and small step-downs where tool results are externalized (annotated "→ handle"). Below the right chart, a small callout: "one detail lost at step 10 — fixed with a schema field, not a better prompt."*

---

## Key Takeaways

- **Separate state from conversation.** State is structured truth in your code; conversation is a rendering of it, regenerated each turn. This turns context management from unbounded growth into a decision you control.
- **Four techniques:** rolling summarization, tool-result externalization by handle, a durable scratchpad, and goal re-injection near the end of the context.
- **Three memory tiers** — working, session, long-term — and the long-term tier is both an injection surface (Module 11) and a PII store with a retention policy.
- **Summarization is lossy and loses unpredictably.** Summarize into a schema, never summarize identifiers or qualifiers, keep the raw trajectory for debugging.
- **Summarization loss is fixed with schema fields, not better prompts.**

---

## Next

Your agent survives long tasks. It does not survive a tool that returns a 500, a model that emits a malformed call at step 7, or a question whose answer does not exist — the fourteen-step spin from Lesson 6.2 is still entirely possible and still costs real money.

**Lesson 6.4** bounds every dimension, classifies every error, and makes every exit distinguishable — which is the lesson that turns your agent from a demo into something you could deploy.
