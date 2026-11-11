# Module 7 quiz — Agent Architectures: LangGraph & MCP

Ten scenario questions on moving your hand-rolled agent into real agent infrastructure: porting it to a graph and proving the port, making runs durable and pausable for a human, deciding whether more than one agent is ever worth it, and connecting tools through MCP without trusting what you should not. They test reasoning, not recall. Most give you a measurement, a failing run or a design proposal and ask what it shows or what to do next.

**Covers**

- [07.1 · From Loop to Graph: Porting to LangGraph](../lessons/module-07/lesson-01.md) — the loop-to-graph mapping, reducers, what the framework adds and hides, proving the port, code graph vs event orchestrator.
- [07.2 · Checkpointing, Interrupts, and Human-in-the-Loop](../lessons/module-07/lesson-02.md) — durable runs, `interrupt()` and resume, time travel, idempotency after resume.
- [07.3 · Multi-Agent Patterns, and When They're a Mistake](../lessons/module-07/lesson-03.md) — four patterns, lossy boundaries, independence as the deciding question, isolation as the security case.
- [07.4 · MCP: Consuming and Building Servers](../lessons/module-07/lesson-04.md) — hosts, clients, servers and primitives, description injection, the stateless 2026-07-28 revision.

**Pass mark:** 70% (7 of 10). Unlimited retries; answers and explanations appear after you submit.

The hands-on build is in [the Module 7 practical](../projects/module-07-practical.md).
