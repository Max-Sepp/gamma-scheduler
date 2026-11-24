# Reading list

The ten papers to read first for the Kubernetes value-aware scheduler, an extension of the
MIT IROP report (Select trees with opaque scores and value gradients). The scope is HPO and
Monte Carlo jobs sharing one cluster, so the list covers app-informed scheduling, the HPO
baselines and the one trace used for the multi-tenant run.

**Status key:** `Unread` -> `Reading` -> `Read`

**Role key:** **Baseline** = run as an evaluation arm; **Position** = closest related work;
the paper must say how we differ; **Cite** = background.

> Bibliographic details were written from memory except where linked. Check authors, venue
> and year before citing.

---

## App-informed scheduling

### 1. TetriSched
- **Status:** Unread
- **Title:** TetriSched: Global Rescheduling with Adaptive Plan-ahead in Dynamic Heterogeneous Clusters
- **Authors:** Alexey Tumanov, Timothy Zhu, Jun Woo Park, Michael A. Kozuch, Mor Harchol-Balter, Gregory R. Ganger
- **Venue:** EuroSys 2016
- **Summary:** Jobs declare space-time utility functions in STRL, which has an n-choose-k
  (`nCk`) primitive with MAX, MIN and SUM combinators. The scheduler compiles them into a
  MILP and replans as the cluster changes.
- **Relevance:** The closest related work. `nCk` is almost the same as `Select(k, ...)`. The
  difference: STRL utility is declared once at submission; we report live scores during the
  run.
- **Role:** Position

### 2. SLAQ
- **Status:** Unread
- **Title:** SLAQ: Quality-Driven Scheduling for Distributed Machine Learning
- **Authors:** Haoyu Zhang, Logan Stafman, Andrew Or, Michael J. Freedman
- **Venue:** ACM SoCC 2017
- **Summary:** Gives resources to the ML jobs whose loss is improving fastest per unit of
  resource.
- **Relevance:** Close to the value-gradient idea (report section 3.4). We must explain how a
  general score interface differs from SLAQ's ML-specific loss signal.
- **Role:** Position

### 3. Pollux
- **Status:** Unread
- **Title:** Pollux: Co-adaptive Cluster Scheduling for Goodput-Optimized Deep Learning
- **Authors:** Aurick Qiao, Sang Keun Choe, Suhas Jayaram Subramanya, Willie Neiswanger, Qirong Ho, Hao Zhang, Gregory R. Ganger, Eric P. Xing
- **Venue:** OSDI 2021
- **Summary:** Each training job reports a goodput model (throughput $\times$ statistical
  efficiency). The scheduler reallocates GPUs across jobs and tunes batch size inside them.
- **Relevance:** The main example of an app-reported signal driving a cluster scheduler.
- **Role:** Position

### 4. Optimus
- **Status:** Unread
- **Title:** Optimus: An Efficient Dynamic Resource Scheduler for Deep Learning Clusters
- **Authors:** Yanghua Peng, Yixin Bao, Yangrui Chen, Chuan Wu, Chuanxiong Guo
- **Venue:** EuroSys 2018
- **Summary:** Fits each job's convergence curve online and allocates resources to minimise
  completion time.
- **Relevance:** Full-curve prediction, which report section 3.4 rejects in favour of local
  signals. The contrast for [value-reporting.md](research-exploration/value-reporting.md).
- **Role:** Position

### 5. Themis
- **Status:** Unread
- **Title:** Themis: Fair and Efficient GPU Cluster Scheduling
- **Authors:** Kshiteej Mahajan, Arjun Balasubramanian, Arjun Singhvi, Shivaram Venkataraman, Aditya Akella, Amar Phanishayee, Shuchi Chawla
- **Venue:** NSDI 2020
- **Summary:** Finish-time fairness through an auction in which applications bid for
  resources.
- **Relevance:** Sharing between tenants without rewarding inflated scores. The reference
  for the cross-tenant part of
  [optimisation-objective.md](research-exploration/optimisation-objective.md).
- **Role:** Position

### 6. GRASS
- **Status:** Unread
- **Title:** GRASS: Trimming Stragglers in Approximation Analytics
- **Authors:** Ganesh Ananthanarayanan, Michael Chien-Chun Hung, Xiaoqi Ren, Ion Stoica, Adam Wierman, Minlan Yu
- **Venue:** NSDI 2014
- **Summary:** Speculation for approximate jobs that need only a subset of tasks, under a
  deadline or an error bound.
- **Relevance:** Subset completion under an error bound, which is the Monte Carlo
  workload's shape.
- **Role:** Position

---

## Hyperparameter optimisation

### 7. ASHA
- **Status:** Unread
- **Title:** A System for Massively Parallel Hyperparameter Tuning
- **Authors:** Liam Li, Kevin Jamieson, Afshin Rostamizadeh, Ekaterina Gonina, Jonathan Ben-Tzur, Moritz Hardt, Benjamin Recht, Ameet Talwalkar
- **Venue:** MLSys 2020 ([arXiv 1810.05934](https://arxiv.org/abs/1810.05934))
- **Summary:** Asynchronous Successive Halving. Promotes the top $1/\eta$ of each rung without
  waiting for the whole rung.
- **Relevance:** The main HPO baseline (H1). Already cited as [4] in the IROP report.
- **Role:** Baseline

### 8. Hyperband
- **Status:** Unread
- **Title:** Hyperband: A Novel Bandit-Based Approach to Hyperparameter Optimization
- **Authors:** Lisha Li, Kevin Jamieson, Giulia DeSalvo, Afshin Rostamizadeh, Ameet Talwalkar
- **Venue:** JMLR 2018
- **Summary:** Several successive-halving brackets with different trade-offs between number
  of configurations and budget. Rungs are synchronous.
- **Relevance:** The second HPO baseline (H2). Its barriers are where contention hurts.
- **Role:** Baseline

### 9. HyperSched
- **Status:** Unread
- **Title:** HyperSched: Dynamic Resource Reallocation for Model Development on a Deadline
- **Authors:** Richard Liaw, Romil Bhardwaj, Lisa Dunlap, Yitian Zou, Joseph E. Gonzalez, Ion Stoica, Alexey Tumanov
- **Venue:** ACM SoCC 2019
- **Summary:** Moves resources to promising trials so the best model is found by a deadline.
- **Relevance:** Resource-aware HPO, the closest work to W1. It works inside one search; we
  trade resources across tenants.
- **Role:** Position

---

## Trace

### 10. Philly trace
- **Status:** Unread
- **Title:** Analysis of Large-Scale Multi-Tenant GPU Clusters for DNN Training Workloads
- **Authors:** Myeongjae Jeon, Shivaram Venkataraman, Amar Phanishayee, Junjie Qian, Wencong Xiao, Fan Yang
- **Venue:** USENIX ATC 2019
- **Summary:** Characterises Microsoft's Philly cluster and releases the trace.
- **Relevance:** Arrival times and job sizes for W10.
- **Role:** Cite

---

## Systems docs to read

- **Kueue:** [elastic workloads](https://kueue.sigs.k8s.io/docs/concepts/elastic_workload/),
  partial admission (KEP-420), preemption. Baseline.
- **Volcano:** PodGroup `minMember`, gang scheduling, preemption. Baseline.
- **Ray Tune on KubeRay:** `ASHAScheduler`, `HyperBandScheduler`. Baseline.
- **Kubernetes scheduler framework:** plugin extension points (Score, Permit, PostFilter).
  Implementation.
- **Kubernetes Workload API:**
  [v1.35](https://www.kubernetes.io/blog/2025/12/29/kubernetes-v1-35-introducing-workload-aware-scheduling/) /
  [v1.37](https://kubernetes.io/blog/2026/09/08/kubernetes-v1-37-advancing-workload-aware-scheduling/),
  mutable `minCount`. Position.
