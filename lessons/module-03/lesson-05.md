---
id: "03.5"
module: 3
minutes: 12
practice_minutes: 90
prerequisites: ["03.4"]
objectives:
  - "Wire a regression suite into CI that blocks a prompt or model change on measured quality loss."
volatility: implementation
sources:
  - title: "Promptfoo command line (exit code 100, PROMPTFOO_PASS_RATE_THRESHOLD, -o)"
    url: https://www.promptfoo.dev/docs/usage/command-line/
  - title: "Promptfoo configuration reference (test metadata, threshold, assertion metric)"
    url: https://www.promptfoo.dev/docs/configuration/reference/
  - title: "Promptfoo Python provider and Python assertions"
    url: https://www.promptfoo.dev/docs/providers/python/
  - title: "Promptfoo CI/CD integration (caching, GitHub Action)"
    url: https://www.promptfoo.dev/docs/integrations/ci-cd/
  - title: "OpenAI, \"OpenAI to acquire Promptfoo\" (2026-03-09)"
    url: https://openai.com/index/openai-to-acquire-promptfoo/
  - title: "Braintrust"
    url: https://www.braintrust.dev/docs
  - title: "LangSmith evaluation"
    url: https://docs.langchain.com/langsmith/evaluation
last_verified: "2026-11-11"
---

# 03.5 · Regression Suites and CI Gating

Everything you have built in this module works only if someone runs it. Evaluation that depends on discipline is evaluation that stops happening in week five, during a rush, on the change that most needed checking.

This lesson makes it automatic. The mechanics are familiar — you have wired test suites into CI before. What is different is that this suite is non-deterministic, costs money per run, and has a measured resolution below which its results are noise. Those three properties change the design, and getting them wrong produces a gate that either flaps constantly and gets disabled, or costs so much it gets moved to a nightly job nobody reads.

---

## Key Concept 1: Two tiers, because evals cost money

An ordinary test suite is free to run. This one is not: 60 items × a generation call × two judge calls (position-swapped) is roughly 180 model calls, which is real money and real minutes on every push.

So split it:

**Smoke tier — every commit.**
- 12–15 items, at least one per stratum, chosen for coverage rather than difficulty
- Cheap model where possible for mechanical checks; skip the judge or judge a subset
- Target: under 90 seconds, under a few cents
- Catches: broken pipelines, malformed prompts, crashed retrieval, obvious regressions

**Full tier — on merge to main, and on demand.**
- The complete dev set, all metrics, judge included with position swapping
- Target: a few minutes, a dollar or so
- Catches: real quality regressions, per-stratum failures, cost drift

**Held-out tier — at milestones only.**
- Your 30% held-out set, run at module boundaries and before releases
- Never in automated CI, because a set that runs on every merge is a set you are tuning against (Lesson 3.2)

Cache aggressively. Embeddings, retrieval results for unchanged corpora, and judge verdicts for unchanged (answer, rubric-version) pairs are all cacheable and cut the bill substantially.

---

## Key Concept 2: Gate on a band, not a number

This is where naive implementations fail.

If you write `assert score >= 0.85`, your gate will flap. The system is non-deterministic (Lesson 1.3), the judge has measured disagreement (Lesson 3.4), and a 60-item set has limited resolution (Lesson 3.2). Runs will vary by a few points with no code change at all. A gate that fails randomly gets disabled within a fortnight, and then you have no gate.

Gate on the **noise band you measured**:

```yaml
gate:
  baseline_source: eval-report.md   # the current committed baseline
  overall:
    max_drop: 0.04        # from judge trust boundary (Lesson 3.4)
  per_stratum:
    max_drop: 0.10        # smaller n, wider band
  hard_failures:
    - refusal_recall < 0.70     # non-negotiable floor, absolute
    - citation_validity < 0.95
  cost:
    max_increase_pct: 25
```

Four ideas in there worth naming:

**Compare to the committed baseline**, not to a fixed number. The bar rises as the system improves.

**Wider bands per stratum**, because a 6-item stratum has much worse resolution than the 60-item whole. Do not apply the same threshold to both.

**Hard floors are absolute, not relative.** Some properties must hold regardless of what the baseline says. Refusal behavior is the canonical one.

**Cost is gated.** A change that improves quality 2 points and triples cost should require a human decision, not sail through.

**Reduce variance where you can:** pin the model version explicitly rather than using a floating alias, pin the prompt version, use `temperature=0` for evaluation runs (accepting it is not full determinism), and run the judge with position swapping so its variance is averaged rather than sampled.

---

## Key Concept 3: The report must name the stratum

A gate that says `FAILED: score 0.81 < 0.85` is nearly useless. The engineer's next question is "what broke?" and answering it means running things locally and bisecting.

Make the CI output answer it:

```
EVAL GATE: FAILED

Overall:  0.847 → 0.812  (−0.035, within band)

Per stratum:
  simple_factual   (12)  0.93 → 0.94   +0.01
  multi_hop        (10)  0.86 → 0.85   −0.01
  ambiguous         (8)  0.79 → 0.77   −0.02
  out_of_scope      (8)  0.88 → 0.38   −0.50   ← FAIL (max_drop 0.10)
  long_context      (6)  0.81 → 0.80   −0.01
  adversarial       (6)  0.74 → 0.76   +0.02

Hard failures:
  refusal_recall    0.38 < 0.70        ← FAIL

Cost: $0.0021 → $0.0019 per request (−9%)

Changed since baseline: prompts/answer.md v3 → v4

Failing examples (3 of 5):
  q-012  "How do I configure SAML?"  → answered from general knowledge
  q-031  "What's the pricing?"       → invented a price
  q-044  "Does it support GraphQL?"  → confident yes, not in corpus
```

Notice the overall score dropped only 0.035 — inside the band, and a gate on the aggregate alone would have **passed this**. The per-stratum check is what caught it, and the failing examples make the cause obvious in ten seconds: prompt v4 broke refusal.

This is the Lesson 3.1 worked example, caught automatically.

---

## Key Concept 4: The eval report as an artifact

Every full-tier run appends to `eval-report.md`. Structure each entry:

```markdown
## 2026-09-14 — Module 5: added cross-encoder reranking

**Change:** retrieve 50 → rerank to 5 (was: retrieve 5 directly)
**Prompt version:** answer v3 (unchanged)
**Dataset:** golden v1.2 (60 items)

| Stratum | Before | After | Δ |
|---|---|---|---|
| multi_hop | 0.71 | 0.86 | +0.15 |
| ... | | | |
| **Overall** | **0.812** | **0.871** | **+0.059** |

Cost: $0.0019 → $0.0031 (+63%)
p95 latency: 2.1s → 3.4s

**Decision:** kept. The multi-hop gain is well outside the noise band
and multi-hop is our second-priority failure mode per the charter.
Cost increase accepted; Module 10 will address latency.
```

Two things make this the most differentiating artifact in your Module 12 portfolio.

**It shows a decision, not just a number.** The trade-off is stated and resolved against the charter.

**It will eventually contain a reverted change,** and that entry is the strongest page in the file. "We tried X, it cost 4 points on adversarial phrasing, we reverted" is direct evidence of engineering discipline that almost no candidate portfolio contains.

---

## Key Concept 5: Putting the suite into a tool

You can build all of this with `make` and a Python script, and the first time you should, because it teaches you what the parts are. In a team you will usually map it onto an eval tool. *Checked 2026-11-11:*

- **Promptfoo 0.125.x** — open-source CLI (MIT), config in YAML, runs locally and in any CI. OpenAI announced an agreement to acquire Promptfoo on 2026-03-09 and said it will remain open source; the closing had not been confirmed when this was checked, so re-check the licence and repository before you standardize on it.
- **Hosted alternatives:** **Braintrust** and **LangSmith** run the same kind of experiments with datasets, scorers, and a UI for comparing runs, and both have CI integrations. Pick one if your team already uses its tracing (Lesson 10.4).

The mapping from this module onto Promptfoo:

| This module | Promptfoo |
|---|---|
| Golden dataset item with its stratum | a test with `vars` and `metadata: {stratum: out_of_scope}` |
| Your pipeline under test | a Python provider: `id: 'file://spine_provider.py'` exposing `call_api(prompt, options, context)` |
| Mechanical checks (refused? citation valid?) | `python` or `javascript` assertions |
| Calibrated judge (Lesson 3.4) | an `llm-rubric` assertion, with your rubric text and judge model |
| Per-stratum and per-metric reporting | assertion `metric:` labels, test `metadata` |
| Smoke vs full tier | two config files, or one config filtered by metadata |
| The gate | `promptfoo eval` exits with code 100 when tests fail or the pass rate is below `PROMPTFOO_PASS_RATE_THRESHOLD` |

```yaml
# promptfooconfig.yaml (smoke tier, abbreviated)
providers:
  - id: 'file://spine_provider.py'
tests:
  - vars: {question: "Does it support GraphQL?"}
    metadata: {stratum: out_of_scope}
    assert:
      - type: python
        value: file://checks/refused.py
        metric: refusal_recall
```

```bash
PROMPTFOO_PASS_RATE_THRESHOLD=95 npx promptfoo@0.125.1 eval -c promptfooconfig.yaml -o results.json
```

**The 95 in that line is a decision, not a constant.** By default Promptfoo allows no failures at all (100%), which on a non-deterministic system is a gate that flaps. Set the pass rate per suite from what a failure costs:

- An **out-of-scope question answered with an invented price** is a support incident. That suite's hard-floor tests should allow no failures, or close to none.
- A **style or verbosity** suite where a failure means a slightly worse answer can tolerate several percent.
- And the threshold must sit **below the pass rate your unchanged system scores run to run** (your Lesson 3.4 noise), or it fails on no change at all.

Two numbers live side by side: a **pass-rate threshold** for per-item assertions (did this item pass its checks?), and the **baseline band** from Key Concept 2 for aggregate scores (did quality drop by more than noise?). The tool gives you the first out of the box; the comparison with the committed baseline is still a short script you own, reading `results.json`.

---

## Worked Example: The gate that earned its keep

A GitHub Actions workflow:

```yaml
on:
  pull_request:
    paths: ['spine/**', 'prompts/**']
jobs:
  smoke:
    steps:
      - run: make eval-smoke        # 15 items, ~80s, ~$0.04
      - run: make eval-gate TIER=smoke

  full:
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    steps:
      - run: make eval-full         # 60 items, ~4min, ~$0.90
      - run: make eval-gate TIER=full
      - run: make eval-report-append
```

A pull request edits the answer prompt to reduce verbosity. Smoke tier: **passes** — 15 items, only two of them out-of-scope, and the aggregate stays inside the band. This is the honest limitation of a smoke tier, and it is why it is not the only tier.

Merge to main. Full tier: **fails**, with the report above. Reverted in eleven minutes.

Total cost of catching it: about a dollar and one revert. Cost of not catching it: a documentation assistant confidently inventing pricing for however long it takes a customer to complain.

---

## Practice

**1. Build both tiers.** `make eval-smoke` and `make eval-full`. Select the smoke set for stratum coverage — at least one item per stratum, weighted toward your top-priority failure modes.

**2. Write the gate config** with thresholds derived from your own measurements: overall band from your Lesson 3.4 trust boundary, wider per-stratum bands, hard floors from your charter's unacceptable-answer list, and a cost ceiling.

**3. Make the report actionable.** Per-stratum table, hard-failure list, cost delta, what changed since baseline, and up to five failing examples with their questions. Print it to CI output, not to a file nobody opens.

**4. Wire it in.** Smoke on PR, full on merge. Add caching for embeddings and judge verdicts.

**5. Break it deliberately.** Make a change you know harms one stratum — the "be concise and direct" edit from Lesson 3.1 is a good one. Push it. **Confirm the gate fails and names the right stratum.** If it passes, your bands are too wide or your smoke set has a coverage hole; fix and repeat until it catches.

**6. Commit the baseline** to `eval-report.md` with the pre-retrieval numbers from Lesson 3.3.

**7. Port the smoke tier to Promptfoo (40 minutes).** Install a pinned version (`npm install -D promptfoo@0.125.1`). Write a Python provider that calls your pipeline, turn your 15 smoke items into tests with `metadata: {stratum: ...}`, express the refusal and citation checks as `python` assertions with `metric:` labels, and add your judge as an `llm-rubric` assertion. Choose `PROMPTFOO_PASS_RATE_THRESHOLD` and write one sentence in `docs/eval-gate.md` justifying it from what a failure costs and from your measured run-to-run noise. Re-run the step 5 breaking change. **You know it worked when** `promptfoo eval` exits with code 100 on the breaking change, exits 0 on the unchanged system three runs in a row, and the failing items in `results.json` carry the right stratum.

---

## Visual Aid

*CI pipeline flowchart. `commit` → `smoke tier (15 items, ~90s, ~$0.04)` → gate diamond → PR status. `merge to main` → `full tier (60 items, ~4min, ~$0.90)` → gate diamond → `append to eval-report.md`. A third branch, dashed and off to the side, labeled `held-out tier — manual, at milestones only`, with an annotation: "not in CI, by design." Beside the full-tier gate diamond, an expanded callout shows the per-stratum table with one row highlighted in red and an arrow labeled "this row is why the gate exists — the aggregate passed."*

---

## Key Takeaways

- **Two tiers make evaluation affordable in CI:** a fast cheap smoke set on every commit, the full set on merge. The held-out set stays out of CI on purpose.
- **Gate on a band derived from your measured noise,** not a fixed number. A flapping gate gets disabled, and then you have no gate.
- **Per-stratum thresholds and absolute hard floors** catch what the aggregate hides — including the exact failure mode from Lesson 3.1, where the overall drop was inside the band.
- **The report must name the stratum and show failing examples.** A gate that reports only a number costs an engineer an hour every time it fires.
- **`eval-report.md` is a portfolio artifact.** It shows decisions and trade-offs, and the entry where you reverted a change is the strongest evidence of discipline in your repository.
- **A tool runs the suite; you still own the thresholds.** Promptfoo (or hosted Braintrust / LangSmith) maps the tiers, strata, and judge onto config, but the pass rate is set per suite from what a failure costs and must sit above your measured noise.

---

## Sources

Checked 2026-11-11.

- Promptfoo command line (exit code 100, `PROMPTFOO_PASS_RATE_THRESHOLD`, `-o`): https://www.promptfoo.dev/docs/usage/command-line/ — checked 2026-11-11 (Promptfoo 0.125.1, MIT)
- Promptfoo configuration reference (test `metadata`, `threshold`, assertion `metric`): https://www.promptfoo.dev/docs/configuration/reference/ — checked 2026-11-11
- Promptfoo Python provider and Python assertions: https://www.promptfoo.dev/docs/providers/python/ and https://www.promptfoo.dev/docs/configuration/expected-outputs/python/ — checked 2026-11-11
- Promptfoo CI/CD integration (caching, GitHub Action): https://www.promptfoo.dev/docs/integrations/ci-cd/ — checked 2026-11-11
- OpenAI, "OpenAI to acquire Promptfoo" (2026-03-09): https://openai.com/index/openai-to-acquire-promptfoo/ — checked 2026-11-11
- Braintrust: https://www.braintrust.dev/docs — checked 2026-11-11
- LangSmith evaluation: https://docs.langchain.com/langsmith/evaluation — checked 2026-11-11

---

## Next

The harness is live, the baseline is committed, and the gate will stop a regression from reaching main. From this point on, every change you make in this course has a number attached to it, and every claim you make in an interview about this system can be backed with evidence.

**Module 4** is the first time you get to move that number. You will build the retrieval layer — embeddings chosen on your own corpus rather than a leaderboard, chunking strategies measured against each other, two vector stores compared honestly, and hybrid search that fixes the queries dense vectors quietly fail. And for the first time, you will be able to prove the improvement rather than assert it.
