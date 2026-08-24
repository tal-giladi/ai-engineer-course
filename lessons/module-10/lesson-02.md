# Lesson 10.2 — Serving Open-Weights Models with vLLM

**Module 10: Serving, Deployment & Performance** · ~12 min
**Objective:** M10-LO2 — Serve an open-weights model with vLLM behind the same OpenAI-compatible interface used since Module 1.

---

## Introduction

In Lesson 1.2 you built a client with a seam, and the argument was that a hosted frontier model and a local open-weights model should be a config change apart. Nine modules later, this is where that claim gets tested with a model you trained yourself.

The lesson has two halves. First, why vLLM is dramatically faster than the obvious approach, which matters because the obvious approach is what makes people conclude self-hosting is impractical. Second, the capacity model — how many concurrent requests a GPU actually supports, why that number depends on context length, and how to work out whether self-hosting is cheaper for *your* volume rather than for a benchmark.

---

## Key Concept 1: Why naive serving is slow

Write a loop that loads a model with `transformers` and calls `generate()` per request, and you will get poor throughput. Two reasons.

**Static batching wastes the GPU.** Naive batching collects N requests, runs them together, and waits for all of them to finish before starting the next batch. Since responses have wildly different lengths, a batch finishing in 20 tokens waits on one generating 800. Most of the GPU sits idle.

**KV-cache fragmentation wastes memory.** During generation, the model caches key/value tensors for every token so far. Naive implementations pre-allocate the maximum possible sequence length per request. A request that generates 50 tokens holds an allocation sized for 4,096. Most of your GPU memory is reserved and unused, which directly caps concurrency.

---

## Key Concept 2: What vLLM does

**Continuous batching.** Requests join and leave the running batch at token boundaries. A finished request frees its slot immediately and a waiting request takes it. The GPU stays busy. This alone is worth several times the throughput of static batching on realistic traffic.

**PagedAttention.** KV-cache memory is managed in fixed-size pages, like virtual memory, allocated on demand as a sequence grows. Fragmentation largely disappears, so far more sequences fit in the same GPU, and concurrency rises accordingly.

Both are transparent — you get them by using vLLM rather than by configuring anything.

Two more features that matter here: an **OpenAI-compatible server** (which is why nothing above your client changes) and **multi-LoRA serving** (Lesson 9.4), letting one base model serve many adapters with per-request selection.

---

## Key Concept 3: Running it

```yaml
vllm:
  image: vllm/vllm-openai:latest
  command: >
    --model <base-model>
    --enable-lora
    --lora-modules extract=/adapters/docs-extract-v1.2
    --max-model-len 8192
    --gpu-memory-utilization 0.90
    --quantization awq
  volumes: ["./adapters:/adapters", "~/.cache/huggingface:/root/.cache/huggingface"]
  deploy:
    resources:
      reservations:
        devices: [{driver: nvidia, count: 1, capabilities: [gpu]}]
```

Then, in your application:

```bash
LLM_BASE_URL=http://vllm:8000/v1
LLM_MODEL=extract
```

That is the entire change. Your RAG pipeline, agent loop, and eval harness are untouched.

**The knobs that matter:**

- **`--gpu-memory-utilization`** (default ~0.90) — the fraction of GPU memory vLLM claims. Whatever is left after weights becomes KV cache, which *is* your concurrency. Raise it to increase capacity; leave headroom or you will OOM under load.
- **`--max-model-len`** — the maximum context. **Setting this higher than you need directly costs concurrency** (Key Concept 4). Set it to what you actually use.
- **`--tensor-parallel-size`** — shard across multiple GPUs for models that do not fit on one.
- **`--quantization`** — see Lesson 9.4; measure the quality cost on your task.
- **`--max-num-seqs`** — cap on concurrent sequences.

---

## Key Concept 4: The capacity model

This is the mental model most engineers new to self-hosting lack, and it explains behavior that otherwise looks arbitrary.

```
GPU memory = model weights + KV cache + overhead
KV cache per sequence ≈ context_length × per-token-cost
max concurrent sequences ≈ KV cache memory / KV cache per sequence
```

The consequence: **doubling your maximum context length halves your concurrency.**

That is a scheduling fact, not a tuning detail, and it has direct architectural implications:

- Trimming retrieved context (Lesson 5.3) does not just save money — it doubles your serving capacity
- A single long-context request can occupy the memory of many short ones
- Sizing a deployment means knowing your *actual* context distribution, not your maximum

When the KV cache fills, vLLM queues or preempts. Watch for the preemption metric in production — it means you are over capacity and latency is about to degrade non-linearly.

---

## Key Concept 5: Self-hosting economics

Self-hosting is not automatically cheaper. The honest comparison:

**Hosted API:** pay per token, zero idle cost, scales instantly, no operations.

**Self-hosted:** pay per GPU-hour whether or not you use it, fixed capacity, you own upgrades, drivers, monitoring, and on-call.

The break-even is a volume calculation. At low or spiky volume, a GPU idling most of the day is far more expensive than per-token pricing. At sustained high volume, the fixed cost amortizes and self-hosting wins by a large margin.

Do the arithmetic:

```
hosted_monthly  = requests × avg_tokens × price_per_token
selfhost_monthly = gpu_hourly × 730 × instances

# and check you can actually serve it:
required_rps = requests / (30 × 86400)
capacity_rps = measured throughput at your context length
```

That last check is essential. A cost calculation that assumes capacity you do not have is not a cost calculation. And add operational cost honestly — engineer time is not free, and self-hosting adds on-call surface.

---

## Worked Example 1: Throughput

Same 8B model, same hardware, same 200-request workload:

| | Naive `transformers` loop | vLLM |
|---|---|---|
| Throughput | 3.1 req/s | **41 req/s** |
| p95 latency (32 concurrent) | 14.2 s | **1.9 s** |
| GPU utilization | 22% | **89%** |
| Max concurrent (8k context) | 6 | **48** |

Thirteen times the throughput on identical hardware. This is why "self-hosting was too slow" is usually a serving-stack problem rather than a hardware problem.

## Worked Example 2: The context/concurrency trade

Same model, same GPU, varying `--max-model-len`:

| `--max-model-len` | Max concurrent | Throughput | p95 (16 concurrent) |
|---|---|---|---|
| 2,048 | 184 | 78 req/s | 0.9 s |
| 4,096 | 96 | 61 req/s | 1.2 s |
| 8,192 | 48 | 41 req/s | 1.9 s |
| 16,384 | 23 | 26 req/s | 3.4 s |
| 32,768 | 11 | 14 req/s | 7.1 s |

Almost perfectly inverse. The measured context distribution for this workload:

```
p50: 2,100 tokens   p95: 5,800 tokens   p99: 7,900 tokens   max: 12,400
```

Setting `--max-model-len 32768` "to be safe" would have cost **four times the capacity** to serve the 0.1% of requests above 12,400 tokens. The right answer is 8,192, with those rare long requests routed to a hosted model — which is one line of routing logic through the Lesson 1.2 seam.

## Worked Example 3: Break-even

Extraction workload, 800 input / 150 output tokens average.

| Monthly requests | Hosted API | Self-hosted (1 GPU) | Winner |
|---|---|---|---|
| 50,000 | $105 | $1,050 | **hosted** |
| 500,000 | $1,050 | $1,050 | break-even |
| 2,000,000 | $4,200 | $1,050 | **self-hosted (4×)** |
| 10,000,000 | $21,000 | $2,100 (2 GPUs) | **self-hosted (10×)** |

Break-even around 500k requests/month for this shape.

**And the capacity check at 2M/month:** 0.77 req/s required against 41 req/s measured capacity. Comfortable — the GPU is idle most of the time, which is why the *next* observation matters: at this volume the GPU is 98% idle and self-hosting still wins on price, because per-token pricing at that volume exceeds a full month of GPU rental.

Note also the spiky-traffic caveat: if those 2M requests arrive in two hours a day, you need capacity for the peak and pay for the trough. Model your actual arrival pattern, not the monthly total.

---

## Practice

**1. Serve your Module 9 adapter with vLLM** in Docker Compose, with `--enable-lora`. GPU passthrough configured, model weights mounted rather than baked into the image.

**2. Switch by configuration only.** Point your Module 1 client at vLLM by changing environment variables. **Run your Module 3 eval suite against it** and confirm results match your Lesson 9.4 measurements. Nothing above the client should change.

**3. Measure your context distribution** from telemetry — p50, p95, p99, max. You have been recording token counts since Lesson 1.4.

**4. Build the capacity table.** Sweep `--max-model-len` at four values, measuring max concurrent sequences, throughput, and p95 latency at fixed concurrency. Pick the value that covers your p99, not your maximum.

**5. Handle the tail.** Route requests exceeding your `--max-model-len` to a hosted profile. One branch through the Lesson 1.2 seam.

**6. Compare against naive serving** at least once — a simple `transformers` generate loop versus vLLM on the same hardware. Seeing the 10× gap yourself is worth the twenty minutes.

**7. Write the break-even analysis** in `docs/serving-economics.md`: your token shape, hosted cost curve, self-hosted cost including operational overhead, the break-even volume, your actual volume and arrival pattern, and a recommendation. **This is a strong system design interview artifact** — it is the kind of reasoning the round is testing for.

---

## Visual Aid

*A chart with `--max-model-len` on the x-axis (2k to 32k, log scale) and two y-axes: max concurrent sequences on the left (descending curve) and p95 latency on the right (ascending curve). Overlaid as a shaded vertical band is the workload's actual context distribution — a histogram peaking at ~2k with a thin tail to 12k — with p50, p95, and p99 marked. A vertical line at 8,192 is labeled "chosen: covers p99." A second vertical line at 32,768 is labeled "'to be safe' — 4× less capacity for 0.1% of requests," with an arrow showing the concurrency drop. Caption: "Context length is a capacity decision."*

---

## Key Takeaways

- **Continuous batching and PagedAttention are why self-hosting is viable** — an order of magnitude over a naive `transformers` loop on identical hardware. "Self-hosting was too slow" is usually a serving-stack problem.
- **vLLM is OpenAI-compatible,** so your fine-tuned model becomes a config change through the Lesson 1.2 seam, with your eval suite verifying the switch.
- **Capacity is KV-cache-bound, and doubling context halves concurrency.** Set `--max-model-len` from your measured p99, not your maximum, and route the tail elsewhere.
- **Trimming retrieved context does not just save money — it multiplies serving capacity.**
- **Break-even is a volume calculation**, and it must include a capacity check and your real arrival pattern. Low or spiky volume favors hosted; sustained high volume favors self-hosted by a wide margin.

---

## Next

Both backends are fast. Your system is still slow, because answering one question takes eleven model calls, and you do not currently know which of them is responsible for your p95.

**Lesson 10.3** decomposes the latency budget from real traces and then spends it deliberately — caching at four levels, batching, routing, and context trimming — with the eval suite run after every single change, because several of these optimizations quietly cost quality and one of them can return a confidently wrong cached answer.
