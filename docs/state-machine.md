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

14. Transition Guards and Hybrid Recovery-Budget Accounting

Policy: Hybrid accounting
Status: Proposed for Virtual Current v0.2; not implemented or validated.

14.1 Budget model

Each task recovery episode is governed by four independent budget dimensions:

1. Recovery-round count.
2. Individual recovery-action count.
3. Elapsed wall-clock time.
4. Resource consumption.

Each recovery episode must have an explicit budget configuration established by policy before recovery begins.

The effective limits are the most restrictive applicable limits among the task, workflow, and system policies. A task-specific policy cannot override a stricter system-wide safety limit.

Illustrative configuration:

- Maximum recovery rounds: 3.
- Maximum executed recovery actions per episode: 8.
- Maximum elapsed recovery time: 30 seconds.
- Maximum cumulative recovery CPU time: 5 CPU-seconds.
- Maximum recovery memory: 256 MiB peak.
- Maximum recovery I/O: 20 MiB.

These values are examples only. Actual limits must be configured and justified for the target workload and platform.

14.2 Definitions

Recovery episode: The continuous recovery process initiated when a task enters "RECOVERING" after a failure or rejected validation result. It ends when the task returns to ordinary execution or scheduling, succeeds after validation, or terminates in "FAILED" or "STOPPED".

Recovery round: A bounded attempt to address a particular failure using a selected recovery plan. It begins when the controller commits to the plan and reserves the applicable budgets. It ends when the plan succeeds, fails, is abandoned, or is terminated by a guard.

Recovery action: A discrete operation intended to restore the task or workflow to a valid executable condition. Examples include checkpoint restoration, task retry, resource reallocation, and recomputation of invalid results.

Recovery action execution: An action that has passed its preconditions and has begun execution. Each execution has a unique identifier and consumes the applicable action budget.

Reassessment: The evaluation performed after new failure evidence, a failed action, a changed resource condition, or another meaningful event. Reassessment determines whether to continue the current round, select another permitted action, start a new round, or terminate recovery.

14.3 Recovery-round accounting

1. A recovery episode begins when the controller accepts a recoverable failure or rejected validation outcome and enters "RECOVERING".
2. A round is counted when the controller commits to a recovery plan and authorizes its execution.
3. A round remains counted if its plan fails, is interrupted, or is abandoned after execution begins.
4. A round does not count if its plan is rejected before commitment and no recovery execution begins.
5. A new round requires reassessment of the latest failure evidence and remaining budgets.
6. The controller must not start a new round when the round budget is exhausted.
7. A new round cannot reset the episode's action, time, or resource counters.
8. Restarting the controller or process must not reset a persisted recovery episode's consumed budgets.
9. Starting a genuinely new task execution or workflow revision requires a new execution record and an explicitly defined budget scope. Creating a new record must not be used to evade applicable system-wide limits.

14.4 Individual-action accounting

1. Every executed recovery action must have a unique action identifier.
2. The action counter increments immediately before the action begins its potentially state-changing execution.
3. A planned action that is rejected before execution is recorded but does not increment the executed-action counter.
4. A failed, timed-out, cancelled, or partially completed action remains counted if execution began.
5. Retrying the same operation counts as another action execution.
6. Splitting one operation into multiple substeps does not automatically make every substep a separate recovery action. The action taxonomy must be defined so that equivalent work cannot evade limits through artificial decomposition.
7. Composite actions must declare their permitted subactions and limits before execution.
8. A subaction that can independently consume significant resources or produce side effects must be individually accounted for.
9. Before each action begins, the controller must verify that the remaining action, time, resource, and safety budgets permit it.
10. An action that cannot be safely interrupted must reserve sufficient budget for its declared worst-case execution or be excluded when the remaining budget is insufficient.

14.5 Wall-clock time accounting

1. The episode's elapsed-time budget starts when the task enters "RECOVERING".
2. Time consumed by diagnosis, strategy selection, waiting, action execution, validation, and reconciliation counts toward the episode deadline.
3. Waiting for external services or resource availability does not pause the deadline by default.
4. Each round has a deadline no later than the episode deadline.
5. A nested action may have its own shorter deadline, but it cannot extend the round or episode deadline.
6. The controller must use a monotonic time source for elapsed-time measurements.
7. Process restarts must preserve the episode's original deadline or recover an equivalent trustworthy elapsed-time record.
8. If time accounting cannot be trusted after a restart, the controller must not assume that the full budget remains available.
9. At deadline expiry, no new recovery action may begin. Running actions must be cancelled or brought to a safe stopping point according to their declared interruption guarantees.
10. A deadline expiry leads to "FAILED" or a policy-required "STOPPED", unless a specifically authorized safety procedure requires a different controlled outcome.

14.6 Resource accounting

The controller must maintain separate measurements for resource dimensions that have materially different meanings.

Examples include:

- Cumulative CPU time.
- Peak memory usage.
- Memory-time, if configured.
- Cumulative bytes read and written.
- Network transfer or request budgets, if applicable.
- Device-specific or monetary costs, if applicable.

Rules:

1. Resource usage is charged to the recovery episode and to the applicable round and action.
2. An action's resource reservation must be checked before execution when reliable estimates or enforceable limits are available.
3. Actual consumption must be measured and reconciled with reservations.
4. Unused reservations may be released after the associated operation ends.
5. Consumption already incurred is never erased when a strategy fails or changes.
6. Peak memory is evaluated separately from cumulative CPU or I/O consumption.
7. Concurrent recovery actions must be accounted for without double-counting shared usage or omitting their consumption.
8. Resource limits must be enforced by the controller, operating environment, or another reliable mechanism. Accounting alone does not guarantee enforcement.
9. If a resource measurement is unavailable or unreliable, the controller must apply a conservative policy rather than assume zero consumption.
10. When a budget is exhausted, no new action may begin if it would violate that budget. Running actions must be constrained or stopped according to their interruption and safety properties.

14.7 Strategy reassessment

After each failed recovery action, the controller must reassess the failure before proceeding.

The reassessment must determine:

1. Whether the original failure cause remains applicable.
2. Whether the attempted action produced partial results or external side effects.
3. Whether the action's failure changes the set of safe recovery strategies.
4. Whether any results or dependencies require invalidation.
5. Whether a different strategy has a plausible chance of success.
6. Whether the remaining round, action, time, and resource budgets permit another action.
7. Whether the next action is safe, authorized, and compatible with current resource conditions.

The controller may continue the current round with another permitted action if the committed plan allows it and all guards pass. Otherwise, it must close the unsuccessful round and decide whether a new round is justified.

A strategy change must be based on the failure evidence and applicable policy. Changing strategies merely to consume the remaining budget is prohibited.

14.8 Transition guards

A state transition is permitted only when its guard conditions are satisfied.

"PENDING → READY"

- All required dependencies have valid results.
- Required dependency versions and assumptions are consistent.
- The task has not been cancelled or invalidated.

"READY → RUNNING"

- Required execution permissions are valid.
- Required resources are available or reserved according to policy.
- The task is eligible under dependency and scheduling rules.
- No applicable cancellation or safety stop has been committed.

"RUNNING → VALIDATING"

- Execution has completed or reached a defined validation boundary.
- The controller has recorded the available outcome and relevant evidence.

"RUNNING → RECOVERING"

- A recoverable failure has been detected.
- Failure evidence has been recorded.
- Entering recovery is permitted by policy.

"RUNNING → UNKNOWN"

- A relevant operation's outcome cannot reliably be established.
- Blind retrying or accepting the result could be unsafe.

"VALIDATING → SUCCEEDED"

- All required validation checks pass.
- Required dependency assumptions remain valid.
- The result can be committed according to the configured consistency and durability policy.

"VALIDATING → RECOVERING"

- A validation failure or invalid dependency has been identified.
- At least one recovery path may be permitted, or reassessment is required to establish whether recovery is possible.

"RECOVERING → RUNNING"

- A permitted action has completed sufficiently to allow execution to resume.
- Required state and resource preconditions pass.
- The task's next execution attempt is authorized.

"RECOVERING → READY"

- Recovery has restored a valid schedulable state, but execution must wait for scheduling or resources.
- Required dependencies and task state have been revalidated.

"RECOVERING → RECOVERING"

- A reassessment or another action remains permitted.
- The transition does not reset episode budgets.
- Every new action passes all applicable guards.

"RECOVERING → FAILED"

- No safe and applicable recovery strategy remains.
- The applicable round, action, time, or resource budget is exhausted.
- An unrecoverable failure has been established.

"UNKNOWN → VALIDATING"

- Evidence establishes the operation's outcome sufficiently to proceed to validation.
- The result is not accepted solely because the external operation appears to have completed.

"UNKNOWN → RECOVERING"

- Reconciliation establishes failure, or a documented safe recovery path is available despite residual uncertainty.
- The selected action cannot duplicate or corrupt an uncertain external effect without explicit authorization and safeguards.

Any active state → STOPPED

- An authorized cancellation or mandatory safety halt is received.
- The system follows the task's declared interruption and cleanup policy.

14.9 Concurrent events and guard priority

When multiple events compete to change a task's state, the controller must serialize or otherwise deterministically resolve the transitions.

The policy must prioritize mandatory safety restrictions and cancellation over starting new work. Budget exhaustion prohibits new recovery actions. A successful completion event cannot override a required validation check.

If cancellation or budget expiry races with action completion, the controller must record the actual operation outcome and apply the defined transition policy. It must not silently discard evidence of a completed external side effect.

14.10 Recovery episode termination

An episode ends when:

- The task resumes normal execution or scheduling after recovery.
- The task reaches "SUCCEEDED" after the required validation.
- The task reaches "FAILED".
- The task reaches "STOPPED".

The implementation must define exactly when a new failure after resumed execution creates a new recovery episode and how broader workflow-level limits constrain repeated episodes.

Task-level episode budgets do not replace workflow-level or system-level safety limits.

14.11 Required audit record

For each recovery episode, round, and action, record as applicable:

- Task, workflow, execution, episode, round, and action identifiers.
- Entry and exit states.
- Triggering event and failure classification.
- Strategy selected and justification.
- Budget limits and remaining budgets at selection time.
- Action start and completion outcomes.
- Actual resource consumption and reservations.
- Validation results and dependency invalidations.
- Cancellation, timeout, or termination reason.
- Checkpoint and journal references needed for reconciliation.

Audit records must follow the system's durability, access-control, and retention policies. Sensitive data must not be logged indiscriminately.

14.12 Acceptance criteria

A simulator must demonstrate that:

1. Failed recovery actions are counted.
2. Strategy changes trigger reassessment.
3. New rounds do not reset episode budgets.
4. Retries count as new action executions.
5. Time spent waiting is included in elapsed recovery time.
6. Resource reservations and actual usage are reconciled.
7. No action begins after an applicable budget is exhausted.
8. Concurrent actions cannot bypass shared limits.
9. Restart does not silently restore consumed budgets.
10. External uncertainty is preserved until reconciliation or explicit policy resolution.
11. Cancellation and deadline races follow documented rules.
12. A task terminates when no safe, budget-compliant recovery remains.

All numerical limits are configuration examples until validated against the intended workloads and platform.
