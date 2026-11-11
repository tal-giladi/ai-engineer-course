---
id: "01.3"
module: 1
minutes: 12
practice_minutes: 60
prerequisites: ["01.2"]
objectives:
  - "Predict and measure the token cost and latency of a call, and explain how context length, sampling parameters, and streaming affect each."
volatility: implementation
sources: []
last_verified: "2026-08-24"
---

# 01.3 · The Physics of an LLM Call: Tokens, Context, Sampling, Streaming

Every design decision you make in the next eleven modules — how many documents to retrieve, whether to add a reranker, how long to let an agent run, whether to fine-tune a smaller model — is a trade against three quantities: tokens, latency, and money. Those three are governed by a handful of mechanical facts about how a model actually processes a request.

You do not need to know how a transformer works. You do need to know these facts cold, because engineers who skip them consistently optimize the wrong thing. This lesson is short and dense on purpose. Treat it as reference material you will come back to.

---

## Key Concept 1: Tokens are not words, and not the same across models

A model does not read characters or words. It reads **tokens** — subword units produced by a tokenizer. Roughly, in English prose, one token is about four characters or three-quarters of a word. But "roughly" hides things that matter:

- **Code tokenizes badly.** Punctuation, indentation, and identifiers fragment. The same information costs noticeably more tokens as code than as prose.
- **Non-English text tokenizes worse.** Languages under-represented in the tokenizer's training can cost several times more tokens for the same content.
- **Rare identifiers explode.** A UUID or a hash can consume dozens of tokens.
- **Different model families use different tokenizers.** The exact same string is a different token count — and therefore a different price and a different fraction of the context window — on two different backends. This is the concrete reason the identical prompt reported different usage numbers in Lesson 1.2.

**Practical rule:** never estimate cost from character count across model families. Count tokens with the tokenizer for the model you are actually calling, or read `usage` from the response.

---

## Key Concept 2: Prefill and decode — the asymmetry that governs everything

This is the single most useful mechanical fact in the lesson.

A model call has two phases:

**Prefill** processes your input. Every input token is processed **in parallel** — the whole prompt goes through in essentially one pass. Prefill is fast and scales well. Doubling your input roughly doubles prefill work, but that work is parallel and cheap per token.

**Decode** produces your output. Tokens are generated **one at a time**, each one requiring a full forward pass through the model that depends on the previous token. Decode is inherently sequential and cannot be parallelized within a single response.

The consequences follow directly, and they are not intuitive:

1. **Output tokens dominate latency.** A 5,000-token prompt returning 50 tokens is usually *faster* than a 500-token prompt returning 500 tokens.
2. **Output tokens are priced higher than input** — typically three to five times — and that pricing reflects the underlying cost structure rather than being arbitrary.
3. **The cheapest latency optimization is asking for less output.** "Answer in two sentences" is a performance change, not a style preference.
4. **Long context is affordable; long answers are not.** This shapes every retrieval decision in Modules 4 and 5.

> **Remember this:** input is wide and parallel, output is narrow and serial. Optimize the serial part first.

---

## Key Concept 3: The context window is a shared budget

The context window is the hard limit on input plus output tokens for a single call. It is shared by everything you send:

```
system prompt + conversation history + retrieved context
  + tool definitions + tool results + the space reserved for output
```

Three things engineers underestimate:

**It fills faster than you expect in agent systems.** In Module 6 you will build an agent whose conversation grows with every tool result. A tool returning a raw API response can consume a large fraction of the window in one call.

**Quality degrades well before the limit.** Models attend less reliably to information buried in the middle of a long context — the "lost in the middle" effect. A document at position 40 of 60 may as well not be there. This is why Module 5 spends a full lesson on context *assembly*, and why "just retrieve more chunks" is usually wrong.

**Output needs reserved space.** If the window is 128k and your input is 127k, you have room for almost no answer. Budget output space explicitly.

---

## Key Concept 4: Sampling parameters, and why `temperature=0` isn't determinism

**Temperature** scales the probability distribution before a token is sampled. Low values concentrate probability on the most likely tokens; high values flatten the distribution and admit unlikely ones. **Top-p** (nucleus sampling) instead restricts sampling to the smallest set of tokens whose cumulative probability exceeds *p*.

Practical guidance: use low temperature (0–0.3) for extraction, classification, and anything feeding structured output; moderate (0.7-ish) for open generation. Adjust one of temperature or top-p, not both — they interact confusingly.

Now the important part, because it burns people: **`temperature=0` is not determinism.** It selects the highest-probability token at each step, which sounds deterministic, but identical inputs can still produce different outputs because of:

- Floating-point non-associativity in GPU kernels, where results vary with batch composition
- **Continuous batching** on the serving side (Module 10) — your request is processed alongside whichever other requests happened to arrive, which changes the arithmetic
- Model version updates behind a stable alias
- Mixture-of-experts routing in some architectures

The engineering consequence is significant: **you cannot write an exact-match test against a model's output and expect it to pass reliably.** This is precisely why Module 3 builds an evaluation harness with tolerance bands instead of assertions, and it is worth internalizing now rather than discovering it in CI.

---

## Key Concept 5: Streaming changes perception, not cost

With streaming, tokens are sent as they are generated. The metric that changes is **time to first token (TTFT)** — how long before the user sees anything.

- **TTFT** is roughly prefill time plus network overhead. It grows with input size.
- **Total latency** is TTFT plus (output tokens × time per output token).
- **Cost is identical.** You pay for the same tokens either way.

Streaming is a user-experience optimization, and a large one: a response with a 400 ms TTFT and a 6-second total feels far better than a 6-second wall of silence. It also has a real engineering consequence you will meet in Module 10 — a streaming response cannot be fully validated before the first byte goes out, which interacts awkwardly with the output-validation patterns from Module 2 and the safety filtering in Module 11.

---

## Worked Example: One prompt, four context sizes

The same question — "What does this function do?" — asked with progressively more surrounding code:

| Input tokens | Output tokens | TTFT | Total latency | Cost |
|---|---|---|---|---|
| 120 | 85 | 0.31 s | 2.4 s | $0.0012 |
| 1,200 | 88 | 0.44 s | 2.6 s | $0.0028 |
| 12,000 | 91 | 1.20 s | 3.5 s | $0.0195 |
| 60,000 | 89 | 4.10 s | 6.4 s | $0.0912 |

Read the columns carefully, because this table contains the whole lesson:

- **Output stayed flat (~88 tokens) while input grew 500×.** Total latency grew only 2.7×, because most of the added work was parallel prefill.
- **TTFT grew 13×.** All of that growth is prefill. This is what your user feels first.
- **Cost grew 76×** — driven almost entirely by input tokens, which is why retrieving 60 chunks when 5 would do is a cost decision, not a quality decision.

Now the determinism check — same prompt, `temperature=0`, five runs:

```
Run 1: "This function validates the input dictionary and returns..."
Run 2: "This function validates the input dictionary and returns..."
Run 3: "This function validates the input dictionary and returns..."
Run 4: "The function validates an input dictionary, returning..."     ← differs
Run 5: "This function validates the input dictionary and returns..."
```

Four out of five identical, one different. That is normal. Build for it.

---

## Practice

**1. Instrument your client.** Extend the Lesson 1.2 `LLMClient` so every call returns, alongside the text: `tokens_in`, `tokens_out`, `ttft_ms`, `total_ms`, and `cost_usd`. Take token counts from the response `usage` field; compute cost from a per-model price table in config, not hardcoded.

**2. Reproduce the table.** Run one prompt at four context sizes (roughly 100 / 1k / 10k / 50k input tokens) on at least two of your backends. Produce the same five-column table. Confirm the asymmetry holds on your own numbers.

**3. Test determinism.** Run one prompt five times at `temperature=0` on each backend. Record how many outputs were byte-identical. Write down one sentence on what this means for testing — you will need that conclusion in Module 3.

**4. Measure the output lever.** Take one prompt and add "Answer in one sentence." Compare total latency and cost against the unconstrained version. This is the cheapest optimization in the course and you should feel its size.

---

## Visual Aid

*Timeline diagram of a single streamed call, drawn left to right. A short "network" segment, then a wide shaded block labeled "PREFILL — all input tokens processed in parallel," then a long series of small evenly-spaced ticks labeled "DECODE — one token at a time." A vertical marker at the end of prefill is labeled "Time To First Token"; a marker at the final tick is labeled "Total latency." Beneath, a second identical timeline with a 10× longer prefill block but the same number of decode ticks, showing TTFT moving right substantially while total latency moves only a little. Caption: "Growing the input moves TTFT. Growing the output moves everything."*

---

## Key Takeaways

- **Tokens are model-specific.** The same string costs different amounts on different backends. Read `usage`; never estimate from characters across families.
- **Prefill is parallel, decode is serial.** Output tokens dominate latency and are priced several times higher than input. Shortening answers is the cheapest optimization available.
- **The context window is a shared budget** across system prompt, history, retrieved context, tool results, and reserved output space — and quality drops before the limit does.
- **`temperature=0` is not determinism.** Batching, floating-point ordering, and model updates all introduce variation. Evaluation needs tolerance bands, not exact assertions.
- **Streaming improves TTFT, not cost,** and complicates validating output before it is sent.

---

## Next

You now have real numbers for a single call. Production is millions of calls, and nobody inspects them one at a time. Worse, the cost and latency you will care about are almost never per-*call* — one user question in the system you are about to build will trigger a dozen model calls, and the per-call numbers will look fine while the per-request bill is alarming.

**Lesson 1.4** builds the telemetry layer that makes that visible, including the one field that will still be paying dividends in Module 10.
