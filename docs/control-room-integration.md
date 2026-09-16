# Control Room Integration

**Status: Concept / Research.** Nothing described here is implemented, deployed, or
scheduled.

## Starting point

The Control Room is the existing surface for reaction administration, including the
reserved buyback and liquidity reinvestment budgets drawn from the reaction allocation.
Today that is manual: an operator decides and acts.

Autopilot would sit **on top of that surface**, not beside it. Same reserved budgets,
same limits, same immutable reaction constraints - executed by declared rules instead
of by hand.

The relationship matters because it bounds the design: **automating an action must not
widen the set of actions available.** If an operator cannot do something manually
through the Control Room, no strategy may do it either.

## Authority model

```
              +--------------------------------+
              |    Immutable reaction core     |   <- reachable by neither
              |    supply . fees . threshold   |
              |    migration . principal       |
              +--------------------------------+
                              ^
              +---------------+----------------+
              |     Reserved budgets (70%)     |   <- the only writable surface
              +--------------------------------+
                    ^                      ^
             manual |                      | declared rule
        +-----------+------+     +---------+----------+
        |   Control Room   |     |     Autopilot      |
        |    (operator)    |     |     (strategy)     |
        +------------------+     +--------------------+
```

Autopilot's authority is a **subset** of Control Room authority, never a superset.

## Manual and automated administration together

Both paths act on the same budgets, so they interact.

**Shared accounting.** A manual spend and an automated spend draw from one balance.
Budget checks must be against live remaining balance at execution time, not against a
figure cached when the strategy was declared.

**Precedence.** If a manual action and a strategy execution target the same budget in
the same block, one must lose. The draft assumes manual action takes precedence and the
strategy execution reverts cleanly on insufficient budget - consistent with treating
Autopilot as a convenience over the manual surface.

**Pause authority.** An operator must be able to disable a strategy. This is the one
place where automation is clearly subordinate: a strategy behaving unexpectedly needs a
stop that does not require a protocol upgrade.

> **Open question.** Is `enabled: false` sufficient, or is a global per-reaction
> "halt all automation" switch needed? A global halt is a single point of control and
> therefore a trust consideration; per-strategy disable is finer-grained but slower in
> an incident.

## What administrative access must never reach

Unchanged from the reaction's existing guarantees, and restated because automation is
exactly the place where such boundaries erode:

- immutable launch fee configuration;
- initial supply;
- ignition threshold;
- whether or when migration occurs;
- principal of the migrated Uniswap V4 position.

Automating an action does not widen the set of actions available.

## Observability

For an external participant to trust declared automation, the strategy set should be as
inspectable as the module set in a reaction manifest:

- which strategies exist;
- their budgets and remaining balance;
- their trigger and condition definitions;
- execution history with outcomes.

> **Under research.** Whether strategy definitions should be immutable once declared,
> amendable within pre-declared bounds, or freely editable by the operator. Free
> editing makes observability near-worthless - a participant can only see what the
> strategy is *now*, not what it will be.

## Related documents

- [Execution model](execution-model.md)
- [Safety invariants](../research/safety-invariants.md)
