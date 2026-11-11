# Module 2 practical — An invoice extractor that never returns an invalid object

This practical puts the whole module together on messy, realistic input. You build an extractor that turns invoice text into a typed object using the strongest structuring each backend supports, repairs only what is worth repairing, stops on a budget you can justify from your own data, and records every failure instead of hiding it.

## Task

Build an extractor for messy invoice text that **never returns an invalid object**. Inputs include mixed formats, missing fields, two languages, and one OCR-mangled scan.

## Deliverable

`exercises/m2-invoice-extractor/` containing the schema, the repair budget with a data-backed justification, and a failure log showing what was refused and why.

## How it is assessed

| Criterion | Full credit |
|---|---|
| Never returns invalid | No code path constructs a default object on failure |
| Strongest mechanism | Uses the strongest structuring available per backend, with a documented fallback ladder |
| Terminal classification | Refusals, truncation, and impossible-field cases do not enter the repair loop |
| Budget justified | Cites the candidate's own recovery table, not a convention |
| Failure log | Records input, attempts, errors, and reason for refusal |

**Common failure:** returning a default-constructed object when the budget is exhausted. That reintroduces the Lesson 2.1 silent wrong value through the error handler itself.
