# Module 11 quiz — Security, Safety & Reliability

Ten scenario questions on defending and bounding an agent system you own: why prompt injection is architectural and which defenses actually break the risk chain, where the safety control for a code-execution tool lives, why a cross-tenant cache leak is invisible to monitoring, how to report a red-team campaign honestly, and how to respond to provider failures and side-effecting tools that time out. They test reasoning, not recall. Most describe an attack, a trace or a failure and ask what it means or what to do next.

**Covers**

- [11.1 · Prompt Injection: Direct, Indirect, and Why It Isn't Solved](../lessons/module-11/lesson-01.md) — one channel for instructions and data, indirect injection, the three-part risk chain, defenses by strength.
- [11.2 · Tool Permissions, Sandboxing, and Least Privilege](../lessons/module-11/lesson-02.md) — assume compromise, scoped credentials, argument validation, rate limits, conditional approval gates, sandbox hardening.
- [11.3 · PII, Output Filtering, and Compliance Basics](../lessons/module-11/lesson-03.md) — the data inventory, redaction at capture, output as an exfiltration channel, cache scoping, retention.
- [11.4 · Red-Teaming Your Own Agent](../lessons/module-11/lesson-04.md) — the attack grid, five categories, automated campaigns with a calibrated judge, regression tests, honest reports.
- [11.5 · Reliability and Cost: Retries, Fallback Routing, Kill Switches](../lessons/module-11/lesson-05.md) — retry classes, idempotency keys, fallback tiers with measured quality, circuit breakers, cost caps, kill switches.

**Pass mark:** 70% (7 of 10). Unlimited retries; answers and explanations appear after you submit.

The hands-on part of this module is the [Module 11 practical](../projects/module-11-practical.md): red-team a deliberately vulnerable agent, then fix and regression-test every exploit.
