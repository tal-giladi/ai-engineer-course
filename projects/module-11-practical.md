# Module 11 practical — Red-team a vulnerable agent

This practical puts the module's defensive method to work on a system built to be broken. You attack a deliberately vulnerable agent in your own local sandbox, enumerate attacks from a channel × target grid rather than improvising, and treat every exploit the way the module treats a bug: understand the mechanism, fix it with the right kind of control, and lock the fix in with a regression test. The part that matters most is honesty about what you could not fix without giving up a feature, and what you recommend instead.

## Task

Attack the deliberately vulnerable agent in `exercises/m11-vulnerable-agent/` in your local sandbox. Document every successful exploit with its mechanism, then fix and regression-test each one.

## Deliverable

An attack log, the fixes, a regression suite, and a note on any vulnerability that could not be fixed without removing a feature, with your recommendation.

## How it is assessed

| Criterion | Full credit |
|---|---|
| Systematic | Attacks derived from a channel × target grid, not improvised |
| Category coverage | All five categories attempted, including resource exhaustion |
| Mechanism documented | Each success explains *how* it worked, not just that it did |
| Fixes match mechanism | Architectural controls where architectural, not prompt patches |
| Regression tests | Each fix has a test asserting attack fails **and** functionality survives |
| Honest open items | Anything unfixed stated with reasoning and a recommendation |

**The unfixable-without-removing-a-feature note is the graded part.** It tests whether you can state a genuine security/functionality trade-off and make a recommendation, which is the judgment a hiring panel is actually assessing.
