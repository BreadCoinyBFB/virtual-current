TXP Virtual Current — Event Schema and Transition Contracts

Document: "docs/event-schema.md"
Specification: v0.4-draft
Status: Proposed; not implemented or validated
Related documents: "docs/state-machine.md"

1. Purpose

This document defines the event representation, event-ordering rules, and transition contracts for TXP Virtual Current.

Its purpose is to make state-machine behavior sufficiently explicit for a deterministic simulator to validate.

The specification distinguishes:

- Event observation from underlying execution.
- Event processing order from real-world occurrence order.
- Operation completion from result validation.
- Successful execution from authorization to use its result.
- Recovery planning from actual recovery-action execution.
- Missing evidence from evidence of failure.

2. Event envelope

Every event accepted by the Virtual Current controller must use a versioned event envelope.

The following JSON object illustrates the proposed structure. Fields marked optional may be absent when they do not apply; required fields must be present.

{
  "schema_version": "0.4",
  "event_id": "evt-unique-id",
  "event_type": "task.execution_completed",
  "workflow_id": "workflow-001",
  "workflow_revision": 1,
  "task_id": "task-007",
  "execution_id": "exec-007-002",
  "recovery_episode_id": null,
  "recovery_round_id": null,
  "action_id": null,
  "producer": {
    "component_id": "task-runner-01",
    "instance_id": "instance-01"
  },
  "ordering": {
    "controller_epoch": 1,
    "sequence_number": 142,
    "producer_sequence_number": 38,
    "causal_parent_event_ids": ["evt-start-007"],
    "ordering_confidence": "controller_ordered",
    "occurrence_order": "known_before_deadline"
  },
  "timestamps": {
    "occurred_at": "2026-10-25T10:00:00Z",
    "observed_at": "2026-10-25T10:00:01Z",
    "accepted_at": "2026-10-25T10:00:01Z",
    "monotonic_elapsed_ns": 8123456789
  },
  "payload": {
    "execution_outcome": "completed",
    "result_reference": "result-007-002"
  },
  "evidence": {
    "evidence_reference": "evidence-007-002",
    "integrity_status": "verified"
  },
  "correlation_id": "corr-007",
  "idempotency_key": "execution-completion-007-002"
}

This example is illustrative, not a finalized JSON Schema. The implementation must define requiredness, types, enumerations, length limits, and validation rules for every field.

3. Field definitions

3.1 Identity and version fields

- "schema_version": identifies the event-envelope schema version.
- "event_id": globally unique event identifier within the defined identity scope.
- "event_type": registered event type.
- "workflow_id": identifies the workflow.
- "workflow_revision": identifies the workflow definition or revision to which the event belongs.
- "task_id": identifies the affected task when applicable.
- "execution_id": identifies a particular task execution attempt.
- "recovery_episode_id": identifies the recovery episode when applicable.
- "recovery_round_id": identifies a committed recovery round when applicable.
- "action_id": identifies an individual recovery-action execution when applicable.

Identifiers must not be reused to represent different executions or actions.

An event that refers to a stale workflow revision must be reconciled against that revision. It must not silently modify the state of a newer revision.

3.2 Producer fields

"producer.component_id" identifies the logical component that generated the event.

"producer.instance_id" identifies the particular component instance or process.

Producer identity supports diagnostics and provenance. It does not, by itself, establish that the producer is trusted.

3.3 Ordering fields

"ordering.controller_epoch" identifies a controller generation or epoch.

"ordering.sequence_number" identifies the event's position in the controller's accepted event stream within the specified epoch and ordering scope.

"ordering.producer_sequence_number" optionally records the producer's own sequence number.

"ordering.causal_parent_event_ids" identifies events known to precede or cause this event.

"ordering.ordering_confidence" identifies what the controller can establish about event order.

Proposed values:

- "controller_ordered": accepted into a defined controller ordering.
- "causally_ordered": a causal relationship is established, but global ordering may not be.
- "partially_ordered": only some ordering relationships are established.
- "unresolved": ordering remains uncertain.

"ordering.occurrence_order" optionally represents the relationship between an underlying operation and a relevant deadline or event.

Proposed values:

- "known_before_deadline"
- "known_at_or_after_deadline"
- "unknown"
- "not_applicable"

The controller must not assign "known_before_deadline" unless supported by trustworthy evidence and the configured time semantics.

A controller sequence number establishes processing order, not necessarily physical occurrence order.

3.4 Timestamp fields

- "occurred_at": the producer's claimed occurrence time, when known.
- "observed_at": the time the producer or receiving component observed the event.
- "accepted_at": the time the controller accepted the event into its ordered stream.
- "monotonic_elapsed_ns": optional monotonic elapsed-time information within a documented clock domain.

Wall-clock timestamps must include an explicit timezone or use a consistently defined UTC representation.

Monotonic measurements must identify or unambiguously belong to a known clock domain and process or system epoch. Monotonic timestamps from unrelated machines cannot be compared as though they shared a clock.

A timestamp is evidence, not automatic proof of the underlying operation's occurrence time. Clock skew, buffering, process suspension, and delayed delivery must be considered.

3.5 Payload

"payload" contains the event-type-specific data.

The payload must conform to the registered schema for its "event_type". Unknown payload fields must be handled according to the versioning policy.

Payloads should reference large outputs, logs, or artifacts rather than embedding unlimited content in the event envelope.

3.6 Evidence fields

Evidence identifies the information supporting an event or claimed outcome.

- "evidence_reference": a reference to supporting evidence, when available.
- "integrity_status": the result of applicable integrity checks.

Proposed integrity values:

- "verified"
- "unverified"
- "failed"
- "not_applicable"

Integrity verification does not independently prove that a claimed execution outcome is true. The controller must evaluate evidence provenance and the relevant operation contract.

3.7 Correlation and idempotency

"correlation_id" groups events associated with the same operation or diagnostic episode.

"idempotency_key" identifies a logical operation for deduplication or safe-retry handling when applicable.

The same "event_id" delivered repeatedly must not cause multiple committed state transitions.

Distinct events describing the same logical operation must not automatically be discarded as duplicates. Deduplication must use event identity and operation semantics, not timestamps alone.

An idempotency key in Virtual Current does not guarantee that an external service performs an operation exactly once. That guarantee requires support from the external system or another appropriate coordination mechanism.

4. Event-type registry

The initial registry should include at least the following categories.

Task lifecycle

- "task.ready"
- "task.execution_started"
- "task.execution_completed"
- "task.execution_failed"
- "task.validation_started"
- "task.validation_passed"
- "task.validation_failed"
- "task.cancel_requested"
- "task.stopped"

Recovery lifecycle

- "recovery.entered"
- "recovery.plan_committed"
- "recovery.action_started"
- "recovery.action_completed"
- "recovery.action_failed"
- "recovery.round_closed"
- "recovery.budget_exhausted"
- "recovery.terminated"

Dependencies and checkpoints

- "dependency.invalidated"
- "checkpoint.created"
- "checkpoint.restored"
- "journal.reconciled"

Resources and safety

- "resource.limit_approached"
- "resource.limit_exhausted"
- "safety.restriction_activated"
- "safety.restriction_cleared"
- "workflow.pause_requested"
- "workflow.stop_requested"

External operations

- "external_operation.submitted"
- "external_operation.outcome_unknown"
- "external_operation.reconciled"

The registry is extensible, but each event type must define its payload schema, valid source states, required evidence, and transition effects before it is enabled.

5. Event acceptance and ordering contract

When an event arrives, the controller must:

1. Validate the envelope and schema version.
2. Authenticate or otherwise establish the producer's permitted role.
3. Verify identifiers and workflow revision.
4. Check whether the event has already been accepted.
5. Validate causal references and available ordering metadata.
6. Assign its controller ordering token.
7. Preserve uncertainty if the underlying occurrence order cannot be established.
8. Evaluate the event against the current committed state and transition guards.
9. Commit the accepted event and resulting state change according to the durability policy.

Events with invalid schemas, unauthorized producers, conflicting identifier reuse, or impossible references must be rejected or quarantined according to policy.

A delayed event must not automatically overwrite a newer committed state. The controller must reconcile it against the execution record, causal history, and current workflow revision.

An event that has been accepted into the log is not necessarily an event that changes the task state. Some events are informational; others may be rejected as stale or inapplicable after reconciliation.

6. Transition contract model

Each permitted transition must define:

- "contract_id": stable identifier for the transition rule.
- "from_states": permitted source states.
- "trigger_events": event types that may request the transition.
- "preconditions": conditions that must be true before committing.
- "guards": safety, budget, authorization, dependency, and ordering checks.
- "actions": authorized operations performed as part of the transition.
- "postconditions": facts that must hold after commitment.
- "failure_policy": behavior when a guard or action fails.
- "audit_requirements": records required to explain the decision.

A transition is permitted only when its source state, event, preconditions, and guards are compatible.

The contract evaluator must distinguish a rejected transition from a failed task execution. A transition rejection does not prove that the underlying task failed.

7. Initial transition contracts

TC-001: PENDING → READY

Trigger: Dependency state changes or a readiness check occurs.

Preconditions:

- All required dependencies have accepted, valid results.
- Required dependency versions match the workflow revision.
- The task has not been cancelled or invalidated.

Postconditions:

- The task is eligible for scheduling.
- No execution is implicitly started by this transition.

Failure behavior: Remain "PENDING" or record the relevant dependency failure and initiate reassessment.

TC-002: READY → RUNNING

Trigger: "task.ready" or an authorized scheduling decision.

Preconditions:

- The task remains eligible under the current workflow revision.
- Required permissions are valid.
- Required resources are available or reserved.
- No mandatory safety restriction or applicable cancellation prohibits execution.

Actions:

- Create a unique execution identifier.
- Record the execution start.
- Reserve applicable resources.

Postconditions:

- Exactly one execution attempt is associated with the committed transition.
- The task enters "RUNNING".

Failure behavior: Remain "READY", return to a valid waiting state, or stop according to the reason and policy. Do not claim the task started if no execution began.

TC-003: RUNNING → VALIDATING

Trigger: "task.execution_completed" or a defined validation boundary.

Preconditions:

- The event refers to the active execution or has been reconciled with its execution record.
- Available execution evidence is sufficient to establish the relevant outcome.

Actions:

- Close or reconcile the execution record.
- Record the output reference and resource consumption.
- Begin the required validation procedure.

Postconditions:

- The task enters "VALIDATING".
- The output is not yet accepted solely because execution completed.

Failure behavior: If the outcome cannot be established, enter or remain in "UNKNOWN" where appropriate.

TC-004: VALIDATING → SUCCEEDED

Trigger: "task.validation_passed".

Preconditions:

- The result passed all required validation checks.
- Required dependencies remain valid.
- The result belongs to the applicable workflow revision.
- Applicable safety, cancellation, and acceptance policies permit downstream use.
- Required state and result records have been committed according to the durability policy.

Postconditions:

- The task enters "SUCCEEDED".
- The result is marked valid for its declared scope.
- Eligible dependents may proceed.

Failure behavior: Reject the transition if any guard fails. Reassess invalid results or apply the applicable stop policy. A rejected transition does not itself imply that the execution failed.

TC-005: RUNNING or VALIDATING → RECOVERING

Trigger: A recoverable execution failure, rejected validation result, or relevant dependency invalidation.

Preconditions:

- The triggering evidence has been recorded.
- Recovery is permitted by the applicable policy.
- The transition does not violate a mandatory safety restriction.
- The recovery episode can be initialized or continued within its configured limits.

Actions:

- Record the recovery trigger.
- Establish the applicable budget scope.
- Identify affected results and dependencies.
- Begin recovery assessment.

Postconditions:

- The task enters "RECOVERING".
- No recovery action begins until its own guards pass.

TC-006: RECOVERING → RUNNING or READY

Trigger: An authorized recovery action reaches its required completion condition.

Preconditions:

- The action was permitted and its outcome has been recorded.
- The action did not violate the applicable budgets.
- The task state and required dependencies have been re-established.
- Required resources and permissions permit resumption.

Postconditions:

- "RUNNING" is used when execution is authorized to resume immediately.
- "READY" is used when the task is valid for scheduling but must wait for the scheduler.
- Required validation still occurs before a result is accepted.

Failure behavior: Reassess the action outcome. If the recovery action failed, follow the bounded recovery policy rather than resuming automatically.

TC-007: RECOVERING → FAILED

Trigger: Recovery budget exhaustion or a determination that no safe, applicable strategy remains.

Preconditions:

- The applicable failure or budget condition is established.
- Any required safe-stop and audit operations are performed or accounted for.
- No policy-required system-wide stop changes the required terminal handling.

Postconditions:

- The task enters "FAILED".
- The reason and consumed budgets are recorded.
- Unaffected tasks may continue only if workflow and safety policies permit.

TC-008: Active state → STOPPED

Trigger: An authorized cancellation or mandatory stop policy.

Preconditions:

- The stop instruction is authenticated or otherwise authorized.
- The required stop and cleanup policy is identified.

Actions:

- Prevent prohibited new work.
- Safely interrupt work where possible.
- Reconcile or preserve the status of active external operations.
- Record completed side effects.

Postconditions:

- The task enters "STOPPED" when its defined safe stopping condition is reached.
- Completed execution evidence is preserved.

TC-009: Active state → UNKNOWN

Trigger: An operation outcome becomes uncertain.

Preconditions:

- Available evidence cannot establish whether the operation succeeded or failed.
- Assuming either outcome would violate the applicable correctness or safety policy.

Postconditions:

- The task or operation is marked uncertain.
- Potentially unsafe retries are blocked.
- Reconciliation becomes the required next step.

The implementation must define how "UNKNOWN" applies when only one operation within a task is uncertain, so that unrelated task state is not unnecessarily lost.

TC-010: UNKNOWN → VALIDATING or RECOVERING

Trigger: An external-operation reconciliation result or newly available evidence.

Preconditions:

- The evidence is linked to the correct operation and execution.
- The reconciliation result is sufficiently trustworthy for the relevant decision.

Postconditions:

- If successful completion is established, proceed to "VALIDATING".
- If failure is established, proceed to "RECOVERING" when recovery is permitted.
- If uncertainty remains, preserve "UNKNOWN" and follow the reconciliation budget.

An uncertain external operation must not be retried merely because its original response was missing.

8. Global arbitration contract

Before committing any state transition, the controller must evaluate:

1. Mandatory safety restrictions.
2. Authorized cancellation and stop requirements.
3. Hard time, action, and resource limits.
4. Event identity, causality, and ordering.
5. Transition-specific preconditions and guards.
6. Result validity and downstream-use authorization.
7. Persistence and audit requirements.

The controller must record the facts that occurred separately from the decision it made about those facts.

For example, an execution can have the outcome "completed", while its result remains ineligible for downstream use because a cancellation or acceptance policy applies.

9. Versioning and compatibility

Every event and transition contract must be interpreted under a defined schema and workflow revision.

Schema changes must specify whether they are backward-compatible. Unknown event types must not silently trigger arbitrary state transitions.

Replaying an event history must use a compatible transition-policy version or explicitly document how the replay is migrated. A new policy must not silently reinterpret old records as if they had originally been processed under that policy.

10. Simulator acceptance criteria

The first simulator must demonstrate that:

- Invalid envelopes are rejected.
- Duplicate event delivery does not duplicate committed transitions.
- Controller sequence order is distinct from occurrence-time claims.
- Out-of-order events cannot silently overwrite newer state.
- Stale workflow revisions are handled explicitly.
- Every transition checks its contract.
- Recovery actions cannot begin without budget and safety approval.
- Successful execution is distinct from validated success.
- Unknown external outcomes remain uncertain until reconciled.
- Concurrent events produce a consistent committed state.
- Restart and event replay preserve the documented ordering and durability guarantees.

11. Implementation status and limitations

This is a proposed schema and contract specification. The example JSON is illustrative and is not yet a validated machine-readable schema.

The design does not establish exactly-once external execution, global event ordering across unsynchronized systems, or crash consistency without additional implementation mechanisms.

The next milestone is to create a formal, machine-readable transition table and a deterministic event-processing model, then test them against the acceptance scenarios before implementing the production runtime.
