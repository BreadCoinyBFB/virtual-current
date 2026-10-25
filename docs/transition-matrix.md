TXP Virtual Current — Formal Transition Matrix and Deterministic Event Processor

Version: v0.5-draft
Status: Proposed; not implemented or benchmarked
Depends on: "docs/state-machine.md", "docs/event-schema.md"

1. Objective

Define a formal transition matrix and deterministic event-processing algorithm for TXP Virtual Current (VC).

The implementation must:

1. Reject state transitions that violate the transition contracts.
2. Process duplicate, delayed, and conflicting events safely.
3. Preserve uncertainty rather than inventing an operation outcome.
4. Enforce cancellation, safety, and recovery budgets.
5. Commit state changes and corresponding audit records consistently.
6. Support reproducible simulation and automated testing.

The controller's decisions must be deterministic relative to the same initial state, policy version, accepted event history, and recorded evidence.

Determinism does not imply that external operations or distributed event delivery are deterministic.

2. Canonical states

State| Meaning
"PENDING"| Dependencies or readiness conditions are not yet satisfied.
"READY"| Eligible for scheduling, but execution has not started.
"RUNNING"| An execution attempt is active or authorized to run.
"VALIDATING"| Execution evidence or a result is being checked.
"SUCCEEDED"| Required validation passed and the result was accepted.
"RECOVERING"| Failure assessment, reconciliation, or an authorized recovery plan is active.
"FAILED"| The task cannot proceed under the applicable failure policy.
"STOPPED"| The task has reached its defined safe stopping condition.
"UNKNOWN"| The task outcome cannot currently be established safely.

"UNKNOWN" requires special handling: uncertainty may concern the entire task or only one external operation. An implementation should represent operation-level uncertainty separately where possible, so that an uncertain operation does not unnecessarily erase known task progress.

3. Formal transition matrix

The matrix describes permitted state changes. Every transition still requires its specified guard conditions.

From| To| Required trigger or condition
"PENDING"| "READY"| Required dependencies and readiness checks pass.
"PENDING"| "FAILED"| A terminal dependency or policy failure prevents execution.
"PENDING"| "STOPPED"| Authorized cancellation or stop policy.
"PENDING"| "UNKNOWN"| A material outcome cannot be established.
"READY"| "RUNNING"| Scheduling, authorization, resource, and safety checks pass.
"READY"| "PENDING"| Readiness or dependency conditions become unsatisfied.
"READY"| "STOPPED"| Authorized cancellation or stop policy.
"READY"| "FAILED"| An unrecoverable pre-execution condition is established.
"RUNNING"| "VALIDATING"| Execution completion is established sufficiently to begin validation.
"RUNNING"| "RECOVERING"| Recoverable failure or required recovery is established.
"RUNNING"| "UNKNOWN"| A material execution outcome is uncertain.
"RUNNING"| "STOPPED"| Stop requested and safe stopping completed.
"RUNNING"| "FAILED"| A terminal failure is established and no recovery is applicable.
"VALIDATING"| "SUCCEEDED"| Result validation and acceptance requirements pass.
"VALIDATING"| "RECOVERING"| Validation fails and recovery is permitted.
"VALIDATING"| "UNKNOWN"| Evidence is insufficient to decide the outcome safely.
"VALIDATING"| "STOPPED"| Stop policy prevents acceptance and safe stopping is complete.
"VALIDATING"| "FAILED"| Validation or policy establishes terminal failure.
"RECOVERING"| "READY"| Recovery restores eligibility; scheduler must decide when to run.
"RECOVERING"| "RUNNING"| Recovery permits immediate execution resumption.
"RECOVERING"| "VALIDATING"| Reconciliation or recovery establishes an outcome requiring validation.
"RECOVERING"| "UNKNOWN"| Recovery cannot resolve a material uncertainty.
"RECOVERING"| "FAILED"| No safe strategy remains or a hard recovery limit is exhausted.
"RECOVERING"| "STOPPED"| Authorized stop is safely completed.
"UNKNOWN"| "VALIDATING"| Reconciliation establishes a candidate completed result.
"UNKNOWN"| "RECOVERING"| Reconciliation establishes failure and recovery is permitted.
"UNKNOWN"| "READY"| Uncertainty is resolved and task eligibility is re-established.
"UNKNOWN"| "STOPPED"| Safe stopping is established and required reconciliation is complete.
"UNKNOWN"| "FAILED"| Evidence establishes terminal failure or policy mandates termination.

3.1 Terminal-state restrictions

"SUCCEEDED", "FAILED", and "STOPPED" are terminal for a particular task execution lifecycle.

A terminal state cannot be reopened by an ordinary delayed event.

If a new attempt is authorized after a terminal state, the controller must create a new execution or explicitly authorized task revision according to policy. It must preserve the earlier terminal record.

A correction to a previously accepted result must use an explicit correction, revocation, or new-revision mechanism. It must not silently rewrite history.

3.2 Forbidden transitions

Any transition not listed in the matrix is forbidden unless a later specification explicitly adds it.

Examples:

- "PENDING → SUCCEEDED" without an established execution or explicitly authorized imported result.
- "READY → SUCCEEDED" without satisfying the result-acceptance contract.
- "RUNNING → SUCCEEDED" without validation.
- "RECOVERING → SUCCEEDED" without the required validation and acceptance process.
- "SUCCEEDED → RUNNING" through a delayed execution-start event.
- "FAILED → READY" through ordinary event delivery.
- "STOPPED → RUNNING" through ordinary event delivery.
- "UNKNOWN → SUCCEEDED" merely because a timeout occurred or a response was missing.

A forbidden transition must not mutate the committed task state. The controller records the rejection and any relevant evidence.

4. Transition guard evaluation

For every proposed transition, evaluate the following in order:

1. Identity: Does the event refer to the correct workflow, revision, task, execution, and recovery episode?
2. Integrity and authority: Is the event structurally valid, and is its source permitted to assert the relevant fact?
3. Deduplication: Has this event already been processed?
4. Ordering and causality: Is the event applicable to the current state, and is its relationship to competing events understood?
5. Safety: Would the transition authorize prohibited work or violate a mandatory restriction?
6. Cancellation: Does an authorized stop request prohibit new work?
7. Budgets: Would the proposed action exceed the applicable time, round, action, or resource budget?
8. Transition preconditions: Are the contract-specific requirements satisfied?
9. Result acceptance: If success is proposed, have validation and dependency requirements passed?
10. Durability: Can the event, decision, state update, and required audit information be committed consistently?

If any mandatory guard fails, reject or defer the transition as appropriate. Do not convert a guard failure into an unsupported claim that the underlying operation failed.

Safety restrictions and cancellation must prevent prohibited new work even when lower-priority events are waiting to be processed. The controller must still reconcile actual effects of operations that may already have started.

5. Deterministic event processor

5.1 Inputs

The processor receives:

- A committed workflow state.
- An accepted event or a bounded batch of events.
- The applicable workflow revision.
- The transition-policy version.
- Recovery-budget state.
- Available evidence and operation-reconciliation records.
- Safety and cancellation status.

5.2 Outputs

Each processing decision produces:

- Accepted, rejected, deferred, or quarantined event status.
- The resulting task and operation states.
- Any authorized actions.
- Updated recovery-budget accounting.
- A transition-decision record.
- Reasons for rejected or deferred transitions.
- Any unresolved ordering or outcome uncertainty.

5.3 Processing algorithm

Step 1 — Validate

Validate schema, field types, event type, identity references, source permissions, workflow revision, and payload.

Reject malformed or unauthorized events. Quarantine events that require manual or policy-defined reconciliation.

Step 2 — Deduplicate

Check "event_id" and the relevant operation identity.

If the event has already been committed, do not apply its transition again. Return the existing processing result where possible.

If the same identifier appears with conflicting contents, treat it as an integrity conflict rather than silently choosing one version.

Step 3 — Record receipt

Persist the accepted event or an equivalent durable ingestion record according to the configured durability policy.

Assign a controller ordering token within a defined ordering scope. A single task's conflicting state transitions must be serialized or protected by an equivalent concurrency-control mechanism.

Step 4 — Establish ordering

Use available controller sequence numbers, causal relationships, operation evidence, and clock information.

Do not treat event arrival time or a producer timestamp as universally authoritative.

If two events have no established causal order, the controller may choose a deterministic processing order for administrative purposes, but it must preserve the fact that their real-world occurrence order is unknown.

Step 5 — Reconcile uncertainty

Before authorizing a potentially unsafe retry, determine whether an earlier operation may already have taken effect.

If the outcome cannot be reconciled sufficiently, retain "UNKNOWN" for the affected operation and block unsafe retries.

Step 6 — Apply global restrictions

Check mandatory safety rules, authorized cancellation, hard deadlines, and applicable recovery budgets.

Do not start new prohibited work. Record the status of operations already in flight and safely interrupt them when the operation contract permits.

Step 7 — Evaluate the transition contract

Find the registered contract for the event and proposed transition.

Check source state, preconditions, guards, and postconditions. An event may be accepted for audit while its requested transition is rejected.

Step 8 — Prepare the decision

Construct the proposed state change, resource-accounting changes, action authorization, and audit record.

Check that the resulting state is consistent with the current workflow revision and that no conflicting state update has committed in the meantime.

Step 9 — Commit atomically

Commit the state transition and the records needed to recover or explain it as one logical transaction, or use a documented write-ahead log or equivalent crash-consistency protocol.

Do not report a committed transition before its required durable commit has succeeded.

External side effects cannot generally be made atomic with a local state database. They require explicit operation tracking, idempotency support where available, and reconciliation after uncertain outcomes.

Step 10 — Reevaluate

Reevaluate newly enabled transitions, queued events, dependencies, safety status, and budgets.

The processor must not automatically execute every newly eligible task. Scheduling and execution remain subject to their own contracts.

5.4 Reference pseudocode

process(event):
    validate_schema_and_authority(event)

    if event_id_already_committed(event.event_id):
        return previous_decision(event.event_id)

    if identifier_conflicts_with_existing_record(event):
        quarantine(event)
        return CONFLICT

    record_ingestion_and_assign_order(event)
    reconcile_causality_and_occurrence_evidence(event)

    if event_requires_unresolved_external_reconciliation(event):
        preserve_unknown_operation_state(event)
        block_unsafe_retry(event)
        persist_decision()
        return DEFERRED

    restrictions = evaluate_safety_cancellation_and_budgets(event)

    if restrictions.prohibit_transition:
        record_rejection_or_deferral(event, restrictions)
        persist_decision()
        return REJECTED_OR_DEFERRED

    contract = find_transition_contract(event)

    if contract_is_missing_or_invalid(contract):
        quarantine_or_reject(event)
        persist_decision()
        return REJECTED

    proposed = evaluate_contract(contract, current_state, event)

    if not proposed.guards_pass:
        record_rejection_or_deferral(event, proposed.reasons)
        persist_decision()
        return REJECTED_OR_DEFERRED

    commit_state_budget_and_audit_changes(proposed)
    return COMMITTED

This pseudocode specifies control flow, not a production-ready implementation. Each persistence operation must follow the crash-consistency rules defined by the implementation.

6. Determinism and replay

For reproducible replay, the processor must preserve or identify:

- Initial committed state.
- Ordered accepted event history.
- Event payloads and evidence references needed by the decision.
- Workflow revision.
- Transition-policy version.
- Recovery-budget configuration and accounting history.
- Relevant reconciliation decisions.
- Any external facts that influenced a transition.

Given equivalent inputs and policy versions, the simulator must produce equivalent transition decisions and state outcomes.

If evidence was unavailable during the original execution, replay must not invent it. A later replay with newly obtained evidence must be identified as a reconciliation or revised evaluation, not silently represented as the original decision.

7. Recovery budget integration

The existing hybrid budget policy remains in force.

Track independently:

- Recovery rounds committed.
- Recovery actions started.
- Elapsed recovery wall-clock time.
- Applicable resource consumption.

Count a recovery round when its plan is committed and authorized for execution. Count an action immediately before its potentially state-changing execution begins.

Failed, timed-out, cancelled-after-start, and partially completed actions remain counted. A new recovery round does not reset the recovery episode's budgets.

Budget exhaustion prohibits new recovery actions. In-flight actions must be reconciled or safely interrupted according to their contracts.

After a failed action, reassess the evidence and remaining budgets. Select a different permitted strategy when justified. Do not repeat a strategy indefinitely or retry an operation with an unresolved external outcome.

8. Testable acceptance criteria

The v0.5 simulator must pass the following tests.

TC-01 — Normal success

Input sequence: "PENDING → READY → RUNNING → VALIDATING → SUCCEEDED", with all required guards satisfied.

Expected: The result is accepted only after validation. Replaying the same history yields the same committed state.

TC-02 — Invalid transition

Input: An ordinary completion event attempts "RUNNING → SUCCEEDED".

Expected: Reject the transition because validation has not occurred. Preserve the event for audit.

TC-03 — Duplicate event

Input: Deliver the same completion event twice.

Expected: One committed event effect and no duplicate transition or recovery-budget charge.

TC-04 — Conflicting identifier

Input: Deliver two different event payloads with the same "event_id".

Expected: Detect the conflict and quarantine or reject the conflicting record.

TC-05 — Out-of-order notification

Input: A late execution-start notification arrives after a valid completion record.

Expected: Do not reopen the execution or overwrite its terminal outcome. Reconcile the late event against the execution history.

TC-06 — Deadline race

Input: A completion notification arrives after the deadline, but trustworthy evidence establishes completion before the deadline.

Expected: Apply the configured deadline and result-acceptance policy to the established occurrence time. Do not use notification arrival time as a substitute for operation completion time.

TC-07 — Unknown deadline relationship

Input: Completion is reported, but reliable evidence cannot establish whether it occurred before or after the deadline.

Expected: Preserve uncertainty and apply the configured reconciliation policy. Do not assume the favorable outcome.

TC-08 — Cancellation race

Input: Cancellation is authorized while an execution is active and completion is reported concurrently.

Expected: Prohibit additional work, reconcile the actual completion and side effects, and apply the configured cancellation-acceptance policy.

TC-09 — Recovery budget exhaustion

Input: A recovery action would exceed the action or elapsed-time budget.

Expected: Do not authorize the new action. Record the budget decision and follow the safe-stop or terminal-failure policy.

TC-10 — Recovery strategy reassessment

Input: A recovery action fails, while another permitted strategy remains safe and budget-compliant.

Expected: Reassess the evidence and select the alternative if justified. Do not blindly repeat the failed strategy.

TC-11 — Uncertain external side effect

Input: A request times out after submission, and no authoritative operation status is available.

Expected: Mark the operation "UNKNOWN"; do not issue a potentially duplicating retry until reconciliation or a safe idempotency contract permits it.

TC-12 — Crash during commit

Input: Simulate a crash during the state-and-audit commit.

Expected: On restart, recover a consistent committed state or replay from a durable log without duplicating committed effects.

TC-13 — Concurrent conflicting events

Input: Two events concurrently propose incompatible transitions for the same task.

Expected: Serialize or reconcile them through the defined concurrency mechanism. Never commit an invalid combined state.

TC-14 — Stale workflow revision

Input: A validly formed event references an obsolete workflow revision.

Expected: Preserve its record but prevent it from silently modifying the current revision.

TC-15 — Terminal-state protection

Input: A delayed ordinary event attempts to reopen "SUCCEEDED", "FAILED", or "STOPPED".

Expected: Reject the transition. An authorized new execution must use a new execution identity or explicit revision mechanism.

TC-16 — Replay consistency

Input: Replay the same durable event history with the same initial state, policy version, and evidence.

Expected: The simulator produces equivalent transition decisions and final state.

TC-17 — Safety precedence

Input: A valid task-start event arrives while a mandatory safety restriction prohibits execution.

Expected: The start is not authorized. The controller records the restriction and does not mistake the rejected start for a completed execution.

9. Definition of done

v0.5 is ready to proceed to implementation only when:

- Every canonical state has defined outgoing-transition rules.
- Every permitted transition has explicit guards and postconditions.
- Forbidden transitions are rejected consistently.
- Event ordering and occurrence uncertainty are represented separately.
- Duplicate delivery and conflicting identifiers are handled.
- Recovery budgets are enforced without reset loopholes.
- Unknown external outcomes cannot trigger unsafe automatic retries.
- Crash and concurrency behavior have explicit test expectations.
- All acceptance tests can be run by an automated simulator.
- Known limitations are documented instead of hidden.

Next milestone: v0.6 — Minimal Deterministic VC Simulator, using synthetic events and a small reference implementation to exercise these contracts before integrating Virtual Current with TXP's runtime or other planned VC Program features.
