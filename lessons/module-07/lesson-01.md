# Lesson 7.1 — From Loop to Graph: Porting to LangGraph

**Module 7: Agent Architectures — LangGraph & MCP** · ~14 min
**Objective:** M7-LO1 — Port a hand-rolled loop to LangGraph and name what the framework adds and what it hides.

---

## Introduction

You built an agent without a framework, on purpose. Now you are going to put it in one, also on purpose, and the order is what makes this lesson worth anything.

Two honest reasons to learn LangGraph. **Leverage:** checkpointing, interrupts, streaming, and retry policies are real infrastructure that takes weeks to build well, and Lesson 6.5 left you with a blocking-approval problem that the framework solves properly. **Hiring:** LangChain and LangGraph are named explicitly in a large share of AI engineering postings, and a candidate who cannot discuss them reads as unfamiliar with the ecosystem regardless of what they have built by hand.

The candidate who can say *"I wrote the loop myself first, then ported it — here is what the framework gave me and here is what it hid"* is stronger than one fluent in either alone. This lesson produces that answer.

---

## Key Concept 1: The conceptual shift

Your loop is control flow with branches. A graph is nodes and edges, with a typed state object threaded through.

The mapping is close to one-to-one:

| Your loop | LangGraph |
|---|---|
| `while True:` | a cycle in the graph |
| `if not response.tool_calls: return` | a **conditional edge** |
| `dispatch(call)` | a **`ToolNode`** |
| `AgentState` from Lesson 6.3 | the graph's **state schema** |
| appending to `messages` | a **reducer** on the messages field |
| your budget checks | a conditional edge, or a check inside each node |

Note the third and fourth rows. The state/conversation separation you built by hand in Lesson 6.3 is what LangGraph *requires* — so students who skipped that lesson find graphs confusing, and you should find them familiar.

```python
class State(TypedDict):
    messages: Annotated[list, add_messages]   # reducer: append, don't replace
    goal: str
    findings: list[Finding]
    budget_used: BudgetUsage

builder = StateGraph(State)
builder.add_node("agent", call_model)
builder.add_node("tools", ToolNode(tools))
builder.add_edge(START, "agent")
builder.add_conditional_edges("agent", should_continue,
                              {"tools": "tools", "end": END})
builder.add_edge("tools", "agent")
graph = builder.compile()
```

**Reducers** are the one genuinely new idea. A reducer defines how a node's output merges into existing state — `add_messages` appends rather than overwrites. This matters because nodes can run concurrently, and without a defined merge two parallel nodes writing the same field is a race. In your hand-rolled loop you avoided this by being single-threaded.

---

## Key Concept 2: What the framework adds

**Persistence.** Checkpoint state after every node. Runs survive process death, can be inspected mid-flight, and can be resumed. This is Lesson 7.2 and it is the biggest single win.

**Interrupts.** Pause before or after a node, surface state, wait for input, resume. This solves Lesson 6.5's blocking-approval problem properly — the process does not stay alive waiting for a human.

**Streaming.** Stream tokens, node transitions, and state updates. Building this over a multi-node graph by hand is genuinely fiddly.

**Retry policies per node.** Declarative rather than hand-written per tool.

**A visual graph.** Underrated. Once a system has eight nodes and conditional routing, a rendered diagram is a real reasoning aid — and a good artifact for your Module 12 README.

---

## Key Concept 3: What the framework hides

This is the half that people do not learn until production, and knowing it is the interview answer.

**The exact prompt sent to the model.** The framework assembles messages — system prompt, tool schemas, history, possibly trimming. When output is wrong and you need to see the literal input, you need the escape hatch, and you should find it on day one rather than during an incident.

**When and how history is trimmed.** Message trimming may be automatic and configurable. If you do not know the policy, you cannot explain why turn 12 lost context that turn 8 had.

**Where retries happen.** A node retry, a framework-internal API retry, and your own tool retry are three different layers that can compound into nine attempts when you intended three — and into nine charges.

**Tool result formatting.** `ToolNode` decides how results become messages. Your Lesson 6.1 capping and projection may or may not survive the port, and a 40,000-token result quietly reappearing is a real regression.

**Token and cost accounting.** Framework-internal calls may not flow through your Lesson 1.4 telemetry unless you wire callbacks. A port that loses cost visibility is a bad trade.

**Find the escape hatches immediately:** verbose/debug output, callback handlers, and streaming of raw model inputs. Do this before you need them.

---

## Key Concept 4: Prove the port

A port is a refactor of a system whose behavior you have measured. Treat it accordingly: **run the Module 6 trajectory eval on both implementations and require the difference to fall inside your noise band.**

This is not ceremony. Ports lose things — a budget check that no longer fires, a tool result no longer capped, a repeated-action detector that lived in the loop you deleted. The eval catches them; reading the code does not.

---

## Worked Example 1: The port, measured

Same tools, same prompts, same 40-task trajectory eval.

| | Hand-rolled | LangGraph | Δ |
|---|---|---|---|
| Task success | 0.82 | 0.84 | +0.02 (within noise) |
| Mean steps | 5.1 | 5.0 | — |
| Mean cost/run | $0.019 | $0.021 | +11% |
| p95 cost/run | $0.038 | $0.044 | +16% |
| Lines of my code | 340 | 135 | **−60%** |

Success unchanged, code down 60%. The cost increase is real and worth understanding rather than shrugging at: it came from `ToolNode` including full tool results where the hand-rolled version projected and capped them (Lesson 6.1). Restoring the projection inside the tool functions brought cost back to $0.019.

**That is the lesson in one line:** the port silently dropped an optimization, the eval caught it, and reading the diff would not have.

## Worked Example 2: Debugging what the framework hides

A ported agent starts refusing questions it previously answered. Trajectories look normal. Prompts unchanged. Tools unchanged.

Reading the code reveals nothing, because the cause is not in the code.

Enabling verbose callbacks and dumping the literal model input shows it: the framework's message trimming was dropping the system prompt's refusal-policy section on longer conversations, keeping recent messages and cutting from the front — including part of the system message.

Two-line fix once visible. Two hours to find, all of it spent learning where the escape hatch was.

The generalizable finding, and the one worth carrying into an interview: **the framework's convenience is assembling the prompt for you; the cost is that the prompt is no longer in your code.**

---

## Practice

**1. Port the loop.** Define a typed `State` with reducers, `call_model` and `ToolNode` nodes, a conditional edge for termination, and a cycle back. Keep your Module 6 tools and prompts unchanged — isolate the variable.

**2. Preserve every Module 6 control.** Budgets, repeated-action detection, per-tool error policies, typed termination. Some become conditional edges, some live inside nodes. **This is where things get silently dropped** — check each one deliberately against your Lesson 6.4 list.

**3. Wire telemetry through.** Callback handlers so every framework-internal model call still emits a `CallRecord` with your `request_id`. If cost visibility does not survive the port, the port is not done.

**4. Run the trajectory eval on both.** Same 40 tasks. Report the table above. **Investigate any difference outside your noise band** — there is almost always something, and finding it is the exercise.

**5. Find three escape hatches.** Dump the literal messages sent to the model; find where message trimming is configured; find where retries are configured. Write down how to reach each one.

**6. Write `docs/framework-tradeoff.md`.** What you gained, what you lost visibility into, how to see each hidden thing again, and one paragraph on when you would *not* use a framework. This document is interview material for Module 12 — a candidate with a considered position on frameworks stands out sharply from one with a preference.

**7. Render the graph** and put the diagram in your README.

---

## Visual Aid

*Two diagrams side by side, connected by matching-coloured arrows between corresponding elements. Left, the Lesson 6.2 flowchart with its eight numbered decision points. Right, the LangGraph diagram: START → `agent` node → conditional edge → `tools` node → back to `agent`, with `END` as the exit. Coloured arrows link "while True" to the cycle, "if not tool_calls" to the conditional edge, "dispatch" to ToolNode, "AgentState" to the state schema. Below the right diagram, a red-bordered box lists what is now inside the framework and no longer visible: "prompt assembly · message trimming · retry layering · tool result formatting · token accounting," each with a small key icon labeled with its escape hatch.*

---

## Key Takeaways

- **The mapping is close to one-to-one:** loop→cycle, branch→conditional edge, dispatch→ToolNode, state→typed schema. Reducers are the one new idea, and they exist because nodes can run concurrently.
- **The framework adds real infrastructure** — persistence, interrupts, streaming, declarative retries, a visual graph — that takes weeks to build well.
- **The framework hides the exact prompt, the trimming policy, the retry layering, tool result formatting, and token accounting.** Find the escape hatches on day one.
- **Prove the port with your trajectory eval.** Ports silently drop optimizations and controls; the eval catches what reading the diff does not.
- **A considered position on frameworks is an interview asset.** Write it down while the port is fresh.

---

## Next

You have the same agent with better infrastructure. Now the payoff that motivated the port: your hand-rolled agent died with its process and blocked on approval for as long as a human took to answer.

**Lesson 7.2** makes runs durable, resumable after a crash, pausable for human approval without holding a process open — and adds the single best debugging tool in agent development: rewinding to any earlier step, editing state, and re-running from there.
