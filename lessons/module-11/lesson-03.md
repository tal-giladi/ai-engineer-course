---
id: "11.3"
module: 11
minutes: 11
practice_minutes: 75
prerequisites: ["11.2"]
objectives:
  - "Handle PII and sensitive output: detection, redaction, filtering, and what to log versus what never to log."
volatility: implementation
sources: []
last_verified: "2026-08-24"
---

# 11.3 · PII, Output Filtering, and Compliance Basics

This lesson is aimed at an engineer who will be asked about data handling in a design review, not at a compliance officer. It is about knowing where your system keeps user content, which is more places than you think, and what to do about each.

Over ten modules you have quietly built six or seven stores of user data. Most teams inventory two of them.

---

## Key Concept 1: The data inventory

Every place user content persists in the system you have built:

| Store | Contains | Built in |
|---|---|---|
| **Trace store** | Full prompts, responses, tool arguments, retrieved documents | 10.4 |
| **Checkpoint store** | Complete agent state, conversation history | 7.2 |
| **Semantic cache** | Queries and their answers | 10.3 |
| **Embedding cache** | Query text and vectors | 10.3 |
| **Telemetry JSONL** | Inputs, outputs, metadata | 1.4 |
| **Fine-tuning dataset** | Production traces as training data | 8.2 |
| **Long-term memory** | Accumulated user facts across sessions | 6.3 |
| **The provider** | Everything you send, subject to their terms | 1.2 |

The trace store and the fine-tuning dataset are the ones that surprise people. The trace store holds *more* than your primary database — it holds the full text of everything, including retrieved documents the user may not have been entitled to see in aggregate. The fine-tuning dataset takes production content and bakes it into weights, which is functionally permanent.

**Write this inventory down.** For each store: what it contains, where it lives, who can read it, and how long it is kept. That document is the deliverable of this lesson, and it is a document a hiring panel will recognize.

---

## Key Concept 2: Input handling

**Detection and redaction before the boundary.** Emails, phone numbers, payment details, national IDs, credentials, internal hostnames. Regex catches the structured ones; named-entity models catch names and addresses more reliably.

**Detection is imperfect.** It misses unusual formats and produces false positives that damage legitimate content. Treat it as risk reduction, not a guarantee — and log the detection rate so you know roughly what it is catching.

**Tokenized placeholders** preserve utility while keeping real values out:

```python
text, vault = redact(text)
# "Contact <PERSON_1> at <EMAIL_1> about order <ORDER_1>"
answer = model(text)
answer = restore(answer, vault)      # substitute back on the way out
```

The model reasons over structure; the real values never leave your boundary. This works well for extraction and routing, less well where the actual values matter to the answer.

**Decide what you send to which provider.** This may force a self-hosted model for sensitive paths — which is a genuine architectural input, and one where Module 9's economics and Module 10's serving work become the enabling capability rather than an optimization.

---

## Key Concept 3: Output filtering

Check responses before they reach the user:

- **PII that should not be there** — the model surfacing another user's data from context
- **Secrets** — API keys, connection strings, internal hostnames leaked from tool results or retrieved documents
- **Unsupported claims** — this is the Lesson 5.4 grounding check, reused as a safety control
- **Model-composed URLs** — the citation-link exfiltration from Lesson 11.2 that no tool control caught. Validate that every URL in an answer points to a document actually retrieved, or strip it.

That last one is worth emphasizing: **the output is an exfiltration channel**, and it is the one people forget because nothing about it looks like a tool call.

**Streaming complicates this.** You cannot fully validate a response you have already started sending (Lesson 10.1). Options: filter per chunk with a small lookahead buffer, or hold the first N tokens until a fast check passes. Neither is perfect; pick deliberately and document the gap.

---

## Key Concept 4: Cache scoping — the cross-tenant bug

A specific, common, serious bug.

A semantic cache keyed on the query embedding alone will serve user A's cached answer to user B when their queries are similar. If those users are different tenants, you have leaked one customer's data to another — silently, with no error, and with citations that make it look authoritative.

```python
# WRONG
key = embedding(query)

# RIGHT
key = (tenant_id, user_permission_scope, corpus_version, embedding(query))
```

The key must include **everything that affects what the answer should be**: tenant, permission scope, any filter applied, and the corpus version (so a re-index invalidates the cache rather than serving stale answers).

Test for it explicitly. Two tenants, similar queries, distinct data, and assert no crossover. This belongs in your regression suite permanently, because it is exactly the kind of thing a well-meaning refactor reintroduces.

---

## Key Concept 5: Retention, access, and disclosure

**Retention.** Set it per store and implement expiry — do not rely on intent.

| Store | Reasonable default |
|---|---|
| Traces | 7–30 days (sample longer if needed) |
| Checkpoints (completed) | 24–72 hours |
| Checkpoints (awaiting approval) | Long enough for a human, plus a timeout |
| Semantic cache | Hours to days, matched to corpus change rate |
| Telemetry | Aggregate indefinitely, raw content briefly |
| Fine-tuning datasets | Documented, with provenance and consent basis |

**Sampling reduces exposure.** Capturing 10% of traces answers most operational questions at a tenth of the risk.

**Access control.** The trace store is a production data store. Restrict it accordingly. "Everyone on the team can read all traces" is a common and poor default.

**Deletion requests.** If a user asks for their data to be deleted, you now need to reach traces, checkpoints, caches, memory, and telemetry. Know how you would do that. And note the honest limit: data already baked into a fine-tuned model cannot be deleted from weights, which is a reason to be careful about what enters a training set.

**User-facing honesty.** Disclose AI involvement, show citations so answers are verifiable, and make refusal visible rather than dressing it up as an answer.

---

## Worked Example 1: Auditing the stores

A team greps their own stores for PII patterns:

| Store | Emails found | API keys | Records scanned |
|---|---|---|---|
| Trace store | 4,182 | **31** | 42,000 |
| Checkpoints | 890 | 6 | 11,400 |
| Telemetry JSONL | 3,940 | 28 | 42,000 |
| Fine-tuning dataset | **112** | **2** | 1,032 |

Two findings stand out.

**Thirty-one API keys in the trace store.** Users pasting credentials into support questions — "my key sk-... isn't working." Redaction was applied to the model call and not to the trace capture, so the keys were stripped before reaching the provider and stored in full in the trace database.

**Two API keys in the fine-tuning dataset.** Those would have gone into model weights. The Lesson 8.2 filters removed failures and duplicates; they did not scan for secrets.

Fix: redact at capture for every store, not only on the provider path, and add a secret scan to the dataset pipeline. Re-scan after: zero.

## Worked Example 2: The cache leak, demonstrated

Two tenants, similar questions:

```
Tenant A: "What's our contracted rate limit?"
  → retrieves A's contract → "Your plan allows 500 requests/minute."
  → cached, key = embedding(query)

Tenant B: "What is our contracted rate limit?"
  → similarity 0.987 > threshold 0.97
  → CACHE HIT
  → "Your plan allows 500 requests/minute."   ← A's contract terms
```

Tenant B receives tenant A's contract data, confidently, with a citation to a document B cannot access.

No error. No log line. Nothing in the trace distinguishes it from a correct response — the cache hit *is* the response.

Fix: include tenant in the cache key. One line. Then the regression test that keeps it fixed:

```python
def test_cache_does_not_cross_tenants():
    ask(tenant="A", q="What's our contracted rate limit?")
    r = ask(tenant="B", q="What is our contracted rate limit?")
    assert "500" not in r.answer
    assert all(c.tenant == "B" for c in r.citations)
```

---

## Practice

**1. Write the data inventory** — `docs/data-inventory.md`. Every store from Key Concept 1: contents, location, access, retention, and sensitivity classification.

**2. Audit your own stores.** Scan traces, checkpoints, telemetry, caches, and your fine-tuning dataset for emails, phone numbers, keys, and connection strings. **Report what you find**, including zero.

**3. Add redaction at capture** for every store, not only on the provider path. Verify by re-scanning.

**4. Scan your fine-tuning dataset for secrets** specifically. What goes into weights does not come out.

**5. Scope every cache key** by tenant, permission scope, applied filters, and corpus version. **Write the cross-tenant regression test** and keep it permanently.

**6. Add output filtering** — PII, secrets, and model-composed URLs validated against retrieved sources. Handle the streaming case deliberately and document the gap.

**7. Implement retention.** Actual expiry jobs for traces, checkpoints, and caches. Verify old records are gone.

**8. Restrict access** to the trace store. Note who has it and why.

**9. Write the deletion runbook** — for a user deletion request, which stores are touched and how. Note honestly what cannot be deleted.

---

## Visual Aid

*A data-flow diagram of the full system with every persistence point drawn as a cylinder: trace store, checkpoint store, semantic cache, embedding cache, telemetry, fine-tuning dataset, long-term memory, and an external "provider" cloud. Each cylinder is tagged with a coloured badge showing sensitivity (high/medium/low) and its retention period. Red flags mark the trace store ("31 API keys found") and the fine-tuning dataset ("permanent — cannot be deleted from weights"). A redaction filter icon sits on the arrow into each store, with one arrow — into the trace store — drawn without a filter and circled in red, labeled "redaction applied to the provider path only."*

---

## Key Takeaways

- **You have built seven or more stores of user content.** The trace store and the fine-tuning dataset are the ones teams miss, and they are the two highest-risk.
- **Redact at capture for every store**, not only on the path to the provider — otherwise you strip secrets before sending them out and keep them in full at home.
- **The output is an exfiltration channel.** Validate model-composed URLs against retrieved sources; no tool control catches this.
- **Cache keys must include tenant, permission scope, filters, and corpus version.** An embedding-only key serves one customer's data to another, silently and with citations.
- **Set and implement retention, restrict access to traces, and know your deletion path** — including the honest limit that weights cannot be un-trained.

---

## Next

You have defended against the attacks you thought of. The attacks that matter are usually the ones you did not.

**Lesson 11.4** makes that systematic: enumerate from your threat model rather than improvising, automate the volume with an attacker model, and — the part that makes it professional rather than performative — turn every successful attack into a permanent regression test.
