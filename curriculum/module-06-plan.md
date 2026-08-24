# Module 6 Plan — The Agent Loop from Scratch

**Duration:** 60 min · **Lessons:** 5 · **Depends on:** Modules 2, 3
**Outline reference:** `curriculum/course-outline.md` §8, Module 6

---

## Prerequisites

**From Module 2:** structured outputs and the validate-repair-retry pattern — a tool call *is* a structured output, and this module reuses that machinery directly.
**From Module 3:** the eval harness. Agents are evaluated on trajectories, not just final answers, which this module extends the harness to handle.
**From Module 5 (soft):** the RAG pipeline becomes the agent's first tool. Students who reordered can substitute any tool.
**From the student:** state machines, retry semantics, idempotency, and sandboxing — ordinary distributed-systems knowledge that transfers almost completely.

---

## Framing: Why Hand-Rolled

Interview research is explicit that panels probe whether a candidate has built something *agentic* — a system that plans, calls tools, holds state, and recovers from failure — rather than something that answers prompts. A candidate who has only assembled a framework graph cannot answer "what happens when the model returns a malformed tool call on step 7 of 20?" This module builds the loop by hand for exactly that reason. Module 7 then ports it to LangGraph, and the port is only meaningful because the mechanism underneath is understood.

---

## Module Learning Objectives

**M6-LO1** — Implement tool calling end to end — schema, dispatch, result formatting, parallel calls — against the raw API.
**M6-LO2** — Write an agent loop by hand and explain every control-flow decision inside it.
**M6-LO3** — Manage state, memory, and a shrinking context window across a long-running task.
**M6-LO4** — Make the loop survive failure: tool errors, malformed calls, step and token budgets, explicit termination.
**M6-LO5** — Execute untrusted or dangerous tool calls inside a sandbox.

---

## Lesson 6.1 — Tool Calling: The Protocol Beneath Every Agent

*12 min · serves M6-LO1*

**Concept.** Strip the mystique: tool calling is a structured output whose schema happens to describe a function signature, plus a convention for feeding the result back. Walk the full round trip — you declare tools as JSON Schema, the model emits a tool-call message with arguments, *your code* executes it (the model never runs anything), you append a tool-result message, the model continues. Cover tool design as the actual skill: descriptions are prompt real estate and are the single biggest lever on whether the model calls the right tool; enums and constrained types prevent whole classes of invalid call; fewer, well-named tools beat many overlapping ones; and the result format matters, because a 40,000-token raw API dump will consume the context window by step three. Cover parallel tool calls and the ordering hazards they introduce. Reuse Module 2's validation directly — tool arguments are untrusted structured input.

**Example.** One tool declared three ways — a bare name, a name with a good description, and a name with a description plus enum-constrained arguments — with the model's tool-selection accuracy measured across 40 queries at 52%, 81%, and 94%. Then a tool returning raw JSON versus a summarized result, with the context consumption compared.

**Practice.** Students wrap the Module 5 RAG pipeline as a tool, plus a calculator and a clock, and measure selection accuracy across a query set that requires choosing correctly between them. They then improve one tool's description and re-measure.

**Summary.** A tool call is a validated structured output that your code executes. Descriptions drive selection accuracy; result formatting drives context survival.

**Key terms:** tool/function calling, tool schema, tool-call message, tool-result message, parallel tool calls, tool description, result summarization.
**Visual aid:** Message-sequence diagram of one full round trip, with the message roles labeled and the "your code runs this" boundary drawn in bold.
**Hook:** "One tool call is a request/response. An agent is what happens when you put that in a loop and let the model decide when to stop."

---

## Lesson 6.2 — Writing the Agent Loop by Hand

*13 min · serves M6-LO2*

**Concept.** The loop, in about forty lines, with every decision made explicitly. Walk it: append the user message; call the model with tools declared; if the response contains tool calls, execute them, append results, and loop; if it contains a final answer, return. Then interrogate every choice the forty lines quietly make — *who decides it's done*, the model or your code, and what happens when they disagree; *how many tool calls per turn* are permitted; *what happens on a malformed call*; *what the model sees* of a tool's failure; and *whether the loop can ever refuse to execute* a call the model requested. Introduce the taxonomy honestly: a plan-then-execute agent decides everything up front and is predictable but brittle; a ReAct-style interleaved loop reasons and acts alternately and is flexible but harder to bound. Most production systems are a constrained hybrid, and the constraints are the engineering.

**Example.** The loop built live, run on a multi-step question requiring three retrievals and a calculation, with every message in the growing conversation printed. Then the same loop given a question it cannot answer, spinning for fourteen steps — the motivating failure for Lesson 6.4.

**Practice.** Students implement `spine/agent/loop.py` from scratch — no framework — and run it on a five-question multi-step set, logging the full trajectory for each. They then answer, in writing, the five interrogation questions above for their own implementation.

**Summary.** The loop is small. The engineering is in the decisions it encodes about control, termination, and failure — and being able to name those decisions is what an interview probes.

**Key terms:** agent loop, ReAct, plan-and-execute, trajectory, turn, terminal condition, control inversion.
**Visual aid:** Annotated flowchart of the loop with each decision point numbered and cross-referenced to the lesson that hardens it.
**Hook:** "Every iteration appends to the conversation. Run it long enough and you'll hit a wall you already met in Module 1."

---

## Lesson 6.3 — State, Memory, and Context Management

*12 min · serves M6-LO3*

**Concept.** An agent's context grows monotonically while its window does not, so context management is not an optimization but a correctness requirement. Separate the two things people conflate: **state** (the structured facts of the task — goal, findings so far, open sub-tasks — which should live in *your* data structures) and **conversation history** (the message list the model sees, which is a rendering of state plus the raw exchange). Once separated, the management techniques become obvious: summarize old turns into a running brief, truncate or externalize large tool results and pass a handle instead, keep a durable scratchpad the agent reads and writes, and re-inject the goal periodically because it drifts to the top of a long context and gets attended to less. Cover memory tiers — working memory (this run), session memory (this conversation), and long-term memory (across sessions, usually a vector store, which is the Module 4 pipeline pointed at the agent's own history). Cover the honest failure mode: summarization loses exactly the detail that later turns out to matter.

**Example.** A twenty-step agent run instrumented with context size per step, hitting the window at step 12 and failing. Then the same run with tool-result externalization and rolling summarization, completing in 19 steps at 40% of peak context — with one detail lost in summarization, shown and discussed rather than hidden.

**Practice.** Students separate state from history in their loop, add rolling summarization and tool-result externalization, and re-run the failing twenty-step task. They report peak context, step count, and any information lost to summarization.

**Summary.** Keep state in your code, render the minimum into the context, externalize big results, and re-inject the goal. Summarization is lossy — decide deliberately what you are willing to lose.

**Key terms:** agent state, conversation history, context window pressure, rolling summarization, tool-result externalization, scratchpad, working/session/long-term memory, goal drift.
**Visual aid:** Stacked-area chart of context composition over 20 steps — system, goal, history, tool results — before and after management.
**Hook:** "It survives long tasks now. It does not survive a tool that returns a 500."

---

## Lesson 6.4 — Failure Recovery, Budgets, and Termination

*12 min · serves M6-LO4*

**Concept.** The lesson that separates a demo from a system, and the one interviewers reliably probe. Cover the failure taxonomy: the model emits a malformed tool call (Module 2's repair loop applies, with a budget); a tool raises (decide per tool whether the model should see the error and retry, or whether the loop should abort — a transient timeout is worth retrying, a validation error usually means the model misunderstood and needs the error text, and an auth failure should never be retried by the model); the model loops on the same failing action; the model declares success falsely. Cover **budgets** as first-class parameters — max steps, max tokens, max wall-clock, max cost per run, and max calls per individual tool — and why a cost budget is the one that prevents an incident rather than an annoyance. Cover **termination**: success, budget exhaustion, unrecoverable error, and explicit user cancellation, each returning a distinguishable result so callers can react. Connect back to Module 1: an unbounded agent is the most expensive way ever invented to produce nothing.

**Example.** The fourteen-step spin from Lesson 6.2 with its real cost computed, then bounded — a repeated-action detector breaking the loop at step 4, and a cost budget capping the worst case. Then a table of four termination reasons and the different caller behavior each should trigger.

**Practice.** Students add budgets, a repeated-action detector, per-tool error policies, and typed termination results. They then run a chaos test — a supplied tool set where 30% of calls fail randomly, one hangs, and one returns plausible garbage — and report completion rate and cost under failure.

**Summary.** Bound every dimension, classify every error, distinguish every exit. An agent without a cost budget is an incident waiting for a trigger.

**Key terms:** step budget, token budget, cost budget, wall-clock budget, repeated-action detection, error policy, transient vs terminal failure, termination reason, chaos testing.
**Visual aid:** State diagram of the loop with all four termination transitions and the budget counters drawn on the edges.
**Hook:** "It's bounded and it recovers. It also just executed a tool call the model wrote — and you have no idea what was in it."

---

## Lesson 6.5 — Sandboxed Execution and Dangerous Tools

*13 min · serves M6-LO5*

**Concept.** The moment an agent's output becomes an action, the trust model changes completely — and the model's tool call is derived from input you do not control. Classify tools by blast radius: **read-only and safe** (retrieval, calculation), **read-only and sensitive** (customer records, internal search — the exfiltration surface), **write and reversible** (draft an email, create a ticket), **write and irreversible** (send the email, issue the refund, delete the row). Cover the controls that match each tier: schema validation and allowlists at the argument level, capability scoping so the agent holds the least privilege it needs rather than the caller's full rights, human approval gates on irreversible actions, and real isolation for code execution — containers with no network, dropped capabilities, read-only mounts, CPU and memory and wall-clock limits — using this audience's existing Docker fluency. Cover the code-execution tool honestly: it is enormously useful and it is arbitrary remote code execution by design, so the sandbox is the entire safety story. Flag forward: Module 11 covers the case where the *input* is adversarial, not merely unknown.

**Example.** A code-execution tool run unsandboxed against a benign-looking task that reads an environment variable and puts an API key in its output — then the same task in a hardened container where it cannot. Then an approval gate on an irreversible tool, showing the interrupt-and-resume shape that Module 7 formalizes with checkpointing.

**Practice.** Students classify every tool in their agent by blast radius, implement a Docker sandbox for code execution with network disabled and resource limits set, and add an approval gate to one irreversible tool. They then attempt three escapes from their own sandbox and document the results.

**Summary.** Classify by blast radius, scope capabilities to the minimum, gate the irreversible, and isolate execution for real. The model chooses the arguments — your sandbox decides what those arguments can do.

**Key terms:** blast radius, capability scoping, least privilege, allowlist, human-in-the-loop approval, container sandbox, network isolation, resource limits, arbitrary code execution.
**Visual aid:** Blast-radius quadrant — reversibility against sensitivity — with each tool type placed and the required control listed per quadrant.
**Hook:** "You built the loop, bounded it, and locked down its tools. Module 7 is where you hand it to a framework — and you'll finally be able to tell what the framework is doing for you and what it's hiding."

---

## Spine Increment

**Deliverable:** `spine/agent/` — a hand-written agent loop with the RAG pipeline as a tool, separated state and history, context management, full budgets, error policies, typed termination, and a Docker-sandboxed code-execution tool with an approval gate.

**Acceptance criteria:**
1. No agent framework imported. The loop is the student's own code.
2. Trajectories logged with the Module 1 correlation ID, so every model call in a run is linkable.
3. Peak context stays under 60% of the window on a 20-step task.
4. Chaos test passes: bounded cost and a typed termination reason under 30% tool failure.
5. Code execution runs with no network, dropped capabilities, and enforced resource limits; the irreversible tool cannot fire without approval.
6. The eval harness scores trajectories — steps, cost, tool-selection accuracy — not just final answers.

---

## Standalone Exercise

Build a research agent that plans, searches, reads, and writes a sourced brief, and cannot exceed its step or cost budget. Deliverable: `exercises/m6-research-agent/` with the agent, five logged trajectories, the budget justification, and a short analysis of the worst trajectory and what caused it.

---

## Dependencies

| Lesson | Requires |
|---|---|
| 6.1 | Module 2 (structured outputs) |
| 6.2 | 6.1 |
| 6.3 | 6.2 |
| 6.4 | 6.2, 6.3 |
| 6.5 | 6.1, 6.4 |

**Downstream:** Module 7 ports this loop to LangGraph and formalizes the 6.5 approval gate as an interrupt. Module 10 traces and optimizes it. Module 11 attacks it — 6.5 handles unknown input, Module 11 handles hostile input.

---

## Assessment Focus (Phase 4)

The module most likely to be examined in an interview, so questions are mechanism-level. Given a trajectory log, identify why the agent looped. Given a tool that returns 40,000 tokens, predict the failure and propose two fixes. Given four tool types, assign controls by blast radius. Given a malformed tool call at step 7 of 20, walk the recovery path. Given an agent with no cost budget, compute the worst case. One practical: bound a supplied runaway agent without lowering its task success rate.
