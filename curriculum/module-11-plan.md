# Module 11 Plan — Security, Safety & Reliability

**Duration:** 60 min · **Lessons:** 5 · **Depends on:** Modules 6, 10
**Outline reference:** `curriculum/course-outline.md` §8, Module 11

---

## Prerequisites

**From Module 6:** the agent with tools, blast-radius classification, and sandboxing — Lesson 6.5 handled *unknown* input; this module handles *hostile* input.
**From Module 10:** the deployed service, its tracing, and its client.
**From Module 3:** the eval harness, which becomes the security regression suite — every successful attack becomes a permanent test case.
**From Modules 4, 5, 7:** the retrieval corpus and MCP servers, both of which are attack surfaces here.

---

## Framing and Scope

This is defensive security for systems the student owns. Students attack **their own** agent in a local sandbox to build the defensive instinct that hiring managers look for and that appears in postings as "prompt injection defense" and "guardrails." Nothing in this module targets third-party systems, and the red-teaming lesson is explicitly scoped to the student's own deployment.

Reliability shares the module because the two concerns share a shape: both are about what the system does when conditions are not the happy path, and both are engineered with budgets, boundaries, and fallbacks rather than with hope.

---

## Module Learning Objectives

**M11-LO1** — Explain prompt injection, direct and indirect, and why untrusted content plus tools plus an exfiltration path is the dangerous combination.
**M11-LO2** — Design tool permissions on least-privilege lines, with sandboxing and approval gates for consequential actions.
**M11-LO3** — Handle PII and sensitive output: detection, redaction, filtering, and what to log versus what never to log.
**M11-LO4** — Red-team their own agent systematically and turn each successful attack into a regression test.
**M11-LO5** — Keep the system inside a cost and reliability envelope: retries with backoff, provider fallback routing, rate limits, and runaway-agent kill switches.

---

## Lesson 11.1 — Prompt Injection: Direct, Indirect, and Why It Isn't Solved

*13 min · serves M11-LO1*

**Concept.** The root cause, stated precisely because it is usually stated vaguely: an LLM receives instructions and data in the same channel with no reliable mechanism to distinguish them, so any text that reaches the context can act as an instruction. This is not a bug with a patch pending; it is a property of the architecture. Cover **direct injection**, where the user is the attacker and the target is usually your system prompt, your tools, or your guardrails. Cover **indirect injection**, which is the one that matters for everything built in this course: instructions arrive inside *content the system retrieves* — a document in the Module 4 corpus, a web page an agent fetches, a tool description from an MCP server in Lesson 7.4, a code comment in a file the agent reads. The user is not attacking; the data is. Then the composition rule that makes this actionable: risk requires all three of **untrusted content**, **tool access**, and **an exfiltration path** — and removing any one of the three breaks the chain, which is the design lever. Cover mitigations honestly and in order of real effectiveness: architectural (separate the agent that reads untrusted content from the agent that holds sensitive tools — the legitimate multi-agent case from Lesson 7.3), boundary controls (egress allowlists, no rendering of model-supplied URLs or images), then detection and instructional defenses, which reduce incidence and never eliminate it.

**Example.** A poisoned document added to the spine corpus containing an instruction to append a summary of prior conversation to a URL — retrieved normally, followed by the agent, exfiltrating data. The attack is then broken three ways in sequence: by removing the tool, by an egress allowlist, and by architectural separation, showing that the boundary controls hold where instructional pleading does not.

**Practice.** Students poison their own corpus with three injection payloads, confirm the attack succeeds against their current agent, then map their system's three-part risk chain — where untrusted content enters, which tools are reachable from there, what exfiltration paths exist — and document it as a threat model.

**Summary.** Instructions and data share a channel and always will. Design so untrusted content never sits in the same context as both sensitive tools and an egress path. Detection is a supplement, never the plan.

**Key terms:** prompt injection, direct vs indirect injection, instruction/data confusion, untrusted content, exfiltration path, egress allowlist, architectural isolation, threat model.
**Visual aid:** The three-circle risk diagram — untrusted content, tool access, exfiltration — with the dangerous intersection shaded and each break point labeled.
**Hook:** "You mapped which tools are reachable from untrusted content. Most of them shouldn't be."

---

## Lesson 11.2 — Tool Permissions, Sandboxing, and Least Privilege

*12 min · serves M11-LO2*

**Concept.** The defense that survives when the model is fully compromised — assume the attacker controls the agent's next tool call and ask what it can do. This inverts the design question from "how do I stop bad instructions?" to "what is the worst outcome if bad instructions win?", which is answerable with ordinary engineering. Cover **capability scoping**: the agent runs with its own narrow identity, never the caller's full rights and never a service account with broad access — a read-only database role for a read-only agent, scoped API tokens per tool, per-tenant credentials in multi-tenant systems. Cover **argument-level control**: allowlists on destinations, path restrictions, parameterized queries, and bounded numeric ranges — because the model chooses the arguments, and validating a tool call is validating untrusted input, which is Module 2's discipline with consequences attached. Cover **approval gates** on irreversible actions, now implemented durably with Module 7's interrupts, and the human-factors caveat that a gate firing on every action trains people to approve reflexively — so gates must be rare enough to be read. Cover **sandbox hardening** beyond Lesson 6.5: no network by default, dropped capabilities, read-only mounts, resource and wall-clock limits, no credentials in the sandbox environment. Cover **rate limits per tool per session**, which turn a successful exploit into a bounded one.

**Example.** The Lesson 11.1 exfiltration attempted again against a hardened agent — the tool exists but its egress allowlist rejects the destination, the credential is read-only, and the per-session rate limit caps repeated attempts. The injection still *works* on the model; it accomplishes nothing.

**Practice.** Students harden every tool in their agent: scoped credentials, argument allowlists, per-tool rate limits, approval gates on irreversible actions, and a hardened sandbox. They re-run their three Lesson 11.1 payloads and document what each defense stopped, plus any that still succeed.

**Summary.** Assume the model is compromised and constrain what a compromised model can reach. Least privilege, validated arguments, bounded rates, and rare gates that are actually read.

**Key terms:** least privilege, capability scoping, scoped credentials, argument allowlist, approval gate, approval fatigue, sandbox hardening, per-tool rate limiting, blast-radius containment.
**Visual aid:** Tool permission matrix — tool, credential scope, argument constraints, rate limit, approval required — filled in for the spine agent.
**Hook:** "Exfiltration is contained. The system still writes every prompt and response into a trace store, and some of those contain things they shouldn't."

---

## Lesson 11.3 — PII, Output Filtering, and Compliance Basics

*11 min · serves M11-LO3*

**Concept.** The data-handling obligations that come with an LLM system, aimed at an engineer who will be asked about them rather than at a compliance officer. Cover the surfaces PII actually reaches, which is more than people expect: prompts, model responses, the Module 10 trace store, the Module 7 checkpoint store, the semantic cache from Lesson 10.3, the fine-tuning datasets built from production traces in Lesson 8.2, and any provider that receives the request. Cover **input handling** — detection and redaction before the boundary, tokenized placeholders that let the model work with structure while the real value stays out, and the honest limitation that detection is imperfect. Cover **output filtering** — checking responses for leaked PII, secrets, and content the retrieved context did not support, which is the Lesson 5.4 grounding check reused as a safety control. Cover **retention** — trace and checkpoint stores need expiry policies, and the semantic cache is a cross-user leak risk if cache keys are not scoped per tenant, which is a real and commonly missed bug. Cover **provider posture** — data residency, retention, and training-use terms differ, and this is a genuine architectural input that sometimes forces self-hosting, connecting back to Module 9's economics. Cover **user-facing honesty**: disclose AI involvement, expose citations, and make refusal visible.

**Example.** A trace store searched for email addresses and API keys, both present. Then the redaction layer added and re-verified. Then the semantic cache leak demonstrated concretely: user A's cached answer served to user B because the cache key was the embedding alone with no tenant scope.

**Practice.** Students add PII detection and redaction at the boundary, scope the semantic cache per tenant, set retention on traces and checkpoints, and audit their own trace store and fine-tuning dataset for leaked PII, reporting what they found.

**Summary.** PII spreads to every store the system keeps — traces, checkpoints, caches, training sets. Redact at the boundary, scope caches per tenant, expire what you keep, and know your provider's terms.

**Key terms:** PII detection, redaction, tokenized placeholder, output filtering, secret scanning, data retention, cache key scoping, cross-tenant leak, data residency, training-use terms, AI disclosure.
**Visual aid:** Data-flow diagram with every persistence point marked and labeled with its retention policy and redaction status.
**Hook:** "You've built defenses against attacks you thought of. Now go looking for the ones you didn't."

---

## Lesson 11.4 — Red-Teaming Your Own Agent

*12 min · serves M11-LO4*

**Concept.** Systematic adversarial testing of a system you own, in an environment you control. Cover the method rather than a trick list, because trick lists expire: enumerate from the threat model built in Lesson 11.1 — every untrusted input channel crossed with every tool and every sensitive data store — and generate test cases per cell rather than improvising. Cover the attack **categories** at the level that stays true: instruction override, context manipulation, tool abuse (getting a legitimate tool to do something illegitimate through its arguments), data exfiltration through any channel that leaves the system, and resource exhaustion, which is the one people forget and the one that shows up as a bill. Cover **automation** — an attacker model generating variants against your agent, scored by an outcome judge, which is Module 3's machinery pointed at security and scales far past manual probing. Cover the discipline that makes this professional rather than performative: **every successful attack becomes a permanent test case in the Module 3 suite**, so it can never silently return — this is the security equivalent of a regression test and is exactly what an interviewer is probing when they ask how you handle injection. Cover honest reporting: what was tried, what worked, what was fixed, what remains open with a stated risk acceptance.

**Example.** A structured red-team run over the spine agent — forty generated attacks across five categories — with six successes: two injections, one tool-argument abuse, one exfiltration via a citation URL, and two resource exhaustions. Each is triaged, fixed, and converted into a regression case, with one accepted as low-risk and documented rather than fixed, modeling honest risk acceptance.

**Practice.** Students run a systematic red-team against their own deployed agent, at minimum three attacks per category derived from their threat model, using an automated attacker for at least one category. They fix what they can, convert every success into a regression test, and write a security report with an open-issues section.

**Summary.** Enumerate from the threat model, automate the volume, fix what you can, test what you fixed, and document what you accepted. A security report with honest open issues is stronger than one claiming none.

**Key terms:** red-teaming, threat enumeration, instruction override, tool abuse, exfiltration channel, resource exhaustion, automated adversarial generation, security regression suite, risk acceptance.
**Visual aid:** Red-team coverage matrix — input channels against attack categories — with cells shaded by outcome: not attempted, blocked, succeeded-then-fixed, accepted.
**Hook:** "Attackers are one source of failure. Providers going down, rate limits, and your own agent spending money in a loop are the others."

---

## Lesson 11.5 — Reliability and Cost: Retries, Fallback Routing, Kill Switches

*12 min · serves M11-LO5*

**Concept.** Operating a system whose critical dependency is a third party with variable latency, rate limits, and outages — and an internal component that can spend money autonomously. Cover **retries** done correctly: exponential backoff with jitter, retry only on genuinely transient status, and the LLM-specific hazard that a retried non-idempotent tool call can double-execute a side effect, so idempotency keys matter more here than in ordinary services. Cover **fallback routing** through the Lesson 1.2 seam — provider B when provider A is down, a smaller model when the large one is saturated, a degraded non-agentic path when everything is failing — with the quality implication measured rather than assumed, since a fallback that quietly halves quality is an incident nobody notices. Cover **circuit breakers** so a failing provider stops absorbing the latency budget. Cover **rate limit handling** as capacity planning rather than error handling: token-bucket accounting against the provider's actual limits, queue with backpressure from Lesson 10.1, and shed load explicitly. Then the AI-specific one: **cost controls**. Per-run budgets from Lesson 6.4, per-tenant daily caps, anomaly alerting on cost-per-request, and a kill switch that can disable an agent or a tool without a deploy — because the runaway loop that costs a fortune is the incident this category actually produces.

**Example.** A chaos test on the deployed service — provider A returning 500s, then rate-limiting, then timing out — with the fallback chain engaging and a quality-under-degradation table showing exactly what users receive on each fallback tier. Then a cost anomaly: an injected prompt that induces a retry loop, caught by the per-tenant cap and the alert, with the kill switch flipped from config.

**Practice.** Students implement retries with jitter and idempotency keys, a fallback chain with measured quality per tier, a circuit breaker, per-run and per-tenant cost caps with alerting, and a config-driven kill switch. They run a chaos test and report availability, quality, and cost under each failure condition.

**Summary.** Retry transient failures only, fall back with known quality cost, break the circuit on a sick provider, and cap spend per run and per tenant with a kill switch you can reach without a deploy.

**Key terms:** exponential backoff, jitter, idempotency key, fallback routing, graceful degradation, circuit breaker, token bucket, load shedding, cost cap, cost anomaly detection, kill switch.
**Visual aid:** Fallback chain diagram — tiers from primary to degraded — with availability, latency, quality, and cost annotated per tier.
**Hook:** "The system is built, measured, deployed, defended, and bounded. Module 12 turns it into the thing that gets you hired."

---

## Spine Increment

**Deliverable:** `spine/security/` and hardened service configuration — a documented threat model, scoped tool permissions with argument allowlists, PII redaction with scoped caches and retention, a security regression suite, and a full reliability and cost-control layer.

**Acceptance criteria:**
1. A threat model naming every untrusted input channel, reachable tool, and exfiltration path.
2. All three Lesson 11.1 payloads contained; each defense's contribution documented.
3. Every tool has scoped credentials, argument constraints, and a rate limit; irreversible actions gated durably.
4. Trace store, checkpoint store, and fine-tuning dataset audited for PII; cache keys scoped per tenant; retention policies set.
5. A red-team report covering five categories, with every success converted into a regression test and open issues stated with accepted risk.
6. Chaos test passes with a documented quality-per-tier fallback table; cost caps and kill switch demonstrated.

---

## Standalone Exercise

Attack a supplied deliberately vulnerable agent in a local sandbox, document every successful exploit with its mechanism, then fix and regression-test each one. Deliverable: `exercises/m11-vulnerable-agent/` with the attack log, the fixes, the regression suite, and a note on any vulnerability that could not be fixed without removing a feature — and what you would recommend.

---

## Dependencies

| Lesson | Requires |
|---|---|
| 11.1 | Modules 6, 10; corpus from 4 |
| 11.2 | 11.1, Lessons 6.5 and 7.2 |
| 11.3 | Module 10 stores; Lessons 8.2, 10.3 |
| 11.4 | 11.1–11.3, Module 3 harness |
| 11.5 | Module 10 service, Lessons 1.2 and 6.4 |

**Downstream:** Module 12 presents the threat model, security report, and reliability design as portfolio and interview material — this module produces several of the strongest artifacts in the repository.

---

## Assessment Focus (Phase 4)

Threat reasoning and operational design. Given an architecture, identify where untrusted content meets tools and egress. Given a poisoned document, trace the attack and name the cheapest effective break point. Given a semantic cache serving another tenant's answer, diagnose and fix. Given a retried tool call that sent two emails, name the missing control. Given a fallback chain, state what quality users receive at each tier. One practical: harden a supplied vulnerable agent so that a provided attack set fails, without removing functionality.
