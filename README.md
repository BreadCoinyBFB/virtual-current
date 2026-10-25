# virtual-current
TXP - Virtual Current is a Virtual version of Alternating Current and Direct Current -- READ MORE IN README.md

TXP Virtual Current (VC)

Complete Concept, Proposed Architecture, Technical Specification, and Research Status

Project: The X Project (TXP)
Technology: Virtual Current (VC)
Program: TXP Virtual Current Program
Document type: Technical concept and research proposal
Status: Conceptual specification consolidated; implementation and experimental validation have not begun.
Version: VC Research Specification 1.x
Date: October 25, 2026

---

1. Executive Summary

Virtual Current (VC) is a proposed TXP execution-control and recovery architecture designed to coordinate how software tasks execute, how their outcomes are validated, how computing resources are allocated, and how workflows recover from failures.

The central idea is to manage software execution as an evolving workflow with two conceptual phases:

- Forward phase: Execute eligible tasks, process their results, and make progress through the workflow.
- Backward phase: Validate results, inspect affected dependencies, determine whether previous results remain valid, and select appropriate recovery actions when necessary.

VC combines this execution model with resource-aware concurrency, adaptive resource allocation, checkpointing, persistent recovery records, bounded retries, and feedback-informed scheduling.

The concept takes inspiration from the distinction between alternating current (AC) and direct current (DC). However, Virtual Current is a software execution metaphor and proposed control model, not literal electrical current flowing through code.

Its proposed goal is to improve the balance between execution performance, computational efficiency, and reliability while maintaining explicit safety constraints.

Important research qualification: VC is currently a conceptual TXP research project. Its specification describes intended behavior, not an implemented product. Its performance, reliability, and novelty have not been established experimentally.

2. What Is Virtual Current?

Virtual Current is intended to govern the progression of a software workflow by coordinating task execution, validation, resource allocation, and recovery.

A conventional workflow scheduler may determine which task runs next. A recovery system may restore a checkpoint after a failure. An adaptive scheduler may change resource assignments according to observed workload.

VC proposes coordinating these responsibilities under a unified execution-cycle model.

The proposed system considers:

1. Which tasks are ready to execute.
2. Which tasks can safely run concurrently.
3. How much CPU, memory, and other resources are available.
4. Whether task outputs satisfy validation requirements.
5. Which results remain valid after a failure.
6. Which recovery action offers an acceptable balance of speed, wasted computation, and reliability.
7. Whether execution should continue, pause, recover, or stop.
8. How verified outcomes can inform future scheduling decisions.

The research objective is to determine whether a precisely defined VC control algorithm can improve upon conventional approaches for selected workloads.

3. Core Terminology

3.1 Virtual Current

The proposed execution-control architecture that coordinates forward execution, backward validation, resource-aware scheduling, and selective recovery.

3.2 Coding Charge

“Coding charge” is a conceptual TXP term describing the changing execution condition of a workflow.

For technical analysis, this concept is divided into three independently measurable components:

- Control flow: The order in which tasks and branches execute.
- Task state: The current status of each task.
- Resource allocation: The assignment of CPU time, memory, concurrency, and other available resources.

Coding charge does not represent literal electrical charge, and the term alone does not establish a new computing mechanism.

3.3 Forward Phase

The stage in which ready tasks execute, results are produced, and workflow progress occurs.

3.4 Backward Phase

The stage in which outcomes and affected dependencies are examined, results are validated, and recovery requirements are determined.

Backward processing does not mean that every CPU instruction is reversed.

3.5 Virtual Transformers

A proposed TXP term for controls that adjust the allocation and scheduling of available computing resources.

Virtual Transformers may increase or decrease task concurrency, adjust resource budgets, prioritize important work, or defer noncritical tasks.

They cannot create physical CPU capacity or RAM simply by changing a software parameter.

3.6 Workflow Cycle

A conceptual unit of progress involving forward execution, outcome assessment, and any necessary validation or recovery decisions.

The precise cycle boundary must be defined for each implementation and workload.

3.7 Recovery Budget

The limits placed on recovery attempts, elapsed recovery time, and resource consumption.

3.8 Persistent History

Stored records of workflow progress, checkpoints, recovery events, and validated adaptation statistics.

Historical information can guide future decisions but cannot override current safety requirements.

4. Proposed Architecture

The following components represent a proposed architecture, not completed software modules.

4.1 Workflow Graph

Workflows are represented as directed acyclic graphs (DAGs).

- Nodes represent tasks.
- Edges represent dependencies.
- A task becomes eligible when its required dependencies are satisfied.
- Independent eligible tasks may execute concurrently.
- A failure triggers assessment of affected downstream work.

A DAG does not directly represent arbitrary loops. Repeated operations require an explicit loop mechanism or separate workflow executions.

4.2 Execution Controller

The execution controller determines which ready tasks may run, tracks their states, and coordinates transitions between execution and validation.

4.3 Resource Manager

The resource manager monitors available resources and determines which tasks can run concurrently within configured limits.

4.4 Validation Engine

The validation engine checks task outputs, dependency validity, and relevant completion requirements.

4.5 Recovery Manager

The recovery manager chooses among permitted recovery actions, such as retrying a task, restoring a checkpoint, changing resource allocation, revalidating results, or stopping affected work.

4.6 Checkpoint and Journal System

The checkpoint system stores recoverable workflow snapshots. The durable journal records important state transitions and recovery-critical events.

4.7 Adaptation History

The history system retains approved statistics about scheduling and recovery outcomes, subject to access controls, retention policies, and reset mechanisms.

4.8 Security and Isolation Layer

The security layer enforces task permissions, isolation boundaries, resource limits, and integrity checks.

5. How a Virtual Current Cycle Works

A proposed workflow cycle follows these stages.

Stage 1: Select Ready Tasks

The controller identifies tasks whose dependencies are satisfied.

Stage 2: Allocate Resources

The resource manager evaluates CPU availability, memory pressure, concurrency limits, task importance, and other configured constraints.

Stage 3: Execute Forward

Eligible tasks execute. Independent tasks may run concurrently when resources and safety constraints permit.

Stage 4: Validate Outcomes

At workflow milestones, completion, and meaningful failure or safety events, VC checks relevant results and dependencies.

Stage 5: Assess Backward

VC examines affected work and dependencies in reverse execution order where useful. It determines which results remain valid and which require further checks or recomputation.

Stage 6: Select an Action

VC may:

- Continue with the next eligible task.
- Revalidate an existing result.
- Retry a failed task.
- Restore a compatible checkpoint.
- Change permitted resource allocations.
- Invalidate affected downstream results.
- Pause or stop when safe continuation is impossible.

Stage 7: Record and Adapt

VC records relevant outcomes and recovery events. Validated historical statistics may inform future decisions within approved limits.

Stage 8: Repeat or Finish

Execution continues until the workflow's completion criteria are met or safe continuation is no longer possible.

This model defines the intended behavior. Its exact transition rules and scheduling algorithm remain to be designed.

6. Positive and Negative Outcomes

The earlier VC concept described positive and negative execution paths.

For a rigorous implementation, these should be defined as explicit workflow outcomes rather than ambiguous electrical analogies.

A positive outcome means the relevant task or validation criteria have been satisfied. The workflow may continue when its dependencies and safety requirements permit.

A negative outcome means a task or validation criterion has failed. VC must determine whether the failure is recoverable, which results are affected, and what action is safe.

An unknown outcome is also necessary. For example, an external service may have received a request but become unreachable before confirming whether it completed. VC must reconcile that state before blindly retrying an operation that could have side effects.

7. Resource-Aware Concurrency

VC adopts resource-aware concurrency rather than sequential execution or unrestricted parallelism.

Suppose a workflow has four ready tasks, but current memory limits permit only two to run safely. VC may start two tasks, retain the others in a queue, and reevaluate available resources as tasks finish.

The scheduler considers:

- CPU utilization and availability.
- Memory usage and estimated demand.
- Current concurrency.
- Task importance.
- Dependency constraints.
- Failure and recovery conditions.
- Configured resource limits.

The purpose is not to maximize concurrency at all costs. It is to find useful parallelism without violating safety constraints or causing avoidable resource exhaustion.

8. Virtual Transformers and Hybrid Resource Allocation

Virtual Transformers are proposed as the resource-control part of VC.

The selected strategy is hybrid allocation:

1. Fixed safety limits: Establish maximum permitted resource consumption and other non-negotiable restrictions.
2. Adaptive allocation: Adjust permitted resource assignments in response to workload and resource pressure.
3. Task-aware prioritization: Protect resources needed by critical tasks where possible.
4. Recovery-aware adjustment: Reassess resource allocations after failures and during recovery.
5. Independent safety enforcement: Ensure that adaptive optimization cannot override safety checks.

For example, VC could reduce the number of concurrent tasks when memory pressure becomes excessive, then increase concurrency when resources become available.

This is resource management, not the creation of additional physical processing power.

9. Timing and the 50 Hz Proposal

TXP has proposed an optional 50 Hz target for some VC applications.

The corresponding period is:

T = 1 / f = 1 / 50 = 0.02 seconds = 20 milliseconds.

A 50 Hz target therefore means one defined cycle every 20 milliseconds.

However, this target does not guarantee that every compilation, workflow, validation, or recovery operation can finish within that period.

The selected timing policy is adaptive timing with an optional 50 Hz target.

When a cycle exceeds its target, VC should:

1. Finish work safely where possible.
2. Record the overrun.
3. Defer noncritical work when appropriate.
4. Preserve correctness and safety rather than forcing an incomplete operation to appear successful.

A future implementation must define exactly what counts as a cycle and measure deadline misses. The frequency target itself does not make computation faster.

10. Recovery Policy

VC's recovery strategy balances three objectives:

- Recovery speed.
- Computational efficiency and minimized wasted work.
- Reliability and correctness.

The proposed optimizer considers the failure type, available resources, and task importance.

Fixed Safety Rules

Fixed rules enforce correctness, dependency integrity, resource limits, and safe stopping conditions.

Adaptive Recovery

Within those rules, VC may select among permitted recovery strategies according to observed conditions and validated history.

Bounded Recovery

Recovery uses three combined budgets:

- Maximum attempts.
- Maximum elapsed recovery time.
- Maximum permitted resource consumption.

When the budgets are exhausted, VC stops the affected task and preserves results that remain valid. It continues independent work only when that work is demonstrably safe.

A system-wide safety issue may require pausing or stopping the entire workflow.

11. Selective Invalidation and Resumption

A task failure does not automatically mean every previous result must be discarded.

VC proposes selective invalidation:

1. Identify the failed task.
2. Determine which downstream results depend on affected inputs or outputs.
3. Preserve results whose validity can be established.
4. Revalidate results whose validity is uncertain.
5. Recompute results that are no longer valid.
6. Resume only after required validation succeeds.

The workflow DAG helps identify affected dependencies, but graph connectivity alone does not establish whether a result is still valid.

12. External Side Effects

Internal rollback cannot automatically undo an external operation.

Examples include sending a payment request, submitting a remote job, modifying a remote file, or sending a message.

VC therefore proposes:

- Operation records.
- Idempotency protections where supported.
- Compensating actions where supported.
- Explicit tracking of completed, failed, attempted, and uncertain operations.
- Reconciliation with external systems before potentially unsafe retries.

Exactly-once execution cannot be guaranteed merely by writing a durable journal. It may require cooperation from the external system, stable operation identifiers, and appropriate idempotency or reconciliation mechanisms.

13. Checkpoints and Persistent State

VC proposes two complementary forms of persistence.

Durable Journal

The journal records recovery-critical state transitions and events.

Workflow Checkpoints

Checkpoints capture snapshots of workflow state, including:

- Task statuses.
- Dependency information.
- Validated outputs or references.
- Recovery metadata.
- Workflow version and checkpoint identity.
- Journal position.

Checkpoints may be periodic or triggered by important events, such as expensive tasks, risky operations, or workflow milestones.

The design must account for checkpoint overhead, storage costs, crash consistency, and compatibility with the workflow version.

The proposed checkpoint format does not automatically preserve arbitrary process memory, open connections, or external side effects. Those require separate mechanisms.

14. Learning From Previous Runs

VC proposes both short-term and long-term adaptation.

Short-term adaptation uses observations from the current workflow and recent resource conditions.

Long-term adaptation uses validated statistics retained across workflows.

Recent observations may be more relevant when conditions change. Long-term statistics may reveal recurring patterns.

However, old or unreliable observations must not be trusted automatically. The system needs rules for weighting evidence, discarding stale information, resetting history, and avoiding overreaction to isolated failures.

Historical statistics inform decisions; they do not prove that a recovery action is safe in a new situation.

15. Proposed Optimization Objective

A candidate mathematical objective is:

J(a) = w_t T(a) + w_w W(a) + w_r R(a)

where:

- a represents a candidate scheduling or recovery action.
- T(a) represents expected recovery time.
- W(a) represents expected wasted computation.
- R(a) represents residual reliability risk.
- w_t, w_w, and w_r are configurable weights.

The system would seek to minimize this objective subject to hard safety constraints.

TXP would define allowed weight ranges, while the adaptive controller could adjust weights within those ranges. A separate safety mechanism would enforce non-negotiable requirements.

This is a proposed optimization framework, not a validated formula. Its estimators, weights, constraints, and action-selection algorithm remain unresolved.

16. Security and Isolation

VC proposes risk-based isolation.

Trusted, low-risk tasks may use lighter isolation where appropriate. Untrusted or high-risk tasks should receive stronger sandboxing and tighter permissions.

All modes remain subject to safety and resource limits.

The design should address:

- Filesystem and network permissions.
- Task-to-task isolation.
- CPU and memory limits.
- Checkpoint integrity.
- Journal integrity.
- Access controls for persistent history.
- Retention, expiration, and reset.
- Recovery from corrupted or incompatible state.

A task must not gain additional permissions simply because it is part of a recovery operation.

17. Relationship to the TXP Virtual Current Program

Virtual Current belongs to the broader TXP Virtual Current Program, which includes other proposed technologies such as CloudRAM or CloudR, Cloud Storage, EmulatorZone, Virtual CPU, A++, Right-On-Time (ROT), and 6G Ethernet.

Those are separate concepts and projects. VC's proposed workflow controller should not be assumed to implement them or depend on them automatically.

Potential future integrations may be researched, but each requires its own architecture, interfaces, resource assumptions, and validation.

Virtual Current is also distinct from the proposed A++ programming language and ROT compilation model. A compiler or runtime could potentially use VC-like workflow coordination, but that is a possible integration rather than an established capability.

18. Existing Research and Prior Art

Many individual components of VC overlap with established computer science research.

Adaptive Workflow Scheduling

The Pegasus workflow system has published work on revising resource allocations during execution based on runtime job-performance feedback.

Reference: Lee et al., Adaptive workflow processing and execution in Pegasus (2009).

https://ntu-repository.worktribe.com/output/460808

Checkpointing and Recovery

Research on computational workflow scheduling has studied task ordering, checkpoint placement, rollback, and recovery under failure-prone execution environments.

Reference: Aupy, Benoit, Casanova, and Robert, Checkpointing Strategies for Scheduling Computational Workflows (2016).

https://www.jstage.jst.go.jp/article/ijnc/6/1/6_2/_article

Uncertainty-Aware Workflow Management

Research published in 2025 proposed a checkpointing mechanism integrated with an uncertainty-aware workflow scheduling algorithm for unreliable or preemptible cloud resources.

Reference: Oikonomou, Kolomvatsos, and Anagnostopoulos, A proactive and uncertainty driven management mechanism for workflows of processing tasks (2025).

https://link.springer.com/article/10.1007/s00607-025-01468-1

Adaptive and Fault-Tolerant Scheduling

Research published in October 2026 describes an adaptive Spark workflow scheduler combining task-interaction graphs, resource provisioning, monitoring, and fault-tolerant micro-checkpointing.

Reference: Raj et al., A meta Proximal policy optimization driven task interaction graph scheduler with predictive resource provisioning and fault-tolerant micro-checkpointing for adaptive spark workflow scheduling in multi-cloud systems (2026).

https://www.sciencedirect.com/science/article/abs/pii/S0952197626022529

These examples demonstrate that adaptive scheduling, dependency graphs, checkpointing, and recovery are not individually new concepts.

The cited research does not establish that every possible implementation of Virtual Current already exists. It doe
