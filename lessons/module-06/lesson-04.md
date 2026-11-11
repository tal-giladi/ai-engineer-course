---
id: "06.4"
module: 6
minutes: 12
practice_minutes: 90
prerequisites: ["06.3"]
objectives:
  - "Make the loop survive failure: tool errors, malformed calls, step and token budgets, explicit termination."
volatility: implementation
sources: []
last_verified: "2026-08-24"
---

# 06.4 · Failure Recovery, Budgets, and Termination

This is the lesson that separates a demo from a system, and it is the one interviewers probe hardest, because it is where production experience shows.

An agent is a loop that spends money, calls external services, and decides for itself when to stop. Every one of those properties is a liability without bounds. The fourteen-step spin from Lesson 6.2 is not an exotic edge case — it is the default behavior of an unbounded agent asked a question it cannot answer, which happens every day in production.

Your existing distributed-systems instincts transfer almost completely here. The one genuinely new thing is that the *cost* budget matters as much as the timeout, because this is the first system you have built where an infinite loop has a per-iteration price.

---

## Key Concept 1: The failure taxonomy

Four kinds of failure, four different responses.

**1. Malformed tool call.** The model emitted arguments that fail validation. This is Module 2's repair loop applied to tool arguments: return the validation error as the tool result, let the model correct, and **bound the attempts per call**. Two is usually right (Lesson 2.3). Beyond that, treat the tool as unavailable for this run.

**2. Tool execution failure.** The tool raised. The response depends on the error, and getting this policy right per tool is most of the work:

| Failure | Response | Why |
|---|---|---|
| Timeout, 503, connection reset | Retry with backoff in *your code*, invisible to the model | Transient. The model does not need to know. |
| Validation error from the tool itself | Return the error to the model | It misunderstood the tool; the message helps. |
| Auth failure, 403 | Fail the run, alert | **Never let the model retry this.** It cannot fix it, and repeated auth failures look like an attack. |
| Not found, empty result | Return as a normal result | This is information, not an error. |
| Rate limited | Back off in your code; if persistent, remove the tool for this run | The model retrying makes it worse. |

The general rule: **your code handles what the model cannot fix; the model sees what it can act on.** A traceback tells the model nothing useful and costs 400 tokens.

**3. Repeated action.** The model calls the same tool with the same arguments it already tried. Step 6 identical to step 1 in the Lesson 6.2 spin. Detect it by hashing (tool name, normalized arguments):

```python
sig = hash_call(call.name, call.arguments)
if sig in state.attempted:
    state.repeat_count += 1
    if state.repeat_count >= 2:
        return terminate(Reason.LOOP_DETECTED)
    return "You already tried this exact call and got: {prior_result}. " \
           "Try a different approach, or conclude the information is " \
           "unavailable."
```

That message matters as much as the detection. Telling the model *what it already tried and that it should consider concluding* is often enough to produce a graceful "this is not in the documentation" — the Lesson 5.4 refusal behavior, at the loop level.

**4. False completion.** The model returns an answer that does not address the goal. Detect with a verification step — a cheap call comparing answer to goal — or by requiring a structured completion where the model must state which sub-goals it satisfied. Note the cost: verification is a call per run, so it may belong only on high-stakes paths.

---

## Key Concept 2: Budgets are first-class parameters

Five dimensions. Set all of them explicitly, at construction:

```python
@dataclass(frozen=True)
class Budget:
    max_steps: int = 15
    max_tokens: int = 60_000
    max_wall_clock_s: float = 90.0
    max_cost_usd: float = 0.25
    max_calls_per_tool: dict[str, int] = field(default_factory=dict)
```

**`max_steps`** — the coarse safety net. Set it from your trajectory data: measure p95 steps on tasks that succeed, add headroom.

**`max_tokens`** — catches the case where few steps consume enormous context.

**`max_wall_clock_s`** — because a user is waiting, and 30 seconds of correct work is still a bad experience if they left at 12.

**`max_cost_usd`** — the one that prevents an incident rather than an annoyance. Everything else bounds a single run's inconvenience; this bounds your exposure. It is also the dimension people omit, because the others feel like "real" engineering.

**`max_calls_per_tool`** — per-tool limits. An agent that calls `search_docs` twenty times is spinning; one that calls `send_email` twice may have sent two emails.

Check budgets **before** each model call and **before** each tool execution, not after. Exceeding a budget and then doing the work anyway is not a budget.

---

## Key Concept 3: Termination must be typed

`while True` has one exit. A real agent has several, and the caller needs to distinguish them:

```python
class Reason(Enum):
    COMPLETED = "completed"
    BUDGET_EXHAUSTED = "budget_exhausted"
    LOOP_DETECTED = "loop_detected"
    UNRECOVERABLE_ERROR = "unrecoverable_error"
    CANCELLED = "cancelled"
    REFUSED = "refused"           # concluded it cannot be answered

@dataclass
class AgentResult:
    reason: Reason
    answer: str | None
    state: AgentState
    trajectory: list[Step]
    budget_used: Budget
    partial: bool
```

Why each matters to a caller:

- `COMPLETED` → show the answer
- `REFUSED` → show the refusal; this is a **good** outcome, not a failure
- `BUDGET_EXHAUSTED` → show partial findings, offer to continue; alert if frequent
- `LOOP_DETECTED` → show partial findings; **this is a bug report** — investigate the query
- `UNRECOVERABLE_ERROR` → apologize, alert, do not retry blindly
- `CANCELLED` → user left; Module 10 makes sure this actually stops the spend

`BUDGET_EXHAUSTED` and `REFUSED` returning the same shape to a caller is a common design error. One means "I could not finish"; the other means "I finished, and the answer is no."

**Always return partial state.** Fourteen steps of findings are worth something even without a final answer.

---

## Key Concept 4: Chaos testing

You cannot verify failure handling on tools that work. Build a chaos harness — a wrapper injecting realistic failures:

```python
class ChaosTool:
    def __call__(self, args):
        r = self._rng.random()          # seeded for reproducibility
        if r < 0.15: raise TimeoutError()
        if r < 0.25: raise ConnectionError()
        if r < 0.30: return PLAUSIBLE_GARBAGE      # the nasty one
        if r < 0.32: raise PermissionError()
        return self._real(args)
```

The plausible-garbage case is the most valuable and the most often skipped. A tool that returns *wrong but well-formed* data tests whether your agent notices, and it usually does not — which is a finding worth having before production rather than after.

Seed the RNG so failures are reproducible.

---

## Worked Example: The spin, bounded

The Lesson 6.2 fourteen-step spin, with this lesson's controls applied:

```
step 1  search_docs("enterprise tier SLA")      → 3 irrelevant
step 2  search_docs("SLA")                      → 3 irrelevant
step 3  search_docs("service level agreement")  → 2 irrelevant
step 4  search_docs("enterprise tier SLA")      → REPEAT DETECTED
        → returned to model: "You already tried this exact call and
          got 3 irrelevant excerpts. Try a different approach, or
          conclude the information is unavailable."
step 5  model → "The documentation does not cover enterprise tier
        SLAs. It covers rate limits and uptime reporting, but no
        service level agreement terms."

TERMINATED: REFUSED
4 steps, 6,100 tokens, 5.1 s, $0.009
```

Versus 14+ steps, 38,400 tokens, $0.058, and no answer. **Six times cheaper, and the outcome is correct** — an honest refusal instead of an abandoned run.

**Chaos test results**, 50 runs at 30% tool failure, before and after:

| | Before | After |
|---|---|---|
| Completed successfully | 34% | 78% |
| Refused cleanly | 0% | 12% |
| Crashed (exception escaped) | 41% | 0% |
| Ran to budget with no result | 25% | 10% |
| Mean cost per run | $0.081 | $0.019 |
| p95 cost per run | $0.310 | $0.038 |

The p95 column is the one to internalize. Before, the worst 5% of runs cost nearly four times the mean. That long tail is what an unbounded agent does to a monthly bill, and it is invisible in per-call metrics.

And the finding from the plausible-garbage injection: in 6 of 50 runs the agent accepted wrong-but-well-formed tool output and produced a confident wrong answer. No control in this lesson catches that — it is a grounding and verification problem, and worth recording as a known limitation.

---

## Practice

**1. Implement per-tool error policies.** A table mapping error type to response per tool. Transient errors retry in your code with backoff; auth failures terminate; not-found returns as a normal result.

**2. Add repeated-action detection** with normalized argument hashing, and write the corrective message carefully — it should include what was already tried and explicitly permit concluding unavailability.

**3. Implement all five budget dimensions,** checked before each model call and each tool execution. Derive `max_steps` and `max_cost_usd` from your own trajectory data, not from these defaults.

**4. Add typed termination.** Six reasons, `AgentResult` with partial state and trajectory. Ensure `REFUSED` and `BUDGET_EXHAUSTED` are distinguishable by a caller.

**5. Build the chaos harness** with seeded randomness, including the plausible-garbage case.

**6. Run 50 chaos runs before and after** your changes. Report the table above with your own numbers, including p95 cost.

**7. Re-run the spin.** Confirm it now terminates as `REFUSED` within your budget, and record the cost difference.

**8. Extend your eval harness** to score trajectories, not just answers: task success rate, mean steps, mean and p95 cost, tool selection accuracy, and the distribution of termination reasons. Append to `eval-report.md`.

---

## Visual Aid

*A state diagram. Central state `RUNNING` with a self-loop labeled "step." Five outgoing transitions to terminal states, each labeled with its trigger: → `COMPLETED` ("model returns final answer"), → `REFUSED` ("concluded unavailable — a good outcome," drawn in green), → `LOOP_DETECTED` ("repeat threshold," drawn amber with a note "investigate the query"), → `BUDGET_EXHAUSTED` ("any of 5 dimensions," with the five listed in a small box), → `UNRECOVERABLE_ERROR` ("auth failure, unhandled"). A sixth transition `CANCELLED` comes in from outside, labeled "client disconnect → Module 10." Along the self-loop, two small guard boxes: "check budgets BEFORE the call" and "check repeat signature."*

---

## Key Takeaways

- **Classify every failure and respond per class.** Your code handles what the model cannot fix; the model only sees errors it can act on. Never let the model retry an auth failure.
- **Repeated-action detection is the highest-value single control** — and the corrective message should permit the model to conclude unavailability, which turns a spin into a clean refusal.
- **Five budget dimensions,** checked before the work, not after. The cost budget is the one that prevents an incident rather than an annoyance, and it is the one most often omitted.
- **Termination must be typed.** `REFUSED` and `BUDGET_EXHAUSTED` are completely different outcomes and callers must be able to tell them apart. Always return partial state.
- **Chaos testing is the only way to verify any of this,** and the plausible-garbage case will find a failure the other controls do not catch.

---

## Next

Your agent is bounded, recovers from failure, and exits cleanly. It also just executed a tool call whose arguments were written by a model, based on text it retrieved from a corpus you do not fully control.

Right now the only thing standing between a badly-chosen tool call and your infrastructure is that nothing has tried yet.

**Lesson 6.5** classifies your tools by blast radius, scopes their capabilities, gates the irreversible ones, and puts code execution inside a container that can actually contain it.
