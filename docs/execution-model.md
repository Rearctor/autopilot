# Autopilot - Execution Model

**Status: Concept / Research.** Nothing described here is implemented, deployed, or
scheduled. Structures and field names are drafts.

## The chain

```
Revenue  --->  Strategy  --->  Execution
   |              |               |
   |              |               +-- a bounded onchain action
   |              +-- a declared rule over accrued revenue
   +-- reaction allocation inflow (70% share), both denominations
```

Each arrow is a place where something can be constrained, and each is treated
separately below.

## Revenue

Autopilot operates on the **reaction allocation** - the 70% share - and never on the
protocol share. The allocation accrues from two sources whose behaviour differs:

| Phase | Source | Denomination | Cadence |
| :--- | :--- | :--- | :--- |
| Charging | Bonding-curve trading fees | USDC | Bursty; concentrated around launch activity |
| Expansion | Migrated Uniswap V4 pool fees | USDC and reaction token | Continuous with pool volume |

A strategy written against Charging revenue may behave very differently once the same
strategy is fed pool fees. The model therefore treats the ignition boundary as a
first-class concern rather than an implementation detail.

## Strategy

A strategy is a declared rule of the form:

> **when** `trigger` holds **and** all `conditions` hold **and** `cooldown` has elapsed,
> perform `action`, spending no more than the lesser of available budget and
> `budget_limit`.

Four properties are deliberately separated:

- **Trigger** - the event or threshold that makes execution *eligible*.
- **Conditions** - predicates that must additionally hold for execution to *proceed*.
- **Budget limit** - the hard ceiling on cumulative spend.
- **Cooldown** - minimum interval between executions.

Separating trigger from conditions matters: a trigger answers "is it time?", a
condition answers "is it safe?". Collapsing them produces rules that are hard to reason
about and harder to constrain.

## Execution

An execution is a single bounded action against a declared budget. The draft treats
executions as:

- **Atomic** - fully applied or fully reverted.
- **Isolated** - failure affects only this strategy's state.
- **Budget-bounded** - cannot spend more than remains.
- **Non-reentrant into protocol mechanics** - see the safety invariants.

### Who initiates

Autopilot is "automated" in the sense that the *decision* is declared in advance. Some
party still submits a transaction.

| Model | Description | Cost |
| :--- | :--- | :--- |
| Creator-initiated | Creator submits when conditions hold | Not really automation |
| Permissionless keeper | Anyone may submit when conditions hold | Needs an incentive |
| Incentivised keeper | Caller compensated from strategy budget | Reintroduces an outflow path needing care |
| Piggyback | Evaluated during an unrelated transaction | Imposes cost on an unrelated party |

> **Open question.** Initiation model is unresolved. Permissionless-with-no-reward is
> the most conservative: nothing executes unless someone cares enough to pay gas, which
> is a safe failure mode but may mean strategies simply never run.

## Pre- and post-ignition execution

The same strategy spans a boundary where its inputs change character.

**Pre-ignition concerns.** Any strategy that trades the reaction's own token during
Charging is acting on the bonding curve - the same curve that determines progress
toward 5,042 USDC. A buyback executing pre-ignition moves the reaction closer to
ignition using the reaction's own fee revenue.

That is a significant interaction. It does not violate "no early graduation" - the
threshold is still the threshold, and nothing skips it - but it does mean automated
spending can accelerate arrival at it.

> **Under research.** Whether any strategy that trades the reaction token should be
> permitted to execute pre-ignition at all. A defensible default is that
> token-trading strategies activate only at Expansion. This is the current working
> assumption, not a decision.

**Post-ignition concerns.** Execution happens against a Uniswap V4 pool with real
depth and external participants. Slippage, sandwiching, and interaction with the
Rearctor hook all become relevant. A strategy that is safe against a curve is not
automatically safe against a pool.

## State a strategy needs

Minimum per-strategy state for the above to be checkable:

| Field | Purpose |
| :--- | :--- |
| `spent_total` | Enforce `budget_limit` |
| `last_execution_at` | Enforce `cooldown` |
| `execution_count` | Observability; rate anomaly detection |
| `enabled` | Administrative pause |
| `last_failure` | Diagnose inert strategies without offchain tooling |

## Related documents

- [Control Room integration](control-room-integration.md)
- [Draft strategy schema](../specs/strategy.schema.json)
- [RFC 0001 - Trigger model](../rfcs/0001-trigger-model.md)
- [Safety invariants](../research/safety-invariants.md)
