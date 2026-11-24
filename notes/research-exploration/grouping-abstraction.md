# Area 1: The grouping abstraction

## What the prototype assumed

```
WorkNode ::= Select(k, WorkNode...)
           | Leaf(Workload)
```

- k and n are fixed at submission.
- The tree is done when k children are done; any completion beyond k is worth nothing.
- Children are interchangeable except for their scores.

The workload survey (in `ignored-notes/`) found that this fits few real workloads. Strict
k-of-n is rare; growing groups, stop-on-stability and all-of-n with duplicates are common.

## The groupings

Nine distinct groupings, each with one test workload. For each: how the group completes,
how its membership changes, what the scheduler can do to it, and how it relates to
`Select(k)`.

### 1. Value-ranked race that grows (W1, HPO)
- **Completes:** when the budget runs out, a goal metric is reached or the best member is
  stable. The group's value is the best member's score.
- **Membership:** grows. ASHA keeps adding trials while there is capacity. Hyperband adds
  rounds: brackets -> rungs -> trials, keeping the top $1/\eta$ of each rung behind a barrier.
- **Scheduler actions:** admit new members, stop low-ranked ones.
- **vs. Select:** `Select(1, ...)` with n growing over time and, for Hyperband, nested rounds.

### 2. Stop on stability (W2, Monte Carlo)
- **Completes:** when a group statistic converges (CI half-width $\le \epsilon$) or at a deadline.
  Value rises with the number of samples, with diminishing returns.
- **Membership:** fully elastic. Members are fungible; any can stop at any time and its
  work still counts.
- **Scheduler actions:** grow when the cluster is idle, shrink when it is busy.
- **vs. Select:** no k. Completion is a predicate on a statistic, not a count.

### 3. Hard quorum (W3, CodedMatMul)
- **Completes:** when any k of n members finish. Completions beyond k are worth nothing.
- **Membership:** fixed n, chosen at submission.
- **Scheduler actions:** stop the rest once k finish; choose n according to contention.
- **vs. Select:** exactly `Select(k, ...)`.

### 4. All of n with first-wins duplicates (W4, Spark)
- **Completes:** when every member finishes. Each member can have several attempts, and
  the first attempt to finish wins. Groups form a DAG of stages with a barrier between them.
- **Membership:** fixed members; attempts are added and removed.
- **Scheduler actions:** duplicate a straggler, stop losing attempts.
- **vs. Select:** `Select(n, Select(1, task, task'), ...)`, with dependencies between groups.

### 5. Quota of groups with a worthless-group abort (W5, RL rollouts)
- **Completes:** when a batch holds B informative sub-groups. Each sub-group (a prompt's G
  samples) needs all its members, and is worthless if their rewards are all equal. A
  training-step barrier sits on top.
- **Membership:** nested and over-provisioned; extra sub-groups are started and the tail is
  aborted.
- **Scheduler actions:** abort a sub-group once it is known to be worthless; stop once the
  quota is filled.
- **vs. Select:** groups of groups, with an abort predicate on a sub-group's values.

### 6. Elastic gang (W6, elastic DDP)
- **Completes:** when training finishes. Value is throughput, concave in size.
- **Membership:** all members run together; the size changes within $[\min, \max]$, and each
  change costs a restart.
- **Scheduler actions:** resize within the range.
- **vs. Select:** no subset choice. The group is all-or-nothing at any size in the range.

### 7. Fixed population with replace and pause (W7, PBT)
- **Completes:** when the total budget runs out.
- **Membership:** fixed size. At each interval the worst members are replaced by copies of
  the best. Members are paused and resumed from checkpoints.
- **Scheduler actions:** pause and resume members; choose which to run when capacity is
  short.
- **vs. Select:** no completion count; members are replaced in place rather than dropped.

### 8. Fail-fast all of n (W8, CI matrix)
- **Completes:** when every member passes. Any failure makes the group worthless.
- **Membership:** fixed.
- **Scheduler actions:** cancel the rest on the first failure; run likely failures first.
- **vs. Select:** `Select(n, ...)` with an abort predicate.

### 9. Open-ended allocation (W9, fuzzing)
- **Completes:** never. Value is the rate of new results (coverage, crashes).
- **Membership:** members get budget in proportion to how productive they are.
- **Scheduler actions:** move resources to the most productive members.
- **vs. Select:** no completion condition at all.

## What the groupings need beyond Select(k)

- **Utility, not a count:** groupings 2, 6 and 9 have value that rises smoothly.
- **Growing and replaced membership:** groupings 1, 5 and 7.
- **Rounds and dependencies:** Hyperband rungs, Spark stages, RL training steps.
- **Abort predicates:** groupings 5 and 8.
- **Attempts per member:** grouping 4.
- **Actions beyond run and stop:** duplicate (4), pause (7), resize (2, 6).

## Open questions

1. **Is k-of-n the primitive, or a special case?** A generalisation is `Group(U, children)`,
   where `U(completed set)` is a utility and Select(k) is a step at k. More expressive
   utilities make the scheduler's problem harder; TetriSched needs a MILP for this.
2. **Should groupings be one construct or several?** One `Group(U, ...)` with predicates and
   a membership policy, or a small set of named shapes.
3. **Who owns membership?** The application knows which member is expendable (bias in RL,
   replacement in PBT). The scheduler knows how much the group should get.

