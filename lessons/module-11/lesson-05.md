---
id: "11.5"
module: 11
minutes: 12
practice_minutes: 90
prerequisites: ["11.4"]
objectives:
  - "Keep the system inside a cost and reliability envelope: retries with backoff, provider fallback routing, rate limits, and runaway-agent kill switches."
volatility: implementation
sources: []
last_verified: "2026-08-24"
---

# 11.5 · Reliability and Cost: Retries, Fallback Routing, Kill Switches

Your system's most critical dependency is a third-party API with variable latency, aggressive rate limits, occasional outages, and a deprecation schedule you do not control. Inside it sits an agent that spends money autonomously.

Most of this is familiar distributed-systems work and your instincts transfer. Two things do not. **Retried tool calls can double-execute side effects** in a way that ordinary idempotent HTTP retries do not prepare you for — and this got worse in Module 7 when you made runs resumable. And **cost is a failure dimension**: a runaway loop here has a per-iteration price, which no other system you have built does.

---

## Key Concept 1: Retries that do not make things worse

**Exponential backoff with jitter.** Without jitter, every client that failed during an outage retries simultaneously and re-creates the outage on recovery.

```python
delay = min(base * (2 ** attempt), cap) * (0.5 + random() * 0.5)
```

**Retry only genuinely transient failures:**

| Status | Retry? |
|---|---|
| 429 rate limited | Yes — respect `Retry-After` if present |
| 500, 502, 503, 504 | Yes |
| Timeout, connection reset | Yes |
| 400 bad request | **No** — it will fail identically |
| 401, 403 | **No** — and alert; repeated auth failures look like an attack |
| 422 content filtered | **No** — retrying produces the same refusal |

This is the Lesson 6.4 taxonomy at the provider layer.

**The idempotency problem.** A tool with a side effect that times out *may have succeeded*. Retrying sends the second email, issues the second refund, creates the second ticket.

This matters more here than in a conventional service for two reasons: your agent may retry at the loop level *and* your HTTP client may retry *and* the framework may retry (Lesson 7.1's layered-retry hazard), and Module 7's checkpointing means a crashed run resumes and re-executes the step.

```python
def issue_refund(args, *, idempotency_key: str):
    if prior := already_processed(idempotency_key):
        return prior
    ...
```

Derive the key from the trace ID and step index, so it is stable across retries and resumes.

**Audit your retry layers.** Three layers each retrying three times is 27 attempts and 27 charges. Decide which layer owns retries and disable the others.

---

## Key Concept 2: Fallback routing

The Lesson 1.2 seam, finally used for its most important purpose.

```python
FALLBACK_CHAIN = [
    Tier("primary",  profile="frontier",     timeout=20),
    Tier("secondary",profile="frontier-alt", timeout=20),
    Tier("local",    profile="local-tuned",  timeout=10),
    Tier("degraded", mode="rag_only",        timeout=5),   # no agent loop
]
```

Four tiers, each with different quality. The last one is worth noting: when everything is failing, a plain RAG answer with citations is far better than an error page, and it does not need an agent loop.

**Measure quality at every tier.** A fallback that silently halves quality is an incident nobody notices — users get worse answers and no alert fires. Run your Module 3 eval suite against each tier and record the results. This is the step most teams skip and it is what makes the fallback chain a design rather than a hope.

**Tell the system which tier answered.** Put it in the trace (Lesson 10.4) and in the response metadata, so degraded answers are attributable afterward.

---

## Key Concept 3: Circuit breakers and rate limits

**Circuit breakers.** A provider returning errors after 20 seconds is consuming your entire latency budget to fail. After N consecutive failures, open the circuit: fail fast to the next tier, retry a single probe after a cooldown, close on success.

**Rate limits are capacity planning, not error handling.** You know your provider quota. Track consumption with a token bucket and shape traffic *before* hitting the limit — queue with the backpressure from Lesson 10.1, shed low-priority work explicitly, or route to a tier with separate quota.

Waiting to be told you are over the limit means every user in that window sees a failure.

---

## Key Concept 4: Cost controls

The dimension unique to this field, and the one that produces incidents.

**Per-run budget.** Lesson 6.4 — already built. Verify it checks *before* the call, which is where the Lesson 11.4 red team found a bug.

**Per-tenant daily cap.** A single customer, or a single bug, must not consume the whole budget. Cap, alert at a threshold, and degrade rather than fail when the cap is hit.

**Anomaly detection on cost per request.** Alert on the *rate*, not the total — a monthly total tells you after the fact. Cost per request tripling within an hour is the signal, and your Lesson 1.4 telemetry with `call_site` breakdown is what makes it diagnosable in seconds.

**A kill switch reachable without a deploy.** This is the control that determines how long an incident lasts:

```python
if flags.get(f"agent.{name}.enabled") is False:
    return degraded_response(reason="temporarily unavailable")
```

Granular: disable one tool, one agent, one tenant, or the whole agent path. Config or a feature-flag store — never a code change, because a deploy during an incident is ten to thirty minutes of continued spend.

Test that it works. An untested kill switch is a comforting idea.

---

## Worked Example 1: Chaos test with quality per tier

Provider A subjected to 500s, then 429s, then timeouts.

| Condition | Availability | Tier serving | Quality | p95 |
|---|---|---|---|---|
| Normal | 100% | primary | 0.87 | 2.9 s |
| A returns 500s | 100% | secondary | 0.86 | 3.4 s |
| A + B failing | 100% | local | **0.79** | 2.1 s |
| All models failing | 100% | degraded (RAG only) | **0.61** | 1.4 s |
| Everything down | 0% | — | — | — |

Availability holds to the last tier. **The quality column is the finding**: on the local tier users get 0.79 instead of 0.87, and on the degraded tier 0.61 — and without this table nobody would know, because there is no error.

Two actions followed: an alert when more than 5% of traffic serves below the primary tier, and a visible notice to users on the degraded tier.

## Worked Example 2: A cost incident, contained

An injected prompt (surviving from Lesson 11.4) induced a retrieval-and-fetch loop.

**Without controls, projected:**

```
14:02  cost/request $0.003 → normal
14:07  cost/request $0.019 → 6× (unnoticed)
14:31  cost/request $0.047
16:00  cumulative additional spend: ~$1,900
```

**With controls, actual:**

```
14:02  cost/request $0.003
14:07  cost/request $0.019
       → anomaly alert: 6× baseline, call_site=agent.step
14:09  per-tenant daily cap reached for tenant_4471
       → that tenant degraded to rag_only; others unaffected
14:14  engineer flips flag: agent.fetch_url.enabled = false
       → no deploy, effective in 30 seconds
14:16  root cause identified from traces (Lesson 10.4)
14:48  fix deployed, flag re-enabled

Additional spend: $61
```

Three controls, in the order they fired: **anomaly alerting** made it visible in five minutes, **the per-tenant cap** bounded the blast radius to one customer, and **the kill switch** stopped it in thirty seconds without a deploy. The trace store located the cause.

$61 instead of $1,900. And the per-tenant cap is the underrated one — without it, one tenant's incident degrades everyone.

---

## Practice

**1. Implement retries** with exponential backoff and jitter, retrying only genuinely transient statuses. Respect `Retry-After`.

**2. Audit your retry layers.** Count how many places can retry the same call — your code, your HTTP client, your framework. Pick one owner and disable the rest. **Report the multiplier you found.**

**3. Add idempotency keys** to every tool with a side effect, derived from trace ID and step. Then deliberately crash after a side effect and confirm resume does not duplicate it.

**4. Build the fallback chain** — at least three tiers ending in a non-agentic degraded mode.

**5. Measure quality at every tier** with your Module 3 eval suite. **Produce the quality-per-tier table.** This is the deliverable most teams do not have.

**6. Add a circuit breaker** per provider, and record the tier that served each request in the trace and the response metadata.

**7. Implement cost controls:** per-run budget checked *before* each call, per-tenant daily cap with degradation, and anomaly alerting on cost per request broken down by `call_site`.

**8. Build a kill switch** — config or feature flag, granular to tool and agent level, effective without a deploy. **Test it**, and time how long it takes to take effect.

**9. Run a chaos test** covering 500s, 429s, timeouts, and a cost anomaly. Report availability, serving tier, quality, latency, and spend under each condition.

---

## Visual Aid

*A vertical fallback ladder, four rungs. Each rung is a box labeled with tier name and profile, annotated on the right with availability, quality score, p95 latency, and cost. Quality is shown as a coloured bar shrinking down the ladder — 0.87 green, 0.86 green, 0.79 amber, 0.61 red — making the silent degradation visible at a glance. A dashed horizontal line across the amber tier is labeled "alert if >5% of traffic serves below this." To the left, three control icons attached at different points: a circuit-breaker symbol between tiers, a cost-cap gauge beside the whole ladder, and a large switch icon at the top labeled "kill switch — config, no deploy, 30 s."*

---

## Key Takeaways

- **Retry only transient failures, with jittered backoff,** and audit how many layers can retry the same call — three layers of three retries is 27 charges.
- **Idempotency keys are mandatory once runs are resumable.** A timed-out side effect may have succeeded, and Module 7's checkpointing re-executes on resume.
- **Fallback tiers must have measured quality.** Silent degradation is an incident nobody notices; alert when traffic serves below the primary tier and tell users when it is degraded.
- **Rate limits are capacity planning.** Shape traffic before the limit rather than handling errors after it.
- **Cost is a failure dimension.** Per-run budgets checked before the call, per-tenant caps that bound the blast radius, anomaly alerting on the rate, and a kill switch that works without a deploy — tested and timed.

---

## Next

You have built a complete production AI system: measured, retrievable, agentic, tuned, served, observable, defended, and bounded. Twelve modules of work, and every claim in it is backed by a number in an eval report.

What you do not have yet is the version of it that gets you hired. A reviewer will spend four minutes on your repository, an interviewer will push every claim you make three levels deep, and a system design round will ask you to do all of this again on a whiteboard for a problem you have never seen.

**Module 12** turns eleven modules of artifacts into a portfolio and rehearses both rounds — and you will find that almost every document a hiring panel wants has been accumulating since Lesson 1.1.
