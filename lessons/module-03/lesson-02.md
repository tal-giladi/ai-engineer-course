---
id: "03.2"
module: 3
minutes: 13
practice_minutes: 90
prerequisites: ["03.1"]
objectives:
  - "Build a golden dataset for a task — sourcing, labeling, sizing, and splitting it — and justify every choice."
volatility: concept
sources: []
last_verified: "2026-08-24"
---

# 03.2 · Building a Golden Dataset

Of everything you build in this course, the golden dataset is the artifact most likely to still be in use in two years.

Your embedding model will be replaced. Your framework will be replaced — twice, probably. The frontier model you are calling today will be deprecated. The fine-tuned adapter from Module 8 will be retrained against a different base. All of it is fungible.

The dataset is not. It encodes what your system is supposed to do, which does not change when the technology underneath it does. Every model swap, every prompt change, every architectural decision for the lifetime of the project gets evaluated against it. Teams that have one move confidently; teams that do not argue.

It is also smaller and more achievable than people expect. You are going to build a good one in about ninety minutes.

---

## Key Concept 1: Small and deliberate beats large and scraped

The instinct is that more data is better. For evaluation, it is not — and the reason is specific.

A large scraped dataset is drawn from whatever was easy to collect, which means it is dominated by the common case. Five thousand examples of which 4,800 are simple lookups will tell you, precisely and repeatedly, that your system handles simple lookups. The failure you actually need to find is in the 40 adversarial cases that are not in there at all.

Sixty deliberately chosen examples covering eight categories tell you something about all eight.

There is a real statistical constraint alongside this, and you should know it rather than pretend otherwise: **a 60-item set cannot reliably detect a 2% change.** With 60 items, differences smaller than roughly 5–8 percentage points are inside the noise. That is a genuine limitation, and the honest response is not to inflate the set with easy examples — it is to know your resolution and refuse to act on differences below it. Lesson 3.5 turns that into a CI threshold policy.

Practical sizing: **40–60 items per major task** for a working harness, growing over time as production surfaces failures you did not anticipate. Aim for at least 5 items in any stratum you want to reason about separately.

---

## Key Concept 2: Stratification is the whole game

An unstratified dataset is a thermometer that only reads room temperature.

**Stratification** means deliberately including known proportions of known categories. You decide the categories from your Lesson 3.1 charter, and you record the proportions, because the proportions are how you read the results later.

For a documentation-QA system, a reasonable set of strata:

| Stratum | Count | Why it exists |
|---|---|---|
| Simple factual | 12 | The common case. Establishes the ceiling. |
| Multi-hop | 10 | Requires combining two or more sources. Where retrieval breaks. |
| Ambiguous | 8 | Underspecified questions. Tests clarification vs guessing. |
| **Out-of-scope** | **8** | **Not in the corpus. Tests refusal.** |
| Long-context | 6 | Needs a large amount of retrieved material. |
| Adversarial phrasing | 6 | Negations, unusual wording, terms the corpus never uses. |
| Conversational | 5 | Depends on prior turns. Tests query rewriting. |
| Near-miss | 5 | Similar to something in the corpus but subtly different. |

Note that **out-of-scope is 13% of the set**. Nothing in ordinary traffic would produce that proportion. It is there because refusal is the behavior most likely to silently degrade — as the Lesson 3.1 worked example demonstrated, where a one-line prompt change cost 46 points on exactly this stratum and would have been invisible in an aggregate.

Two rules follow:

**Over-sample what you fear.** The strata exist in proportion to the *risk* they represent, not their frequency in traffic. Your dataset is not a sample of production; it is a test suite.

**Never report only the aggregate.** A weighted average across these strata hides the one number that matters. Every result in this course is reported per stratum.

---

## Key Concept 3: Sourcing

**Real user queries** are the best source. If the system is live, sample from logs across time and user segments. If it is not live, get five minutes each from three people who would use it and write down what they actually ask — their phrasing will be worse and stranger than yours, which is the point.

**Your own hard cases.** Every question you wondered about while building. These cluster in exactly the ambiguous and near-miss strata that are hardest to source otherwise.

**Synthetic generation** is legitimate and widely used — generate variations with a model to fill a thin stratum. Two conditions: **label it as synthetic** in the dataset, and **verify each item by hand**, because a model generating test cases for a model produces cases that are suspiciously easy for that model.

**Production failures**, once you have them. In Lesson 2.3 you built a dead-letter store. Those records are already-vetted hard cases. Module 10 formalizes this into a loop where production failures flow back into the dataset continuously.

---

## Key Concept 4: Labeling that two people would agree on

A label is only as good as the rubric behind it, and rubric ambiguity is the most common defect in a homemade dataset.

The test is concrete: **have two people label the same ten items and compare.** If they disagree on three, your rubric is underspecified — not because either person is wrong, but because the rubric permitted both readings. Fix the rubric and re-label. Every ambiguity you leave in becomes noise in every measurement you take for the next two years.

For generative tasks, a "label" is usually not a single string — it is a set of requirements:

```yaml
- id: q-047
  stratum: multi-hop
  source: real_user_log
  question: "Can I use batch endpoints with streaming responses?"
  must_contain:
    - "batch endpoints do not support streaming"
  must_not_contain:
    - fabricated version numbers
  must_cite: ["api/batch.md#limitations"]
  acceptable_refusal: false
  notes: "Requires combining the batch page and the streaming page."
```

This form is checkable mechanically for the hard requirements and by judge for the soft ones — which is exactly the split Lessons 3.3 and 3.4 build.

---

## Key Concept 5: The held-out set, and leakage

Split roughly 70/30 into a **development set** you iterate against and a **held-out set** you look at rarely.

The reason is not ceremony. The moment you tune against a set, it stops measuring generalization and starts measuring how well you tuned. This happens gradually and invisibly: you check a failing item, adjust a prompt, check again. Twenty repetitions later the dev set is effectively training data and its scores are inflated.

Rules that keep this honest:

- **Run the held-out set at milestones**, not continuously — end of module, before a release.
- **Never fix an individual held-out failure.** Fix the class of problem on the dev set; if you fixed a real class, the held-out set moves too.
- **If you have looked at it repeatedly, retire it.** A burned held-out set should be replaced, not trusted.
- **Record the split.** Item IDs in a committed file, not a random seed you might change.

One rule matters more than all of them, and it applies in Module 8: **the held-out set must never enter a fine-tuning dataset.** In Lesson 8.2 you will build training data from production traces, and those traces may include the very questions in your held-out set. If they leak, your Module 8 comparison of fine-tuned versus baseline is worthless — and it will look *excellent*, which is worse. Lesson 8.2 requires an explicit leakage check for exactly this reason. Set up now to make it easy: keep the held-out IDs and their question hashes in one file.

---

## Worked Example: Sharpening a rubric

Two labelers annotate ten items. They disagree on three, all of the same shape:

> **Q:** "What's the rate limit?"
> **Labeler A:** ambiguous — should ask which endpoint.
> **Labeler B:** simple factual — the docs have a default rate limit; answer that.

Neither is wrong. The rubric said "mark ambiguous if the question could refer to multiple things," which is satisfied, and also said "mark simple factual if a direct answer exists," which is also satisfied.

The revision:

```diff
- Mark AMBIGUOUS if the question could refer to multiple things.
+ Mark AMBIGUOUS only if answering the most likely interpretation
+ would be MISLEADING. If a reasonable default answer exists and is
+ correct for the common case, mark SIMPLE FACTUAL and require the
+ answer to state which case it covers.
```

Re-labeled: all three become simple factual, with an added `must_contain: ["specify which endpoint the limit applies to"]`.

Agreement on the ten-item overlap moved from 70% to 100%. That 30% would otherwise have been permanent noise in every measurement derived from this set.

---

## Practice

**1. Define strata.** From your Lesson 3.1 charter, write your strata table: name, target count, and one sentence on why it exists. Over-sample your feared cases. Include an out-of-scope stratum of at least 6 items.

**2. Source 40–60 items.** Mix real questions, your own hard cases, and synthetic fill for thin strata. Tag every item with its `source`.

**3. Write the rubric,** then test it: label a 10-item overlap sample twice (ideally with another person; if solo, label, wait an hour, re-label blind). Compute agreement. **If it is below 90%, revise the rubric and repeat.** Do not proceed on a rubric you cannot apply consistently.

**4. Label everything** in the YAML form above.

**5. Split 70/30** and commit the held-out IDs to `spine/eval/holdout_ids.txt`, with question hashes alongside for the Module 8 leakage check.

**6. Write the datasheet** — `spine/eval/dataset.md`: strata and proportions, sources, rubric, agreement rate, split method, and known limitations. That last section is a hiring artifact: "our set under-represents multilingual queries" reads as judgment; its absence reads as unawareness.

---

## Visual Aid

*A horizontal stacked bar, 60 segments wide, one per item, coloured by stratum. Below it, a second bar of the same width showing what the equivalent production traffic distribution looks like — overwhelmingly simple-factual, with out-of-scope a sliver. Arrows connect the two bars at the out-of-scope band, annotated "2% of traffic, 13% of the dataset." Caption: "A golden set is not a sample of production. It is a test suite, weighted by risk."*

---

## Key Takeaways

- **The dataset outlives everything else you build.** Models, frameworks, and prompts are fungible; the specification of correct behavior is not.
- **40–60 stratified items beat thousands of scraped ones** — and know your resolution: a 60-item set cannot detect a 2% change, so do not act on differences that small.
- **Stratify by risk, not by frequency.** Over-sample refusal, adversarial phrasing, and near-misses. Report per stratum, never only in aggregate.
- **Test the rubric with two labelers on ten items.** Below 90% agreement, the rubric is the problem and the noise is permanent.
- **Hold out 30%, look at it rarely, never fix individual failures in it, and record the IDs and hashes** — Module 8's fine-tuning comparison depends on proving it never leaked into training.

---

## Next

You have labeled examples. The obvious next step is to score your system against them — and the obvious way to do that, accuracy, will tell you almost nothing useful and may actively mislead you.

**Lesson 3.3** covers choosing metrics that fit the task shape, why an aggregate of 0.88 can conceal a category failing at 0.20, and how to produce the baseline number that every remaining module in this course is measured against.
