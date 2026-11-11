---
id: "09.4"
module: 9
minutes: 13
practice_minutes: 90
prerequisites: ["09.3"]
objectives:
  - "Ship the result: merge or serve adapters, quantize, and account for the quality cost of quantization."
volatility: implementation
sources: []
last_verified: "2026-08-24"
---

# 09.4 · Shipping Adapters: Merging, Quantizing, Serving

A trained adapter in a Colab notebook is not a deployed model. Between the two sit four decisions — merge or mount, quantize or not, how to version, and what happens when the base model is deprecated — and each one has a wrong answer that costs you later.

This is also where the Lesson 8.1 total-cost-of-ownership argument stops being an estimate and becomes a set of concrete artifacts you now maintain. The versioning discipline in Key Concept 4 is the part that looks like paperwork and is actually the difference between being able to roll back on a Friday and not.

---

## Key Concept 1: Merge or mount

A LoRA adapter can be **merged** into the base weights, producing one self-contained model, or kept **separate** and applied at load time.

**Merged:**
- One artifact, simplest to serve and reason about
- Marginally faster inference — no adapter computation
- Full model size on disk (gigabytes, not megabytes)
- One model per fine-tune in memory

**Separate (mounted):**
- Adapter stays tens of megabytes
- **One base model in memory serves many adapters** — vLLM supports multi-LoRA serving with per-request adapter selection
- Swap or roll back an adapter without redeploying the base
- Slight per-request overhead

**The rule:** merge if you serve exactly one fine-tune and want operational simplicity. Keep separate if you serve several, if you iterate frequently, or if you want rollback to be a config change rather than a redeploy.

For most teams shipping their first fine-tune, and certainly for this course, **keep it separate**. The operational flexibility outweighs a small latency difference, and multi-LoRA serving is the feature that makes the routing pattern from Lesson 9.3 cheap.

---

## Key Concept 2: Quantization, and measuring what it costs

Quantization reduces weight precision for inference — 16-bit to 8-bit or 4-bit. Memory drops proportionally, throughput usually improves, and quality drops by an amount that is **task-dependent and must be measured**.

That last point is the whole concept. The internet is full of confident claims that 4-bit quantization is free. On some tasks it nearly is. On others it costs several points.

The pattern is reasonably consistent:

- **Narrow classification and extraction** — often near-zero cost. These tasks have wide margins.
- **Structured generation** — small cost, occasionally a rise in format errors.
- **Multi-step reasoning** — the largest cost. Errors compound across steps.
- **Long-context tasks** — noticeable degradation, especially at 4-bit.

Which means your fine-tuned extraction model may quantize almost free while your agent's reasoning model does not — and you cannot know which without running your own evaluation.

**Formats:** GPTQ and AWQ for GPU serving; GGUF for CPU and local. Note that inference quantization is a separate decision from the QLoRA *training* quantization in Lesson 8.3 — you can train in 4-bit and serve in 16-bit, or the reverse.

**Always evaluate the quantized artifact.** Not the unquantized one you trained. They are different models.

---

## Key Concept 3: Serving with vLLM

The payoff for the Lesson 1.2 seam. vLLM exposes an OpenAI-compatible server, so your fine-tuned model becomes a profile in the client you wrote in Module 1:

```bash
vllm serve <base-model> \
  --enable-lora \
  --lora-modules extract=/adapters/docs-extract-v1.2 \
                 scope=/adapters/scope-router-v1.0 \
  --max-lora-rank 16 \
  --quantization awq
```

Request an adapter by name in the `model` field, exactly like any other model:

```python
client = LLMClient("local-tuned")     # base_url → vLLM
result = client.complete(messages, model="extract")
```

**Nothing above the client changes.** Not your RAG pipeline, not your agent loop, not your eval harness. That is the entire argument for the seam, arriving eight modules later.

Module 10 covers vLLM's performance characteristics — continuous batching, KV-cache capacity, and the concurrency implications.

---

## Key Concept 4: Versioning and provenance

A fine-tuned model is a deployed artifact with a lifecycle. Treat it like one.

**Every adapter needs a version record:**

```yaml
adapter: docs-extract
version: 1.2.0
base_model: <model-id>
base_commit: abc123f              # exact, not a floating tag
training:
  method: QLoRA
  rank: 16
  epochs: 3
  learning_rate: 2e-4
data:
  datasheet: spine/finetune/datasheet.md
  examples: 800
  window: 2026-05-01..2026-07-30
  leakage_check: passed (17 removed)
evaluation:
  report: eval-report.md#module-8
  holdout_accuracy: 0.94
  known_regressions: [refusal −0.45, mitigated by scope router]
serving:
  quantization: awq
  quantized_eval: eval-report.md#quantization
rollback_to: 1.1.0
```

Why each field earns its place:

**`base_commit`, exactly.** A floating tag means you cannot reproduce the training run, and adapters are tied to specific base weights. A base model updating under you can break an adapter silently.

**Datasheet link.** Six months from now, "what was this trained on?" must have an answer.

**Known regressions.** Whoever deploys this needs to know the refusal behavior is mitigated elsewhere, or they will remove the mitigation.

**`rollback_to`.** Rollback should be a config change: point the serving config at the previous adapter. This is the concrete payoff of keeping adapters unmerged.

**Deployment discipline:** adapters are config, so canary them like config — route a small percentage of traffic, compare online quality (Module 10), then ramp.

---

## Key Concept 5: The deprecation event

Your base model will be deprecated. Plan for it now, because the plan takes ten minutes and its absence costs a scramble.

When it happens you re-run the whole chain: rebuild or reuse the dataset (this is why the datasheet exists), retrain on the new base, re-evaluate against a fresh baseline, re-quantize, re-benchmark, and canary.

Roughly two to five days of work if your datasheet, training script, and eval harness are in order. Considerably more if the dataset lives in someone's Colab drive and the hyperparameters are in a Slack message.

**What makes it cheap:** the dataset in version control, the training script in the repo and runnable end to end, the eval harness automated (Module 3), and provenance recorded. Everything in this course has been building toward that being true.

**What to write down:** which base model, where its deprecation notices are published, an estimate of the re-training effort, and who owns it. Half a page in `docs/model-lifecycle.md`.

---

## Worked Example: Three serving configurations

Fine-tuned 8B extraction adapter, evaluated on the untouched held-out set in each configuration.

| | Merged, bf16 | Merged, AWQ 4-bit | Unmerged + AWQ base |
|---|---|---|---|
| GPU memory | 16.4 GB | **5.8 GB** | 6.1 GB |
| p95 latency | 340 ms | **295 ms** | 318 ms |
| Throughput (req/s) | 42 | **61** | 57 |
| **Held-out accuracy** | **0.94** | **0.93** | **0.93** |
| Format adherence | 0.99 | 0.99 | 0.99 |
| Adapter swap | redeploy | redeploy | **config change** |
| Serve N adapters | N × 16 GB | N × 5.8 GB | **6.1 GB total** |

Quantization cost **one point of accuracy** and bought a 65% memory reduction plus 45% more throughput. For this task, clearly worth it — and note that it had to be measured, because on a reasoning task the same change might have cost five points.

**Chosen: unmerged plus quantized base.** It gives up 23 ms and 4 req/s against the merged-quantized option, and buys the two properties that matter operationally: multi-adapter serving from one base (the scope router from Lesson 8.4 and the extraction adapter share memory) and rollback as a config change.

**The reasoning-task counterexample**, run on the same hardware with the agent's model:

| | bf16 | AWQ 4-bit |
|---|---|---|
| Multi-step task success | 0.84 | **0.76** |
| Format adherence | 0.98 | 0.94 |

Eight points. The same quantization decision, a different task, a completely different answer. **Not quantized in production.**

That pair of results is the lesson: quantization is not a policy, it is a per-model measurement.

---

## Practice

**1. Serve your adapter with vLLM** in Docker, with `--enable-lora`. Reach it through your Module 1 client by **changing configuration only**. Verify nothing above the client changed.

**2. Evaluate three configurations** on the untouched held-out set: merged bf16, merged quantized, unmerged with quantized base. Report memory, p95 latency, throughput, and full quality metrics for each.

**3. Measure quantization cost on your own task.** Do not quote a figure from a blog post. If you have both an extraction-style and a reasoning-style workload, measure both — the difference is instructive.

**4. Choose a configuration and justify it**, including the operational properties, not only the numbers.

**5. Serve two adapters from one base** if you have two (the distilled router from Lesson 9.3 and your extraction adapter). Confirm per-request adapter selection works and note the memory saving.

**6. Write the version record** for every adapter, in the YAML shape above. Exact base commit, datasheet link, eval link, known regressions, rollback target.

**7. Test rollback.** Point serving at the previous adapter version and confirm it takes effect without a redeploy. Time it. If it takes more than a minute you have not finished.

**8. Write `docs/model-lifecycle.md`** — half a page: base model and where deprecations are announced, re-training effort estimate, what would need to change, and who owns it.

---

## Visual Aid

*Two panels. Left, a serving-configuration comparison table rendered as a small heatmap — rows are the three configurations, columns are memory, latency, throughput, accuracy, and operational flexibility — with the chosen row outlined. Right, a paired bar chart headed "quantization cost is task-dependent": two groups of bars, "extraction task" showing bf16 0.94 vs AWQ 0.93 (nearly identical, green), and "multi-step reasoning" showing bf16 0.84 vs AWQ 0.76 (a visible gap, red). Beneath, in bold: "Same quantization. Same hardware. Different answer. Measure yours."*

---

## Key Takeaways

- **Merge for a single fine-tune and operational simplicity; keep adapters separate for multi-adapter serving, frequent iteration, and rollback as a config change.** Separate is usually right for a first deployment.
- **Quantization's quality cost is task-dependent and must be measured on your own held-out set** — near-free for extraction, potentially several points for multi-step reasoning. Evaluate the quantized artifact, not the one you trained.
- **vLLM serves your adapter behind the same OpenAI-compatible interface** from Lesson 1.2, so nothing above your client changes. That is the seam paying off eight modules later.
- **Every adapter needs a version record** with the exact base commit, datasheet link, evaluation link, known regressions, and a rollback target — and rollback should take under a minute.
- **Plan for base-model deprecation now.** With the dataset, script, and eval harness in order it is days; without them it is a scramble.

---

## Next

You have models worth serving and a way to serve them. What you do not have is a system anyone else can call.

**Module 10** turns everything from Modules 1 through 9 into a service: a streaming FastAPI endpoint that handles concurrency and cancellation, vLLM behind it, caching and routing tuned to a stated SLO, end-to-end tracing that finally makes an agent run legible, and one command that brings the whole stack up from a clean clone. It is also where your existing engineering seniority converts most directly into hireability.
