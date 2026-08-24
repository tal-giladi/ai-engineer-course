# Module 7 Plan — Agent Architectures: LangGraph & MCP

**Duration:** 55 min · **Lessons:** 4 · **Depends on:** Module 6
**Outline reference:** `curriculum/course-outline.md` §8, Module 7

---

## Prerequisites

**From Module 6:** the hand-written agent loop, its budgets, error policies, and approval gate. This module is a *port*, and the port has no pedagogical value without the original.
**From Module 3:** the eval harness, extended in Module 6 to score trajectories — used here to prove the port did not regress.

---

## Framing: Frameworks as Vocabulary and as Leverage

Two honest reasons to learn LangGraph. First, leverage: checkpointing, interrupts, streaming, and retry policies are real infrastructure that takes weeks to build well. Second, hiring: LangChain and LangGraph appear by name in a large share of postings, and a candidate who cannot discuss them reads as unfamiliar with the ecosystem regardless of what they have built by hand. The order matters — because students wrote the loop first, they can say precisely what the framework adds and what it hides, which is a stronger interview answer than fluency in either alone.

MCP gets equal billing because it is the interoperability layer the ecosystem converged on: it decouples tool implementations from agent implementations, which is the same seam argument made for model providers in Lesson 1.2.

---

## Module Learning Objectives

**M7-LO1** — Port a hand-rolled loop to LangGraph and name what the framework adds and what it hides.
**M7-LO2** — Use checkpointing, interrupts, and human-in-the-loop approval for consequential actions.
**M7-LO3** — Evaluate multi-agent patterns — supervisor, handoff, parallel workers — and argue when a single agent is the better design.
**M7-LO4** — Build an MCP server exposing their tools and consume third-party MCP servers as a client.

---

## Lesson 7.1 — From Loop to Graph: Porting to LangGraph

*14 min · serves M7-LO1*

**Concept.** The conceptual shift is from *loop with branches* to *graph with edges*: nodes are units of work, edges are transitions, conditional edges are the branches, and a typed state object is threaded through every node — which is exactly the state/history separation students already built in Lesson 6.3, now enforced by the framework. Map each part of the hand-written loop onto its graph equivalent explicitly: the while loop becomes a cycle, the "is it done?" branch becomes a conditional edge, the tool dispatch becomes a `ToolNode`, and the state dict becomes a typed schema with reducers that define how concurrent updates merge. Then the honest accounting. **What the framework adds:** persistence, streaming, interrupts, retry policies, and a visual graph that is genuinely useful for reasoning about complex flows. **What it hides:** the exact prompt sent to the model, when and how history is trimmed, and where the retries happen — all of which become invisible right when a production bug requires seeing them. Teach the reflex of finding the framework's escape hatches on day one.

**Example.** The Module 6 loop ported node by node, run side by side with the original on the same eval set, showing near-identical trajectory scores and a 60% reduction in the student's own code. Then a deliberate debugging exercise: a wrong answer where the cause is a prompt the framework assembled, found only by enabling verbose tracing — the concrete demonstration of what "hides" costs.

**Practice.** Students port their loop to LangGraph with a typed state schema, run both against the Module 6 trajectory eval, and confirm no regression. They then write `docs/framework-tradeoff.md`: what they gained, what they lost visibility into, and how to see it again.

**Summary.** Same mechanism, better infrastructure, less visibility. Know the escape hatches before you need them, and prove the port with the eval you already have.

**Key terms:** graph, node, edge, conditional edge, state schema, reducer, `ToolNode`, cycle, framework escape hatch.
**Visual aid:** The Module 6 flowchart and the LangGraph diagram side by side, with each loop element linked to its graph counterpart by an arrow.
**Hook:** "Here's the payoff. Your hand-written agent died if the process died. This one can stop mid-run, wait a day for a human, and resume exactly where it was."

---

## Lesson 7.2 — Checkpointing, Interrupts, and Human-in-the-Loop

*13 min · serves M7-LO2*

**Concept.** The framework capability that is genuinely hard to build yourself. **Checkpointing** persists graph state after every node, which makes an agent run durable, resumable after a crash, and inspectable mid-flight — turning a long agent run from a fragile in-memory process into something with the reliability properties of a workflow engine. **Interrupts** pause execution at a defined point and surface state to a caller: `interrupt_before` for approval on a consequential node, `interrupt_after` for review of a result. Together these formalize the Lesson 6.5 approval gate into infrastructure and add the things a hand-rolled gate lacked — durability across restarts, and **time travel**, the ability to rewind to an earlier checkpoint, edit state, and re-run from there, which is the single best debugging tool in agent development. Cover thread IDs as the persistence key and their relationship to the Module 1 correlation ID. Cover the operational reality: checkpoints are state you must now store, size, and expire, and they may contain sensitive data — a Module 11 concern raised here.

**Example.** An agent interrupted before an irreversible tool call, its full state inspected, approved, and resumed — then the same run resumed after killing the process entirely, proving durability. Then time travel: rewinding to step 3, editing one state field, and re-running to a different outcome, used to find the exact step where a trajectory went wrong.

**Practice.** Students add a checkpointer, gate their irreversible tool with `interrupt_before`, and demonstrate three things: approval-then-resume, resume-after-crash, and a time-travel debug session in which they rewind, edit, and produce a corrected trajectory. They document the checkpoint retention policy they would set in production.

**Summary.** Checkpointing makes agents durable and debuggable; interrupts make consequential actions supervised. Both are state you now own — including its retention and its sensitivity.

**Key terms:** checkpointer, thread ID, `interrupt_before`/`interrupt_after`, human-in-the-loop, time travel, state rewind, checkpoint retention.
**Visual aid:** Timeline of a checkpointed run — nodes as ticks, checkpoints as saved states, the interrupt as a pause, and a rewind arrow branching to an alternate path.
**Hook:** "One agent, durable and supervised. The obvious next thought is 'what if I had five?' — and that thought is usually wrong."

---

## Lesson 7.3 — Multi-Agent Patterns, and When They're a Mistake

*13 min · serves M7-LO3*

**Concept.** Multi-agent architecture is heavily marketed and frequently the wrong choice, so this lesson teaches the decision, not just the patterns. Cover the patterns: **supervisor** (a router agent delegates to specialists and collects results), **handoff** (control transfers laterally, each agent owning a phase), **parallel workers** (independent sub-tasks fanned out and results merged — the only pattern with a clean latency argument), and **debate/critic** (one agent produces, another critiques, useful where verification is genuinely cheaper than generation). Then the case against: every agent boundary is a lossy serialization of context, costs an extra model call, multiplies the failure surface, and makes evaluation dramatically harder because a bad final answer must now be attributed across agents. State the honest heuristic — reach for multiple agents when sub-tasks need genuinely different tools or permissions, when parallelism buys real wall-clock, or when context isolation is a security requirement (a Module 11 pattern: never give the agent reading untrusted content the tools that can exfiltrate). Otherwise, one agent with good tools usually wins on cost, latency, and debuggability.

**Example.** The spine task implemented both ways — single agent with four tools, and a supervisor with three specialists — compared on task success, cost, p95 latency, and lines of code. The single agent wins on all four. Then a second task where parallel workers legitimately win, and the structural difference between the two cases named explicitly.

**Practice.** Students implement one supervisor-with-specialists version of a supplied task, evaluate it against a single-agent baseline on success, cost, and latency, and write a decision record. Reaching the conclusion "single agent was better here" is an acceptable and expected outcome — the deliverable is the argument, not the architecture.

**Summary.** Multi-agent buys tool separation, permission isolation, and parallelism. It costs context, money, latency, and debuggability. Make the trade explicit and measure it.

**Key terms:** supervisor pattern, handoff, parallel workers, critic/debate, context serialization loss, attribution problem, permission isolation.
**Visual aid:** Four architecture diagrams with a cost/latency/debuggability rating under each, and the deciding question printed beneath.
**Hook:** "Every tool you've built lives inside your process. There's a protocol for making them available to any agent, including ones you didn't write."

---

## Lesson 7.4 — MCP: Consuming and Building Servers

*13 min · serves M7-LO4*

**Concept.** The Model Context Protocol is the integration seam for tools, the way the OpenAI-compatible API is the seam for models — the same architectural argument students already accepted in Lesson 1.2, applied one layer up. Cover the model: a **host** application runs **clients** that connect to **servers**, and servers expose three primitive types — **tools** (functions the model can invoke), **resources** (data the host can read), and **prompts** (reusable templates the user can invoke). Cover transports: stdio for local subprocess servers, HTTP for remote ones. Cover consuming — connecting an agent to existing servers for filesystem, database, or SaaS access, and gaining a capability without writing an integration. Cover building — wrapping the spine project's retrieval as an MCP server so any MCP-capable host, including a coding agent this audience already uses, can query the corpus. Cover the security reality plainly, since it is a genuine and current concern: an MCP server is untrusted code and its tool descriptions are untrusted text that enters the model's context, so tool descriptions are an injection vector and server trust is a real decision. Module 11 attacks this directly.

**Example.** The spine retrieval exposed as an MCP server over stdio, then consumed from a coding agent — the student's own course corpus becoming queryable from their editor. Then a third-party server connected and its tools appearing in the agent with no integration code written.

**Practice.** Students build an MCP server exposing spine retrieval as a tool and the corpus index as a resource, verify it from at least one external MCP host, and separately connect their agent as a client to one third-party server. They document, for each server they connect to, why they trust it.

**Summary.** MCP decouples tools from agents. Build servers to make your capabilities portable, consume servers to gain capabilities cheaply, and treat every server — and every tool description it sends — as untrusted input.

**Key terms:** Model Context Protocol, host, client, server, tools/resources/prompts, stdio transport, HTTP transport, server trust, tool-description injection.
**Visual aid:** MCP topology — one host, several clients, several servers — with the trust boundary drawn between host and servers and the injection vector marked.
**Hook:** "Your agent is architected, durable, and interoperable. It's also still running on someone else's model, at someone else's price. Modules 8 and 9 are about owning that."

---

## Spine Increment

**Deliverable:** `spine/graph/` — the Module 6 agent ported to LangGraph with a typed state schema, checkpointing, an interrupt-gated irreversible tool, plus `spine/mcp_server/` exposing retrieval over MCP.

**Acceptance criteria:**
1. Trajectory eval scores match the Module 6 hand-rolled baseline within the measured noise band — the port is proven, not assumed.
2. `docs/framework-tradeoff.md` names three things the framework hides and how to see each one.
3. Checkpointing demonstrated three ways: approve-and-resume, resume-after-crash, and a time-travel debug session.
4. A multi-agent decision record with measured cost and latency for both architectures.
5. The MCP server is verified from an external host, with a written trust note for every server consumed.

---

## Standalone Exercise

Rebuild the Module 6 research agent as a LangGraph graph with a human approval gate before publication, then write a comparison of the two implementations across code size, debuggability, failure handling, and what you can no longer see. Deliverable: `exercises/m7-graph-research-agent/`.

---

## Dependencies

| Lesson | Requires |
|---|---|
| 7.1 | Module 6 complete |
| 7.2 | 7.1 |
| 7.3 | 7.1, 7.2 |
| 7.4 | Module 6 tools; independent of 7.1–7.3 and reorderable |

**Downstream:** Module 10 deploys and traces the graph. Module 11 attacks the MCP surface and the checkpoint store. Module 12 presents the framework trade-off document as interview material.

---

## Assessment Focus (Phase 4)

Architectural judgment. Given a hand-written loop, map it to graph components. Given a requirement for day-long human approval, name the two features that make it possible. Given a proposed five-agent design, argue for or against with cost and attribution reasoning. Given a production bug caused by a framework-assembled prompt, describe how to see it. Given an MCP server from an unknown publisher, list what you check before connecting. One practical: add checkpointing and an approval gate to a supplied graph and demonstrate resume-after-crash.
