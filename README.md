# Rearctor Autopilot

**Status: Concept / Research**

This repository documents a design direction. Nothing described here is implemented,
deployed, or scheduled.

## Overview

Autopilot would extend the Control Room from manual administration into a rule-based
execution layer. The Control Room is the surface through which a reaction's reserved
buyback and liquidity reinvestment budgets are administered. Autopilot would allow
those budgets to be spent according to rules declared in advance, rather than through
discretionary action taken one transaction at a time.

Automation would operate strictly within the immutable constraints established by the
underlying reaction. It could not change the trading-fee configuration, alter initial
supply, move the ignition threshold, or withdraw migrated liquidity principal. Those
constraints hold regardless of who - or what - is operating the reaction.

## Execution model

<div align="center">

**Revenue → Strategy → Automated execution**

</div>

| Step | Concept |
| :--- | :--- |
| **Revenue** | Fees accrue to the reaction allocation, both during curve trading and from the migrated pool after ignition. |
| **Strategy** | A rule declared over that revenue: what to do, under which conditions, within which limits. |
| **Automated execution** | The resulting onchain action, bounded by the strategy's own constraints. |

## Example strategies

| Strategy | Concept |
| :--- | :--- |
| **Automated buybacks** | Spend accrued revenue on the reaction's own token on a defined trigger. |
| **Fee-triggered reinvestment** | Route revenue back into liquidity once accrued fees pass a threshold. |
| **Volume-based execution** | Condition execution on observed trading volume rather than on elapsed time. |

These are illustrative. They are not a committed strategy set.

## Safety constraints

Any execution layer operating on a reaction's budget needs its own limits:

- **Budget limits** - a hard ceiling on what a strategy may spend.
- **Execution cooldowns** - a minimum interval between executions.
- **Safety conditions** - preconditions that must hold for an execution to proceed.

Beneath those sits the protocol floor. The reaction's fee configuration is immutable,
migrated liquidity has no principal-withdrawal function, and no strategy can reach
past either. Autopilot would be constrained by the reaction, not the reverse.

## Control Room integration

The Control Room already exposes reaction administration, including the reserved
buyback and liquidity reinvestment budgets drawn from the reaction allocation.
Autopilot would sit on top of that surface: the same reserved budgets, operating
within the same immutable reaction constraints, but executed by declared rules instead
of by hand.

Administrative access would remain unable to change immutable launch fees or withdraw
migrated liquidity principal. Automating an action does not widen the set of actions
available.

## Open research questions

- How should strategy parameters be bounded so that automation cannot drain a
  reaction's allocation faster than it accrues?
- What conditions can be evaluated onchain cheaply enough to gate execution, and what
  belongs off-chain?
- How should a strategy behave across the ignition boundary, when revenue shifts from
  curve trading to the migrated pool?
- Should strategies be immutable once declared, or amendable within pre-declared
  bounds?
- How should a reaction's active strategies be made observable to participants?

## Research workspace

Current design work is organized across architecture notes, draft specifications and
RFCs. Everything below is concept and research: no implementation exists.

**Architecture**
- [Execution model](docs/execution-model.md) - Revenue, Strategy, Execution, and the ignition boundary
- [Control Room integration](docs/control-room-integration.md) - manual and automated administration over the same reserved budgets

**Draft specifications**
- [Strategy schema](specs/strategy.schema.json) - draft JSON Schema for a declared strategy
- [Example: buyback strategy](specs/examples/buyback-strategy.json) - a strategy conforming to the draft schema

**RFCs**
- [RFC 0001 - Trigger model](rfcs/0001-trigger-model.md) - eligibility, conditions, cooldowns and manipulability

**Open research**
- [Safety invariants](research/safety-invariants.md) - candidate invariants an implementation would have to satisfy

## Links

[Website](https://rearctor.io) · [Docs](https://rearctor.io/docs) · [GitHub](https://github.com/Rearctor) · [X](https://x.com/JoinRearctor) · [Telegram](https://t.me/rearctor)
