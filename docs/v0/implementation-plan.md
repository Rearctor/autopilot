# Autopilot - v0 Implementation Plan

## Status

**Concept / Research.** This plan describes work that has not started. No phase is
complete, in progress, or scheduled.

There are **no dates** in this document. Phase ordering is a dependency order, not a
timeline. No implementation exists, no contracts exist, no tests exist, no benchmarks
exist, and no strategy has ever executed.

Companion document: [v0 design specification](design-spec.md).

## Phase A - Strategy schema stabilization

**Objective.** A strategy format precise enough that two readers agree on what a given
strategy will do, and on what it cannot do.

**Inputs.**
- [specs/strategy.schema.json](../../specs/strategy.schema.json)
- [docs/v0/design-spec.md](design-spec.md)
- Issues #1 and #2

**Deliverables.**
- A decision on whether `volume-observed` remains a permitted trigger kind.
- If retained: the conditions it must carry, expressed in the schema rather than left
  optional.
- Defined semantics for partial execution, retry, idempotency and budget reservation.
- A decision on whether strategy definitions are immutable once declared.
- Schema revised accordingly, with examples still validating.

**Blockers.**
- Issue #1 open. Trigger manipulability unassessed.
- Issue #2 open. Execution and failure semantics undefined.

**Exit criteria.**
- Every entry in the design spec's failure-semantics table is answered or explicitly
  deferred with a stated consequence.
- No trigger kind remains in the schema whose safety depends on an optional condition.
- Schema validates every example.

## Phase B - Trigger evaluator prototype

**Objective.** A non-production evaluator for declared triggers, used to measure how
cheaply each kind can be manipulated.

**Inputs.** Phase A output; the trigger analysis in
[rfcs/0001-trigger-model.md](../../rfcs/0001-trigger-model.md).

**Deliverables.**
- An evaluator for each retained trigger kind.
- Adversarial exercises: manufactured volume against a windowed trigger; timestamp drift
  against an interval trigger; forced reverts to suppress a strategy.
- A written cost-to-manipulate assessment per trigger kind.

**Blockers.** Phase A incomplete. Windowing unresolved.

**Exit criteria.**
- Each retained trigger kind has a documented manipulation cost, or is removed.
- The revert-suppression vector has either a proposed mitigation or an explicit
  acceptance of the risk.

**Explicitly not an exit criterion:** performance, gas efficiency, or production
suitability.

## Phase C - Budget accounting model

**Objective.** An accounting model that satisfies invariants I-6, I-7 and I-8 under
concurrent manual and automated spending.

**Inputs.** Phase A output; the budget accounting section of the design spec.

**Deliverables.**
- A reference model of reserved budgets across denominations.
- Defined interaction between manual Control Room spends and automated executions,
  including precedence.
- A worked treatment of the ignition boundary, where denomination mix changes.

**Blockers.**
- Phase A incomplete. Budget reservation semantics undefined.
- Overlap with Reaction Modules unresolved: module settlement and strategy execution may
  both draw on the reaction allocation.

**Exit criteria.**
- Concurrent manual and automated spend in one block cannot exceed the balance that
  existed at block start.
- No path exists by which a USDC budget is spent in reaction tokens or the reverse.
- Precedence between manual and automated spending is specified, not incidental.

## Phase D - Safety invariant harness

**Objective.** Convert invariants I-1 through I-13 into executable checks.

**Inputs.** Phases B and C; [research/safety-invariants.md](../../research/safety-invariants.md).

**Deliverables.**
- An executable check per invariant, each able to fail.
- Adversarial cases: a strategy attempting to reach protocol parameters; a reinvestment
  action that removes before adding; forced revert mid-execution with other strategies
  holding state; duplicate submission against one budget.
- A record of which invariants the prototype violates.

**Blockers.** Phases B and C incomplete. I-12 cannot be tested while strategy mutability
is undecided.

**Exit criteria.**
- Each check fails when its invariant is deliberately broken. A test that cannot fail
  proves nothing.
- Violations recorded, not suppressed.
- I-2 is tested against capability, not net outcome: a remove-then-add sequence must
  fail the check even when it nets positive.

## Phase E - Control Room integration research

**Objective.** Understand what the Control Room surface would need for declared
automation to be operable and observable.

**Inputs.** Phases A-D; [docs/control-room-integration.md](../control-room-integration.md).

**Deliverables.**
- An assessment of what a participant must be able to read to trust a declared strategy
  set.
- A decision on pause granularity: per-strategy, per-reaction, or both.
- Identification of any protocol change automation would require - with the expectation
  that the answer should be none.
- An analysis of overlap with Reaction Modules over the shared reaction allocation.

**Blockers.** Phases A-D incomplete.

**Exit criteria.**
- Autopilot authority is demonstrably a subset of Control Room authority, with no
  operation reachable through automation that is not reachable manually.
- Pause authority is specified and cannot be used to reach any protocol parameter.
- Any required protocol change is eliminated or escalated as a blocking finding.

## Status of every phase

| Phase | Status |
| :--- | :--- |
| A - Strategy schema stabilization | Not started. Blocked on Issues #1 and #2. |
| B - Trigger evaluator prototype | Not started. Blocked on Phase A. |
| C - Budget accounting model | Not started. Blocked on Phase A. |
| D - Safety invariant harness | Not started. Blocked on Phases B and C. |
| E - Control Room integration research | Not started. Blocked on Phases A-D. |

No phase has begun. No deliverable in this document exists.

## Open design issues

- [#1 - Evaluate trigger manipulability and safety constraints](https://github.com/Rearctor/autopilot/issues/1)
- [#2 - Define strategy execution and failure semantics](https://github.com/Rearctor/autopilot/issues/2)
