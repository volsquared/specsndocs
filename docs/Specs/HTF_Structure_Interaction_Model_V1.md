# HTF Volume Structure Interaction Model V1

## Purpose

Define a small deterministic model for tracking how price interacts with
completed higher-timeframe volume-structure zones such as HVN1 and HVN2.

The existing HTF Volume Structure Producer remains responsible for
identifying the zones. This model only answers: **what has price done
relative to a particular HTF structure zone?**

It does not decide entries, exits, reversals, stops, or profit taking.

The same interaction information can later support four consumers:

1.  Continuation-entry context and structural stop placement.
2.  Reversal qualification and structural stop placement.
3.  Broader market-context / market-theory construction.
4.  Profit-taking at destination structure.

Execution timeframe and structure timeframe are independent. A 1H HVN
can, for example, be projected onto a 1-minute or 5-minute execution
chart.

## Structure Identity

Each tracked structure should retain:

-   Source timeframe.
-   Source profile / completed HTF period.
-   Structure type, e.g. HVN1 or HVN2.
-   `ZoneLow`.
-   `ZoneHigh`.

Daily, 4H, 1H and other structures use the same interaction principles.
Their relative importance is downstream policy.

## Price Position

For each closed execution-chart bar:

``` text
Close > ZoneHigh  -> ABOVE
Close < ZoneLow   -> BELOW
otherwise         -> INSIDE
```

A wick or intrabar penetration does not by itself change the
completed-bar side classification.

## Breach Definition

A breach requires a completed candle to close beyond the complete zone.

### Upward breach

Price was previously established below the zone and subsequently:

``` text
Close > ZoneHigh
```

### Downward breach

Price was previously established above the zone and subsequently:

``` text
Close < ZoneLow
```

A wick through a boundary, penetration into the zone, or close inside
the zone is not a breach.

Bars spent `INSIDE` must preserve the last established external side so
that:

``` text
BELOW -> INSIDE -> ABOVE
```

is one upward breach, while:

``` text
ABOVE -> INSIDE -> BELOW
```

is one downward breach.

Repeated closes on the same external side do not increment the breach
count.

## Breach Count

`BreachCount` counts alternating completed breaches through the same
structure during the current interaction episode.

``` text
BELOW -> ABOVE   BreachCount = 1
ABOVE -> BELOW   BreachCount = 2
BELOW -> ABOVE   BreachCount = 3
```

Remaining on the same side for multiple bars does not add breaches.
Touches and closes inside the zone do not add breaches.

## Significant Move

A breach can produce meaningful displacement away from the structure.

The configurable threshold is:

``` text
X * ATR
```

Use the normal ATR available to the study for the bar being evaluated.
V1 does not add frozen-at-breach ATR semantics.

After an upward breach, measure from the upper boundary:

``` text
UpExcursion = HighestPriceSinceBreach - ZoneHigh
```

A significant upward move occurs when:

``` text
UpExcursion >= X * ATR
```

After a downward breach, measure from the lower boundary:

``` text
DownExcursion = ZoneLow - LowestPriceSinceBreach
```

A significant downward move occurs when:

``` text
DownExcursion >= X * ATR
```

The zone midpoint is not used.

## Chop Around Structure

Configuration:

``` text
N = alternating breach-count threshold
X = significant-move threshold in ATR
```

If:

``` text
BreachCount >= N
```

without the interaction episode producing the required `X * ATR`
significant move, classify interaction with that structure as:

``` text
IN_CHOP
```

This means price is repeatedly oscillating around this particular
structure without meaningful displacement. It does not necessarily mean
the entire market is globally in chop.

## Significant Move Ends the Oscillation Sequence

Once a breach produces at least `X * ATR` displacement, the current
back-and-forth oscillation sequence is complete.

A later return to the same HVN is a **retest / new interaction
episode**, not another breach in the previous chop sequence.

The return occurs on the first later completed bar whose range overlaps the
zone:

``` text
High >= ZoneLow AND Low <= ZoneHigh
```

The bar does not need to close inside the zone. Intrabar contact on a forming
bar is not confirmed until that bar completes. A gap completely across the
zone without completed-bar range overlap is not a return.

This distinguishes:

``` text
breach
-> reverse breach
-> reverse breach
-> no meaningful displacement
-> CHOP
```

from:

``` text
breach
-> >= X * ATR displacement
-> later return
-> RETEST / NEW EPISODE
```

## Minimal State and Facts

Keep formal state deliberately small:

``` text
UNTESTED
ACTIVE
IN_CHOP
DISPLACED
```

`DISPLACED` is the persistent post-significant-move phase: the prior episode
is complete and the model is waiting for a later return before starting a new
episode. This state is required to distinguish that phase from an active
oscillation sequence.

Useful facts may include:

``` text
CurrentSide
LastExternalSide
BreachCount
LastBreachDirection
LastBreachBarIndex
MaxExcursionSinceBreach
MaxExcursionSinceBreachATR
SignificantMoveOccurred
InteractionEpisodeId
InteractionState
```

Do not create separate V1 states for `ACCEPTED`, `REJECTED`,
`RECLAIMED`, `LOST`, `DOUBLE_INVALIDATED`, etc. Downstream consumers can
derive interpretations from the simpler facts if needed.

## Interaction Episode Lifecycle

``` text
Structure available
    |
    v
UNTESTED
    |
first relevant interaction / breach
    v
ACTIVE
    |
    +-- repeated alternating breaches without X*ATR --> IN_CHOP
    |
    +-- significant X*ATR move ----------------------> DISPLACED
                                                         |
IN_CHOP -- significant X*ATR move ----------------------+
                                                         |
                                                         v
                                               later return creates
                                               a new ACTIVE episode
```

`IN_CHOP` does not end or freeze the episode. Alternating completed breaches
continue increasing `BreachCount` beyond `N`, and significant displacement
continues to be evaluated. Any later `X * ATR` displacement transitions
`IN_CHOP` to `DISPLACED` and completes the episode.

A new episode resets episode-local breach/chop tracking while allowing
historical statistics to be retained separately if desired.

## Downstream Use Cases

### Continuation Entry

A Trigger may combine structure proximity/interaction with other entry
dimensions.

The structure may also provide structural invalidation for stop
placement, typically beyond the opposite zone boundary with any
tolerance defined downstream.

### Reversal Qualification

If the market has been moving strongly downward, an upward completed
breach of a meaningful HTF HVN can provide structural evidence before
considering a counter-directional long.

``` text
market falling
-> approach 1H HVN1 from below
-> Close > ZoneHigh
-> upward breach
-> reversal-long setup may become eligible downstream
```

This is not automatically a long signal. The inverse applies to a
downward reversal after a strong upward move.

### Market Context / Market Theory

A broader MCtx component may combine interaction histories across
structures such as previous-day HVNs, 4H/1H HVNs, previous-day high/low,
opening price, opening range, global/overnight range and other
structural references.

This larger model is outside V1.

### Profit Taking / Destination Structure

Trade management may use an approaching/touched HTF structure as
destination information when a position is already significantly
profitable.

The interaction model publishes structure-relative facts. It does not
determine how much to exit or whether to retain a runner.

## Separation of Responsibilities

### HTF Volume Structure Producer

Owns completed HTF volume-structure calculation:

``` text
HTF VAP/profile
-> buckets
-> HVN/LVN structure
-> completed structure records
```

### Structure Interaction Model

Owns deterministic price-relative facts:

``` text
side
breach
breach count
excursion
significant move
structure-local chop
interaction episode
```

### Market Context

May consume interactions from multiple structures plus other market
information to construct broader context.

### Trigger

May consume structure interaction plus other confluence to determine
whether a setup is actionable.

### Trade Management

May consume structure location/interaction as structural invalidation or
destination information.

No entry, exit, stop, or profit-taking decision is hard-coded into this
interaction model.

## Configuration

Minimum V1 parameters:

``` text
N = alternating breach count required for IN_CHOP; V1 default 4
X = significant displacement threshold expressed as an ATR multiple; V1 default 1.5
```

Both parameters remain configurable for empirical tuning.

## V1 Design Principles

1.  Keep the model small.
2.  Use completed-bar closes for breaches.
3.  Treat zones as areas, not single prices.
4.  Measure displacement from the breached outer boundary.
5.  Count alternating breaches, not touches.
6.  Distinguish repeated oscillation from a genuine move followed by
    retest.
7.  Use ATR-normalised displacement so the definition adapts to
    volatility.
8.  Keep interpretation and trading decisions downstream.
9.  Do not create states unless persistent state is genuinely required.
10. Let empirical testing justify future complexity rather than
    designing it in advance.

## Items for Codex / Implementation Review

Review the behavioural model above and challenge implementation details
including:

-   exact episode initialization/reset mechanics;
-   newly appearing or expiring HTF structures;
-   recalculation/reload reconstruction;
-   persistence and publication contract;
-   simultaneous multiple HTF zones;
-   invalid/missing ATR or structure data;
-   deterministic closed-bar processing order;
-   whether `ACTIVE` needs to be explicit;
-   historical/statistical retention after an episode completes.

Do not expand the behavioural state vocabulary unless a concrete
implementation or trading requirement justifies it.
