# DrainLab

DrainLab is a maintenance-aware Kubernetes/OpenShift rescheduling benchmark. It compares multiple planning algorithms that decide how to evacuate workloads from nodes scheduled for maintenance while preserving availability and using cluster capacity efficiently.

The project is designed as an offline, explainable planning and experimentation tool. It will use [KWOK](https://kwok.sigs.k8s.io/) to create reproducible simulated clusters without requiring a large physical Kubernetes or OpenShift environment.

> **Status:** design and initial implementation phase.

## The problem

Before maintaining, replacing, or upgrading a node, an operator normally cordons and drains it. Kubernetes then schedules replacement pods using its regular pod-by-pod scheduling process. That process is reliable and fast, but it does not create a global maintenance plan for all affected workloads in advance.

DrainLab asks a broader question:

> Given a set of nodes that must become unavailable before a deadline, what sequence of workload movements will complete the maintenance safely with the least disruption and additional capacity?

A useful plan must account for more than whether each pod eventually fits. It must also respect availability during every step, wait for replacements when necessary, limit concurrent evictions, and explain when the maintenance request is infeasible.

## Goals

- Model node-maintenance rescheduling as a reproducible optimization problem.
- Compare native scheduling, greedy placement, constraint optimization, and local search under identical scenarios.
- Produce both a target placement and an ordered, availability-aware execution plan.
- Explain constraint violations, trade-offs, and infeasible plans.
- Benchmark solution quality and runtime as cluster size and utilization increase.
- Provide a Quarkus API and a small Vue dashboard for creating scenarios and comparing results.
- Keep execution dry-run and simulation-first until plan validation is mature.

## Non-goals

- Replacing `kube-scheduler`, Karpenter, Karmada, or the Kubernetes Descheduler.
- Claiming complete OpenShift behavior from KWOK simulations.
- Moving live production workloads during the initial project stages.
- Supporting every Kubernetes storage and scheduling feature in the first version.
- Using private employer manifests, metrics, or infrastructure data.

## Algorithms

Every planner will implement the same interface and receive the same immutable cluster snapshot and maintenance request.

| Planner | Role | Expected trade-off |
| --- | --- | --- |
| Native drain simulation | Operational baseline using sequential cordon/drain behavior and normal scheduling | Representative behavior, limited global look-ahead |
| First-Fit Decreasing | Greedy vector-bin-packing baseline | Very fast, potentially fragmented placements |
| OR-Tools CP-SAT | Bounded global constraint optimizer | Strong solution quality, increasing runtime at scale |
| Local search | Custom improvement heuristic over an initial placement | Adjustable balance between runtime and quality |

Possible later experiments include best-fit variants, tabu search, simulated annealing, and hybrid approaches that warm-start CP-SAT with a greedy or local-search solution.

## Initial constraint model

### Hard constraints

- CPU and memory requests must fit on the destination node.
- Nodes in maintenance cannot receive workloads.
- Node selectors and required node affinity must be satisfied.
- Taints and tolerations must be respected.
- Required pod anti-affinity and topology rules must remain valid.
- PodDisruptionBudgets must remain satisfied at every execution step.
- Concurrent evictions must stay within configured limits.
- Excluded workloads must never be moved by the planner.

### Soft constraints and objectives

DrainLab will use lexicographic priorities where possible instead of hiding operational trade-offs inside one unexplained weighted score:

1. Find a feasible plan.
2. Preserve workload availability throughout execution.
3. Minimize additional nodes or capacity required.
4. Minimize weighted workload disruption and pod movements.
5. Minimize resource fragmentation and peak node utilization.
6. Complete the plan before the maintenance deadline.

Weights will still be supported for experiments within an objective level, such as assigning a higher movement cost to stateful or high-priority workloads.

## Plan output

A planner returns more than a final pod-to-node assignment. It returns an ordered plan with preconditions and explanations.

```text
Step 1: cordon node-03
Step 2: create replacement for api-2 on node-08
Step 3: wait until api-2 is Ready
Step 4: evict the old api-2 pod from node-03
Step 5: move worker-4 from node-03 to node-05
Step 6: verify disruption budgets and mark node-03 maintenance-ready
```

Each result will include:

- feasibility status and reasons;
- ordered actions and validation conditions;
- final workload placement;
- objective breakdown;
- runtime and solver status;
- optimality bound or gap when the planner can provide one;
- warnings for unsupported or approximated Kubernetes behavior.

## Architecture

```text
Scenario generator ──> KWOK cluster
                           │
                           v
                    Snapshot collector
                           │
                           v
                      Planner API
                 ┌─────────┼──────────┐
                 │         │          │
              Native     Greedy    CP-SAT    Local search
                 │         │          │           │
                 └─────────┴──────────┴───────────┘
                           │
                           v
                   Plan validator/executor
                           │
                    dry-run by default
                           │
                           v
                  Results and Vue dashboard
```

The optimizer core will remain independent of Kubernetes clients so algorithms can be unit-tested with plain in-memory models. Kubernetes integration will be implemented at the snapshot and plan-execution boundaries.

## Proposed technology stack

- Java 21
- Quarkus and Maven
- Fabric8 Kubernetes Client
- Google OR-Tools CP-SAT Java API
- Vue 3 and TypeScript
- KWOK for Kubernetes API and workload simulation
- JUnit for model, constraint, and planner tests
- JMH or a dedicated benchmark runner for algorithm-level measurements
- Docker/Podman and GitHub Actions for reproducible builds

## Benchmark design

DrainLab will generate deterministic scenarios from versioned configurations and random seeds.

### Scenario dimensions

| Dimension | Initial values |
| --- | --- |
| Nodes | 10, 100, 500, 1,000 |
| Cluster utilization | 50%, 70%, 85%, 95% |
| Node types | homogeneous and heterogeneous |
| Nodes under maintenance | 5%, 10%, 20% |
| Workload importance | uniform and weighted |
| Disruption policy | permissive, moderate, strict |
| Affinity complexity | none, sparse, dense |

### Metrics

- feasible-plan rate;
- planning wall-clock time;
- number and weighted cost of pod movements;
- additional nodes or capacity required;
- maintenance completion steps and estimated duration;
- average, maximum, and variance of node utilization;
- unused CPU and memory fragmentation;
- objective value and optimality gap;
- plan-validation failures;
- successful realization by the simulated scheduler.

Results will be emitted in a machine-readable format so experiments can be repeated and charts regenerated.

## Research questions

1. When does global CP-SAT planning materially improve on native drain behavior or greedy bin packing?
2. How quickly does CP-SAT runtime grow with cluster size, utilization, and affinity density?
3. Can local search approach CP-SAT solution quality within a much smaller time budget?
4. How much additional capacity is required to preserve strict disruption budgets during maintenance?
5. Which constraints most frequently make maintenance plans infeasible?
6. How does optimizing the movement sequence differ from optimizing only the final placement?

## Validation strategy

Validation will happen at three levels:

1. **Model validation:** small hand-checkable scenarios with known feasible and optimal solutions.
2. **Simulation validation:** apply plans to KWOK and verify assignments, readiness transitions, and disruption constraints.
3. **Compatibility validation:** run a smaller subset against a local Kubernetes cluster and, later, OpenShift Local/CRC.

KWOK simulates Kubernetes API resources and their lifecycle; it does not reproduce kubelet execution, real application performance, storage behavior, networking, or every OpenShift-specific controller. Results will be described as simulated Kubernetes/OpenShift-oriented experiments until separately validated.

## Safety model

- Dry-run is the default and only mode during the initial milestones.
- The planner will use synthetic or explicitly anonymized data.
- Live-cluster mutation will be isolated behind a separate adapter and explicit configuration if it is added later.
- System-critical pods, DaemonSets, unmanaged pods, and unsupported persistent-volume configurations will initially be excluded.
- Every executable plan must pass an independent validator rather than trusting the planner that produced it.

## Roadmap

### Milestone 1 — Domain model and deterministic scenarios

- Define nodes, workloads, resources, policies, maintenance requests, and plans.
- Implement JSON scenario import/export.
- Add small fixtures with known outcomes.
- Create a deterministic synthetic workload generator.

### Milestone 2 — Greedy baseline and validation

- Implement First-Fit Decreasing for CPU and memory.
- Implement final-placement validation.
- Add objective and fragmentation metrics.
- Publish the first repeatable benchmark results.

### Milestone 3 — CP-SAT planner

- Translate placement variables and constraints into OR-Tools CP-SAT.
- Support solver time limits and reproducible seeds.
- Report bounds and optimality gaps.
- Compare CP-SAT against the greedy baseline.

### Milestone 4 — Maintenance sequencing

- Model cordon, replacement, readiness, eviction, and drain-complete actions.
- Enforce PodDisruptionBudgets at every step.
- Add deadlines and maximum concurrent evictions.
- Implement the native sequential-drain baseline.

### Milestone 5 — Local search

- Start from greedy and randomized feasible placements.
- Implement move, swap, and node-emptying neighborhoods.
- Compare solution quality under equal time budgets.
- Explore warm-starting CP-SAT with heuristic solutions.

### Milestone 6 — KWOK integration

- Create repeatable KWOK clusters from benchmark scenarios.
- Collect snapshots through the Kubernetes API.
- Apply plans to simulated resources in a controlled environment.
- Validate planner predictions against observed scheduling results.

### Milestone 7 — API and visualization

- Expose scenarios, planner runs, plans, and results through Quarkus.
- Build a Vue interface for cluster state and maintenance timelines.
- Visualize placements, constraint failures, runtime, and solution quality.

### Milestone 8 — OpenShift-oriented validation

- Test representative scenarios on OpenShift Local/CRC.
- Document differences from KWOK and upstream Kubernetes.
- Evaluate OpenShift-specific constraints that should enter the model.

## Proposed repository structure

```text
drainlab/
├── planner-core/          # Domain model, objectives, validation
├── planner-greedy/        # Greedy baselines
├── planner-cpsat/         # OR-Tools implementation
├── planner-local-search/  # Custom heuristics
├── kubernetes-adapter/    # Snapshot and simulated execution adapters
├── benchmark/             # Scenario generation and experiment runner
├── api/                   # Quarkus REST API
├── ui/                    # Vue application
├── scenarios/             # Versioned example and benchmark inputs
├── docs/                  # Design notes and experiment reports
└── README.md
```

The structure may be simplified during the first milestones; modules should only be split when their boundaries become useful in code.

## Related work

- [OPSche: A Kubernetes Scheduler Plugin for Cluster-Wide Placement Optimisation](https://arxiv.org/abs/2608.06987)
- [OPSche reproducibility artifact](https://zenodo.org/records/19052813)
- [Kubernetes Scheduler Simulator](https://github.com/kubernetes-sigs/kube-scheduler-simulator)
- [Kubernetes Descheduler](https://github.com/kubernetes-sigs/descheduler)
- [Kubernetes Scheduler Plugins](https://github.com/kubernetes-sigs/scheduler-plugins)
- [KWOK](https://github.com/kubernetes-sigs/kwok)
- [Cost Minimization in Multi-cloud Systems with Runtime Microservice Re-orchestration](https://arxiv.org/abs/2401.01408)
- [Disruption-aware Microservice Re-orchestration for Cost-efficient Multi-cloud Deployments](https://arxiv.org/abs/2501.16143)
- [Solver-In-The-Loop Cluster Resource Management for Database-as-a-Service](https://www.vldb.org/pvldb/vol16/p4254-konig.pdf)
- [Firmament: Fast, Centralized Cluster Scheduling at Scale](https://www.usenix.org/conference/osdi16/technical-sessions/presentation/gog)

## License

A license will be selected before implementation code is published. Dependencies and referenced projects retain their respective licenses.
