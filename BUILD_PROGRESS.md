# BUILD_PROGRESS — Tal's Academy format conversion

Goal: convert the existing AI Engineer course (written 2026-08-24, updated 2026-11-11) to the Tal's
Academy import format (`tals-academy/docs/new-course-instructions.md`) so that
`npm run check-course -- ../course-creator/ai-engineer` reports 0 problems. Content stays the same in
substance; only format changes, plus new multiple-choice quizzes.

## Resume procedure

1. Read this file and `curriculum/status/*.log`.
2. A module is done when its log ends with `module done`. Skip done modules.
3. For a module in progress, skip lessons whose `lesson-MM done` line is in the log.
4. Run the dry-run import after every module:
   `cd C:/Users/TalGiladi/OneDrive/repos/tals-academy && npm run check-course -- ../course-creator/ai-engineer`
5. Commit after every finished module.

## Units

- [x] Foundations: front-matter on all 53 lessons, `# NN.M · Title` H1s, intro paragraph lifted
      under the H1 (the old `**Module…** · ~N min` / `**Objective:**` block moved into front-matter),
      `_sidebar.md` rebuilt (running numbers, no curriculum/brief/output links, rubrics as references)
- Per module: lesson `.quiz.yaml` files (3-5 questions), `practice_minutes` set from the Practice
  section, module quiz converted to `assessments/module-NN-quiz.quiz.yaml` (8-10 scenario MCQs) plus a
  short intro page, practical items moved to `projects/module-NN-practical.md`
  - [ ] Module 1
  - [ ] Module 2
  - [ ] Module 3
  - [ ] Module 4
  - [ ] Module 5
  - [ ] Module 6
  - [ ] Module 7
  - [ ] Module 8
  - [ ] Module 9
  - [ ] Module 10
  - [ ] Module 11
  - [ ] Module 12
- [ ] Final QA: check-course 0 problems, link check, TODO_FOR_TAL.md, push

## Who is working on what

(updated by the main session) Agents work one module each, two at a time, writing only that
module's quiz/project/status files and `practice_minutes`; the main session commits per module.

## Decisions

- Fixed lesson section set (kept from the original course): intro paragraphs under the H1, then
  `## Key Concept N: …`, `## Worked Example…`, `## Practice`, `## Visual Aid`, `## Key Takeaways`,
  optional `## Sources`, `## Next`.
- `last_verified`: 2026-11-11 for the eight lessons fact-checked that day (1.2, 2.2, 2.3, 3.5, 7.1,
  7.2, 7.4, 10.5); 2026-08-24 (date written) for the rest.
- `sources`: taken from each lesson's `## Sources` list; lessons without one have `sources: []`.
- `prerequisites`: the previous lesson in course order.
- `output/` is planning-only and not imported; `output/ai-engineer-course-complete.md` is left stale.
