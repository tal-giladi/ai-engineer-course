---
id: "02.1"
module: 2
minutes: 11
practice_minutes: 60
prerequisites: ["01.4"]
objectives:
  - "Define a typed contract between model and application using JSON Schema and Pydantic, and enforce it with schema-constrained generation."
volatility: concept
sources: []
last_verified: "2026-08-24"
---

# 02.1 · Why Free-Form Text Is a Bad API

You would never accept an unvalidated response body from a third-party HTTP API and start indexing into it. You would define a schema, parse into a type, and handle the failure case. Everyone agrees on this. It is unremarkable engineering.

Now consider what almost every early LLM integration does: it calls a model, gets back a string, and pulls out the part it wants with `split()`, a regex, or `json.loads()` on a substring. The upstream in this case is *less* reliable than a third-party API — it is non-deterministic, its behavior shifts with every prompt edit and model version, and it has no versioned contract at all.

This lesson is about why that gap exists, why it fails in a particularly nasty way, and what replaces it.

---

## Key Concept 1: The boundary problem

Every LLM integration has the same shape:

```
your code  →  a string  →  the model  →  a string  →  your code
```

The two arrows in the middle are the model's world, where everything is text. The two arrows on the outside are your world, where you need a `Decision`, a `float`, a `list[Citation]`. Somewhere, text becomes a value.

The question is *where and how*, and there are only three answers:

1. **Parse the prose afterwards.** Regex, string splitting, heuristics. Always available, always fragile.
2. **Ask for a format and parse that.** "Respond in JSON." Better, still unenforced — the model may comply 97% of the time.
3. **Constrain the generation itself** so invalid output cannot be produced. Strongest, and not available everywhere.

Most projects start at (1), move to (2) after the first incident, and reach (3) only if someone knows it exists. This module takes you straight to (3), with a defined ladder down to (2) for backends that lack it.

---

## Key Concept 2: The failure is silent, and that is the real problem

If parsing failed loudly, this would be a minor annoyance. It does not.

Consider a classifier whose prompt asks for one of `positive`, `negative`, `neutral`, parsed with:

```python
sentiment = response.strip().lower()
```

The model usually returns `positive`. Sometimes it returns:

- `Positive` — handled by `.lower()`, fine
- `positive.` — trailing period, now an unrecognized value
- `Sure! The sentiment is positive.` — chatty preamble
- `**positive**` — markdown emphasis
- `positive (though with some reservations)` — hedged
- `The sentiment appears mixed, leaning positive` — refused the categories entirely

Now the critical question: **what does your code do with `"positive."`?**

If you wrote `if sentiment == "positive"`, it falls through to your else-branch and is silently recorded as something else. No exception. No log line. Just a wrong value in your database, indistinguishable from a correct one. The dashboard still renders. The tests still pass, because your test fixture returns the clean string.

This is the actual danger. A crash is a gift — it tells you where and when. Silent misclassification of 5% of your traffic is a defect you discover in a quarterly review, if ever.

> **The rule:** an LLM is an untrusted, non-deterministic upstream. Anything you cannot validate, you do not know.

---

## Key Concept 3: Why patching the parser is a losing game

The instinctive response to each failure is a small fix. Watch what happens.

**Failure:** trailing punctuation. **Fix:** `sentiment.strip(".!*")`.
**New failure:** chatty preamble. **Fix:** regex for the last word.
**New failure:** `"The sentiment appears mixed, leaning positive"` — last word is `positive`, so it now parses as a confident positive, which is *worse than failing*. You have converted an unhandled case into a wrong answer.
**Fix:** allowlist match against the three valid values, else raise.
**New failure:** now 4% of traffic raises, and you have no strategy for what to do with it.

Four fixes in, you have arrived at "validate against a schema and handle the invalid case" — which is where you should have started, having spent a week and shipped one genuinely bad regression on the way.

The deeper reason patching fails: **the output distribution is not stable.** It shifts when you edit the prompt, when the provider updates the model behind an alias, when input arrives in another language, when the input is unusual. Your parser encodes assumptions about a distribution that will move. A schema encodes requirements about a value, which do not.

---

## Key Concept 4: Contract-first

The alternative inverts the order. Instead of *generate, then interpret*, you *define, then constrain*:

```python
class Sentiment(BaseModel):
    label: Literal["positive", "negative", "neutral"]
    confidence: float = Field(ge=0.0, le=1.0)
    rationale: str = Field(max_length=200)
```

Three things this buys you immediately:

- **The type is the documentation.** Anyone reading your code knows exactly what the model produces.
- **Validation is free and total.** `Sentiment.model_validate(payload)` either gives you a correct object or raises — no partial states.
- **The schema can be pushed down to the model** (Lesson 2.2), so many of these failures become impossible rather than merely detected.

And one thing it does not buy you, which must be said plainly now so you are not surprised in Module 3: **a valid object is not a correct object.** The model can return perfectly-shaped output with a wrong label and a fabricated rationale. Schema enforcement solves the *shape* problem completely and the *correctness* problem not at all. Those are different problems with different solutions, and conflating them is a common and expensive mistake.

---

## Worked Example: The 8% you cannot see

A supplied extractor pulls three fields from support tickets — priority, category, customer tier — by parsing prose. It runs over 500 tickets. Manual review of a 100-ticket sample finds:

| Outcome | Count | Visible? |
|---|---|---|
| Correct | 92 | — |
| Raised an exception | 0 | would have been |
| **Wrong value, parsed cleanly** | **8** | **No** |

Breaking down the eight:

- 3× priority parsed as `high` from `"not high — this is a routine request"` (the regex matched the token, not the meaning)
- 2× category defaulted to `other` because the model wrote `Billing/Payments` and the allowlist had `billing`
- 2× customer tier read as `enterprise` from a sentence mentioning enterprise customers generally
- 1× the model returned a two-sentence apology and every field defaulted

Zero exceptions. Every one of those 500 rows looks identical in the database. The three `"not high"` cases are the worst: a negation flipped into its opposite, which no amount of monitoring on error rates would ever surface.

---

## Practice

You are given `exercises/m2-prose-extractor/` — a working prose-parsing extractor and 200 input records with known-correct labels.

**1. Run it.** Execute the extractor over all 200 records and diff against the labels.

**2. Catalogue, do not fix.** For every mismatch, record: the input, the model's raw output, the parsed value, the correct value, and *which assumption in the parser broke*. Group them into distinct failure shapes.

**3. Count the invisible ones.** Separate mismatches that raised an exception from those that produced a clean wrong value. Note the ratio.

**4. Write one paragraph** answering: if this ran in production with only error-rate monitoring, how long would it take to notice? Be honest — the answer is usually "never."

Resist fixing anything. The purpose is to feel the size of the surface, because the temptation to patch is what this lesson is inoculating against.

---

## Visual Aid

*A "failure gallery" — a grid of eight cards. Each card shows the same prompt at the top, then one real model output verbatim in monospace, then a red annotation naming the parser assumption it violates ("assumes no preamble," "assumes no markdown," "assumes last word is the answer," "assumes the model picks from the list"). Two cards are outlined in heavier red and marked "PARSES CLEANLY — WRONG VALUE," visually separating the failures you can detect from the ones you cannot.*

---

## Key Takeaways

- **The model is an untrusted, non-deterministic upstream.** Apply the same discipline you would to any third-party API, with more suspicion rather than less.
- **Parsing failures are usually silent,** producing plausible wrong values rather than exceptions. Silent wrong values are far more expensive than crashes.
- **Patching the parser is a losing game** because the output distribution shifts with prompts, model versions, and unusual inputs. Each patch can also convert an unhandled case into a confidently wrong one.
- **Define the contract first,** as a typed schema. The type documents, validates totally, and can be pushed down into generation.
- **Valid shape is not correct content.** Schemas solve the shape problem completely and the correctness problem not at all — that is Module 3's job.

---

## Next

You have a catalogue of every way prose parsing breaks on real inputs. The answer is not a better parser. The answer is to stop letting the model produce invalid output in the first place.

**Lesson 2.2** covers the three mechanisms for doing that — prompt-and-hope, tool-call shaping, and constrained decoding — which backends support which, and the schema design choices that move field accuracy by twenty points without changing a word of the prompt.
