# Problem statement

## One-line summary

A Kubernetes cluster decides how many pods a workload gets and which keep running, but it
sees no application context. Applications decide how many pods they want and which are worth
keeping, but they can't see cluster contention. Many batch and ML workloads *start more pods
than they need*, so each side makes scaling decisions with half the information.

## Background

The problem builds on the MIT IROP report (Select / value procs on SigmaOS), carried over to
Kubernetes.

In Kubernetes the decision of how much to run is split:

- **The cluster** (kube-scheduler, Kueue/Volcano admission, priority preemption) sees node
  capacity, queue depth, quotas and other tenants' demand. It can't see which pods matter
  to the application or which are making useful progress.
- **The application** (HPO controllers such as Katib and Ray Tune, Monte Carlo drivers) sees
  its own progress and how many results it needs. It can't see how contended the cluster
  is or whether its extra pods delay other work.

### Groupings in scope

Workloads start more members than they need, or members whose value differs, in nine
distinct groupings. Examples: a hyperparameter search where the best trial wins, Monte Carlo
that stops at a target precision, and coded computation that needs any k of n. All nine are
described in
[grouping-abstraction.md](research-exploration/grouping-abstraction.md#the-groupings), with
one test workload each in [workloads.md](baselines/workloads.md).

Kubernetes can express part of this. Gang scheduling (`minMember`, `minCount`) and Kueue's
partial admission let a job say "admit at least k". These choices are made **once, at
admission, and don't say which** pods matter. Once pods run, the cluster treats them as
interchangeable.

## Why the current split goes wrong

1. **Cluster-side scaling destroys value at random.** Preempting or shrinking a
   hyperparameter search may kill the best trial, because the cluster can't see which trial
   is ahead.
2. **App-side scaling ignores load.** Pruning thresholds and sample counts are fixed ahead
   of time. They waste resources when the cluster is busy and leave them idle when it is
   quiet.
3. **Nobody gives resources back gracefully.** A job that filled an idle cluster can't
   shrink for a new arrival and grow back when it leaves, except through blunt preemption.

## The problem to solve

> **How can a Kubernetes application report the live value of each computation to the
> control plane, so the cluster can choose which pods to run and stop using both
> application value and cluster contention, without exposing cluster state to tenants?**

### Design requirements

- **Dynamic:** decisions continue while the job runs, driven by live scores, not only at
  admission.
- **Contention-aware:** the same job grows when the cluster is idle and shrinks when it is
  busy, with no application code involved.
- **Narrow and safe:** applications report information up; cluster state does not flow down
  to tenants (report section 3.1).
- **Kubernetes-native:** CRDs, a controller and scheduler-framework plugins. Score reporting
  must not overload the API server.

### Out of scope

- Sub-second tasks. Pod start-up takes seconds, so the target is jobs lasting minutes or
  hours.
- Predicting full value curves (rejected in report section 3.4).
- Node placement, which stays with kube-scheduler.

## Research questions

1. **Performance:** under varying contention, does a general value-aware controller match
   or beat ASHA/Hyperband and Kueue/Volcano?
2. **Multi-tenancy:** on a shared cluster, does yielding and reclaiming capacity improve
   total utility and fairness compared with static gang and quota scheduling?
3. **Signal:** is the score alone enough, or does the application need to report something
   forward-looking?
