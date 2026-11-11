# Module 9 practical — Fix a verbose model with preference tuning

This practical puts Lessons 9.1 and 9.2 to work on a model with one known flaw: it is too verbose. You build a preference set that targets that one dimension, audit it for confounds, run preference tuning while reading sampled outputs at checkpoints, and then prove two things at once: responses got shorter, and task accuracy did not fall. The second half is what separates a real improvement from a model that has merely been changed.

## Task

Take the supplied SFT model in `exercises/m9-verbosity-fix/` with a known verbosity problem and correct it with preference tuning **without losing task accuracy**.

## Deliverable

Preference set, confound audit before and after, training run with sampled outputs at checkpoints, and a before/after evaluation covering both response length and task accuracy.

## How it is assessed

| Criterion | Full credit |
|---|---|
| One dimension | Pairs vary on verbosity only; correctness, citation, and formatting matched |
| Confound audit | Length ratio (chosen / rejected) reported before and after; here it should sit well below 1, because brevity is the trained dimension (the Lesson 9.1 exception to the 0.9–1.1 target). Every other dimension is checked as matched |
| Judge ceiling stated | The judge's measured agreement rate acknowledged as the data's quality ceiling |
| Sampled outputs | Generated at checkpoints during training and read, not curves alone |
| Both axes measured | Response length **and** task accuracy, before and after |
| Capability check | Full stratified set re-run; regressions reported |

**Common failure:** fixing verbosity and not re-checking task accuracy. Preference tuning trades against other dimensions, and a model that is now concise and 4 points less accurate has not been improved — it has been changed.
