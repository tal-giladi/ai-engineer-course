# Module 2 quiz — Structured output and prompting as engineering

Ten scenario questions on treating the model as an untrusted upstream: why prose parsing fails silently, how to pick and design a schema-enforcement mechanism, when a repair loop should stop, and how versioned, well-ordered prompts make cost and caching explainable. They test reasoning, not recall. Most describe a parser, a schema, a recovery table or a prompt layout and ask what it shows or what to do next.

**Covers**

- [02.1 · Why Free-Form Text Is a Bad API](../lessons/module-02/lesson-01.md) — silent wrong values, why parser patches fail, contract-first schemas, shape vs correctness.
- [02.2 · Structured Outputs and Schema-Constrained Generation](../lessons/module-02/lesson-02.md) — the three rungs, explicit degradation, schema design as prompt design, which tool to reach for.
- [02.3 · Validation, Repair Loops, and Failure Budgets](../lessons/module-02/lesson-03.md) — retryable vs terminal, the three budget dimensions, dead letters, setting the budget from data.
- [02.4 · Prompts as Versioned Artifacts (and How Caching Changes the Math)](../lessons/module-02/lesson-04.md) — prompt versions in telemetry, one change per evaluation, typed variables, stable-to-variable ordering.

**Pass mark:** 70% (7 of 10). Unlimited retries; answers and explanations appear after you submit.

The hands-on part of this module is the [Module 2 practical](../projects/module-02-practical.md): an invoice extractor that never returns an invalid object.
