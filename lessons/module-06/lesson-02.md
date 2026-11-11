---
id: "06.2"
module: 6
minutes: 13
practice_minutes: 75
prerequisites: ["06.1"]
objectives:
  - "Write an agent loop by hand and explain every control-flow decision inside it."
volatility: concept
sources: []
last_verified: "2026-08-24"
---

# 06.2 · Writing the Agent Loop by Hand

You are about to write an agent in roughly forty lines with no framework. It will work.

The point is not that forty lines is impressive. The point is that those forty lines make about eight control decisions, and in a framework all eight are made for you, invisibly, by someone whose priorities were not yours. When your production agent loops forever, or stops early, or executes a tool call it should have refused, the fix lives in one of those decisions.

Interviewers know this. "What happens when the model returns a malformed tool call on step 7 of 20?" is a question that separates people who assembled a graph from people who built a system.

---

## Key Concept 1: The loop

```python
def run_agent(user_message: str, tools: list[Tool]) -> AgentResult:
    messages = [
        {"role": "system", "content": SYSTEM_PROMPT},
        {"role": "user", "content": user_message},
    ]

    while True:
        response = client.complete(messages, tools=tool_schemas(tools))
        messages.append(response.message)

        if not response.tool_calls:
            return AgentResult(answer=response.text, messages=messages)

        for call in response.tool_calls:
            result = dispatch(call)
            messages.append({
                "role": "tool",
                "tool_call_id": call.id,
                "content": format_result(result),
            })
```

Read it once more. Append the user message. Call the model with tools declared. If it returned tool calls, execute them, append the results, loop. If it returned text, we are done.

That is a working agent. It will genuinely solve multi-step problems — search, read, calculate, search again, answer.

It is also unsafe to deploy, and the rest of this module is about why.

---

## Key Concept 2: The eight decisions hiding in there

**1. Who decides it is finished?**
`if not response.tool_calls: return` hands termination entirely to the model. The model decides the task is done and you believe it. That is one valid policy. Alternatives: a verification step that checks the answer against the goal, a required "final answer" tool the model must call explicitly, or a code-side check that required sub-tasks completed. What happens when the model says done and it is not?

**2. How many tool calls per turn?**
The `for` loop executes every call the model emits. It could emit twelve. Do you cap that? Do you execute them concurrently (Lesson 6.1) or serially?

**3. What happens on a malformed tool call?**
`dispatch(call)` will raise on validation failure and the exception escapes the loop, killing the run. Should it instead return the validation error to the model as a tool result so it can correct? Almost always yes — this is Module 2's repair loop, applied to tool arguments — but that is a decision, and it needs a budget.

**4. What does the model see when a tool fails?**
`format_result` decides. Full traceback? A summary? "Tool unavailable"? This directly shapes whether the model retries sensibly, gives up, or hallucinates a result.

**5. Can the loop refuse to execute a call?**
Nothing here checks whether the requested call is permitted. Lesson 6.5.

**6. What bounds the loop?**
`while True`. Nothing. Lesson 6.4.

**7. What happens as `messages` grows?**
It grows monotonically until the context window ends the run. Lesson 6.3.

**8. What is observable?**
Nothing is recorded except the final answer. Every call should carry the `request_id` from Lesson 1.4, and the whole trajectory should be logged — otherwise you cannot debug a fifteen-step run, and Module 10's tracing has nothing to build on.

Being able to enumerate these eight, for your own implementation, is the deliverable of this lesson.

---

## Key Concept 3: Three architectures

**Plan-then-execute.** The model produces a full plan first, then the plan is executed step by step. Predictable, inspectable, cheap to bound, and easy to gate for approval. Brittle: when step 3 returns something unexpected, the plan is stale and the agent has no mechanism to adapt.

**ReAct (reason + act, interleaved).** The loop above. The model reasons, acts, observes the result, reasons again. Flexible and adaptive; harder to bound, and prone to wandering. This is what most production agents are.

**Constrained hybrid.** What actually ships. An interleaved loop with hard structure around it: a required first step, a bounded step count, a fixed set of permitted tools per phase, mandatory verification before finishing. **The constraints are the engineering.** A production agent is not a smarter loop — it is the same loop with well-chosen restrictions.

---

## Key Concept 4: Trajectories are the unit of debugging

For a RAG system, the artifact you inspect is a query and its retrieved chunks. For an agent, it is the **trajectory**: the complete ordered sequence of model messages, tool calls, arguments, results, and timings for one run.

You cannot debug an agent from its final answer. A wrong answer at step 15 might originate from a bad tool result at step 3 that the model then reasoned confidently from.

Log the trajectory from the first run. Structured, with `request_id`, per-step token counts, and per-step latency. Module 10 turns this into distributed tracing; Module 7 makes it inspectable mid-run with checkpointing. Everything downstream assumes it exists.

---

## Worked Example 1: A working multi-step run

**Question:** *"What's the rate limit for batch uploads, and how long would 5,000 items take at that rate?"*

```
step 1  model → tool_call: search_docs(query="batch upload rate limit")
        tool  → "Batch endpoints: 100 items per request, 10 requests/minute."
        [847 tokens, 1.2 s]

step 2  model → tool_call: calculator(expression="5000 / 100")
        tool  → "50"
        [1,102 tokens, 0.8 s]

step 3  model → tool_call: calculator(expression="50 / 10")
        tool  → "5"
        [1,290 tokens, 0.7 s]

step 4  model → "The batch endpoint accepts 100 items per request at
        10 requests per minute. 5,000 items would need 50 requests,
        taking approximately 5 minutes. [cites api/batch.md#limits]"
        [1,510 tokens, 1.9 s]

TERMINATED: model returned text
Total: 4 steps, 4,749 tokens, 4.6 s, $0.0071
```

This is what you want. Retrieval it could not have done alone, arithmetic it should not do in-head, a cited answer.

## Worked Example 2: The same loop, spinning

**Question:** *"What's the SLA for the enterprise tier?"* — not in the corpus.

```
step 1   search_docs("enterprise tier SLA")        → 3 irrelevant excerpts
step 2   search_docs("SLA")                        → 3 irrelevant excerpts
step 3   search_docs("service level agreement")    → 2 irrelevant excerpts
step 4   search_docs("enterprise SLA guarantee")   → 3 irrelevant excerpts
step 5   search_docs("uptime commitment")          → 1 irrelevant excerpt
step 6   search_docs("enterprise tier SLA")        → 3 irrelevant  ← repeat of step 1
step 7   search_docs("SLA terms")                  → 0 results
...
step 14  search_docs("enterprise service level")   → 2 irrelevant

STILL RUNNING. 14 steps, 38,400 tokens, 31 s, $0.058 and counting.
```

Nothing here is a bug in the ordinary sense. Every step executed correctly. The model is being *persistent* — it believes the information exists and has no mechanism to conclude otherwise, and no budget to stop it.

Three separate failures compound:

- **No termination condition** other than the model volunteering to stop
- **No repeated-action detection** — step 6 is byte-identical to step 1
- **No refusal path** — the agent has no way to conclude "this is not in the corpus," which is exactly the Lesson 5.4 behavior, now needed at the loop level

At 200 requests a day this pattern is $12/day for zero value, and it will not appear in your per-call metrics as anything unusual (Lesson 1.4).

Lesson 6.4 fixes all three.

---

## Practice

**1. Write the loop.** `spine/agent/loop.py`. No framework — no LangChain, no LangGraph, nothing. Fewer than sixty lines including trajectory logging.

**2. Log trajectories.** Structured records: step index, model message, tool calls with arguments, tool results, tokens, latency, `request_id`. Write JSONL to `telemetry/trajectories/`.

**3. Run it on five multi-step questions** that genuinely require two or more tool calls. Confirm each completes and inspect every trajectory by hand. Reading these is the point of the exercise.

**4. Reproduce the spin.** Ask something not in your corpus. Let it run — with a manual kill at step 20 so you keep your money. Record steps, tokens, cost, and how many actions were repeats.

**5. Answer the eight questions in writing.** In `docs/agent-design.md`, state your loop's current policy for each of the eight decisions, and what you intend it to be. Several answers will be "nothing, currently" — that is the honest starting point and Lessons 6.3–6.5 fill them in.

**6. Classify your architecture.** Which of the three is your loop, and which constraints would move it toward a constrained hybrid?

---

## Visual Aid

*A flowchart of the loop with each decision point numbered 1–8 in a circle. `append user message` → `call model with tools` [8: observability] → diamond `tool calls returned?` [1: who decides done]. NO → `return answer`. YES → `for each call` [2: how many per turn] → `validate` [3: malformed call] → `authorize` [5: can we refuse] — this box drawn dashed and greyed, labeled "not implemented yet" → `execute` → `format result` [4: what the model sees] → `append` [7: context growth] → back to the model, with the return edge labeled [6: what bounds this]. Three of the eight boxes are dashed grey. Caption: "The dashed boxes are the difference between this and something you can deploy."*

---

## Key Takeaways

- **An agent loop is about forty lines** — call the model with tools, execute what it asks for, append results, repeat until it returns text.
- **Those forty lines make eight control decisions.** Frameworks make all eight for you, invisibly. Knowing yours is what lets you debug production and answer interview questions.
- **Three architectures:** plan-then-execute (predictable, brittle), ReAct (flexible, unbounded), and the constrained hybrid that actually ships — where the constraints, not the loop, are the engineering.
- **The trajectory is the unit of debugging.** A wrong answer at step 15 usually originates at step 3, and the final answer tells you nothing about which.
- **A loop with no budget is not a bug you can see** in per-call metrics. Fourteen correct steps producing nothing is a cost incident with a clean error log.

---

## Next

Your loop works on short tasks. On long ones, `messages` grows with every tool result until the context window ends the run — and long before that, quality degrades as the goal drifts to the top of a context the model attends to poorly.

**Lesson 6.3** separates the agent's *state* from its *conversation*, which turns context management from an unbounded problem into a rendering decision you control.
