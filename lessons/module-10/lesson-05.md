# Lesson 10.5 — Packaging: Docker Compose, Configuration, CI

**Module 10: Serving, Deployment & Performance** · ~11 min
**Objective:** M10-LO5 — Package the system as reproducible Docker Compose with externalized config and a CI pipeline.

---

## Introduction

This is the most familiar ground in the course for you, so it is short and focused on what is AI-specific.

Three things differ from packaging an ordinary service: model containers are unusual (huge images, GPU passthrough, weights as volumes, long warm-ups), the configuration that changes most often is not the configuration you are used to externalizing, and CI has to run tests that are non-deterministic and cost money.

There is also a reason to care beyond correctness. In Module 12 a hiring reviewer will spend roughly four minutes on your repository. If `docker compose up` does not work from a clean clone, they will not spend a fifth. **Reproducibility is a hiring signal**, and this lesson is where you earn it.

---

## Key Concept 1: The topology you have accumulated

Across ten modules, the stack has grown to:

```yaml
services:
  api:          # FastAPI (10.1)
  postgres:     # pgvector + BM25 (4.3, 4.4) + checkpoints (7.2)
  qdrant:       # optional alternative store (4.3)
  vllm:         # fine-tuned model serving (9.4, 10.2)
  redis:        # semantic + embedding cache (10.3)
  otel:         # tracing collector (10.4)
  langfuse:     # LLM trace UI (10.4)
  mcp-docs:     # your MCP server (7.4)
```

Two things that ordinary services do not need:

**Health checks with realistic timings.** vLLM takes 60–180 seconds to load weights and warm up. A default 30-second health check start period will mark it unhealthy and restart it, forever. Set `start_period` generously.

```yaml
vllm:
  healthcheck:
    test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
    interval: 10s
    timeout: 5s
    retries: 5
    start_period: 180s        # weights load slowly
```

**Dependency ordering that waits for readiness,** not just for start:

```yaml
api:
  depends_on:
    postgres: {condition: service_healthy}
    vllm:     {condition: service_healthy}
```

---

## Key Concept 2: Model containers are different

**Never bake weights into the image.** A model image is 15–30 GB, rebuilds on every code change, and cannot be cached usefully. Mount weights as a volume:

```yaml
volumes:
  - ~/.cache/huggingface:/root/.cache/huggingface
  - ./adapters:/adapters:ro
```

**GPU passthrough** needs the NVIDIA container runtime configured on the host — document this as a prerequisite in your README, because it is the step that stops people.

**Provide a CPU-only path.** Not every reviewer has a GPU. A compose profile that swaps vLLM for a hosted profile means your repository runs for everyone:

```bash
docker compose --profile hosted up      # no GPU required
docker compose --profile local up       # vLLM + GPU
```

This one decision materially increases how many people can actually run your project, which matters in Module 12.

---

## Key Concept 3: What counts as configuration

The twelve-factor instinct — config in the environment — is right, but the *set* of things that qualify is different here. These change most often and must be changeable without a rebuild:

```bash
# Models and routing
LLM_PROFILE=local-tuned
LLM_FALLBACK_PROFILE=frontier
ROUTER_ESCALATION_THRESHOLD=0.72        # Lesson 9.3

# Prompts
PROMPT_VERSION_ANSWER=v3                # Lesson 2.4
PROMPT_VERSION_JUDGE=v2

# Retrieval
RETRIEVE_K=50
RERANK_K=5                              # Lesson 5.3
SEMANTIC_CACHE_THRESHOLD=0.97           # Lesson 10.3

# Budgets and SLO
AGENT_MAX_STEPS=15
AGENT_MAX_COST_USD=0.25                 # Lesson 6.4
SLO_P95_MS=3000

# Sampling
ONLINE_EVAL_SAMPLE_RATE=0.10            # Lesson 10.4
```

Every one of these is something you will want to change during an incident, and every one is something a rollback needs to be able to revert. `RERANK_K` and `SEMANTIC_CACHE_THRESHOLD` are as operationally important as any timeout in a conventional service.

**Secrets never go in the image or the repo.** Provider keys via environment or a secret manager, `.env` gitignored, `.env.example` committed with every variable documented and placeholder values.

**Validate config at startup** with Pydantic Settings. A typo in `RERANK_K` should fail the container immediately, not silently fall back to a default and quietly change your retrieval behavior.

---

## Key Concept 4: CI, with the AI-specific problem

Your pipeline assembles what you already built:

```yaml
on: [pull_request]
jobs:
  fast:
    steps:
      - ruff check . && mypy .
      - pytest tests/unit                       # mocked model calls
      - make eval-smoke                         # 15 items, ~90s, ~$0.04
      - make eval-gate --tier=smoke             # Lesson 3.5

  full:
    if: push to main
    steps:
      - pytest tests/integration                # compose up, real services
      - make eval-full                          # 60 items, ~4 min, ~$0.90
      - make eval-gate --tier=full
      - make cost-gate                          # cost regression check
      - docker build && push
```

**The AI-specific problem:** eval tests are non-deterministic and cost money. Two consequences you have already solved:

- **The two-tier design from Lesson 3.5** keeps per-PR cost at cents while full runs happen on merge.
- **Gate on bands, not exact values** (Lesson 3.5), or the pipeline flaps and someone disables it.

**Gate cost alongside quality.** A change that improves quality two points and doubles cost per request should require a human decision, not sail through green:

```yaml
cost_gate:
  max_increase_pct: 25
  baseline: eval-report.md
```

**Cache aggressively in CI** — embeddings, retrieval results for an unchanged corpus, judge verdicts keyed on (answer, rubric version). This is often the difference between a $0.90 and a $4 full run.

**Mock in unit tests.** Unit tests should not call models. Record fixtures from real responses and replay them.

---

## Key Concept 5: The clean-clone test

The single most valuable thing in this lesson, and it takes fifteen minutes.

```bash
cd $(mktemp -d)
git clone <your-repo> && cd <repo>
# now follow your own README, exactly, changing nothing
```

You will find two or three assumptions every time: a tool you installed months ago, an environment variable set in your shell profile, a model already in your Hugging Face cache, a database migration you ran by hand.

Every one of those is a step a reviewer hits and stops at.

**The target:** clone → `cp .env.example .env` → add one API key → `docker compose --profile hosted up` → working system with a seeded demo corpus. Under five minutes, one manual step.

Include a small seeded corpus in the repository so the system does something on first run. A service that starts with an empty index and nothing to query looks broken.

---

## Worked Example: The gate that paid for itself

A pull request tightens the answer prompt and adds a caching layer.

**Fast tier — passes.** Lint clean, unit tests green, 15-item smoke eval within band. Reviewer approves.

**Merge to main. Full tier:**

```
EVAL GATE: PASSED
  Overall: 0.871 → 0.869  (−0.002, within band)

COST GATE: FAILED
  Cost per request: $0.0026 → $0.0071  (+173%, limit +25%)

  By call_site:
    rag.answer      $0.0011 → $0.0011    —
    agent.step      $0.0009 → $0.0009    —
    validate        $0.0006 → $0.0051    ← +750%

  Changed since baseline: service/cache.py, prompts/answer.md v3→v4
```

Quality was fine. Cost tripled.

The cause: the new cache layer was keyed on the rewritten query, but citation validation ran *before* the cache check, so every request paid for validation even on a cache hit — and the sampling from Lesson 10.3 had been accidentally bypassed by the new code path.

**A quality-only gate would have shipped this**, and it would have surfaced as a bill at the end of the month with no obvious cause. The `call_site` breakdown — a field you added in Lesson 1.4 — located it in about ten seconds.

---

## Practice

**1. Write the full `docker-compose.yml`** with every service, health checks with realistic `start_period` values, and readiness-based dependency ordering.

**2. Add compose profiles** — `hosted` (no GPU) and `local` (vLLM + GPU). Verify both start.

**3. Externalize configuration.** Every model choice, prompt version, retrieval parameter, budget, threshold, and sampling rate. Validate at startup with Pydantic Settings so a typo fails fast.

**4. Commit `.env.example`** with every variable documented and placeholder values. Confirm `.env` is gitignored and no secret is in the repo history.

**5. Seed a demo corpus** so the system works immediately on first run.

**6. Build the CI pipeline** — fast tier on PR, full tier on merge, with the quality gate from Lesson 3.5 and a cost gate. Cache embeddings and judge verdicts.

**7. Verify the cost gate catches something.** Deliberately introduce a cost regression — remove sampling from an expensive call — and confirm the gate fails and names the `call_site`.

**8. Run the clean-clone test.** Fresh directory, follow your own README literally. Fix every step that fails. **Then do it once more**, because the first pass usually reveals fixes that introduce new assumptions.

**9. Time it.** From `git clone` to a working query. If it is over five minutes or more than one manual step, keep going.

---

## Visual Aid

*A service topology diagram: boxes for api, postgres, qdrant, vllm, redis, otel, langfuse, mcp-docs, with dependency arrows and exposed ports labeled. Volume mounts shown as cylinders attached to postgres (data), vllm (weights and adapters, with a note "15 GB — mounted, never baked"), and redis. The vllm box carries a clock icon annotated "start_period: 180s — weights load slowly." Two dashed rectangles overlay subsets of the diagram, labeled "profile: hosted (no GPU)" excluding vllm, and "profile: local" including it. Along the bottom, a CI strip: PR → [lint · unit · smoke eval · quality gate] and merge → [integration · full eval · quality gate · **cost gate** · build], with the cost gate highlighted.*

---

## Key Takeaways

- **Model containers are different:** mount weights rather than baking them, configure GPU passthrough as a documented prerequisite, and set health-check `start_period` for a 60–180 second warm-up.
- **Provide a CPU-only profile.** It determines how many people can actually run your project, which matters directly in Module 12.
- **The config that matters here is model choice, prompt version, retrieval-k, thresholds, budgets, and sampling rates** — all changeable without a rebuild, all validated at startup.
- **CI gates on quality *and* cost.** A quality-only gate ships cost regressions that surface as an unexplained bill; the `call_site` breakdown from Lesson 1.4 locates them instantly.
- **Run the clean-clone test, twice.** Target: clone, one key, one command, under five minutes — and it is a hiring signal, not housekeeping.

---

## Next

Your system is built, measured, optimized, observable, and reproducible. Everything so far has assumed the people using it are ordinary users making ordinary requests.

They will not all be. And more importantly, the *documents your system retrieves* are not all under your control — an agent that reads untrusted content and holds tools that can act is an attack surface with a budget attached.

**Module 11** covers prompt injection and why it is not solved, tool permissions that assume the model is already compromised, PII across the many stores you have now accumulated, systematic red-teaming of your own agent, and the reliability layer that keeps everything inside a cost envelope when providers fail.
