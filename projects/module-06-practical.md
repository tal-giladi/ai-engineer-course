# Module 6 practical — A bounded research agent

This practical puts the whole module into one agent you write yourself. It plans, searches, reads and writes a sourced brief, and it cannot run past its step or cost budget. You set those budgets from your own trajectory data, not from the lesson defaults, and then explain the worst run you saw.

## Task

Build a research agent that plans, searches, reads, and writes a sourced brief, and cannot exceed its step or cost budget.

## Deliverable

`exercises/m6-research-agent/` with the agent, five logged trajectories, the budget justification derived from your own data, and an analysis of the worst trajectory and its cause.

## How it is assessed

| Criterion | Full credit |
|---|---|
| No framework | Loop is the student's own code |
| Budgets | All five dimensions, checked before the call |
| Budget justification | Derived from the student's own trajectory data, not the lesson's defaults |
| Trajectory logging | Structured, with correlation ID, tokens, and latency per step |
| Termination | Typed reasons, with `REFUSED` distinguishable from `BUDGET_EXHAUSTED` |
| Worst-trajectory analysis | Identifies a specific cause, not "it went wrong" |

The worst-trajectory analysis is the graded part. "Step 3's tool result was 4,800 tokens of mostly-irrelevant material, which pushed the goal out of the attended region and caused the model to re-plan from scratch at step 5" is the target depth.
