# gamma-scheduler

A value-aware scheduler for Kubernetes. The project extends the MIT IROP report
([notes/MIT_IROP_Report.pdf](notes/MIT_IROP_Report.pdf)), which built `Select(k, ...)` trees
with app-reported scores and gradients on SigmaOS, and carries the idea over to Kubernetes.

Many batch and ML workloads launch n parallel computations but only need some of them, or
only the best of them. The cluster knows how contended it is but not which pods matter. The
application knows which pods matter but not how contended the cluster is. The project asks
how an application can report its grouping intent and the live value of each computation, so
the cluster can decide which pods to run, stop, pause or duplicate.

## Documentation

Design and research notes are in [notes/](notes/). Start with
[notes/problem.md](notes/problem.md).
