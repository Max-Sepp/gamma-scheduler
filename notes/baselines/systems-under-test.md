# Systems under test

The systems our scheduler is compared against. Each one runs the same workload, on the same
cluster, at the same contention levels. Only the scheduling mechanism differs. No baseline should be written by us. It should be implemented by others and only configured by us.

---

## Lower bound

### L1. Default kube-scheduler
- Plain pods or Jobs with no pruning, early stopping, duplication or resizing. Each workload
  defines what this means for it (full HPO search, fixed sample count, no speculation, and
  so on).
- Shows how large the problem is.

## App-specific

Each is the mechanism a workload already ships with.

### H1. ASHA (W1)
- Ray Tune `ASHAScheduler` on KubeRay (Li et al., MLSys 2020).
- Configure $\eta$ (typically 3), `grace_period`, `max_t`, `max_concurrent_trials`.

### H2. Hyperband (W1)
- Ray Tune `HyperBandScheduler` on KubeRay (Li et al., JMLR 2018).
- Synchronous rungs, so it is sensitive to stragglers and contention.

### H3. Indexed Job with `successPolicy` (W3)
- An Indexed Job with n pods and a `successPolicy` rule `succeededCount: k`. The Job
  succeeds once any k pieces finish and Kubernetes terminates the rest. GA since 1.33.

### H4. Spark speculation (W4)
- `spark.speculation` on Spark on Kubernetes, with the quantile and multiplier tuned.

### H5. Ray Tune PBT (W7)
- Ray Tune `PopulationBasedTraining` on KubeRay. Configure the perturbation interval and
  the quantile that is replaced.

### H6. Job `podFailurePolicy` fail-fast (W8)
- An Indexed Job whose `podFailurePolicy` uses `FailJob` on test failure, so the first
  failing shard ends the Job.

## App-informed competitor

### C1. AdaptDL (W6)
- The open-source implementation of Pollux. Goodput-driven resizing of DL training jobs on
  Kubernetes.

## Generic Kubernetes batch schedulers

### G1. Kueue
- **G1a partial admission:** `minCount = k`, `count = n`.
- **G1b elastic workloads:** admitted jobs can grow and shrink.
- Priority-based preemption on for multi-tenant runs.
- The strongest form of "Kubernetes can already do this".

### G2. Volcano
- PodGroup with `minMember`, gang scheduling and queue-based preemption.

## Upper bound

### U1. Oracle signal
- Our policy fed each job's true future values: full learning curves from tabular HPO
  benchmarks, the true variance for Monte Carlo, or the recorded straggler for CodedMatMul.
- Shows how far the real signal is from the best it could be.
