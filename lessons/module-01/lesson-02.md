---
id: "01.2"
module: 1
minutes: 14
practice_minutes: 75
prerequisites: ["01.1"]
objectives:
  - "Configure a single OpenAI-compatible client that routes to a hosted frontier model, a Hugging Face endpoint, or a local Docker-served open-weights model without changing call sites."
volatility: implementation
sources:
  - title: "Ollama, OpenAI compatibility (endpoints, supported and unsupported fields)"
    url: https://docs.ollama.com/api/openai-compatibility
  - title: "Ollama, Anthropic compatibility (/v1/messages, unsupported features)"
    url: https://docs.ollama.com/api/anthropic-compatibility
  - title: "Ollama model library tags (qwen3.5, gemma4, ministral-3, granite4.2)"
    url: https://ollama.com/library
  - title: "mlx-lm HTTP server"
    url: https://github.com/ml-explore/mlx-lm/blob/main/mlx_lm/SERVER.md
  - title: "Artificial Analysis, \"Sub-32B open weights models now offer GPT-5 level intelligence\" (2026-04-13)"
    url: https://artificialanalysis.ai/articles/sub-32b-open-weights
  - title: "Qwen3.5-9B model card (vendor-reported benchmarks)"
    url: https://huggingface.co/Qwen/Qwen3.5-9B
last_verified: "2026-11-11"
---

# 01.2 · One Interface, Many Models: The OpenAI-Compatible Gateway

You are about to add a dependency to your architecture that is more volatile than any database, queue, or cloud service you have worked with. Model providers deprecate models on months of notice. Prices change. A model that was best-in-class for your task in March is mid-tier by September. Rate limits appear during your busiest hour. And unlike a database migration, switching models is something you may want to do *deliberately and often* — a cheap model for the easy 80% of requests, an expensive one for the rest.

If your application calls a vendor SDK directly from three hundred places, every one of those facts becomes a rewrite. This lesson builds the seam that prevents it, and — more importantly — teaches you exactly where that seam leaks, because an abstraction you trust too much is worse than none at all.

---

## Key Concept 1: One wire protocol won

The OpenAI chat-completions request shape became the de-facto standard, and essentially the whole ecosystem now speaks it:

- Hosted frontier providers, either natively or through a compatibility endpoint
- Hugging Face Inference Endpoints
- **vLLM** and **Ollama**, the two most common ways to serve open-weights models locally
- Gateways and routers like LiteLLM and OpenRouter, which normalize dozens of providers behind the one shape

This matters more than it sounds. It means the difference between calling a frontier model at $3 per million tokens and calling a 7B model running in Docker on your own machine can be a base URL and a model name. That is a genuinely unusual property in infrastructure, and building to exploit it is the single highest-leverage architectural decision in this course.

The core request is small: a list of messages with roles, a model name, and some sampling parameters. The core response is a message, a finish reason, and token usage. Everything else is optional.

---

## Key Concept 2: The client and the seam

You want one class in your codebase that the rest of your application talks to. Something close to this:

```python
class LLMClient:
    def __init__(self, profile: str):
        cfg = load_profile(profile)          # base_url, api_key, model, capabilities
        self._http = OpenAI(base_url=cfg.base_url, api_key=cfg.api_key)
        self._model = cfg.model
        self._caps = cfg.capabilities

    def complete(self, messages, *, tools=None, schema=None, **kw): ...
```

Three profiles in configuration, not in code:

```
# .env.frontier
LLM_BASE_URL=https://api.<provider>.com/v1
LLM_MODEL=<frontier-model-id>

# .env.hf
LLM_BASE_URL=https://<endpoint>.endpoints.huggingface.cloud/v1
LLM_MODEL=<served-model-name>

# .env.local
LLM_BASE_URL=http://localhost:8000/v1
LLM_MODEL=<open-weights-model>
```

Your application calls `client.complete(...)`. It never knows which backend answered. Switching is an environment change, and running an A/B across two backends is instantiating two clients.

The `capabilities` field is the part people leave out, and it is the part that makes this honest — which brings us to the leak.

---

## Key Concept 3: What is portable, and what is not

Treating all backends as identical is a bug you will ship at least once. Be explicit about the three tiers.

**Genuinely portable — assume these work everywhere:**
- The message list with `system`, `user`, and `assistant` roles
- `temperature`, `top_p`, `max_tokens`, stop sequences
- Streaming
- Basic tool/function calling (widely supported, with edge-case differences)
- Token usage in the response

**Portable in name, different in behavior:**
- **Tokenization.** Different model families tokenize differently, so the same text is a different number of tokens — and therefore a different price and a different fraction of the context window.
- **System prompt handling.** Some models weight it heavily, some barely distinguish it from a user turn.
- **Tool-calling reliability.** All support it; they are not equally good at it, and a prompt tuned for one may select tools poorly on another.
- **Stop sequences and finish reasons.** Semantics vary at the edges.

**Not portable — must live behind the seam:**
- **Prompt caching.** Mechanism, minimum cacheable length, and TTL all differ; some backends have none.
- **Reasoning / extended thinking parameters.** Provider-specific, and often silently ignored elsewhere.
- **Structured output enforcement.** Ranges from strict schema enforcement to "ask nicely and hope" (Module 2 covers this properly).
- **Safety filtering.** Hosted providers filter; a model you serve yourself does not, which is a real design consideration once you self-host in Module 10.
- **Context window size.** Varies by an order of magnitude or more.

The rule that follows: **a parameter your backend does not support must raise, not silently disappear.** The worst failure mode in this whole design is a request that quietly loses its caching or its schema enforcement and produces subtly worse output at higher cost, with nothing in the logs.

```python
def complete(self, messages, *, cache_prefix=False, **kw):
    if cache_prefix and not self._caps.prompt_caching:
        raise UnsupportedCapability(
            f"prompt_caching unavailable on profile '{self._profile}'"
        )
```

Three lines. They will save you a debugging session that would otherwise take a day.

---

## Key Concept 4: The local-first dev loop

The seam has a second payoff that you can use from today: **build and debug against a model on your own machine, then change only the base URL, key, and model name for production.** Every prompt iteration, parser bug, and retry-loop mistake you make in development is then free, offline, and private.

*Versions checked 2026-11-11.* Two local servers cover almost everyone:

- **Ollama 0.40.x** (any OS). Serves an OpenAI-compatible API at `http://localhost:11434/v1` — `/v1/chat/completions` and `/v1/responses` (stateless only) — and an Anthropic-compatible `/v1/messages` at `http://localhost:11434`. The API key is required by the SDKs but ignored by the server.
- **mlx-lm 0.32.x** on Apple Silicon. `mlx_lm.server --model <hf-repo>` serves an OpenAI-style `/v1/chat/completions` on port 8080. Its own docs say it is not for production; it is for exactly this loop.

```
# .env.dev — the only file that differs from production
LLM_BASE_URL=http://localhost:11434/v1
LLM_API_KEY=ollama            # required by the SDK, ignored by Ollama
LLM_MODEL=qwen3.5:4b
```

The current small open-weights models that run on a laptop (as of 2026-11-11): **Qwen3.5** 0.8B / 2B / 4B / 9B, **Gemma 4** E2B / E4B, **Ministral 3** 3B / 8B, and **IBM Granite 4.2** 3B / 8B. All are in the Ollama library (`qwen3.5:4b`, `gemma4:e4b`, `ministral-3:8b`, `granite4.2:8b`).

**Be honest about what they are.** On reasoning, instruction-following, and tool-use benchmarks, the best of these match frontier models from 2024 — but those numbers are mostly vendor-reported on the model cards. Independent measurement shows where they fall short: Artificial Analysis (2026-04-13) found that even the sub-32B open models that reach GPT-5-level intelligence scores lag badly on factual knowledge and hallucinate far more, because knowledge recall scales with parameter count. Under 10B the gap is wider. They are good enough to exercise your code paths; they are not a preview of your production quality.

**What local development does not tell you:**

- **Provider-specific behaviour.** Ollama's compatibility endpoints do not support `tool_choice` or Anthropic-style prompt caching, and accept some parameters without honouring them (on `/v1/messages`, the thinking `budget_tokens` is accepted but not enforced). Structured-output enforcement also differs from the hosted provider's strict mode. Your capability flags (Key Concept 3) should say so for the `dev` profile.
- **Latency.** A laptop's time-to-first-token and throughput say nothing about a hosted API under load. Measure latency on the production profile (Lesson 1.4).
- **Caching economics.** Cost per request on a hosted provider depends on cache hits you cannot observe locally (Lesson 2.4).
- **Quality.** A prompt tuned on a 4B model is not tuned for the production model, and a prompt that works on the production model may fail locally. **Evaluations that gate a change run on the production model** (Lesson 3.5); the local model is for development speed, not for evidence.

---

## Worked Example: The same call, three backends

```python
QUESTION = "Summarize the trade-offs of optimistic locking in two sentences."

for profile in ["frontier", "hf", "local"]:
    client = LLMClient(profile)
    result = client.complete([{"role": "user", "content": QUESTION}])
    print(profile, "→", result.text[:80])
```

Same call site, three answers of differing quality, at wildly differing costs. This is the property the rest of the course depends on: when Module 9 produces your own fine-tuned model and Module 10 serves it with vLLM, it drops into this client as a fourth profile and nothing above it changes.

**Now the leak, demonstrated.** Add a provider-specific parameter and run it against a backend that lacks it:

```python
client = LLMClient("local")
client.complete(messages, cache_prefix=True)
```

Without the capability check, most SDKs will accept the extra keyword, the server will ignore it, and you will get a correct-looking answer with none of the caching you think you enabled. Your cost model is now wrong and nothing anywhere reports an error. With the check, you get an immediate `UnsupportedCapability`. That is the whole argument for the seam in one example.

---

## Practice

**1. Build the client.** Implement `LLMClient` with profile-based configuration. Keep the public surface small — `complete()` is enough for now; Module 2 adds structured output and Module 6 adds tools.

**2. Wire three profiles.** A hosted frontier model, a Hugging Face endpoint, and a local model in Docker. For the local one, `vllm/vllm-openai` or Ollama both work — the whole point is that your code cannot tell the difference. If your machine cannot run a local model, use two hosted profiles with different model families and note the substitution; the lesson still lands.

**3. Write two tests.**
- **Portability test:** the same fixed prompt runs successfully against all three profiles and returns a non-empty response with token usage populated.
- **Capability test:** requesting an unsupported capability raises `UnsupportedCapability` rather than succeeding silently. This is the test that matters.

**4. Record what differed.** Note the token counts for the identical prompt across backends. They will not match, and understanding why is Lesson 1.3.

**5. Set up the local-first loop (15 minutes).** Install Ollama 0.40.x (or mlx-lm 0.32.x on a Mac), then:

```bash
ollama pull qwen3.5:4b
ollama serve            # skip if the Ollama app is already running
```

Add a `dev` profile pointing at `http://localhost:11434/v1` with model `qwen3.5:4b` and capability flags set to what Ollama actually supports. Run your portability test against `dev` and against your production profile, with no code change between them. **You know it worked when** the only diff between the two runs is the environment file, and your capability test raises `UnsupportedCapability` on `dev` for prompt caching. Then write one line in your README: *"Local model is for development; all gating evals run on the production model."*

---

## Visual Aid

*Architecture diagram. At the top, "Your application" as a wide box with several arrows down into a single narrow box labeled `LLMClient` — the seam, drawn as a horizontal bar. Below it, three arrows fan out to "Frontier API," "HF Endpoint," and "Local vLLM (Docker)." Along the seam bar, small labels list the non-portable features that must be checked there: prompt caching, structured output enforcement, reasoning params, safety filtering, context size. A caption reads: "Everything above the bar is portable. Everything listed on the bar is a lie you must check for."*

---

## Key Takeaways

- **The OpenAI-compatible shape is the ecosystem's common protocol.** Frontier APIs, Hugging Face, vLLM, and Ollama all speak it, so backend choice can be configuration.
- **Put one `LLMClient` between your application and every provider.** Profiles in environment config, not vendor SDKs scattered through your codebase.
- **Three tiers of portability:** genuinely portable (messages, sampling, streaming), portable-in-name (tokenization, system prompt weight, tool-calling quality), and not portable (caching, structured output enforcement, reasoning params, safety filtering).
- **Unsupported capabilities must raise, never no-op.** A silently dropped parameter produces a working system with a wrong cost model and no error to find.
- **This seam pays off repeatedly:** your fine-tuned model in Module 9, your vLLM server in Module 10, and your provider fallback chain in Module 11 all plug in here without touching application code.
- **Develop locally, gate on production.** Ollama or mlx-lm with a small open model makes the dev loop free and offline; it tells you nothing about provider-specific features, latency, caching, or production quality, so evals still run on the production model.

---

## Sources

Versions and endpoints checked 2026-11-11.

- Ollama, OpenAI compatibility (endpoints, supported and unsupported fields): https://docs.ollama.com/api/openai-compatibility — checked 2026-11-11 (Ollama v0.40.16, released 2026-11-09)
- Ollama, Anthropic compatibility (`/v1/messages`, unsupported features): https://docs.ollama.com/api/anthropic-compatibility — checked 2026-11-11
- Ollama model library tags (`qwen3.5`, `gemma4`, `ministral-3`, `granite4.2`): https://ollama.com/library — checked 2026-11-11
- mlx-lm HTTP server: https://github.com/ml-explore/mlx-lm/blob/main/mlx_lm/SERVER.md — checked 2026-11-11 (mlx-lm 0.32.2)
- Artificial Analysis, "Sub-32B open weights models now offer GPT-5 level intelligence" (2026-04-13): https://artificialanalysis.ai/articles/sub-32b-open-weights — checked 2026-11-11
- Qwen3.5-9B model card (vendor-reported benchmarks): https://huggingface.co/Qwen/Qwen3.5-9B — checked 2026-11-11

---

## Next

You can now call three different models with one line of code. Each one costs a different amount, takes a different amount of time, and consumes your context window at a different rate. If you cannot say how much or how long — with numbers — you cannot make a single meaningful architectural decision for the rest of this course.

**Lesson 1.3** covers the mechanical facts underneath every LLM call: what tokens actually cost you, why output is dramatically more expensive than input in wall-clock terms, and why `temperature=0` is not determinism.
