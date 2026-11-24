# Workloads

One test workload per distinct group shape, plus a shared-cluster run that mixes them. The
shapes come from the workload survey (in `ignored-notes/`).

See also: [systems-under-test.md](systems-under-test.md),
[grouping-abstraction.md](../research-exploration/grouping-abstraction.md)

System IDs are from [systems-under-test.md](systems-under-test.md). Every workload also
runs our scheduler.

| Shape | Workload |
|---|---|
| Value-ranked race that grows, with rounds | W1 HPO |
| Stop on stability | W2 Monte Carlo |
| Hard quorum (k of n) | W3 CodedMatMul |
| All of n with first-wins duplicates | W4 Spark speculation |
| Quota of groups with a worthless-group abort | W5 RL rollouts |
| Elastic gang | W6 Elastic DDP |
| Fixed population with replace and pause | W7 PBT |
| Fail-fast all of n | W8 CI test matrix |
| Open-ended allocation | W9 Fuzzing |
| All of the above on one cluster | W10 Shared cluster |

---

## W1. Hyperparameter search (ASHA / Hyperband)

- **Shape:** value-ranked race that grows. The best trial wins, ending on a trial budget, a
  goal metric or a stable best. ASHA keeps adding trials. Hyperband nests brackets -> rungs ->
  trials with a barrier per rung.
- **Signal:** validation accuracy per epoch. Noisy, saturating, and curves can cross.
- **Setup (use both):**
  - Tabular benchmarks (e.g. LCBench, YAHPO Gym): trials replay recorded learning curves
    while burning matching compute. Every final value is known, so the oracle (U1) and
    regret come free. Check licences first.
  - Real training (a small ResNet on CIFAR-10), for real noise and stragglers.
- **Systems:** L1 (full search), H1 ASHA, H2 Hyperband, G1 Kueue, G2 Volcano, U1.
- **Notes:** run ASHA and Hyperband with default concurrency and with concurrency tuned per
  contention level. Use several seeds.
- **Metrics:** best accuracy against wall-clock and compute; regret; compute saved against
  full search.

## W2. Monte Carlo to target precision

- **Shape:** stop on stability. Fungible sample streams run until the CI half-width $\le \epsilon$ or
  a deadline. Any stream can stop at any time and its samples still count.
- **Signal:** CI width after each batch, falling as $1/\sqrt{n}$. Marginal value per core-second can
  be computed directly.
- **Setup:** a simple estimator (option pricing or $\pi$).
- **Systems:** L1 (fixed sample count), G1b Kueue elastic, G2 Volcano, U1 (true variance
  known). No app-specific scheduler exists.
- **Metrics:** time and core-seconds to reach $\epsilon$; jobs that meet $\epsilon$ by their deadline.

## W3. Erasure-coded matrix multiplication

- **Shape:** hard quorum. Any k of n coded pieces (IROP: k = 6, n = 9); completions beyond k
  are worth nothing. The original `Select(k)` case.
- **Signal:** pieces finished.
- **Setup:** the IROP worker and decoder code as pods, with an injected straggler. Use the
  long job; the short one is dominated by pod start-up.
- **Systems:** L1 (Indexed Job, `completions = k`, no redundancy), H3 Indexed Job with
  `successPolicy`, G1a Kueue partial admission, G2 Volcano, U1.
- **Metrics:** time to quorum; wasted core-seconds on pieces outside the k.

## W4. Spark with straggler mitigation

- **Shape:** all of n with first-wins duplicates. Every partition is required, a duplicate
  can race a straggler, and stages form a DAG.
- **Signal:** progress only (fraction of input processed, time against the stage median).
- **Setup:** TeraSort or word count on Kubernetes with delayed tasks, varying their number
  and severity. Speculation is mostly engine-level; the scheduler acts on executors and
  headroom for duplicates.
- **Systems:** L1 (speculation off), H4 Spark speculation, G1 Kueue, G2 Volcano.
- **Metrics:** completion time / no-straggler completion time; resources used by duplicates.

## W5. RL rollouts with DAPO group filtering

- **Shape:** quota of groups. Prompt groups of G samples fill a batch behind a training-step
  barrier, and a group whose rewards are all equal is worth nothing.
- **Signal:** finished siblings' rewards, pass rate per prompt, length against the cap.
- **Setup:** verl with `filter_groups` on KubeRay, with a small model. The scheduler acts on
  rollout worker pods and whole steps. Discarding long trajectories can bias training, so
  the application picks which rollouts to drop and the scheduler sets how many each group
  gets.
- **Systems:** L1 (verl as published, default scheduler), G1 Kueue, G2 Volcano.
- **Metrics:** time per training step; informative groups per GPU-second.

## W6. Elastic distributed training

- **Shape:** elastic gang. All members live together, and the size changes within
  $[\min, \max]$ at a restart.
- **Signal:** throughput, concave in the number of workers.
- **Setup:** PyTorch DDP with `torchrun` elastic, training a small model. Each rescale costs
  a restart.
- **Systems:** L1 (fixed size), C1 AdaptDL, G1b Kueue elastic, G2 Volcano.
- **Metrics:** time to target loss; GPU-hours; number and cost of rescales.

## W7. Population-based training

- **Shape:** fixed population with replace and pause. At each interval the worst members
  copy the best and perturb them; members are paused and resumed from checkpoints.
- **Signal:** per-member validation metric, non-stationary as hyperparameters change.
- **Setup:** Ray Tune PBT on KubeRay, training a small model.
- **Systems:** L1 (population trained to the end with no exploit step), H5 Ray Tune PBT,
  G1 Kueue, G2 Volcano.
- **Metrics:** best accuracy against wall-clock and compute; time members spend paused.

## W8. CI test matrix

- **Shape:** fail-fast all of n. All shards must pass, and the first failure cancels the
  rest.
- **Signal:** pass/fail at completion.
- **Setup:** a real test suite split into shards as an Indexed Job, with injected failures
  and flaky tests.
- **Systems:** L1 (no fail-fast, every shard runs), H6 Job `podFailurePolicy` with
  `FailJob`, G1 Kueue, G2 Volcano.
- **Metrics:** time to first failure report; core-seconds spent after the first failure;
  time to a green run.

## W9. Fuzzing campaign

- **Shape:** open-ended allocation. No completion condition; the budget should follow the
  rate of new coverage.
- **Signal:** coverage growth per core-second; crashes.
- **Setup:** libFuzzer or AFL++ on several targets, one pod per fuzzing instance.
- **Systems:** L1 (equal fixed allocation per target), G1 Kueue, G2 Volcano. No
  off-the-shelf allocator runs on Kubernetes.
- **Metrics:** total coverage and crashes found per core-hour.

## W10. Shared cluster (headline)

- **What:** jobs from W1-W9 arriving per the Philly trace ([papers.md](../papers.md)),
  scaled down to the cluster. Only arrivals and job sizes come from the trace; group
  structure comes from the workloads. Start with W1 and W2 and add the others as they are
  built.
- **Squeeze variant:** one W1 search fills the cluster, a W2 job arrives and later leaves.
  Measures time to yield, time to reclaim and the search's quality loss.
- **Systems:** G1 Kueue with preemption, G2 Volcano. Each job's app-specific mechanism
  (ASHA, Spark speculation, PBT) runs inside it.
- **Metrics:** total utility per the chosen objective
  ([optimisation-objective.md](../research-exploration/optimisation-objective.md)); per-job
  completion time; fairness (Jain's index).

---

## Policy code size

For each workload, count the lines of scheduling-policy code in the application (IROP
Table 2 method). Compare the off-the-shelf version (ASHA/Hyperband/PBT configs, Job specs,
Spark settings, verl `filter_groups`) with the same workload written against our interface.
