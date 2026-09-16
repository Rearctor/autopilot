# RFC 0001 - Trigger Model

- **Status:** Draft / Research. Not accepted, not scheduled, not implemented.
- **Scope:** How an execution becomes eligible, and how eligibility is separated from
  safety.

## Problem

A strategy must decide *when* to act using information available onchain, cheaply, and
without introducing a manipulation surface. Three forces pull against each other:

- **Responsiveness.** A trigger that only fires on a long timer wastes opportunity.
- **Cost.** Evaluating a trigger costs gas, and it is evaluated more often than it
  fires.
- **Manipulability.** Any trigger reading market state can be moved by someone who
  benefits from the resulting execution.

The third is the reason this is an RFC rather than a configuration detail. A buyback
that fires when volume crosses a threshold is a buyback that a third party can *cause*
to fire by generating volume - and then trade against.

## Proposed separation

```
  eligibility                       safety
       |                               |
   [ trigger ]  AND  [ cooldown ]  AND  [ conditions ]  --->  execute
       |                  |                   |
   "is it time?"   "not too soon?"      "is it safe?"
```

Keeping these separate has a concrete benefit: **conditions can be tightened without
changing when a strategy is considered**, and triggers can be tuned without weakening
safety checks. Collapsed into one predicate, every adjustment risks both.

## Candidate trigger kinds

### `accrued-threshold`

Fires when the strategy's claimable balance reaches a declared minimum.

- **Reads:** internal accounting only.
- **Manipulable?** Only by generating real fee revenue, which costs the manipulator
  more than it moves the trigger.
- **Assessment:** the most defensible candidate. Internal state, no oracle, no market
  read.

### `elapsed-interval`

Fires on a wall-clock interval.

- **Reads:** block timestamp.
- **Manipulable?** Marginally, by validators within normal drift.
- **Assessment:** safe but blunt. Fires whether or not there is anything worth doing.

### `volume-observed`

Fires when trading volume over a window exceeds a threshold.

- **Reads:** derived market data over a window.
- **Manipulable?** **Yes, directly.** An adversary can manufacture volume to force
  execution at a moment of their choosing.
- **Assessment:** the most useful and the most dangerous. Included in the draft schema
  to make the problem explicit, not because it is endorsed.

> **Open question.** Whether `volume-observed` should exist at all. If retained, it
> likely requires mandatory conditions - minimum pool liquidity, maximum price
> deviation - rather than optional ones. A trigger that is only safe when paired with
> a condition should arguably carry that condition in its own definition.

## Windowing

Any windowed trigger needs a window definition, and every choice leaks:

| Window | Manipulation cost | Responsiveness |
| :--- | :--- | :--- |
| Short (minutes) | Low - cheap to spike | High |
| Medium (hours) | Moderate | Moderate |
| Long (days) | High - sustained cost | Low |

There is no window that is simultaneously cheap to evaluate, responsive, and expensive
to manipulate. The draft schema constrains `window_seconds` to a 60s..30d range and
takes no position within it.

> **Under research.** Whether time-weighted accumulators are affordable enough onchain
> to be worth their complexity here, or whether windowed triggers should be dropped in
> favour of `accrued-threshold` alone.

## Cooldown as a distinct mechanism

Cooldown is not a trigger and not a condition. It bounds **frequency** regardless of
how eligible a strategy appears.

Its value is bounding worst-case behaviour: however wrong a trigger is, a strategy with
a six-hour cooldown can drain at most `max_spend_per_execution` per six hours. The
six-hour figure is a worked example. Illustrative only. Not protocol defaults or committed parameters. Cooldown
converts an unbounded failure into a rate-limited one.

**Candidate invariant:** `budget_limit / max_spend_per_execution * cooldown_seconds`
gives a lower bound on the time to exhaust a strategy's budget. A declared strategy
where that figure is implausibly short should be treated as suspect.

## Failure and retry

If a strategy is eligible but execution reverts:

- Does the cooldown clock reset?
- Is the attempt counted?
- Is the failure recorded onchain?

Resetting cooldown on failure means a repeatedly-failing strategy is rate-limited,
which is good. Not resetting means it retries immediately and burns gas, which is
worse. The draft assumes **cooldown resets on attempt, not on success** - but this also
means an adversary who can cause reverts can suppress a strategy indefinitely.

> **Open question.** Failure handling is unresolved, and the suppression vector above
> has no proposed mitigation.

## Alternatives considered

**Offchain trigger evaluation with onchain execution.** Trigger logic runs offchain; a
signed authorisation submits the execution. Cheaper and more expressive, at the cost of
a new trust assumption about who evaluates. Not explored here.

**No triggers - manual only.** Every execution requires an operator decision. This is
the current Control Room behaviour and is strictly safer. It is also not automation,
and would make this repository unnecessary.

## Related documents

- [Execution model](../docs/execution-model.md)
- [Safety invariants](../research/safety-invariants.md)
- [Draft strategy schema](../specs/strategy.schema.json)
