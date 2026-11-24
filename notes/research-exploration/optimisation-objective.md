# Area 3: The optimisation objective

## What the prototype assumed

No objective was stated; each benchmark used its own metric. A paper needs one objective
that the policy approximates and that each workload's metric follows from.

## Candidate: value against the price of contention

Every decision to run or stop a member trades benefit to its group against cost to everyone
else:

- **Benefit:** the member's expected marginal contribution to its group's utility, estimated
  from its score ([Area 2](value-reporting.md)).
- **Cost:** the resources it holds $\times$ the current shadow price of those resources. The price
  is near zero when the cluster is idle and high when it is contended.

> **Run a member if and only if its expected marginal value exceeds the shadow price of the
> resources it uses.**

This gives the expected behaviour:
- **Idle cluster:** the price is about 0, so run everything.
- **Contended cluster:** the price is high, so run only the most valuable members.
- **A competitor arrives, then leaves:** the price rises, then falls, so the job shrinks,
  then grows back.

Monte Carlo is the cleanest case: marginal value per core-second follows from the CI width.

## Within a group vs. across groups

Scores are comparable only between siblings. So there are two levels:

- **Across groups:** the cluster sets each group's share using quotas, priorities and
  fairness.
- **Within a group:** scores decide which members use that share.

Letting groups say how much a share is worth to them would be more efficient but gameable.
Themis-style auctions are one answer.

## Open questions

- **Objective:** maximise total utility under fairness constraints, or minimise wasted work
  under deadlines? W10 points to the first.
- **Shadow price:** computed from pending demand, queue length or utilisation, and how fast
  it should react.
