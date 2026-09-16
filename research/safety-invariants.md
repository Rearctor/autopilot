# Safety Invariants

**Status: Concept / Research.** These are candidate invariants for a system that does
not exist. They are written as assertions so that any future design can be tested
against them, not because any implementation has been verified.

An invariant here is a property that must hold **after every operation, for every
reachable state**. Where a property is only expected rather than enforceable by
construction, it is marked as such - the distinction is the point of this document.

---

## Class 1 - Protocol boundary

These concern what automation must never reach. All are intended to hold **by
construction**: the code path should not exist, rather than existing and being guarded.

### I-1. Automation cannot change immutable launch parameters

> For any reaction `r` and any sequence of Autopilot operations, the trading-fee
> configuration, initial supply, and ignition threshold of `r` are unchanged.

**Enforcement:** by construction. No Autopilot operation should accept these as
arguments or hold authority to modify them.

**Test shape:** fuzz arbitrary strategy definitions and execution sequences; assert the
three values are bit-identical before and after.

**Failure impact:** total. The immutability of fee configuration is a core reason a
reaction is inspectable. A violation invalidates the protocol's central claim.

### I-2. Automation cannot withdraw migrated liquidity principal

> For any reaction `r` past ignition, the principal of the migrated Uniswap V4 position
> is non-decreasing under all Autopilot operations.

**Enforcement:** by construction. There is no principal-withdrawal function in
Rearctor's contracts; Autopilot must not introduce one, directly or through a
privileged call into another component.

**Note on liquidity reinvestment.** A reinvestment action *adds* to the position. It
must be strictly additive. An implementation that removes and re-adds liquidity - even
atomically, even netting positive - creates a window in which principal is withdrawable
and therefore breaks this invariant as stated. The invariant should be read as
prohibiting the *capability*, not merely the net outcome.

### I-3. Automation cannot influence whether or when ignition occurs

> No Autopilot operation may prevent, delay, force, or condition migration.

**Enforcement:** by construction.

**Caveat, stated honestly:** a pre-ignition buyback spends reaction revenue on curve
purchases, which increases reserves, which moves the reaction toward the threshold. It
does not change the threshold and does not skip it - but it does influence *when*
arrival happens. Whether this violates I-3 in spirit is unresolved and is the main
argument for restricting token-trading strategies to Expansion. See
[execution-model.md](../docs/execution-model.md).

### I-4. Automation cannot touch the protocol share

> The 30% protocol share is never a source of funds for any strategy.

**Enforcement:** by construction - accounting separation before any strategy is
consulted.

### I-5. Automation cannot tax transfers

> Wallet-to-wallet transfers remain untaxed regardless of strategy configuration.

---

## Class 2 - Budget integrity

### I-6. An execution cannot exceed available reserved budget

> For every execution `e` of strategy `s`:
> `spend(e) <= min(remaining_budget(s), max_spend_per_execution(s))`
> evaluated at execution time, not at declaration time.

**Enforcement:** runtime check. This is the invariant most likely to be violated by a
plausible implementation error - specifically by caching a balance read.

**Test shape:** concurrent manual spend and strategy execution in the same block;
assert the sum never exceeds the balance that existed at block start.

### I-7. Cumulative spend never exceeds `budget_limit`

> `sum(spend(e) for all e in executions(s)) <= budget_limit(s)`

**Enforcement:** runtime, via monotonic `spent_total`.

**Interaction with I-6:** I-6 bounds a single execution, I-7 bounds the sequence.
Neither implies the other. An implementation checking only per-execution limits
satisfies I-6 and can still drain a reaction.

### I-8. Budget accounting is denomination-correct

> A strategy with a USDC budget cannot spend reaction tokens against that budget, and
> vice versa.

**Enforcement:** runtime. Relevant because post-ignition revenue arrives in both
denominations. A conversion introduced for convenience creates a price dependency
inside a budget check - an oracle surface where none is needed.

---

## Class 3 - Isolation

### I-9. Execution failure must not modify unrelated reaction state

> If execution of `s` reverts, then for all strategies `t != s`, `state(t)` is
> unchanged; and reaction state outside `s` is unchanged.

**Enforcement:** by construction, through state separation - not by try/catch around a
shared mutation.

**Test shape:** force a revert mid-execution with multiple strategies holding non-zero
state; assert every other strategy's `spent_total`, `last_execution_at` and
`execution_count` are unchanged.

### I-10. Execution failure must not revert unrelated user transactions

> A failing strategy execution cannot cause a trade, transfer, or migration submitted
> by an unrelated party to revert.

**Enforcement:** by construction. This is the strongest argument for the
settlement-style design in the Reaction Modules RFC: if strategies never execute inside
a user's transaction, this invariant is trivially satisfied.

### I-11. A strategy cannot read or write another strategy's state

> Strategies share only the budget balance, and only through the checks in I-6/I-7.

---

## Class 4 - Observability

These are weaker - they constrain what must be *visible*, not what must be *true*.

### I-12. Declared configuration is readable before it takes effect

> For any strategy `s` that executes at block `b`, the definition of `s` was readable
> at some block `< b`.

**Enforcement:** expected, not enforceable by construction unless strategy definitions
are immutable after declaration. If definitions are freely editable, this invariant
cannot hold and observability is substantially weakened. Unresolved - see
[control-room-integration.md](../docs/control-room-integration.md).

### I-13. Every execution is attributable

> Each execution emits a record identifying strategy, amount, denomination, outcome and
> initiating caller.

---

## Invariants deliberately not asserted

Stating what is *not* guaranteed is as useful as stating what is:

- **No liveness guarantee.** Nothing asserts a strategy will ever execute. Under a
  permissionless-keeper model with no reward, a strategy may remain eligible
  indefinitely and never fire.
- **No execution-quality guarantee.** Nothing asserts a buyback achieves a good price.
  Slippage bounds limit the worst case; they do not promise a good case.
- **No fairness guarantee between strategies.** If two strategies are simultaneously
  eligible against a shared budget, nothing specifies which is served.
- **No manipulation resistance for market-reading triggers.** See
  [RFC 0001](../rfcs/0001-trigger-model.md).

> **Open question.** Should the absence of a liveness guarantee be considered
> acceptable? A strategy that never runs is safe but useless, and a creator may
> reasonably have expected otherwise from something called "Autopilot".

## Related documents

- [Execution model](../docs/execution-model.md)
- [Control Room integration](../docs/control-room-integration.md)
- [RFC 0001 - Trigger model](../rfcs/0001-trigger-model.md)
