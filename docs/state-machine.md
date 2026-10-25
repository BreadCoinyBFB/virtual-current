Virtual Current — State Machine Specification

Project: TXP Virtual Current
File: "docs/state-machine.md"
Specification version: 0.1-draft
Status: Proposed; not implemented or validated

1. Purpose

The Virtual Current state machine defines how workflow tasks progress through forward execution, backward validation, recovery, and termination.

Its objectives are to preserve valid results, detect execution failures, select permitted recovery strategies, prevent invalid dependency use, and maintain an auditable execution history.

Virtual Current is a proposed execution-control and recovery architecture. This specification does not establish technological novelty or demonstrate performance improvements.

2. State definitions

State| Definition
"PENDING"| Waiting for required dependencies to become valid.
"READY"| Dependencies are satisfied; the task is eligible for scheduling.
"RUNNING"| The task is executing or resuming under an authorized strategy.
"VALIDATING"| The task's outcome, output, and relevant dependencies are being checked.
"SUCCEEDED"| The task's result passed the required validation policy.
"RECOVERING"| Failure analysis, recovery selection, or a recovery operation is in progress.
"FAILED"| The task cannot continue successfully under the applicable recovery policy.
"STOPPED"| The task was deliberately cancelled or halted.
"UNKNOWN"| An operation's outcome cannot yet be established reliably.

"SUCCEEDED", "FAILED", and "STOPPED" are terminal for a particular task execution record. Later invalidation or reruns must be represented by a new execution record or workflow revision.

"UNKNOWN" is nonterminal. It requires reconciliation, a permitted wait, or an explicit termination decision.

3. Permitted state transitions

Current state| Trigger| Next state| Required behavior
"PENDING"| All required dependencies become valid| "READY"| Confirm dependency validity.
"READY"| Required resources and permissions are available| "RUNNING"| Start an execution attempt and record its identifier.
"RUNNING"| Execution ends and an outcome is available| "VALIDATING"| Inspect the result and execution evidence.
"RUNNING"| Recoverable failure occurs| "RECOVERING"| Record the failure and evaluate permitted recovery.
"RUNNING"| Outcome is uncertain| "UNKNOWN"| Reconcile before any potentially unsafe retry.
"VALIDATING"| Required validation passes| "SUCCEEDED"| Accept the result and make it available to eligible dependents.
"VALIDATING"| Validation fails and recovery may be possible| "RECOVERING"| Identify the cause and affected dependencies.
"VALIDATING"| An external operation's outcome is unresolved| "UNKNOWN"| Reconcile the operation.
"RECOVERING"| A permitted strategy is selected and ready to execute| "RUNNING" or "READY"| Apply the strategy or prepare a new attempt.
"RECOVERING"| A recovery attempt fails| "RECOVERING"| Reassess the failure and remaining budgets.
"RECOVERING"| No safe strategy remains or a budget is exhausted| "FAILED"| Record the reason recovery terminated.
Any active state| Authorized cancellation or halt| "STOPPED"| Halt safely and record the action.
"UNKNOWN"| Reconciliation establishes a successful external outcome| "VALIDATING"| Verify the result before accepting it.
"UNKNOWN"| Reconciliation establishes an unsuccessful outcome| "RECOVERING"| Evaluate eligible recovery strategies.
"UNKNOWN"| Outcome remains unresolved| "UNKNOWN"| Continue permitted reconciliation or apply an explicit termination policy.

These are permitted transitions, not automatic responses to every event. The transition controller must reject transitions that violate dependency, authorization, resource, or safety constraints.

A system-wide safety failure may require pausing or stopping the whole workflow rather than handling the event as an isolated task failure.

4. Normal forward execution

The normal task lifecycle is:

"PENDING → READY → RUNNING → VALIDATING → SUCCEEDED"

Rules:

1. A task becomes "READY" only when all required dependencies are valid.
2. A scheduler starts the task only when required permissions and resources are available.
3. Successful process termination does not prove that the task's output is correct.
4. Every result must pass its configured validation policy before acceptance.
5. Dependents may use only the results and versions they require that have been established as valid.
6. Each execution attempt must be identifiable in the execution history.

5. Backward validation

Backward validation examines outcomes and workflow history to determine whether results remain valid and whether recovery is needed.

It may inspect:

- Execution status, errors, and failure evidence.
- Output validation results.
- Dependency identities, versions, and validity.
- Resource exhaustion and execution limits.
- Checkpoint and recovery-journal consistency.
- External-operation status where applicable.

Backward validation is a logical workflow operation. It does not imply arbitrary reverse execution of machine instructions or automatic reversal of external side effects.

Validation may occur at task completion, workflow milestones, and immediately after meaningful failures or safety events.

6. Recovery policy B: adaptive, bounded strategy selection

Selected policy: After each failed recovery attempt, reassess the failure and choose a different permitted strategy when justified by available evidence.

6.1 Recovery procedure

Whenever a task enters "RECOVERING", the controller must:

1. Record the triggering event and available failure evidence.
2. Classify the failure, including whether it is transient, resource-related, dependency-related, validation-related, or unrecoverable.
3. Identify invalid results and affected downstream tasks.
4. Check the remaining attempt, elapsed-time, and resource budgets.
5. Construct the set of strategies permitted by the current safety policy.
6. Exclude strategies that are unsafe, inapplicable, unauthorized, or over budget.
7. Select an eligible strategy using the configured recovery-selection policy.
8. Record the chosen strategy and its justification.
9. Execute the recovery action.
10. Validate the outcome before resuming dependent work.

6.2 Strategy changes

When a recovery attempt fails, the controller must reassess rather than blindly repeating the same strategy.

A different strategy should be selected when:

- The previous strategy failed for a reason that the alternative can plausibly address.
- The alternative is permitted by safety and authorization rules.
- Required resources remain available.
- The remaining recovery budget permits another attempt.
- The expected benefit justifies the cost and risk.

A different strategy is not mandatory when no suitable alternative exists. The controller may stop if further recovery would be unsafe, futile, or outside policy.

Repeating a previous strategy is permitted only when the failure evidence supports doing so, such as when a transient condition may have cleared and the operation is safe to repeat. Policy B does not require every strategy to be unique forever; it requires reassessment and justified selection.

6.3 Example strategies

Depending on the implementation and failure type, permitted strategies may include:

- Retry after a transient condition clears.
- Reduce concurrency or request fewer resources.
- Reallocate permitted resources.
- Restore a valid checkpoint.
- Recompute an invalid result.
- Re-run only affected downstream tasks.
- Use an alternate execution method where one exists and is authorized.

These are candidate strategies, not guaranteed capabilities of every Virtual Current implementation.

6.4 Recovery budgets

Recovery must be bounded by configurable limits:

- Attempt budget: maximum permitted recovery attempts.
- Time budget: maximum elapsed time for recovery.
- Resource budget: maximum CPU, memory, or other resources allocated to recovery.

The implementation must define whether the initial execution counts toward any attempt limit and must use consistent accounting.

A recovery attempt must not begin if it would violate a hard safety constraint or the applicable budget. When a budget is exhausted, the task must stop recovering and transition to "FAILED", unless a broader authorized cancellation or safety policy requires "STOPPED".

The controller must never retry indefinitely.

7. Unknown outcomes and external side effects

An operation enters "UNKNOWN" when its outcome cannot be reliably established.

Examples include a remote request timing out after submission or a crash occurring after an external service may have accepted a command.

While in "UNKNOWN":

1. Do not assume success or failure.
2. Do not blindly retry potentially non-idempotent operations.
3. Reconcile through available operation identifiers, durable records, or external-service status.
4. If success is established, validate the resulting state before accepting it.
5. If failure is established, evaluate recovery under Policy B.
6. If the outcome remains unresolved, continue only within configured reconciliation limits or follow an explicit termination policy.

Internal rollback cannot guarantee reversal of external operations. Compensating actions are possible only when the external system supports them and the action is authorized.

8. Dependency invalidation and result preservation

When a result becomes invalid, Virtual Current identifies downstream tasks whose correctness depends on that result.

Only affected results should be invalidated or recomputed when the system can establish that unrelated results remain valid.

A previously successful task whose result is later invalidated must retain its historical execution record. The system creates a new execution record or workflow revision rather than silently rewriting the past.

No dependent task may proceed using a result whose required validity has not been established.

9. Scheduling and resource constraints

Independent "READY" tasks may execute concurrently, subject to:

- CPU, memory, and other resource limits.
- Dependency ordering.
- Task priority and safety requirements.
- Recovery budgets.
- Isolation and authorization policies.

Adaptive scheduling must remain inside hard limits. Resource reallocation cannot create physical compute capacity or justify exceeding permitted limits.

An optional 50 Hz control target corresponds to a 20 ms interval. This is a scheduling or reassessment target, not a guarantee that every workflow cycle, task, or recovery operation finishes within 20 ms.

10. Core invariants

1. Validated-result invariant: Dependents cannot consume invalid or unverified required results.
2. Safety invariant: Adaptive selection cannot override hard safety, authorization, or resource constraints.
3. Bounded-recovery invariant: Recovery is limited by configured attempt, time, and resource budgets.
4. Reassessment invariant: Every failed recovery attempt triggers a new assessment before another attempt begins.
5. Justified-selection invariant: A strategy is selected only if it is applicable, permitted, and supported by the available evidence.
6. Uncertainty invariant: Unresolved external outcomes are not silently classified as success or failure.
7. History invariant: State transitions and recovery decisions are recorded according to the durability policy.
8. Selective-recovery invariant: Unaffected work is preserved only when its validity can be established.
9. Validation invariant: Completing a recovery action does not itself prove recovery success.
10. Termination invariant: Exhausting a recovery budget leads to an explicit terminal or authorized stop policy, not endless retries.

11. Acceptance scenarios

A future deterministic simulator must test at least:

1. Normal execution and successful validation.
2. Execution failure followed by a successful alternative recovery strategy.
3. A failed recovery attempt followed by reassessment and another strategy.
4. No applicable alternative strategy.
5. Attempt-budget exhaustion.
6. Time-budget exhaustion.
7. Resource-budget exhaustion.
8. A task that exits successfully but produces invalid output.
9. Upstream failure with selective downstream invalidation.
10. Preservation of unrelated valid work.
11. A remote timeout with an unknown external outcome.
12. Restart and reconciliation using persisted execution history.
13. Cancellation during execution or recovery.
14. Concurrent failures that compete for limited resources.
15. Rejection of an unsafe or unauthorized recovery strategy.

The tests must verify both final outcomes and the intermediate state transitions.

12. Open specification questions

Before implementation, the project must define:

- The exact event schema and transition guards.
- The recovery-strategy selection algorithm and tie-breaking rules.
- The meaning of an attempt and its accounting.
- Validation policy configuration and versioning.
- Cancellation behavior for running tasks and descendants.
- Checkpoint and journal durability guarantees.
- Ordering of concurrent events and conflict resolution.
- External-operation idempotency and reconciliation requirements.
- Benchmark methodology and comparison baselines.

13. Implementation status

This document specifies proposed behavior only. It is not evidence of an implemented runtime, formal verification, benchmark results, or technological novelty.

The next milestone is to formalize transition guards and recovery-budget accounting, then validate the specification against failure scenarios before implementing a deterministic simulator.
