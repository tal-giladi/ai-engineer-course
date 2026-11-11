# Module 7 practical — Rebuild the research agent as a durable graph

This practical puts the whole module to work on the research agent you built in Module 6. You port it to a LangGraph graph, add a human approval gate before anything is published, and then prove the three properties checkpointing gives you on a running system rather than on paper: approval that survives a restart, a run that survives a crash, and a time-travel debug session on a real failure. The part that matters most is the written comparison of the two implementations, because a considered position on frameworks is what you will be asked about in interviews.

## Task

Rebuild your Module 6 research agent as a LangGraph graph with a human approval gate before publication. Demonstrate approve-and-resume across a process restart, resume-after-crash, and one time-travel debug session.

## Deliverable

`exercises/m7-graph-research-agent/` plus a comparison of the two implementations across code size, debuggability, failure handling, and what you can no longer see.

## How it is assessed

| Criterion | Full credit |
|---|---|
| Port proven | Trajectory eval matches the Module 6 baseline within the noise band |
| Controls preserved | Budgets, repeated-action detection, error policies, typed termination all still present and verified |
| Telemetry preserved | Framework-internal calls still emit `CallRecord` with correlation ID |
| Approve-and-resume | Demonstrated across an actual process restart, not simulated |
| Resume-after-crash | Process killed mid-run; run completes with state intact |
| Time travel | A real failing trajectory rewound, state edited, hypothesis tested |
| Comparison written | Names at least three things the framework hides and how to see each |

**The comparison document is the graded artifact** and is directly reusable as interview material. A candidate with a considered position on frameworks — gains, losses, and when they would not use one — stands out sharply from one with a preference.
