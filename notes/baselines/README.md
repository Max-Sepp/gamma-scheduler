# Baselines

This folder defines the evaluation: the workloads the scheduler runs on and the systems it is
compared against. Each workload runs under every system listed for it, on the same cluster
and at the same contention levels. Only the scheduling mechanism changes.

- [workloads.md](workloads.md): one test workload per group shape (W1 to W9) and a
  shared-cluster run (W10).
- [systems-under-test.md](systems-under-test.md): the baseline systems, how each is
  configured. Baselines that were left out are in `ignored-notes/left-out-baselines.md`.

No baseline is written by us. Each one is a released system or a built-in Kubernetes feature,
used as published and only configured.

## What each baseline tests

- **L1, default kube-scheduler:** that the problem exists and how large it is.
- **H1-H6, app-specific mechanisms:** whether one general mechanism matches what each
  workload already ships with (ASHA, Hyperband, Job `successPolicy`, Spark speculation,
  PBT, Job fail-fast).
- **C1 AdaptDL:** whether our interface beats an existing app-reported signal (goodput).
- **G1 Kueue, G2 Volcano:** whether live value beats static admission, gang scheduling and
  quotas.
- **U1, oracle signal:** how much is lost because the reported values are noisy.

## Which baselines each workload is compared against

Every workload also runs our scheduler, and G1 and G2 run on every workload.

- **W1 HPO:** L1 (full search), H1 ASHA, H2 Hyperband, U1.
- **W2 Monte Carlo:** L1 (fixed sample count), G1b elastic, U1.
- **W3 CodedMatMul:** L1 (`completions = k`), H3 `successPolicy`, G1a partial admission, U1.
- **W4 Spark:** L1 (speculation off), H4 Spark speculation.
- **W5 RL rollouts:** L1 (verl as published).
- **W6 Elastic DDP:** L1 (fixed size), C1 AdaptDL, G1b elastic.
- **W7 PBT:** L1 (no exploit step), H5 Ray Tune PBT.
- **W8 CI matrix:** L1 (no fail-fast), H6 `podFailurePolicy`.
- **W9 Fuzzing:** L1 (equal fixed allocation).
- **W10 Shared cluster:** G1 with preemption and G2 only. Each job's app-specific mechanism
  runs inside it.
