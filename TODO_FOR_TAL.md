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

## Content issues

The small inconsistencies found while writing quizzes were fixed in the lessons on 2026-11-11 (see the
"Fix small content inconsistencies" commit). Nothing open.
