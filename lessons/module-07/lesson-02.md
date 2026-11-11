---
id: "07.2"
module: 7
minutes: 13
practice_minutes: 90
prerequisites: ["07.1"]
objectives:
  - "Use checkpointing, interrupts, and human-in-the-loop approval for consequential actions."
volatility: implementation
sources:
  - title: "LangGraph interrupts (interrupt(), Command(resume=...), static interrupts for debugging, re-execution on resume)"
    url: https://docs.langchain.com/oss/python/langgraph/interrupts
  - title: "LangGraph 1.2.20 source, langgraph/types.py (interrupt(value, *, response_schema=None))"
    url: https://pypi.org/project/langgraph/1.2.20/
  - title: "LangGraph persistence and time travel"
    url: https://docs.langchain.com/oss/python/langgraph/persistence
last_verified: "2026-11-11"
---

# 07.2 · Checkpointing, Interrupts, and Human-in-the-Loop

Lesson 6.5 left you with an unsatisfying approval gate: it worked, and it held a process open for as long as the human took to respond. If that human went to lunch, a worker was tied up for an hour. If the process restarted, fourteen steps of work vanished.

Lesson 7.1 left you with a ported agent and no clear payoff yet — the same behavior, less code.

This lesson is the payoff for both. Checkpointing turns an agent run from a fragile in-memory process into something with the durability properties of a workflow engine, and interrupts turn approval from a blocking call into a state the system can sit in for a week.

It also gives you the best debugging tool in this field, which is worth the lesson on its own.

---

## Key Concept 1: Checkpointing

A **checkpointer** persists the graph's state after every node execution, keyed by a **thread ID**.

```python
from langgraph.checkpoint.postgres import PostgresSaver

graph = builder.compile(checkpointer=PostgresSaver(conn))

config = {"configurable": {"thread_id": request_id}}
result = graph.invoke({"messages": [...], "goal": goal}, config)
```

One argument. Four properties follow:

**Durability.** The process dies at step 11; the run resumes at step 11. Deploys, crashes, and scale-down events stop destroying work.

**Resumability.** Invoke the same thread ID again and execution continues from the last checkpoint rather than starting over.

**Inspectability.** Read the state of a running or finished agent at any step, from outside the process.

**Time travel.** Rewind to any checkpoint, change state, and re-run from there. Key Concept 4.

**Use the `request_id` from Lesson 1.4 as the thread ID.** It already groups every model call in a request; making it the persistence key means one identifier ties together telemetry, traces (Module 10), and checkpoints. Do not invent a second one.

---

## Key Concept 2: Interrupts

An interrupt pauses execution at a defined point and returns control to the caller. *API checked against LangGraph 1.2.20 on 2026-11-11.* You call `interrupt()` inside the node that needs a human, and resume with `Command(resume=...)`:

```python
from pydantic import BaseModel
from langgraph.types import interrupt, Command

class Approval(BaseModel):
    approved: bool
    edited_to: str | None = None          # edit-and-continue (Key Concept 3)

def approve_email(state: State):
    decision = interrupt(
        {"action": "send_email", "args": state["draft"], "reasoning": state["why"]},
        response_schema=Approval,          # optional, new in 1.2: typed, validated resume value
    )
    return {"approved": decision.approved,
            "draft": decision.edited_to or state["draft"]}

graph = builder.compile(checkpointer=checkpointer)

result = graph.invoke(input, config)
if "__interrupt__" in result:
    # process can now exit entirely
    pending = result["__interrupt__"][0]
    await queue_for_approval(pending.value, thread_id=config["configurable"]["thread_id"])
```

The run is now *paused state in a database*, not a blocked process. Approval arrives an hour later, from a different process, possibly a different machine:

```python
result = graph.invoke(Command(resume={"approved": True}), config)   # resumes the paused node
```

`interrupt()` returns the resume value into the node, so the human's answer is ordinary state. With `response_schema=` set to a Pydantic model, the resume value is validated and an invalid answer raises without saving anything, so it can be answered again; with a plain JSON Schema `dict`, the schema is only shown to clients and nothing is validated.

**One rule that changes how you write the node: on resume, the node re-runs from its first line.** `interrupt()` then returns the resume value instead of pausing. Anything before the `interrupt()` call executes twice, so keep side effects after it or in the next node, or make them idempotent (Key Concept 5).

**Static breakpoints are for debugging.** `compile(interrupt_before=[...])` and `interrupt_after=[...]` still exist, but LangGraph's documentation says they are not recommended for human-in-the-loop workflows; use them to step through a graph node by node while debugging.

This is what Lesson 6.5 wanted and could not express.

---

## Key Concept 3: The human-in-the-loop patterns

**Approve / reject.** The gate from Lesson 6.5, now durable. Show arguments *and* reasoning — that requirement has not changed and it is still the part that makes the gate meaningful.

**Edit and continue.** More useful than approve/reject and often overlooked. The human corrects an argument — a recipient, an amount, a query — and the run continues with the corrected state: the correction arrives as the resume value (`edited_to` above) and the node writes it into state.

**Review and steer.** An `interrupt()` in a review node placed right after planning lets a human adjust the plan before execution, which is far cheaper than letting a bad plan run and correcting after.

**Escalation on a signal.** Interrupt conditionally: on low confidence, on high value, on an unusual tool. Better than gating a whole tool class, because it preserves the "gates must be rare" property from Lesson 6.5 while still catching the cases that matter.

---

## Key Concept 4: Time travel

This is the feature to reach for when a trajectory goes wrong, and the one that changes how you debug.

```python
history = list(graph.get_state_history(config))
# pick the checkpoint just before things went wrong
target = history[7]

new_config = graph.update_state(target.config, {"findings": corrected_findings})
result = graph.invoke(None, new_config)         # re-runs from step 7 with the edit
```

Why it matters: agent bugs are almost always **causal**, not local. A wrong answer at step 15 originates from a bad tool result at step 3 that the model reasoned confidently from (Lesson 6.2). Without time travel you re-run the whole task and hope the non-determinism reproduces it. With it, you rewind to step 3, change one thing, and see whether the outcome changes — an actual controlled experiment.

Practical uses:

- **Bisect a failure** — rewind to different steps to find where it went wrong
- **Test a hypothesis** — "if the search had returned the right chunk, would it have answered correctly?" Edit the state and find out
- **Branch** — rewinding creates an alternate history; you can compare two continuations from the same point
- **Recover a production run** — fix the state, resume, rather than restarting a run that already cost $2

---

## Key Concept 5: Checkpoints are state you now own

The operational reality, which is easy to skip and expensive to skip.

**Volume.** A checkpoint per node per run adds up quickly. A 15-step agent at 10,000 runs a day is 150,000 checkpoint writes, each containing full state including message history.

**Retention.** Decide it explicitly. Completed runs: hours or days. Interrupted runs awaiting approval: long enough for a human to actually respond, plus a timeout policy for the ones nobody answers. Implement expiry; do not rely on intent.

**Sensitivity.** Checkpoints contain the full conversation — user input, retrieved documents, tool results. That is PII and possibly customer data, in a new store, with the same protection requirements as your primary database. Lesson 11.3 audits this; note it now.

**Resume semantics.** A node that already executed a side effect and then crashed will re-execute on resume — and so will any code before an `interrupt()` call, every time the node is resumed. **Idempotency keys on tools with side effects are not optional once you have resumability.** This is ordinary distributed-systems discipline and it arrives here for the first time.

---

## Worked Example 1: Three demonstrations

**Approve and resume.**

```
step 6  → approve_email: interrupt({action: send_email, ...})
          process exits. State persisted, thread_id=req_8f3a.

        [47 minutes pass. Service redeployed twice.]

        approval received
        → invoke(Command(resume={approved: True}), config)

step 7  send_email executed
step 8  → COMPLETED
```

Forty-seven minutes and two deploys, zero processes held open, zero work lost.

**Resume after crash.**

```
step 11 → checkpoint written
        → kill -9
        → invoke(None, config)
step 12 → continues, full findings intact
```

Before checkpointing: eleven steps and $0.06 discarded, run restarted from scratch.

**Time travel to diagnose.**

An agent produced a wrong version number at step 15. History shows the state at each step. Rewinding to step 3 and inspecting: `search_docs` returned a v2 changelog entry for a v3 query — a retrieval failure, not a reasoning failure.

Hypothesis test: edit step 3's state to the correct chunk, resume.

```
result: correct answer at step 12
```

Confirmed — the bug is in retrieval, not the agent. Total time: four minutes. Without time travel this is an afternoon of re-running a non-deterministic 15-step task hoping to reproduce.

## Worked Example 2: The idempotency bug that resumability creates

A refund tool executes at step 9. The process crashes *after* the refund succeeds but *before* the checkpoint is written.

On resume, step 9 re-executes. **Two refunds.**

Fix: an idempotency key derived from the thread ID and step index, checked by the tool.

```python
def issue_refund(args, *, idempotency_key: str):
    if already_processed(idempotency_key):
        return prior_result(idempotency_key)
    ...
```

This bug did not exist before checkpointing, because there was no resume. **Every capability has a failure mode**, and this is the one durability introduces.

---

## Practice

**1. Add a checkpointer.** Postgres-backed, using `request_id` as the thread ID.

**2. Convert your approval gate to `interrupt()`.** Call it inside an approval node with `response_schema=` set to a Pydantic model of the decision, and resume with `Command(resume=...)`. The process must be able to exit while awaiting approval. Verify by actually killing it, then resume with an invalid answer (a missing field) and confirm it is rejected and can be answered again.

**3. Demonstrate approve-and-resume** across a process restart. Record the elapsed time and confirm no work was lost.

**4. Demonstrate resume-after-crash.** Kill the process mid-run, resume, confirm the run completes with state intact.

**5. Run a time-travel debug session.** Take a real failing trajectory, use `get_state_history` to find where it diverged, edit the state to test a hypothesis, and re-run. Document what you learned and how long it took.

**6. Implement edit-and-continue** on one gated tool — the human corrects an argument rather than only approving or rejecting.

**7. Add idempotency keys** to every tool with a side effect, derived from thread ID and step. Then deliberately crash after a side effect and confirm resume does not duplicate it.

**8. Write the retention policy** in `docs/checkpoint-policy.md`: retention per state (completed, interrupted, failed), the timeout for unanswered approvals, expected volume, sensitivity classification, and who can read the store. Lesson 11.3 audits against this.

---

## Visual Aid

*A horizontal timeline of an agent run. Steps 1–15 as ticks, with a small database icon under each labeled "checkpoint." At step 6 a red pause bar labeled `interrupt(): approve send_email`, with a dashed box beneath reading "process exits — state lives in Postgres — 47 minutes, 2 deploys," then a green resume arrow. Below the main timeline, a second branch line splits off at step 3 with a curved "rewind" arrow, annotated "time travel: edit state, re-run" and leading to an alternate short timeline ending in a green "correct answer at step 12." A small red warning icon sits at step 9 with the label "side effect + resume = idempotency key required."*

---

## Key Takeaways

- **Checkpointing makes runs durable, resumable, inspectable, and rewindable** for the cost of one compile argument and a database. Use `request_id` as the thread ID so telemetry, traces, and checkpoints share one key.
- **Interrupts turn approval from a blocked process into persisted state.** The process exits; the run resumes hours later from anywhere.
- **Edit-and-continue is more useful than approve/reject,** and an `interrupt()` in a review node after planning is cheaper than correcting a bad plan post-execution. Use `interrupt()` + `Command(resume=...)` for human-in-the-loop; keep static `interrupt_before`/`interrupt_after` for debugging, and remember the paused node re-runs from its first line on resume.
- **Time travel is the best debugging tool in agent development** because agent bugs are causal, not local. Rewind, edit one thing, re-run: a controlled experiment instead of a re-run and a hope.
- **Resumability creates an idempotency requirement.** A side effect that ran before the checkpoint will run again on resume. And checkpoints are a new PII store with a retention policy you must write.

---

## Sources

Checked 2026-11-11.

- LangGraph interrupts (`interrupt()`, `Command(resume=...)`, static interrupts for debugging, re-execution on resume): https://docs.langchain.com/oss/python/langgraph/interrupts — checked 2026-11-11
- LangGraph 1.2.20 source, `langgraph/types.py` (`interrupt(value, *, response_schema=None)`): https://pypi.org/project/langgraph/1.2.20/ — checked 2026-11-11
- LangGraph persistence and time travel: https://docs.langchain.com/oss/python/langgraph/persistence — checked 2026-11-11

---

## Next

You have one durable, supervised, debuggable agent. The natural next thought — "what if I had five of them, each specialized?" — is heavily marketed and usually wrong.

**Lesson 7.3** covers the multi-agent patterns you should be able to name in an interview, and then makes the case against them honestly, with cost and latency numbers from your own system.
