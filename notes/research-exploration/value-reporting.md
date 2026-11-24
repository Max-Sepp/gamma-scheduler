# Area 2: Value reporting

## What the prototype assumed

- Each leaf calls `Score(score, gradient)`.
- The score is opaque and compared only with its siblings.
- The gradient is a local slope meant to separate "behind but climbing" from "plateaued"
  (report section 3.4).

## Observation

In the prototype the gradient is $(\text{score} - \text{prev}) / \Delta t$ (Listing 6): a finite difference of
values the scheduler already receives. The scheduler can compute it itself.

A reported gradient adds information only when it is a **forecast** the scheduler can't
infer from the score history. For example, an HPO trial knows its learning-rate schedule is
about to decay. A Monte Carlo job knows its variance, so it knows how many more samples
reach $\epsilon$.

## Open questions

1. **Is score plus a scheduler-derived slope enough?** Or do applications need to report a
   forecast?
2. **What does a score mean?** Progress, quality, or probability of being in the useful set.
   One declared meaning per group keeps scores comparable.
3. **How often must scores arrive?** This decides whether reports can go through the API
   server or need a sidecar or gRPC path.

## Experiment

Run W1 and W2 with three signals:

1. Score only.
2. Score + slope computed by the scheduler.
3. Score + app-reported forecast (HPO trials that know their LR schedule; Monte Carlo
   variance).

Compare each against the oracle (U1). **Hypothesis:** signal 2 matches the prototype's
reported gradient, and only signal 3 closes the gap to the oracle.
