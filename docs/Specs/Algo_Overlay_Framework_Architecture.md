# Algo Overlay Framework -- Architecture (Draft for Claude Validation)

## Objective

This framework is **not** a trading strategy.

It is a reusable architecture for expressing, testing and combining
trading ideas while keeping each responsibility isolated.

The philosophy is:

> **Measure → Interpret → Detect → Decide → Execute**

The framework is designed to maximise **context-aware decision making**,
rather than optimising individual entry signals.

------------------------------------------------------------------------

# Architecture

``` text
Market Data
    │
    ▼
Sensors
    │
    ▼
Market Context (constellation of small, independent context objects)
    │
    ├──────────────┐
    ▼              │
Trigger Strategy   │
    │              │
    └──────┬───────┘
           ▼
Trade Policy (decomposed)
    ├── EntryGate
    ├── PositionSizer
    ├── StopManager
    ├── TrailingManager
    └── ProfitManager
           │
           ▼
Order Management (candle baton manager — sequencing/state already handled)
```

Trade Policy is not a single component. Each sub-component reads only
the context objects it needs, and each is independently
calibratable/testable.

------------------------------------------------------------------------

# 1. Sensors

Sensors measure objective properties of the market.

They never make decisions.

Examples:

-   ATR
-   EMA Distance
-   VWAP Distance
-   Delta
-   Volume
-   Opening Range Width
-   MACD
-   Weekly ATR Consumed
-   Day Expansion
-   Higher Timeframe OHLC Position

Output should preferably include raw measurements.

Example

``` yaml
WeeklyATR:
  ATR: 420
  RangeCovered: 168
  PercentConsumed: 40
```

------------------------------------------------------------------------

# 2. Market Context

Market Context consumes sensor outputs and converts them into reusable
market facts.

Contexts do **not** recommend actions.

Contexts do **not** know who consumes them.

They simply describe the market.

Market Context is **not a single object**. It is a constellation of
small, independent context objects, each owned and produced by its own
logic, discoverable by consumers through a simple keyed lookup (no
consumer reaches into another producer's internals).

## Decay Class

Every context object carries a decay class and an age/valid-until
field, since different contexts go stale at different rates. The
consumer — not the context object — decides how much staleness it will
tolerate.

-   **STATIC / SESSION** — e.g. Weekly Expansion, Day Expansion. Valid
    for the session or week; recomputed on a defined trigger (weekly
    close, new session open), not on every bar.
-   **SLOW** — e.g. Trend Context, Auction Context. Valid for tens of
    minutes to hours. Consumers check age but don't need tick-level
    freshness.
-   **FAST** — e.g. Delta, Volume, ATR-derived sensors feeding
    anything trigger-adjacent. Must be current to the bar (or tick) or
    it is actively misleading.

Examples

## Trend Context

``` yaml
State: TREND
Strength: HIGH
DecayClass: SLOW
Age: 14m
```

## Auction Context

``` yaml
State: FAILED_LOWER_AUCTION
Strength: STRONG
DecayClass: SLOW
Age: 6m
```

## Weekly Expansion Context

``` yaml
PercentConsumed: 40
State: UNDER_EXPANDED
DecayClass: STATIC
ValidUntil: NextThursdayClose
```

## 4H Engulfing Context

``` yaml
Direction: BULLISH
Status: ACTIVE
DecayClass: SLOW
Age: 37m
```

Contexts are reusable by any downstream component.

------------------------------------------------------------------------

# 3. Trigger Strategy

A Trigger Strategy detects executable opportunities.

Examples:

-   OR Breakout
-   Bullish Engulfing
-   Bearish Engulfing
-   HVN Breakout
-   DOT Acceptance
-   Compression Breakout

A trigger answers only one question:

> "Has my setup occurred?"

It is deliberately unaware of:

-   risk
-   sizing
-   trailing
-   exits
-   other strategies

------------------------------------------------------------------------

# 4. Trade Policy (Decomposed)

This layer is **not a single component**. It is split into
independent sub-components, each combining:

-   Trigger
-   the specific Context object(s) it needs

to determine one piece of trade behaviour:

-   **EntryGate** — accept/reject entry
-   **PositionSizer** — sizing
-   **StopManager** — initial stop
-   **TrailingManager** — trailing behaviour
-   **ProfitManager** — scale-out / exits

Splitting these out means each is independently testable and
independently calibratable — recalibrating TrailingManager doesn't
risk perturbing EntryGate.

Importantly, context is **not only for entries**.

It can influence every stage of the trade lifecycle.

Examples

## Entry

Post-CPI context

→ Reject entry

------------------------------------------------------------------------

## Position Size

Globex context

→ Half size

------------------------------------------------------------------------

## Initial Stop

High volatility context

→ Wider initial stop

------------------------------------------------------------------------

## Trailing Stop

Active 4H Bullish Engulfing

→ Trail more slowly → Allow runners more room

------------------------------------------------------------------------

## Profit Taking

Extreme Trend Stretch

→ Reduce runner → Scale out earlier

------------------------------------------------------------------------

## Weekly Expansion

UNDER_EXPANDED week

→ Allow larger runners → Give breakout trades more room

OVER_EXPANDED week

→ Tighten management → Expect lower expansion probability

------------------------------------------------------------------------

# Key Principle

Context should influence **behaviour**, not just **entries**.

The same context may be consumed differently by different execution
policies.

Example

OR Breakout:

-   Uses Weekly Expansion to decide whether to participate.

Trailing Manager:

-   Uses 4H Engulfing to decide trailing aggressiveness.

Profit Manager:

-   Uses Trend Stretch to determine exit behaviour.

No context knows how it will be used.

------------------------------------------------------------------------

# Design Principles

## Sensors publish measurements.

## Context publishes interpreted market facts.

## Trigger Strategies detect opportunities.

## Trade Policy determines behaviour.

Each layer owns a single responsibility.

------------------------------------------------------------------------

# Example: Weekly Expansion

Hypothesis

If only 40--50% of Weekly ATR has been consumed by Thursday close,
Friday is statistically more likely to become an expansion day.

Implementation

Sensor:

``` text
Weekly ATR
Range Covered
Percent Consumed
```

Context:

``` text
UNDER_EXPANDED
```

Consumers may choose to:

-   participate in OR breakouts
-   hold runners longer
-   avoid fading breakouts

This is a hypothesis suitable for replay validation.

------------------------------------------------------------------------

# Example: Auction Asymmetry

Observed behaviour

1.  Previous candle high breached.
2.  Previous candle low breached.
3.  Sellers fail.
4.  Buyers reclaim.
5.  Price attacks highs again.

Context

``` text
FAILED_LOWER_AUCTION
```

Trigger

Bullish Engulfing (1m)

Execution

Enter long.

The edge comes primarily from the market context.

The trigger simply provides timing.

------------------------------------------------------------------------

# Conflict Resolution

Contexts do not conflict with each other — they are independent,
purely descriptive facts, and both can be true at once (e.g.
WeeklyExpansion = UNDER_EXPANDED and TrendStretch = EXTREME are not in
tension with each other; they are both accurate). Reconciling
multiple simultaneous truths into one action is a **policy problem**,
not a context problem, so the resolution logic lives inside whichever
Trade Policy sub-component consumes those contexts — never in a
shared, global "conflicting context pairs" table (that approach scales
O(n²) with the number of contexts and misplaces knowledge about usage
back into the context layer, which contradicts "no context knows who
consumes it").

Each policy sub-component expresses its own reconciliation as
declarative rules, e.g.:

``` text
TrailingManager — Rule A

IF WeeklyExpansion == UNDER_EXPANDED
THEN Trail = Loose
UNLESS TrendStretch == EXTREME
```

**Local fallback, not a global registry:** if a policy sub-component
encounters a combination of context states it has no rule for, it
skips the signal and logs it (which contexts, what they said, which
trade) rather than guessing. This fallback is scoped to that one
component — it doesn't accumulate into a shared table, and each
component's rule set only grows with what that component actually
needs to handle.

Log the skip rate per component. Low is a non-issue. High means a
policy component is missing rules it needs, not that the framework
needs a smarter arbitration engine.

------------------------------------------------------------------------

# Decision Cycle and Master Clock

None of the four layer contracts (Sensor, Market Context, Trigger
Strategy, Trade Policy) define *when* a decision actually gets made, or
how their outputs assemble into one coherent record of a single moment
in time. This section defines that. It is design only -- no ACSIL code
changes here.

## Master decision chart

One chart+study configuration designates the **master decision chart**,
using the same chart+study wiring convention already used throughout
this framework and the candle baton manager.

-   **Each newly completed bar on the master decision chart creates
    exactly one Chakra decision cycle.** This is edge-gated (the
    transition into a newly closed bar), never level-gated -- a cycle
    does not re-run repeatedly while that bar simply sits closed.
-   **Every other chart and component may update independently, on its
    own declared cadence, without starting an additional decision
    cycle.** A Sensor, Market Context, or Trigger Strategy may compute
    and publish on a closed-bar, intrabar, event-driven, session-
    boundary, or higher-timeframe cadence, entirely on its own schedule
    (each layer's own contract already defines this). None of that
    activity is, by itself, permitted to start a new decision cycle.
    Exactly one thing in the system is authorized to declare "a
    decision cycle happens now": the master clock, triggered solely by
    the master decision chart's own bar-close edge.

## What a decision cycle does

-   **Reads the latest valid, sufficiently fresh Sensor and Market
    Context snapshots.** This is the state consumption pattern already
    established for both layers -- current valid state, freshness
    tolerance chosen by whichever policy is doing the reading, never a
    single cycle-wide freshness rule imposed from outside.
-   **Considers Trigger Strategy events that fired since the last
    cycle.** A trigger firing *between* two decision cycles is not
    acted on immediately -- it sits sticky, per the event pattern,
    until the next cycle evaluates it. At that point the consuming
    policy checks the event's own published `EventValidUntilDateTime`
    against the cycle's evaluation time to decide whether it is still
    actionable. **A master clock slower than a given trigger's own
    expiry tolerance means that trigger can legitimately expire before
    any cycle ever observes it** -- this is expected, by design, not a
    gap: it is exactly why every Trigger Strategy event publishes its
    own actionable boundary rather than leaving expiry to be inferred.
-   **Evaluates only the Trade Policy components relevant to the
    current trade lifecycle**, not every policy unconditionally every
    cycle:

    ``` text
    Flat (no open position):
      EntryGate        -> relevant, evaluates
      PositionSizer     -> relevant only if EntryGate proposes an entry
      StopManager       -> NOT_APPLICABLE
      TrailingManager    -> NOT_APPLICABLE
      ProfitManager     -> NOT_APPLICABLE

    In a trade:
      EntryGate        -> NOT_APPLICABLE
      PositionSizer     -> NOT_APPLICABLE (sizing already fixed at entry)
      StopManager       -> relevant, evaluates
      TrailingManager    -> relevant, evaluates
      ProfitManager     -> relevant, evaluates
    ```

-   **Produces zero or more proposed actions.** A cycle where nothing
    needed to change is a completely normal, expected outcome -- not
    something to avoid, suppress, or treat as a non-event. See
    "Cycle outcomes" below.

## Intrabar decision cycles are named exceptions, not the default

The default and expected model is exactly one cycle per **closed**
master bar. A specific deployment may define an explicit, separately
named "Intrabar Decision Cycle" mode for a genuinely latency-sensitive
case -- but this must be a deliberate, documented, separately
configured exception, never an implicit side effect of some dependency
happening to publish intrabar data. This is the same stance the Trigger
Strategy contract already takes on intrabar firing: closed-bar is the
strong default, intrabar is an explicit, justified exception, never an
accident.

## Cycle outcomes

Two separate result categories, belonging to two separate actors -- never
merged into one list, because doing so is itself a violation of the
boundary this framework already established ("Trade Policy says what
should happen. Order Management decides how to enact it safely.").

**Trade Policy outcome** -- every Trade Policy component relevant to a
given cycle reports exactly one of five. (Amended after the `EntryGate`
contract pass demonstrated a real, unavoidable middle category: a
component can consume a trigger and produce a genuine, durable decision
record that is nevertheless *not yet* a complete, manager-ready action --
`ACCEPT`/`REJECT`/`SKIP_*` are all real decisions, but only an accepted
candidate that survives further staged composition, section "Decision
Cycle and Master Clock" -> `EntryGate` contract, ever becomes something
Order Management can validate.)

-   **`NOT_APPLICABLE`** -- the component was not relevant to this
    cycle's trade-lifecycle state, **and nothing was consumed** -- a true
    no-op cycle for this component. Example: `StopManager` while flat --
    there is nothing for a stop to manage, so it never even runs its own
    condition-checking logic. Note the added clause: a component that is
    substantively irrelevant but still performs required dependency
    bookkeeping (e.g. draining a stale upstream event so it can never
    resurface later, see `EntryGate` section 6) is *not* `NOT_APPLICABLE`
    for that cycle -- it reports `POLICY_DECISION` instead, because it did,
    in fact, produce a durable record. `NOT_APPLICABLE` means truly
    nothing happened, not "nothing important happened."
-   **`NO_ACTION`** -- the component *was* relevant and *did* evaluate,
    but nothing in this cycle gave it a reason to act -- there was nothing
    to consume at all. Example: `EntryGate` evaluates every cycle while
    flat, but if no Trigger Strategy event is pending this cycle, it
    reports `NO_ACTION` -- it looked, and there was genuinely nothing
    there to decide about.
-   **`POLICY_DECISION`** -- the component consumed something (a trigger
    occurrence, an upstream event) and recorded a genuine decision about
    it, but that decision is not by itself a complete, manager-ready
    action. `EntryGate`'s `ACCEPT`/`REJECT`/`SKIP_EXPIRED`/
    `SKIP_UNDECIDABLE`/`SKIP_LIFECYCLE` are all `POLICY_DECISION` at this
    generic level -- an `ACCEPT` additionally produces a durable candidate
    for a later staged-composition step to pick up (its own delivery
    sequence, distinct from this decision's own audit sequence -- see the
    `EntryGate` contract, section 10); a `REJECT`/`SKIP_*` produces no
    further output at all. This is the category most Trade Policy
    sub-components with a staged pipeline in front of Order Management
    will use far more often than raw `PROPOSED_ACTION`.
-   **`NO_CHANGE`** -- the component evaluated, has an existing
    recommendation from a prior cycle, and this cycle's fresh
    calculation confirms that recommendation is still correct. The
    policy component itself performs this comparison and reports
    `NO_CHANGE` directly -- it does not re-report `PROPOSED_ACTION`
    with identical content and rely on something downstream to notice
    nothing changed.
-   **`PROPOSED_ACTION`** -- a **complete, manager-ready action**: a new
    recommendation, or an update to an existing one, that needs nothing
    further before Order Management can evaluate it (e.g. move the
    trailing stop further -- `TrailingManager` has everything it needs to
    say this on its own). *This is the only outcome ever forwarded to
    Order Management.* A component whose own output is inherently
    incomplete (needs quantity, an initial stop, etc. from other
    components first) reports `POLICY_DECISION`, not `PROPOSED_ACTION`,
    until whatever assembles the complete action produces one.
    **(Amended, entry-handoff pass): for an entry proposal specifically,
    "Order Management" here means the Entry Authority component defined by
    `Chakra_OrderManagement_Handoff_Interface_Contract.md`, not
    `K_Util_Candle_Baton_Manager_V1` directly -- see this document's own
    "Order Management" section, below, and that contract's central
    finding. For a post-entry proposal (trailing stop, target), it is the
    existing candle baton manager, unchanged. Which one applies depends on
    trade lifecycle phase, exactly the same phase-gating this section
    already describes for which Trade Policy components evaluate at all.**

**Order Management disposition** -- separately, for each `PROPOSED_ACTION`
forwarded to it, the responsible Order Management component (see the note
above -- the Entry Authority for an entry proposal, the existing candle
baton manager for a post-entry one) reports its own result:

-   **`NO_INSTRUCTION`** -- the manager evaluated the proposal and
    produced nothing executable from it.
-   **`EXECUTABLE_INSTRUCTION`** -- the manager validated the proposal
    and translated it into something concrete and safe to actually
    submit. This is *how* to enact it safely.
-   *(Optionally, a distinct declined/modified disposition carrying a
    reason -- e.g. the proposal was rejected by the existing
    ratchet-only-forward stop guard, or was adjusted before submission.)*

This is not guaranteed to be a 1:1 pass-through: Order Management's own
existing safety nets (e.g. the ratchet-only-forward guard already
governing stop movement) can decline or modify a proposal. A
`PROPOSED_ACTION` with no resulting `EXECUTABLE_INSTRUCTION` is a valid,
expected combination, not an error.

**Only `PROPOSED_ACTION` outputs are forwarded to Order Management. The
responsible component validates each proposal and may produce, modify, or
decline an `EXECUTABLE_INSTRUCTION`. `NOT_APPLICABLE`, `NO_ACTION`,
`POLICY_DECISION`, and `NO_CHANGE` are never forwarded** -- there is
nothing for Order Management to validate in any of those four cases,
including `POLICY_DECISION`: even an `ACCEPT`-shaped decision is, by
definition, not yet complete enough for Order Management to act on. This
preserves the boundary already established for this framework:

> Trade Policy says what should happen.
> Order Management decides how to enact it safely.

Order Management's own contract discipline is unchanged by any of this
-- see "Order Management" below.

## `DecisionFrame`

A `DecisionFrame` is the logical, immutable record of one decision
cycle. "Logical" is deliberate: this section freezes what a
`DecisionFrame` *means* and how long it *lives*, not its physical
storage layout, which is a later, separate design pass.

A `DecisionFrame` contains:

-   **Cycle identity** -- a monotonic cycle sequence (never deliberately
    reset, same discipline already established for `PublicationSeq`/
    `EventSeq` elsewhere in this framework), the manually configured chart
    context this cycle runs in (corrected: not "the instrument" -- Chakra
    does not own an instrument-identity concept of its own; every
    component operates within whatever chart/instrument/account the user
    has manually configured it against, per the framework's own scope
    boundary), and the master decision chart's bar index/time for this
    cycle.
-   **Only the Sensor/Market Context snapshots and Trigger Strategy
    events actually consumed this cycle** -- a snapshot or event enters
    the frame only if at least one policy component that evaluated this
    cycle (i.e. was not `NOT_APPLICABLE`) actually read it. Nothing is
    captured "just in case."
-   **The relevant position snapshot** -- whatever position/trade-
    lifecycle state was relevant to this cycle's evaluations, not a
    full position history.
-   **Policy results** -- the outcome (`NOT_APPLICABLE`/`NO_ACTION`/
    `POLICY_DECISION`/`NO_CHANGE`/`PROPOSED_ACTION`) for every policy
    component evaluated, plus each component's own policy-specific
    decision detail where it produced a `POLICY_DECISION` (e.g.
    `EntryGate`'s `ACCEPT`/`REJECT`/`SKIP_*`).
-   **Proposed actions** -- the actual `PROPOSED_ACTION` payloads --
    complete, manager-ready actions only, not intermediate
    `POLICY_DECISION` output.
-   **Resulting executable changes** -- the `EXECUTABLE_INSTRUCTION`(s)
    Order Management actually produced, which may be fewer than the
    proposals (see above).

**A `DecisionFrame` must never become a giant shared object containing
everything Chakra knows.** It is scoped strictly to what this one cycle
actually used and produced -- the same "latest state, not a historical
bus, not a dumping ground" discipline already governing every other
persistent interface in this framework.

## Bounded runtime retention

Unbounded `DecisionFrame` history is never acceptable. Retention is
bounded by default, with any deeper history available only as an
explicit, deliberate opt-in:

-   **Current frame only**, by default.
-   **Optionally, the previous frame** -- enough for a policy or
    diagnostic tool that wants "did anything change since last cycle"
    without a full log.
-   **Optionally, a fixed-size diagnostic ring** -- bounded, never
    unbounded, matching the bounded-memory discipline already
    established everywhere else in this framework.
-   **Sparse persistent logging at meaningful decision points** -- log
    when a `PROPOSED_ACTION`, an `EXECUTABLE_INSTRUCTION`, or a
    *meaningful* `POLICY_DECISION` occurs (amended alongside
    `POLICY_DECISION`'s own introduction -- omitting it here would
    silently drop exactly the records needed to explain why no trade
    occurred: an accepted candidate, a rejected/expired/undecidable
    trigger, a lifecycle-consumed trigger). Not every routine
    `NOT_APPLICABLE`/`NO_ACTION` cycle, and not every ordinary
    `NO_CHANGE`. Which `POLICY_DECISION`s count as "meaningful" is each
    component's own call (an EntryGate's decisions essentially all
    qualify; a different future component might have some routine
    `POLICY_DECISION` cases not worth logging) -- this is the same "log
    transitions or first occurrence, not every call" discipline already
    required of every Sensor, Market Context, and Trigger Strategy in
    this framework, applied here at the cycle level.
-   **Full per-bar frame recording only in an explicit audit/debug
    mode** -- opt-in, off by default, matching the `DebugLogging`
    input convention already used throughout this codebase.

## Frozen operating principle

> Chakra is a deterministic, bar-driven market decision framework. Each
> newly completed bar on the configured master decision chart creates
> one decision cycle. During that cycle, Chakra reads the latest valid
> and sufficiently fresh sensor and market-context snapshots, detects
> fresh trigger occurrences, evaluates the trade-policy components
> relevant to the current trade lifecycle, and produces zero or more
> proposed actions. No action, an unfinished policy decision, and no
> change are all valid outcomes. Only complete, manager-ready proposed
> actions are forwarded to Order Management, which validates each
> one and may produce, modify, or decline an executable instruction.

> Components may operate at different source cadences. "Every bar"
> means every master decision bar; it does not require every sensor or
> context to recompute, republish, or log on every cycle.

------------------------------------------------------------------------

# Order Management

Sequencing/state handling for an **already-open** trade is covered by the
existing candle baton manager (`K_Util_Candle_Baton_Manager_V1`). Keep the
same contract discipline (decay class / staleness awareness) as the rest
of the pipeline where relevant.

**Amended (Chakra-to-Order-Management handoff contract pass,
`Chakra_OrderManagement_Handoff_Interface_Contract.md`): this section
originally implied one component could serve as all of "Order Management,"
including turning a `PROPOSED_ACTION` into a submitted entry order. That is
inaccurate for the actual codebase and is corrected here rather than left
as a standing misstatement.** `K_Util_Candle_Baton_Manager_V1` is, by its
own already-written design contract (`K_Candle_Baton_Manager_V1_Design.md`,
`Candle_Baton_Decision_Study_Contract_Draft.md`) and its actual
implementation, **a post-entry follower only** — it has no order-submission
logic at all ("Non-goals for V1: no order entry logic... This study is a
slave/follower manager only"). It discovers a trade exists by polling a
separate, already-existing master/leader study (currently
`K_Util_ModeBaton_V1`) for a trade-leader-ID handoff, and takes over
stop/target/BE/trail/close management from there. Turning a `PROPOSED_ACTION`
into a real order therefore requires a distinct **Order Management Entry
Authority** component — new or repurposed, not designed here — that performs
the actual submission and then publishes the *same* pre-existing
master-handoff persistent-variable contract the candle baton manager already
polls. `K_Util_Candle_Baton_Manager_V1` itself needs **no code change** for
this to work; it is already generic over whichever study is wired into its
`Master Trade Chart/Study` input. "Order Management," read precisely, is
therefore at least two physically separate components in this codebase: an
Entry Authority (decides/submits an entry, produces
`NO_INSTRUCTION`/`EXECUTABLE_INSTRUCTION`/declined) and the existing candle
baton manager (manages everything after a trade is confirmed live,
unchanged). See the handoff contract for the full design and the reasoning
behind keeping these as separate persistent-storage instances rather than
one.

------------------------------------------------------------------------

# Versioning

Classification formulas, state-transition structure, precedence, and other
fixed behavioral rules live as code constants. A change to any such rule is
a code commit, and the git hash versions that behavior.

Concrete component specifications may explicitly designate genuine
per-instrument tuning values as ACSIL `Input`s. Each such parameter must
document its units, default, valid range, and change lifecycle. If changing
it can alter historical state, the component must reconstruct from clean
state before publishing `VALID` under the new configuration. Live and replay
runs are comparable only when they use the same code version, market data,
and input values.

Manual ACSIL configuration is the current operating path. A future Bheshma
integration may supply these designated inputs from versioned JSON, but that
loading mechanism is not designed by this architecture revision and must not
be anticipated with interim configuration infrastructure.

------------------------------------------------------------------------

# Questions for Claude — Resolved

1.  Separation of responsibilities — clean. **Resolved.**
2.  Sensors/Context separation — clean, with per-object decay
    classing added (STATIC/SLOW/FAST). **Resolved.**
3.  Trade Policy — split into EntryGate / PositionSizer /
    StopManager / TrailingManager / ProfitManager. **Resolved.**
4.  Hidden coupling — conflict arbitration between contexts was the
    open one; resolved by keeping contexts pure/non-conflicting and
    moving reconciliation logic into each Trade Policy sub-component
    as local declarative rules, with a component-scoped skip-and-log
    fallback for unhandled combinations. **Resolved.**
5.  Extensibility — sound, provided Order Management keeps the same
    contract discipline as the rest of the pipeline. **Resolved.**
6.  Pattern fit — Blackboard (constellation of independent context
    objects) + lightweight Rules Engine at the Policy layer. ECS
    rejected as unnecessary complexity for this use case.
    **Resolved.**
7.  Replay/calibration — determinism via versioned code + versioned
    market data + matching values for explicitly designated tuning inputs
    (no per-candle snapshot storage needed); sparse event-triggered logging
    at decision points only (entries/exits), not every candle. Structural
    behavior remains code-versioned; per-instrument tuning may use documented
    ACSIL inputs with a deterministic change/reconstruction lifecycle.
    **Resolved.**
8.  Decision cadence — one master decision chart drives exactly one
    Chakra decision cycle per newly closed bar (edge-gated); every
    other component keeps its own cadence without starting additional
    cycles; intrabar cycles are a named, explicit exception, never a
    default or an accident. Two separate result categories, not one
    merged list: Trade Policy outcome (`NOT_APPLICABLE`/`NO_ACTION`/
    `POLICY_DECISION`/`NO_CHANGE`/`PROPOSED_ACTION`) and, separately,
    Order Management's own disposition on each forwarded proposal
    (`NO_INSTRUCTION`/`EXECUTABLE_INSTRUCTION`) — conflating the two into
    one list would itself violate the "Trade Policy says what should
    happen, Order Management decides how" boundary. `POLICY_DECISION`
    (added after the `EntryGate` contract pass) covers a genuine, durable
    decision that is not yet a complete manager-ready action — only
    `PROPOSED_ACTION` is ever forwarded to Order Management. The bounded,
    immutable `DecisionFrame` record is also defined in "Decision Cycle
    and Master Clock" above. `DecisionFrame`'s physical storage layout is
    intentionally deferred to a later pass — this round froze
    semantics, lifecycle, and bounded-retention rules only.
    **Resolved.**
