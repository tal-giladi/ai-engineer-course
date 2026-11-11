# Module 6 quiz — The Agent Loop from Scratch

Ten scenario questions on building an agent with no framework: the tool-calling round trip, the control decisions hidden in a hand-written loop, keeping context under control on long tasks, classifying failures and bounding the loop with typed exits, and containing tools by blast radius. They test reasoning, not recall. Most describe a trajectory, a test result or a tool design and ask what it shows or what to do next.

**Covers**

- [06.1 · Tool Calling: The Protocol Beneath Every Agent](../lessons/module-06/lesson-01.md) — the round trip, untrusted arguments, descriptions as prompt text, result size, parallel calls.
- [06.2 · Writing the Agent Loop by Hand](../lessons/module-06/lesson-02.md) — the eight control decisions, three architectures, trajectories, the spin.
- [06.3 · State, Memory, and Context Management](../lessons/module-06/lesson-03.md) — state vs conversation, externalization, goal re-injection, memory tiers, lossy summaries.
- [06.4 · Failure Recovery, Budgets, and Termination](../lessons/module-06/lesson-04.md) — failure taxonomy, repeat detection, five budgets, typed termination, chaos testing.
- [06.5 · Sandboxed Execution and Dangerous Tools](../lessons/module-06/lesson-05.md) — blast radius, scoped credentials, argument constraints, approval gates, sandboxing.

**Pass mark:** 70% (7 of 10). Unlimited retries; answers and explanations appear after you submit.

The hands-on part of this module is the [Module 6 practical](../projects/module-06-practical.md): a research agent with budgets set from your own data.
