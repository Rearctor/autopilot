# Autopilot - v0 Design Specification

## Status

**Concept / Research.** This is a draft design target, not a production specification.

Nothing described here is implemented, deployed, scheduled, or audited. No contracts
exist. No APIs exist. No strategy has ever executed. There is no commitment that this
design will ship, in this form or at all.

Builds on, and does not replace:

- [docs/execution-model.md](../execution-model.md)
- [docs/control-room-integration.md](../control-room-integration.md)
- [rfcs/0001-trigger-model.md](../../rfcs/0001-trigger-model.md)
- [research/safety-invariants.md](../../research/safety-invariants.md)
- [specs/strategy.schema.json](../../specs/strategy.schema.json)

Every numeric figure here that is not a Rearctor protocol constant is illustrative only.
Not protocol defaults or committed parameters.

## Core conceptual model

```
  Revenue  --->  Strategy  --->  Execution
```

Revenue is the reaction allocation. A strategy is a rule declared over it. An execution
is a bounded onchain action. This framing is unchanged from
[docs/execution-model.md](../execution-model.md) and is the organising idea of v0.

## Scope

### Proposed for v0

- A declared strategy set operating on reserved budgets within the reaction allocation.
- Separation of trigger (eligibility), conditions (safety), cooldown (frequency) and
  budget (magnitude).
- A closed action set.
- Per-strategy isolation of state and failure.
- Operator pause authority.
- Execution restricted so that automating an action never widens the action set.

### Under research

- Trigger manipulability, especially market-reading triggers. See Issue #1.
- Execution and failure semantics: retry, partial execution, idempotency, ordering.
  See Issue #2.
- Initiation model and who pays execution gas.
- Whether strategy definitions are immutable once declared.
- Whether token-trading strategies may execute pre-ignition.

### Non-goals

- Arbitrary user-supplied strategy code.
- Any capability the Control Room does not already have. Automating an action must not
  widen the set of actions available.
- Access to the 30% protocol share.
- Any reach into initial supply, fee configuration, the ignition threshold, migration,
  or migrated liquidity principal.
- Price prediction, market making, or discretionary trading judgement.

## Strategy object

Draft format: [specs/strategy.schema.json](../../specs/strategy.schema.json). v0 does not
change it.

| Field | Role |
| :--- | :--- |
| `strategy_id` | Identity within the reaction's strategy set |
| `enabled` | Operator pause |
| `activation_stage` | Earliest stage at which execution may occur |
| `trigger` | Eligibility predicate |
| `conditions` | Safety predicates, all of which must hold |
| `action` | What is performed |
| `budget_limit` | Hard ceiling on cumulative spend |
| `cooldown` | Minimum interval between executions |

## Trigger model

A trigger answers **"is it time?"** and nothing else. Three kinds are drafted, with very
different exposure:

| Kind | Reads | Manipulability |
| :--- | :--- | :--- |
| `accrued-threshold` | Internal accounting only | Low. Moving it requires generating real fee revenue |
| `elapsed-interval` | Block timestamp | Marginal, within validator drift |
| `volume-observed` | Derived market data over a window | **High. An adversary can manufacture volume to force execution** |

`volume-observed` is present in the draft schema **to make the problem explicit, not
because it is endorsed**. Whether it should exist at all is Issue #1.

**Unresolved:** window selection for any windowed trigger. No window is simultaneously
cheap to evaluate, responsive, and expensive to manipulate.

## Conditions

A condition answers **"is it safe?"**. Conditions are evaluated after the trigger and
all must hold.

Drafted kinds: minimum pool liquidity, maximum price deviation, minimum remaining
budget, stage is.

**Open question carried from [RFC 0001](../../rfcs/0001-trigger-model.md):** a trigger
that is only safe when paired with a condition should arguably carry that condition in
its own definition rather than leaving it optional.

## Actions

Closed set: `buyback`, `liquidity-reinvestment`.

Each action declares a per-execution spend cap and a slippage bound. **No default
slippage is proposed anywhere in this repository** - an arbitrary default would be a
silent protocol decision.

**Constraint on liquidity reinvestment:** it must be strictly additive. An
implementation that removes and re-adds liquidity, even atomically and even netting
positive, creates a window in which principal is withdrawable and breaks invariant I-2.

## Budget accounting

- Budgets are reserved from the reaction allocation, never from the 30% protocol share.
- Checks are against **live remaining balance at execution time**, never a value cached
  at declaration.
- Manual Control Room spends and automated executions draw on the same balance.
- Budgets are denomination-specific. A USDC budget cannot be spent in reaction tokens.

**Working assumption:** manual action takes precedence; a strategy execution reverts
cleanly on insufficient budget. Consistent with Autopilot being a convenience over the
manual surface. Not a decision - see Issue #2.

## Cooldowns

Cooldown bounds **frequency** independently of eligibility. Its value is bounding
worst-case behaviour: however wrong a trigger is, spend per unit time is capped.

Candidate diagnostic: `budget_limit / max_spend_per_execution * cooldown_seconds` gives
a lower bound on time to exhaust a budget. A declared strategy where that figure is
implausibly short should be treated as suspect.

**Unresolved:** whether cooldown resets on attempt or on success. Resetting on attempt
rate-limits a failing strategy, and lets an adversary who can force reverts suppress the
strategy indefinitely. No mitigation is proposed. See Issue #1.

## Execution lifecycle

```
  eligible?      trigger holds
      |
      v
  permitted?     cooldown elapsed AND all conditions hold AND enabled
      |
      v
  bounded?       spend <= min(remaining budget, per-execution cap)
      |
      v
  execute        atomic, isolated, recorded
```

Each gate is separable so that one can be tightened without weakening the others.

**Unresolved:** who submits the transaction. Four initiation models are costed in
[docs/execution-model.md](../execution-model.md). Permissionless-with-no-reward is the
most conservative and may mean strategies never run.

## Failure semantics

A failed execution must not:

- revert an unrelated user transaction;
- modify any other strategy's state;
- consume budget it did not spend;
- leave the reaction in a partially-applied state.

**Unresolved and tracked in Issue #2:**

| Question | Why it matters |
| :--- | :--- |
| Partial execution | Is an action that succeeds in part committed or reverted? |
| Retry | Immediate, cooldown-gated, or manual only? |
| Idempotency | Can a duplicate submission double-spend a budget? |
| Budget reservation | Is budget reserved at eligibility or debited at execution? |
| Execution ordering | If two strategies are simultaneously eligible against one budget, which is served? |

**Not asserted:** no fairness guarantee between strategies.

## Pause model

An operator must be able to disable a strategy. This is the one place where automation
is clearly subordinate: a strategy behaving unexpectedly needs a stop that does not
require a protocol change.

**Unresolved:** whether per-strategy `enabled` is sufficient or a per-reaction halt is
needed. A global halt is a single point of control and therefore a trust consideration;
per-strategy disable is finer-grained but slower in an incident.

Pausing must not: release reserved budget to anyone, alter accrued state, or be usable
to reach any protocol parameter.

## Safety invariants

The full set is [research/safety-invariants.md](../../research/safety-invariants.md).
v0 treats these as binding:

| ID | Invariant | Enforcement |
| :--- | :--- | :--- |
| I-1 | Automation cannot change immutable launch parameters | By construction |
| I-2 | Automation cannot withdraw migrated liquidity principal | By construction |
| I-3 | Automation cannot influence whether or when ignition occurs | By construction, with a stated caveat |
| I-4 | Automation cannot touch the protocol share | By construction |
| I-5 | Automation cannot tax transfers | By construction |
| I-6 | An execution cannot exceed available reserved budget | Runtime |
| I-7 | Cumulative spend never exceeds the budget limit | Runtime |
| I-8 | Budget accounting is denomination-correct | Runtime |
| I-9 | Execution failure does not modify unrelated reaction state | By construction |
| I-10 | Execution failure does not revert unrelated user transactions | By construction |
| I-11 | A strategy cannot read or write another strategy's state | By construction |
| I-12 | Declared configuration is readable before it takes effect | Expected, not enforceable unless definitions are immutable |
| I-13 | Every execution is attributable | Runtime |

**Deliberately not asserted:** liveness, execution quality, fairness between strategies,
manipulation resistance for market-reading triggers.

## Pre-ignition vs post-ignition behaviour

| | Pre-ignition (Charging) | Post-ignition (Expansion) |
| :--- | :--- | :--- |
| Revenue source | Curve trading fees | Migrated pool fees |
| Denomination | USDC | USDC and reaction token |
| Cadence | Bursty | Continuous with pool volume |
| Execution venue | Bonding curve | Uniswap V4 pool with outside participants |
| Relevant risks | Interaction with graduation progress | Slippage, sandwiching, hook interaction |

**The honest caveat.** A buyback executing during Charging spends reaction revenue on
curve purchases, which increases reserves, which moves the reaction toward 5,042 USDC.
It does not change the threshold and does not skip it, but it does influence when
arrival happens.

**Working assumption:** token-trading strategies activate only at Expansion. This is the
most conservative reading of invariant I-3 and is an assumption, not a decision.

## Open design issues

- [#1 - Evaluate trigger manipulability and safety constraints](https://github.com/Rearctor/autopilot/issues/1)
- [#2 - Define strategy execution and failure semantics](https://github.com/Rearctor/autopilot/issues/2)

## Related documents

- [Execution model](../execution-model.md)
- [Control Room integration](../control-room-integration.md)
- [RFC 0001 - Trigger model](../../rfcs/0001-trigger-model.md)
- [Safety invariants](../../research/safety-invariants.md)
- [Implementation plan](implementation-plan.md)
