# Module 10 practical — Halve the latency of a slow agent

This practical puts the module's optimization discipline to work on one endpoint. You start from a slow agent service, decompose where its time and money go from traces, and then spend that budget deliberately: free wins first, one change at a time, the full eval suite after every change. The goal is not only a faster endpoint but an honest record of every change, including the ones that did not pay their way.

## Task

Take the supplied slow agent endpoint in `exercises/m10-latency-halving/` and cut p95 latency by half without losing eval score.

## Deliverable

A trace-based budget decomposition, each optimization with its measured effect on latency, cost, and quality, and a note on any change you had to revert.

## How it is assessed

| Criterion | Full credit |
|---|---|
| SLO stated first | Explicit p95, cost, and quality-floor targets before any optimization |
| Budget decomposed | Per-stage latency and cost from traces, not guesses |
| Free wins first | Parallelization and sampling attempted before risky changes |
| One change at a time | Full eval suite run after each, with all three axes recorded |
| Threshold swept | Any cache or routing threshold chosen from a swept curve, per stratum |
| **At least one revert** | A change measured, found to cost quality, and reverted with reasoning |
| Target met | p95 halved, eval score within the noise band |

**The revert is required for full credit.** If every change helped, push harder until one does not: the reverted entry is the single most credible line in an eval report.
