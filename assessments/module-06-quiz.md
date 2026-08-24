# Module 6 Quiz — The Agent Loop from Scratch

**9 questions · ~35 min plus the practical**

This module is examined at mechanism level, because that is how interviewers examine it.

---

## Q1 — Short answer

Describe the tool-calling round trip in five steps. Then state precisely what the model does and does not execute.

---

## Q2 — Applied judgment

One tool, declared three ways, measured on 40 queries requiring correct selection among four tools:

| Declaration | Selection accuracy |
|---|---|
| `search` / *"Searches."* | 52% |
| `search_docs` / *"Search the product documentation for factual information about features, APIs, and configuration."* | 81% |
| Same + *"Use before answering any question about how the product works. Does not cover pricing or account data."* + enum-constrained `section` | 94% |

(a) What are the three levers moving accuracy here?
(b) Which clause in the third declaration is doing disproportionate work, and why?
(c) What does this imply about the "tool description" field in an MCP server you publish?

---

## Q3 — Diagnosis

An agent fails at step 7 of a 10-step task. The trajectory shows every step executing correctly. Context size per step: 1,200 → 8,400 → 18,900 → 27,100 → 34,800 → 41,000 → **failure**.

(a) What is happening?
(b) The tool returning the largest results is `search_docs`, returning ~6,200 tokens per call. Name two fixes and their trade-offs.
(c) Which fix would you apply first?

---

## Q4 — Design

Here is a working agent loop:

```python
while True:
    response = client.complete(messages, tools=schemas)
    messages.append(response.message)
    if not response.tool_calls:
        return response.text
    for call in response.tool_calls:
        messages.append(tool_result(dispatch(call)))
```

Name **five** control decisions this code makes implicitly, and state what each should be in a production system.

---

## Q5 — Diagnosis

An agent asked a question whose answer is not in the corpus:

```
step 1   search_docs("enterprise tier SLA")        → 3 irrelevant
step 2   search_docs("SLA")                        → 3 irrelevant
step 3   search_docs("service level agreement")    → 2 irrelevant
step 4   search_docs("enterprise SLA guarantee")   → 3 irrelevant
step 6   search_docs("enterprise tier SLA")        → 3 irrelevant
...
step 14  search_docs("enterprise service level")   → 2 irrelevant
```

(a) Is any individual step a bug?
(b) Name the three separate controls missing here.
(c) With those controls, how would this run end, and roughly how much cheaper?

---

## Q6 — Classification

For each tool failure, state whether your code should retry silently, return the error to the model, or terminate the run — and why:

(a) `TimeoutError` on an internal search service
(b) `PermissionError: 403` from an external API
(c) `ValidationError: date_from must precede date_to`
(d) A search returning zero results
(e) `RateLimitError: 429`

---

## Q7 — Applied judgment

Chaos test results, 50 runs at 30% tool failure:

| | Before | After |
|---|---|---|
| Completed | 34% | 78% |
| Refused cleanly | 0% | 12% |
| Crashed | 41% | 0% |
| Mean cost/run | $0.081 | $0.019 |
| **p95 cost/run** | **$0.310** | **$0.038** |

(a) Why is the p95 row more important than the mean?
(b) The "refused cleanly" row went from 0% to 12%. Is that an improvement or a regression? Explain.

---

## Q8 — Design

Classify each tool by blast radius and name the required controls:

(a) `search_docs` — reads the public documentation index
(b) `query_customer_db` — reads customer records
(c) `create_draft` — creates an unsent email draft
(d) `issue_refund` — issues a refund
(e) `run_python` — executes arbitrary code

---

## Q9 — Practical

Build a research agent that plans, searches, reads, and writes a sourced brief, and cannot exceed its step or cost budget.

**Deliverable:** `exercises/m6-research-agent/` with the agent, five logged trajectories, the budget justification derived from your own data, and an analysis of the worst trajectory and its cause.

---
---

# Answer Key

## Q1

**Five steps:**
1. You declare tools as JSON Schema alongside your messages
2. The model responds with a tool call — a name and a JSON argument object — instead of text
3. **Your code** validates the arguments, decides whether to permit the call, and executes it
4. You append a tool-result message tagged with the call ID
5. The model continues with the result in context — it may answer, or call another tool

**What the model executes: nothing.** It emits a *request*. Every safety property lives in step 3, which is entirely your code. The model has no ability to run anything, reach the network, or touch a database.

This distinction matters because it locates the entire security surface: not in the model's behavior, but in what your dispatcher permits.

## Q2

**(a)** The three levers: **a descriptive name**, **a description saying what the tool covers**, and **a description saying when to use it plus what it does not cover**, combined with **enum-constrained arguments**.

**(b)** *"Does not cover pricing or account data."* The 52% baseline's failures were largely the model reaching for `search` on questions another tool handled. A negative boundary clause resolves ambiguity between overlapping tools more efficiently than any amount of positive description.

**(c)** The tool description in an MCP server is prompt real estate in **someone else's agent**, and you will not be there to debug it. Write the "use when" and "does not cover" clauses with the same care as your own — and note the inverse (Lesson 7.4): a description you *consume* is untrusted text entering your model's context.

## Q3

**(a)** Context exhaustion. The conversation grows monotonically with every tool result; by step 6 it has consumed the window. Eleven steps of correct work discarded.

**(b) Two fixes:**

**Result projection and capping** — return the top 3 excerpts with source paths, capped at ~1,200 tokens, with explicit truncation the model can see. *Trade-off:* the model loses access to material it might have needed; mitigate with a `fetch_result` tool.

**Rolling summarization with state separation** — keep structured findings in your own state object and render a compact view into the conversation each turn. *Trade-off:* summarization is lossy and loses unpredictably — identifiers and qualifiers are exactly what it drops and what later steps need.

**(c) Projection first.** It is a smaller change, has no information-loss ambiguity, addresses the dominant contributor (6,200 tokens per call), and does not require restructuring the loop. Summarization is the second step once results are already lean.

## Q4 — Five of:

1. **Who decides it is done.** Currently: the model, unilaterally. Should be: a model signal *plus* a code-side check that required sub-goals completed, or a required explicit "final answer" tool.

2. **How many tool calls per turn.** Currently: unlimited. Should be: capped, with concurrency only for genuinely independent tools.

3. **Malformed tool call handling.** Currently: `dispatch` raises and the exception kills the run. Should be: return the validation error to the model as a tool result, bounded by a repair budget.

4. **What the model sees on tool failure.** Currently: undefined. Should be: per-tool error policy — transient errors retried invisibly, actionable errors surfaced, auth failures terminating.

5. **Whether the loop can refuse a call.** Currently: no authorization check at all. Should be: blast-radius classification, argument allowlists, approval gates on irreversible actions.

6. **What bounds the loop.** Currently: `while True`. Should be: step, token, wall-clock, cost, and per-tool budgets, checked *before* each call.

7. **Context growth.** Currently: unbounded append. Should be: state separated from conversation, with rendering per turn.

8. **Observability.** Currently: none. Should be: full trajectory logging with the `request_id` correlation ID.

## Q5

**(a) No.** Every step executed correctly, returned valid results, and was a reasonable next action in isolation. The model is being *persistent* — it believes the information exists and has no mechanism to conclude otherwise. This is why the failure is invisible in error logs.

**(b) Three controls:**
- **No termination condition** other than the model volunteering to stop
- **No repeated-action detection** — step 6 is byte-identical to step 1
- **No refusal path** — nothing lets the agent conclude "this is not in the corpus," which is the Lesson 5.4 behavior needed at loop level

**(c)** With repeated-action detection firing at step 4 and returning a message naming what was already tried and permitting the conclusion that the information is unavailable, the run terminates at roughly step 5 with reason `REFUSED`. Approximately **6,100 tokens and $0.009 versus 38,400 tokens and $0.058 — about six times cheaper**, and the outcome is *correct* rather than absent.

## Q6

**(a) Timeout → retry silently in your code** with backoff. Transient. The model cannot fix it and telling it wastes tokens and may induce a different, worse action.

**(b) 403 → terminate the run and alert.** The model cannot obtain permissions. Letting it retry produces repeated auth failures, which look like an attack to whoever is watching the target system.

**(c) Validation error → return to the model.** It misunderstood the tool's contract; the error text is exactly what it needs to correct. Bound the attempts.

**(d) Zero results → return as a normal result.** This is information, not an error. Treating it as a failure prevents the agent from concluding the material does not exist.

**(e) 429 → back off in your code**; if persistent, remove the tool from the available set for this run and tell the model it is unavailable. The model retrying makes the rate-limit situation worse.

## Q7

**(a)** The mean describes the typical run; **p95 describes your exposure**. Before, the worst 5% of runs cost nearly four times the mean — that long tail is what an unbounded agent does to a monthly bill, and it is completely invisible in per-call metrics and in mean-based dashboards. After bounding, p95 is twice the mean, which is a normal distribution shape.

**(b) An improvement.** Zero clean refusals before meant the agent had no way to conclude a question was unanswerable — it spun until something else stopped it. Twelve percent clean refusals means it now recognizes and reports that case. `REFUSED` is a successful outcome, which is why it must be a distinct termination reason from `BUDGET_EXHAUSTED`.

## Q8

**(a) `search_docs` — read-only, low sensitivity.** Argument validation, per-run rate limit.

**(b) `query_customer_db` — read-only, high sensitivity. The exfiltration surface.** Scoped read-only credential, row-level tenant filter in the credential (not in a model-supplied argument), rate limits, and — critically — it should not share a context with any tool that can send data outward.

**(c) `create_draft` — write, reversible, low sensitivity.** Argument validation, rate limit. No approval needed; nothing is sent.

**(d) `issue_refund` — write, irreversible, high sensitivity.** All of the above plus a scoped credential with an amount ceiling, a destination allowlist, a per-run limit of 1, an approval gate showing arguments and reasoning, and an idempotency key.

**(e) `run_python` — every quadrant at once.** Its control is not classification but **isolation**: a fresh container per execution, `network_mode: none`, dropped capabilities, read-only root, resource and wall-clock limits, and no credentials in the environment. The sandbox is the entire safety story.

## Q9 — Practical rubric

| Criterion | Full credit |
|---|---|
| No framework | Loop is the student's own code |
| Budgets | All five dimensions, checked before the call |
| Budget justification | Derived from the student's own trajectory data, not the lesson's defaults |
| Trajectory logging | Structured, with correlation ID, tokens, and latency per step |
| Termination | Typed reasons, with `REFUSED` distinguishable from `BUDGET_EXHAUSTED` |
| Worst-trajectory analysis | Identifies a specific cause, not "it went wrong" |

**The worst-trajectory analysis is the graded part.** "Step 3's tool result was 4,800 tokens of mostly-irrelevant material, which pushed the goal out of the attended region and caused the model to re-plan from scratch at step 5" is the target depth.
