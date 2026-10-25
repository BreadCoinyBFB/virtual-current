Virtual Current — State Machine Specification

Project: TXP Virtual Current
Document: "docs/state-machine.md"
Specification version: 0.1-draft
Status: Proposed; not implemented or validated

1. Purpose

The Virtual Current state machine defines how a workflow task progresses through execution, validation, recovery, and termination.

Its goals are to:

- Preserve validated results whenever they remain valid.
- Detect unsuccessful or invalid execution outcomes.
- Select recovery actions within predefined safety and resource limits.
- Prevent dependent tasks from consuming invalid results.
- Track uncertain external operations without assuming they succeeded or failed.
- Produce deterministic, auditable state transitions where the underlying events and policies permit it.

Virtual Current is a conceptual execution-control architecture. This document does not claim that its mechanisms are novel or that performance benefits have been demonstrated.

2. State definitions

State| Definition
"PENDING"| The task has unresolved dependencies or is awaiting prerequisite completion.
"READY"| All required dependencies are satisfied and the task is eligible for scheduling.
"RUNNING"| The task is executing.
"VALIDATING"| The task's output, execution status, and relevant dependency assumptions are being checked.
"SUCCEEDED"| The task produced a result that passed the required validation policy.
"RECOVERING"| A permitted recovery action is in progress.
"FAILED"| The task cannot continue successfully under the applicable recovery policy.
"STOPPED"| The task has been deliberately cancelled or halted.
"UNKNOWN"| The outcome of an operation cannot safely be determined yet.

"SUCCEEDED", "FAILED", and "STOPPED" are terminal task states for a particular execution attempt or workflow version. A later invalidation or explicit rerun creates a new task execution record or workflow revision; it must not silently rewrite history.

"UNKNOWN" is a nonterminal uncertainty state. It requires reconciliation, an explicit safe decision, or an authorized termination policy.

3. Normal forward execution

The normal task lifecycle is:

"PENDING → READY → RUNNING → VALIDATING → SUCCEEDED"

Rules:

1. A task becomes "READY" only when every required dependency has a valid result and the task's resource requirements can be considered for scheduling.
2. A scheduler may move a "READY" task to "RUNNING" only when required resources and execution permissions are available.
3. Successful completion of the task's code does not, by itself, establish that its output is correct.
4. The task enters "VALIDATING" before its result is accepted.
5. The task becomes "SUCCEEDED" only after the configured validation checks pass.
6. Dependent tasks may proceed only when the dependency results they require are valid.

4. Backward validation

Backward validation examines execution outcomes and relevant workflow history to determine whether results remain valid and whether recovery is required.

It may evaluate:

- Task completion status and errors.
- Output validation results.
- Dependency versions and validity.
- Resource failures and execution limits.
- Checkpoint and recovery-journal consistency.
- External-operation status where applicable.

Backward validation is a logical workflow operation. It does not mean reversing arbitrary machine instructions, restoring arbitrary process memory, or automatically undoing external side effects.

Validation may happen at task completion, workflow milestones, or immediately after meaningful failures or safety events.

5. Negative outcomes and recovery

A task may enter recovery when execution fails, validation rejects its result, or a recoverable resource or dependency problem is detected.

The general path is:

"RUNNING or VALIDATING → RECOVERING → READY or RUNNING"

The exact destination depends on the recovery strategy. For example, the system may retry a task, restore a valid checkpoint, adjust permitted resource allocation, or rerun affected work.

Recovery rules:

1. Identify the failure or invalidation cause.
2. Determine which results and dependent tasks are affected.
3. Preserve results whose validity can still be established.
4. Select only a recovery strategy allowed by safety policy.
5. Check retry count, elapsed time, and resource budgets.
6. Execute the recovery action and validate its outcome.
7. Resume eligible work only after the relevant results and dependencies pass validation.

If no safe recovery strategy remains, the task transitions to "FAILED". If an authorized cancellation or safety policy requires termination, it transitions to "STOPPED".

Exhausting a task's recovery budget does not automatically require terminating unrelated tasks. However, a workflow-wide safety failure may require pausing or stopping the entire workflow.

6. Unknown outcomes and external side effects

An operation enters "UNKNOWN" when its outcome cannot be established reliably.

Examples include a remote request that times out after submission or a process crash after an external service may have accepted a command.

In "UNKNOWN":

1. Do not assume the operation succeeded.
2. Do not assume the operation failed.
3. Do not retry a potentially non-idempotent operation blindly.
4. Reconcile using available operation identifiers, durable records, or external-service status.
5. Retry only when policy and available evidence establish that doing so is safe.
6. If reconciliation cannot resolve the outcome, retain the uncertainty or stop according to the configured policy.

Internal rollback cannot guarantee reversal of an external action. Compensating actions are possible only when the external system supports them and the compensation is authorized.

7. Dependency invalidation

If a result becomes invalid, Virtual Current identifies downstream tasks that depend on the invalid result.

Only affected results are invalidated or recomputed, provided their continued validity can be established. Independent, still-valid work should be preserved where safe.

A task that is invalidated after previously succeeding must be represented through a new execution record or workflow revision. The history must retain the fact that its earlier execution succeeded under its earlier assumptions.

8. Resource and scheduling constraints

Virtual Current may schedule independent "READY" tasks concurrently, subject to:

- CPU, memory, and other resource limits.
- Dependency ordering.
- Task priorities and safety requirements.
- Recovery budgets.
- Isolation and authorization policies.

Adaptive scheduling must remain within hard safety limits. Resource adaptation cannot create physical compute capacity or justify exceeding those limits.

An optional 50 Hz control target corresponds to a 20 ms interval. It is a scheduling or reassessment target, not a guarantee that every workflow task or recovery cycle completes within 20 ms.

9. Core invariants

An implementation must preserve these invariants:

1. Validated-result invariant: No dependent task may consume a result that is invalid or whose required validity has not been established.
2. Safety invariant: Adaptive policy must not override hard safety, authorization, or resource constraints.
3. Bounded-recovery invariant: Every recovery attempt is subject to configured attempt, time, and resource budgets.
4. Uncertainty invariant: An unresolved external outcome must not be silently classified as success or failure.
5. History invariant: State transitions, recovery decisions, and checkpoint references must be recorded according to the durability policy.
6. Selective-recovery invariant: Unaffected work may be preserved only when its validity can be established.
7. Validation invariant: A retry or recovery action does not count as successful merely because the action itself completed.
8. Termination invariant: When the applicable recovery budget is exhausted, the system follows an explicit stop or failure policy rather than retrying indefinitely.

10. Initial acceptance scenarios

A future deterministic simulator should test at least these scenarios:

- Normal execution: a task completes and passes validation.
- Execution failure: a task fails and enters bounded recovery.
- Validation failure: code exits successfully, but its output is rejected.
- Dependency invalidation: a failed upstream task causes affected downstream results to be reconsidered.
- Resource exhaustion: the scheduler reduces or reallocates permitted work without exceeding resource limits.
- External uncertainty: a timed-out remote operation is reconciled before a potentially unsafe retry.
- Recovery exhaustion: the affected task stops after its configured budget is consumed.
- Independent work: a recoverable failure does not unnecessarily discard unrelated valid results.
- Restart: persisted state and journal records are reconciled after a simulated crash.

11. Open specification questions

Before implementation, the project must define:

- The precise event and transition schema.
- How validation policies are configured and versioned.
- The exact recovery-strategy selection algorithm.
- How task cancellation interacts with running descendants.
- How checkpoint and journal durability guarantees are specified.
- How concurrent events are ordered and conflicting transitions resolved.
- Which external operations can be retried safely.
- How the state machine is tested against a simple baseline.

12. Implementation status

This document describes proposed behavior only. It is not evidence of a working runtime, benchmark results, formal verification, or a claim of technological novelty.

The next milestone is to turn these states and rules into an explicit transition table, then validate the rules against failure scenarios before implementing a deterministic simulator.
