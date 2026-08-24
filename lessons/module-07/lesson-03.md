# Lesson 7.3 — Multi-Agent Patterns, and When They're a Mistake

**Module 7: Agent Architectures — LangGraph & MCP** · ~13 min
**Objective:** M7-LO3 — Evaluate multi-agent patterns and argue when a single agent is the better design.

---

## Introduction

Multi-agent architecture is the most over-recommended pattern in this field. Diagrams with a supervisor and five specialists look like real architecture, they are easy to draw on a whiteboard, and they map onto how humans organize teams — which makes them intuitively appealing and frequently wrong.

You need to be able to name and build these patterns, because interviewers ask and because sometimes they are correct. You also need to be able to argue against them with numbers, because that is the answer that signals judgment rather than fashion-following.

This lesson gives you both, and the practice section explicitly accepts "the single agent was better" as a passing result.

---

## Key Concept 1: The four patterns

**Supervisor.** A router agent receives the task, delegates to specialists, and assembles their results. The supervisor holds the plan; specialists hold expertise.

*Costs:* every delegation is at least two extra model calls — the supervisor deciding, and the specialist working. The supervisor must serialize context into its instruction, and whatever it does not include is lost.

**Handoff.** Control transfers laterally. Agent A owns intake, hands to B for research, B hands to C for drafting. Each fully owns its phase.

*Costs:* the handoff is a lossy serialization boundary. C works from what B chose to pass, and B did not know what C would need.

**Parallel workers.** Fan out independent sub-tasks, run them concurrently, merge. **This is the only pattern with a clean latency argument** — genuinely independent work finishing in parallel wall-clock rather than serial.

*Costs:* merging is real work, and the sub-tasks must be genuinely independent. Where they are not, you get contradictory results.

**Critic / debate.** One agent produces, another critiques, the first revises. Useful where verification is genuinely cheaper than generation — checking whether an answer is grounded is easier than producing a grounded answer.

*Costs:* at least doubles cost, and critics have a strong tendency to always find something, producing revision loops that add words and not quality.

---

## Key Concept 2: The case against

Four costs, and they compound.

**Every boundary is lossy.** This is the fundamental one. Within a single agent, everything the model learned at step 3 is available at step 11. Across an agent boundary, only what the sender chose to serialize survives — and the sender is guessing at what the receiver needs. Rich implicit context becomes a paragraph.

**Every boundary costs a round trip.** Supervisor decides (call), specialist works (calls), supervisor interprets (call). A three-specialist task can easily be five extra model calls of pure coordination.

**The failure surface multiplies.** Five agents means five loops, five budgets, five sets of error policies, and interaction failures that exist in none of them individually — a specialist returning something the supervisor misinterprets, two specialists contradicting each other.

**Attribution becomes hard, and this is the one that hurts most.** With one agent, a bad answer has one trajectory to read. With five, you must determine *which* agent failed and whether it failed because of what it was given. Your Module 3 eval measures a final answer; localizing a failure across agents is a much harder problem, and it makes iteration dramatically slower.

---

## Key Concept 3: When it is genuinely right

Four conditions. If none applies, use one agent.

**1. Genuinely different tools or permissions.** A research agent with web access and a database agent with customer records should probably not be the same agent holding both — which is not primarily an architecture argument, it is the security argument below.

**2. Real parallelism.** Six independent lookups running concurrently is a genuine wall-clock win that no single sequential agent can match.

**3. Context isolation as a security requirement.** This is the strongest reason and it comes from Module 11: **never give the agent that reads untrusted content the tools that can exfiltrate.** Split them, and an injection in retrieved content lands in an agent with nothing dangerous to reach. Lesson 11.1 develops this as the primary architectural defense against prompt injection.

**4. Context genuinely does not fit.** A task requiring more context than one window can hold, where sub-tasks partition cleanly.

Note what is *not* on the list: "different kinds of expertise." A single agent with well-described tools (Lesson 6.1) handles heterogeneous work perfectly well. Specialization by prompt is usually a worse version of specialization by tool description.

---

## Key Concept 4: Build the argument, not the architecture

The professional move is to implement both and measure. That sounds expensive; with your Module 3 harness and Module 6 trajectory eval it is an afternoon.

Compare on: task success, cost per run, p95 latency, lines of code, and — informally but importantly — **time to diagnose a failure**. That last one does not appear in dashboards and dominates the maintenance cost.

---

## Worked Example 1: Single agent wins

**Task:** answer questions requiring documentation lookup, calculation, and changelog checking.

**Architecture A — one agent, four tools.**
**Architecture B — supervisor plus three specialists** (docs, calculation, changelog), each with its own tools and prompt.

40-task trajectory eval:

| | Single agent | Supervisor + 3 |
|---|---|---|
| Task success | **0.84** | 0.79 |
| Mean model calls/run | **5.0** | 11.4 |
| Mean cost/run | **$0.021** | $0.058 |
| p95 latency | **6.2 s** | 14.8 s |
| Lines of code | **135** | 410 |
| Failures needing >1 trajectory to diagnose | **0%** | 64% |

The single agent wins on every axis, including success rate.

The five failures unique to the multi-agent version are the instructive part:

- 2× the supervisor's instruction to the docs specialist dropped a qualifier from the user's question ("for v3"), and the specialist answered for v2 — correctly, for the question it was given
- 2× the calculation specialist received numbers without their units
- 1× two specialists returned contradictory information and the supervisor picked the wrong one

**Every one is a serialization loss at a boundary.** None of them can occur in a single agent, because there is no boundary to lose anything at.

## Worked Example 2: Parallel workers win

**Task:** *"Compare rate limits, auth, and error handling across all three API versions."*

| | Single agent | 3 parallel workers + merge |
|---|---|---|
| Task success | 0.81 | **0.86** |
| Mean model calls/run | **14.2** | 19.0 |
| Mean cost/run | **$0.062** | $0.081 |
| **p95 latency** | 31 s | **12 s** |

Here the parallel version costs 31% more and is **2.6× faster**, because three independent version-lookups genuinely run concurrently. Success is also slightly better, because each worker's context holds one version rather than three, which reduces cross-version confusion.

**The structural difference between the two examples:** in the first, the sub-tasks were *sequentially dependent* — the calculation needed the docs result — so there was nothing to parallelize and the coordination was pure overhead. In the second, they were *genuinely independent*.

That is the question to ask on a whiteboard: **are the sub-tasks independent, or does one need another's output?** If dependent, multi-agent buys you coordination cost and serialization loss for nothing.

---

## Practice

**1. Implement a supervisor architecture** for a task you have already solved with a single agent. Three specialists, real delegation, real result assembly.

**2. Evaluate both** on the same 40-task set: success, model calls, cost, p95 latency, lines of code.

**3. Diagnose five failures in each.** Time yourself. Record how many trajectories you had to read per failure. This is the number that does not appear in the comparison table and matters most in maintenance.

**4. Categorize the multi-agent-only failures.** How many were serialization losses at a boundary? This is the expected finding and you should confirm it on your own system.

**5. Try a parallel-worker version** on a task with genuinely independent sub-tasks. Measure latency specifically — this is the pattern's home ground.

**6. Write `docs/adr-agent-architecture.md`:** the two (or three) architectures, the measured comparison, the choice, the reasoning, and the conditions under which you would revisit. **"Single agent, because the sub-tasks are sequentially dependent and the boundaries cost us five failures" is an excellent decision record** and a strong interview answer.

**7. Note the security case for later.** Even if you choose a single agent, write down which of your tools you would split into a separate context-isolated agent if it were reading untrusted content. Lesson 11.1 picks this up.

---

## Visual Aid

*Four architecture diagrams in a 2×2 grid: Supervisor (one node fanning to three, arrows returning), Handoff (a chain A→B→C), Parallel workers (fan-out to three, fan-in to a merge node), Critic (two nodes in a two-way loop). Under each, a three-line rating: cost, latency, debuggability — with green/amber/red bars. Parallel workers is the only one with a green latency bar. Beneath the grid, a single wide box headed "The deciding question" containing: "Are the sub-tasks independent, or does one need another's output? → Dependent: use one agent. → Independent: parallel workers may win on latency. → Untrusted content plus dangerous tools: split for isolation, regardless of cost (Lesson 11.1)."*

---

## Key Takeaways

- **Four patterns worth naming:** supervisor, handoff, parallel workers, critic. Only parallel workers has a clean latency argument.
- **Every agent boundary is a lossy serialization** of context, an extra round trip, an added failure surface, and — worst — an attribution problem that makes iteration much slower.
- **Four legitimate reasons to split:** genuinely different tools or permissions, real parallelism, context isolation as a security control, and context that genuinely does not fit. "Different kinds of expertise" is not one.
- **The deciding question is independence.** Sequentially dependent sub-tasks make coordination pure overhead.
- **Measure both and write the decision record.** Choosing a single agent with numbers behind it is a stronger result than building the impressive diagram.

---

## Next

Every tool your agent has lives inside your process. Someone else's agent cannot use your retrieval; your agent cannot use their capabilities without you writing an integration for each one.

There is a protocol for that, it is the one the ecosystem converged on, and it appears by name in job postings.

**Lesson 7.4** exposes your retrieval as an MCP server that any MCP-capable host — including the coding agent you already use — can query, connects your agent to third-party servers as a client, and covers the trust question that comes with both.
