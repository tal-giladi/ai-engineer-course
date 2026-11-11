# TODO for Tal

## Ready to import

The course is in Tal's Academy format and `npm run check-course -- ../course-creator/ai-engineer`
reports 0 problems. Repo: `tal-giladi/ai-engineer-course`, branch `main` (commit: see `git log -1`).
Registration details (slug, summary, free/paid suggestion): `curriculum/course-details.md`.

## Decisions taken in the conversion

- `output/ai-engineer-course-complete.md` is stale (predates the 2026-11-11 updates and the format
  conversion). It is NOT imported (`output/` is planning-only) and was not regenerated. Regenerate or
  delete it if you still want a single-file version.
- Practical items from the old module quizzes moved to `projects/module-NN-practical.md` (modules 1-11;
  module 12 had none). They import as the "Projects" section, not as graded items.
- The four rubrics in `assets/` and `assessments/capstone-rubric.md` are top-level sidebar links
  (the "Reference" section).
- `sources:` is filled only for the eight lessons that have a `## Sources` list (1.2, 2.2, 2.3, 3.5,
  7.1, 7.2, 7.4, 10.5). The other 45 lessons have `sources: []` and `last_verified: "2026-08-24"`
  (the date they were written) — they have not been fact-checked against current sources.
- `prerequisites` = the previous lesson in course order. `practice_minutes` are estimates from each
  Practice section (many are 90, the cap).

## Content issues found while writing quizzes (lesson prose NOT changed)

- 1.4 worked example: attempt counts add up to 2,914 calls, text says 4,102 and 4.1 calls/request.
- 1.4 objective is a copy of 1.3's (both map to M1-LO3).
- 2.3 worked example: the "% of total tokens" column doesn't match the token counts.
- 3.4 vs 3.5: judge trust boundary is 0.4 on a 0-3 scale in 3.4 but `max_drop: 0.04` on 0-1 in 3.5.
- 3.2 vs 3.5: 3.2 splits 40-60 items 70/30, 3.5 calls 60 items "the complete dev set".
- 4.2 takeaways say "high double-digit recall gains" from contextual headers; its own table shows +9.
- 6.1 Key Concept 4: unbalanced quote in the result-handle example.
- 7.4 Visual Aid says filesystem server "pinned v1.4"; the code pins `server-filesystem@2.3.0`.
- 7.3 not re-verified on 2026-11-11 (still 2026-08-24) although the rest of module 7 was.
- 8.1: "2,800 fewer bad records × 6 min ≈ 280 hours a year" — that is ~280 hours a month.
- 8.2: says 80/20 split but gives 800/232 of 1,032 (~78/22); near-dup threshold 0.92 vs 0.95.
- Module 9 practical rubric asks for a chosen/rejected length ratio near 0.9-1.1 on a verbosity fix,
  which contradicts 9.1's exemption when brevity is the trained dimension.
- 10.1: says cancellation crosses three layers; Visual Aid lists four steps.
- 10.3 worked example: rows for Changes 2-5 have 3 cells under 4 column headers.
- 12.2 "Next" calls the depth ladder four rungs; 12.3 has five (12.3 Practice says "four levels").
- 12.3 says reranking cost 190 ms p95; 12.2's sample eval entry shows p95 2.1 s → 3.4 s.
- 12.4 Visual Aid: "40-minute budget per step" but the whole round is 40 minutes.
