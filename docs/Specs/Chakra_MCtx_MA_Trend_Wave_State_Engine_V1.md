# Chakra — Market Context: MA Trend Wave State Engine (Revision 5)

Status: **implemented.**

Builds directly on:
- `Chakra_Common_Interface_Envelope_Contract.md` (approved, Revision 8) —
  `ComponentKind`, `K_CHAKRA_STATUS`, `PublicationSeq`, and PV-ownership
  ranges, unmodified.
- `Chakra_Sensor_Interface_Contract.md` — Sensor-consumption rules (state
  pattern, atomic validity) followed exactly.
- `Chakra_MCtx_Interface_Contract.md` (Revision 5) — identity,
  classification, freshness, transition, recalculation, and
  write-last-sequence rules followed exactly; nothing here loosens any of
  them.
- `Chakra_Sensor_Candle_MA_Raw_V1.md` (Revision 2) — the **sole, required**
  Sensor dependency this Market Context consumes for live operation.
  Revision 2 publishes the required `CandleClose` field at Float PV 7.
- `MA_Trend_Wave_State_Engine_Design_Note.md` — the **original conceptual
  source** (states, crossover-then-confirm principle, resume-attempt
  concept). This document is not a mechanical transcription of that note
  — it consolidates a full, subsequent design pass (chop handling,
  pullback/resume counters, invalid-data handling, reconstruction rules)
  into one self-contained, implementation-ready specification.
- **`Trend_Wave_State_Engine_V2.md` is explicitly rejected and was not
  used as a source for this document.** Its shortened logic is not
  carried forward in any form; any resemblance to this document in naming
  alone is coincidental, not derivation.

------------------------------------------------------------------------

# Changes from Revision 4

One pullback-whipsaw refinement:

1. **An opposite Fast/Slow crossover during `UNCERTAINTY` or
   `RESUME_ATTEMPT` now suspends the established directional cycle while
   the opposite direction is `PENDING_CONFIRMATION`.** Its counters and
   diagnostic indices are retained. If the candidate confirms, it becomes
   a fresh cycle at `WaveCount=1`; if Fast/Slow recross into the suspended
   direction first, the prior cycle returns to `UNCERTAINTY` and must use
   the ordinary close-beyond-all-MAs then later-`N` resume path. Consecutive
   cancelled countertrend candidates use the existing `Z` threshold to
   enter `IN_CHOP`; no new tuning input or public payload field is added.

------------------------------------------------------------------------

# Changes from Revision 3

One chop-recovery correction:

1. **`IN_CHOP` may now exit directly to `CONFIRMED` when objective
   directional separation re-establishes without a fresh Fast/Slow
   crossover.** If no crossover fired this bar, `FastMA != SlowMA` and
   `FastSlowDistanceATR >= N` begin a new confirmed cycle in the current
   Fast/Slow ordering direction. This prevents a context from remaining
   indefinitely chopped after directional MA expansion has objectively
   returned.

------------------------------------------------------------------------

# Changes from Revision 2

One contract-alignment correction:

1. **`N` and `Z` remain ACSIL tuning inputs under the MCtx interface's
   Revision 5 parameter boundary.** The earlier rationale incorrectly
   compared them with nonexistent Sensor ATR-multiplier inputs. That analogy
   is removed. Their defaults, validity checks, and mandatory full
   reconstruction on change remain exactly as specified; future Bheshma JSON
   population is deliberately outside this document.

------------------------------------------------------------------------

# Purpose

A descriptive Market Context state engine that classifies moving-average
trend structure — direction, confirmation, pullback, resume attempts, and
chop — from `Chakra_Sensor_Candle_MA_Raw_V1`'s raw measurements. It
describes what the MA geometry is currently doing. **It is not a Trigger
and not a trading strategy.**

------------------------------------------------------------------------

# Scope and ownership

**This Market Context owns:**
- Trend direction.
- Behavioral state.
- Wave lifecycle (confirmed-leg counting).
- Pullback / resume / failure counts.
- Diagnostic lifecycle bar indices.
- Derived MA geometry (`HighestMA`, `LowestMA`, `FastSlowDistanceATR`).

**This Market Context must never:**
- Emit an entry signal.
- Decide which pullback is tradable.
- Contain a maximum-tradable-pullback threshold (a "`Y`" of any kind) —
  that concept belongs entirely to a future Trigger, not here.
- Suppress or alter later pullbacks/resume attempts because some
  consumer did not act on an earlier one — this engine has no memory of,
  or awareness of, any consumer's trading decisions.
- Submit orders or interact with Mode Baton in any way.

**A future Trigger may use a transition into `RESUME_ATTEMPT` together
with `PullbackCount <= Y`** (`Y` defined entirely inside that future
Trigger, never here) as one input to a tradability decision. This
document does not design that Trigger and does not anticipate its exact
shape beyond noting that this is the intended consumption pattern.

------------------------------------------------------------------------

# Non-goals — explicitly excluded

- Any additional states, confidence scores, or trading classifications
  beyond the six behavioral states and their two-value direction defined
  in this document.
- Signal enums, trade-direction fields, or execution policy of any kind.
- Touch/crossover *history* beyond the specific counters and indices this
  document defines (no generic event log).
- A maximum-tradable-pullback threshold (`Y`) — explicitly a future
  Trigger's own concern (see "Scope and ownership").
- Session-boundary resets of any kind (section 15).
- Public or hidden historical output subgraphs (section 18).
- Any change to `K_Util_ModeBaton_V1.cpp`, `K_Util_Candle_Baton_Manager_V1`,
  or any order-management component.

------------------------------------------------------------------------

# 1. Identity and naming

- **`ComponentKind`** is always `K_CHAKRA_KIND_MCTX`.
- **`ProducerTypeID`**:
  ```cpp
  enum K_CHAKRA_MCTX_MATRENDWAVE_TYPE
  {
      K_CHAKRA_MCTX_MATRENDWAVE_TYPE_NONE = 0,
      K_CHAKRA_MCTX_MATRENDWAVE_TYPE_V1   = 1,
  };
  ```
- **`PayloadSchemaVersion`** (int32 PV 20) scoped per `ProducerTypeID`,
  standard envelope convention.
- **One study instance owns this one context concept** — trend/wave state
  and its supporting counters, indices, and geometry, published together
  as one coherent classification act, per the base MCtx contract's own
  "one coherent context concept" rule.

Naming: `K_Chakra_MCtx_MATrendWaveStateEngine_V1`.

**Cadence declaration**: closed-bar cadence only, matching its required
Sensor's own cadence. No intrabar evaluation, no revision of a
already-published bar's classification.

------------------------------------------------------------------------

# 2. Inputs

| Input | Default | Purpose |
|---|---|---|
| `Enabled` | Yes | Standard enable/disable. |
| `Source Sensor Chart/Study` | — | Chart+study reference to a `Chakra_Sensor_Candle_MA_Raw_V1` instance, for live operation (section 19). Same-chart wiring only, matching that Sensor's own local-chart-only design. |
| `N (Separation Confirmation Threshold)` | **`1.0`** | ATR-multiple threshold for initial confirmation, resume confirmation, and directional recovery from chop (sections 8, 11, 12). Must be `> 0` (section 3). Explicit per-instrument tuning input under `Chakra_MCtx_Interface_Contract.md` Revision 5; its formula and transition role remain fixed in code. **`1.0` is a reasonable first test default, a tuning choice, not an architectural decision** — an operator is expected to retune it per instrument. |
| `Z (Failure / Whipsaw Chop Threshold)` | **`2`** | Consecutive failed resume attempts or cancelled countertrend candidates before entering `IN_CHOP` (sections 7A and 12). Each failure class has its own counter; both use this threshold. Must be `>= 1` (section 3). Explicit per-instrument tuning input under the same Revision 5 boundary. **`2` is a reasonable first test default**, same caveat as `N`. |
| `Fast MA Study` / `Fast MA SG` | — | **Reconstruction-path input, independent of the Sensor's own wiring** — must be configured identically to whatever the source Sensor itself reads (section 21). Used only during full recalculation (section 20), never during live operation. |
| `Mid MA Study` / `Mid MA SG` | — | Same, reconstruction-path only. `MidMA` participates in `HighestMA` and `LowestMA` (section 4), but not in crossover direction (section 6) or Fast/Slow separation (section 4's `FastSlowDistanceATR`, sections 8/11/12). |
| `Slow MA Study` / `Slow MA SG` | — | Same, reconstruction-path only. |
| `ATR Study` / `ATR Study SG` | — | Same, reconstruction-path only. |
| `Debug Logging` | No | Standard debug-log toggle; logs decisions/transitions only (section 24). |

**No other classification thresholds are exposed.** `N` and `Z` are the
only two tunable classification parameters this document defines; every
other rule (crossover definition, pullback/resume-attempt/confirmation
conditions, chop entry/exit) is fixed, code-versioned logic.

------------------------------------------------------------------------

# 3. Raw inputs and validation

**Required per closed bar** (from the source Sensor during live
operation, section 19; from direct chart/study arrays during
reconstruction, section 20):

```text
FastMA
MidMA
SlowMA
ATR
CandleHigh
CandleLow
CandleClose
```

**Configuration validity** (checked once; if either fails, this Market
Context cannot function at all — `Status = ERROR`, `ReasonCode =
CONFIGURATION_INVALID`, checked before any per-bar evaluation):

```text
N > 0
Z >= 1
```

**Per-bar data validity** (checked every closed bar, before any
transition logic runs):

```text
ATR > 0
All of FastMA, MidMA, SlowMA, ATR, CandleHigh, CandleLow, CandleClose
    are finite (no NaN/Inf)
```

If per-bar validation fails: see section 17 (invalid-data gaps) for the
exact, non-destructive handling — this is never treated as a silent
warmup case once the dependency should otherwise be available.

------------------------------------------------------------------------

# 4. Derived values

**Computed inside this Market Context — the raw Sensor does not, and
must not, publish any of these three fields:**

```text
HighestMA = max(FastMA, MidMA, SlowMA)
LowestMA  = min(FastMA, MidMA, SlowMA)

FastSlowDistanceATR = abs(FastMA - SlowMA) / ATR
```

`MidMA` participates only in `HighestMA`/`LowestMA` — no formula in this
document compares `MidMA` against `SlowMA` or `FastMA` directly, unlike
`Chakra_MCtx_Candle_MA_Relation_V1.md`'s six-state relationship (a
different, unrelated context, not consumed here).

------------------------------------------------------------------------

# 5. Direction and behavioral state

**Two separate enums — never merged into one, and never sharing values:**

```cpp
enum K_CHAKRA_MCTX_MATRENDWAVE_DIRECTION
{
    K_CHAKRA_MCTX_MATRENDWAVE_DIRECTION_NONE = 0,
    K_CHAKRA_MCTX_MATRENDWAVE_DIRECTION_UP   = 1,
    K_CHAKRA_MCTX_MATRENDWAVE_DIRECTION_DOWN = 2
};

enum K_CHAKRA_MCTX_MATRENDWAVE_STATE
{
    K_CHAKRA_MCTX_MATRENDWAVE_STATE_NEUTRAL              = 0,
    K_CHAKRA_MCTX_MATRENDWAVE_STATE_PENDING_CONFIRMATION = 1,
    K_CHAKRA_MCTX_MATRENDWAVE_STATE_CONFIRMED            = 2,
    K_CHAKRA_MCTX_MATRENDWAVE_STATE_UNCERTAINTY          = 3,
    K_CHAKRA_MCTX_MATRENDWAVE_STATE_RESUME_ATTEMPT       = 4,
    K_CHAKRA_MCTX_MATRENDWAVE_STATE_IN_CHOP              = 5
};
```

**Valid `(State, Direction)` combinations — no other pairing is ever
published:**

| State | Direction |
|---|---|
| `NEUTRAL` | `NONE` |
| `IN_CHOP` | `NONE` |
| `PENDING_CONFIRMATION` | `UP` or `DOWN` |
| `CONFIRMED` | `UP` or `DOWN` |
| `UNCERTAINTY` | `UP` or `DOWN` |
| `RESUME_ATTEMPT` | `UP` or `DOWN` |

**`Direction` while `State == PENDING_CONFIRMATION` means the direction
currently being evaluated, not a confirmed trend.** A consumer must gate
on `State`, never assume `Direction != NONE` alone means a trend is
established.

------------------------------------------------------------------------

# 6. Crossover definitions

```text
Bullish crossover:
    PreviousFastMA <= PreviousSlowMA
    AND CurrentFastMA > CurrentSlowMA

Bearish crossover:
    PreviousFastMA >= PreviousSlowMA
    AND CurrentFastMA < CurrentSlowMA
```

`PreviousFastMA`/`PreviousSlowMA` are this document's own private,
persisted values from the last bar this engine actually evaluated a
crossover against (section 17 governs exactly when this baseline is set
versus silently re-primed).

**A crossover starts a pending cycle. It never confirms a trend by
itself** (section 8 governs confirmation, always on a later bar).

**From `NEUTRAL` or `IN_CHOP`** (`Direction == NONE`):
- Bullish crossover → `Direction = UP`, `State = PENDING_CONFIRMATION`.
- Bearish crossover → `Direction = DOWN`, `State = PENDING_CONFIRMATION`.

For `IN_CHOP`, a fresh crossover retains precedence over section 12's
direct directional-separation recovery. A crossover therefore still starts
`PENDING_CONFIRMATION` and cannot confirm on that same bar.

**From `CONFIRMED` or an ordinary, unsuspended `PENDING_CONFIRMATION`:** an
opposite crossover starts a fresh pending cycle in the opposite direction
(section 7), discarding the prior cycle.

**From `UNCERTAINTY` or `RESUME_ATTEMPT`:** an opposite crossover starts a
countertrend `PENDING_CONFIRMATION` candidate but suspends rather than
discards the established pullback cycle (section 7A).

------------------------------------------------------------------------

# 7. New-cycle initialization

**On the bar where a crossover starts a fresh cycle** (from
`NEUTRAL`/`IN_CHOP`, `CONFIRMED`, or an ordinary unsuspended
`PENDING_CONFIRMATION`, section 6):

```text
WaveCount           = 0
PullbackCount        = 0
ResumeAttemptCount    = 0
FailedResumeCount     = 0

CrossoverBarIndex        = current bar
TrendConfirmedBarIndex   = -1
CurrentPullbackBarIndex  = -1
ResumeAttemptBarIndex    = -1
```

`Direction`/`State` are set per section 6. This is the bar's entire
transition — no further logic (confirmation, pullback, etc.) evaluates on
this same bar (section 14).

------------------------------------------------------------------------

# 7A. Suspended pullback cycle and countertrend whipsaw

On an opposite crossover from `UNCERTAINTY` or `RESUME_ATTEMPT`:

```text
SuspendedDirection          = current Direction
SuspendedCrossoverBarIndex  = current CrossoverBarIndex
State                       = PENDING_CONFIRMATION
Direction                   = opposite direction
CrossoverBarIndex           = current bar
```

`WaveCount`, `PullbackCount`, `ResumeAttemptCount`, `FailedResumeCount`,
`TrendConfirmedBarIndex`, `CurrentPullbackBarIndex`, and
`ResumeAttemptBarIndex` are retained unchanged while the candidate is
pending. The suspended fields and `CountertrendWhipsawCount` are private;
the public payload schema is unchanged.

If the candidate reaches section 8 confirmation, it becomes a fresh cycle:
`WaveCount=1`, all pullback/resume/failure counters reset to `0`, the three
non-crossover lifecycle indices are initialized as section 8 specifies, and
the private whipsaw/suspension state clears.

If Fast/Slow instead recross into `SuspendedDirection` before candidate
confirmation:

```text
CountertrendWhipsawCount++

if CountertrendWhipsawCount >= Z:
    -> IN_CHOP
else:
    -> UNCERTAINTY, Direction = SuspendedDirection
    restore the suspended CrossoverBarIndex
```

The recross bar performs only that transition. Even when its candle closes
beyond all three MAs, section 10 may begin `RESUME_ATTEMPT` only on a later
closed bar. Confirmation then follows section 11; a recross never restores
`CONFIRMED` directly or increments `WaveCount`.

`CountertrendWhipsawCount` resets on any initial, resumed, or direct-from-chop
confirmation and on fresh-cycle initialization. It is deliberately separate
from public `FailedResumeCount`, although both failure classes use `Z`.

------------------------------------------------------------------------

# 8. Initial confirmation (`PENDING_CONFIRMATION` → `CONFIRMED`)

Evaluated only when the state active at bar entry is
`PENDING_CONFIRMATION` and no crossover fired this bar (section 14).

```text
Upward pending cycle (Direction == UP):
    FastMA > SlowMA
    AND FastSlowDistanceATR >= N
    -> CONFIRMED

Downward pending cycle (Direction == DOWN):
    FastMA < SlowMA
    AND FastSlowDistanceATR >= N
    -> CONFIRMED
```

**Actions on confirmation:**
```text
WaveCount             = 1
PullbackCount          = 0
ResumeAttemptCount      = 0
FailedResumeCount       = 0
TrendConfirmedBarIndex  = current bar
```

For a suspended countertrend candidate, this confirmation also discards the
suspended cycle, clears its private state and whipsaw count, and initializes
`CurrentPullbackBarIndex` and `ResumeAttemptBarIndex` to `-1`.

If not met: remains `PENDING_CONFIRMATION`, no changes. There is no
timeout — a pending cycle may remain `PENDING_CONFIRMATION` indefinitely
until either confirmed or superseded by an opposite crossover. For a
suspended candidate, that opposite recross follows section 7A rather than
starting section 7's fresh cycle.

------------------------------------------------------------------------

# 9. Pullback entry (`CONFIRMED` → `UNCERTAINTY`)

Evaluated only when the state active at bar entry is `CONFIRMED` and no
crossover fired this bar.

```text
Confirmed upward cycle (Direction == UP):
    CandleLow <= SlowMA
    -> UNCERTAINTY

Confirmed downward cycle (Direction == DOWN):
    CandleHigh >= SlowMA
    -> UNCERTAINTY
```

A candle may touch, pierce, or close through the Slow MA — this alone
does not reverse the trend, only marks it as uncertain.

**Actions:**
```text
PullbackCount++
CurrentPullbackBarIndex = current bar
```

**Increment only on the `CONFIRMED` → `UNCERTAINTY` transition itself.**
Multiple Slow-MA-touching candles during the same, continuing uncertainty
episode (including a failed resume attempt that returns to
`UNCERTAINTY`, section 12) remain **one** pullback — `PullbackCount` does
not increment again until a genuinely new `CONFIRMED` → `UNCERTAINTY`
transition occurs (which requires first returning to `CONFIRMED`, i.e. a
successful resume, per section 11).

If not met: remains `CONFIRMED`, no changes.

------------------------------------------------------------------------

# 10. Resume attempt (`UNCERTAINTY` → `RESUME_ATTEMPT`)

Evaluated only when the state active at bar entry is `UNCERTAINTY` and no
crossover fired this bar.

```text
Upward uncertainty (Direction == UP):
    CandleClose > HighestMA
    -> RESUME_ATTEMPT

Downward uncertainty (Direction == DOWN):
    CandleClose < LowestMA
    -> RESUME_ATTEMPT
```

**Actions:**
```text
ResumeAttemptCount++
ResumeAttemptBarIndex = current bar
```

**This is an early continuation indication, not a confirmed leg.** No
other counter or index changes.

If not met: remains `UNCERTAINTY`, no changes.

------------------------------------------------------------------------

# 11. Resume confirmation (`RESUME_ATTEMPT` → `CONFIRMED`)

**Confirmation is allowed only on a strictly later closed bar than the
one that entered `RESUME_ATTEMPT`**: `current bar > ResumeAttemptBarIndex`.
By construction (section 14's "no cascade" rule), this is automatically
satisfied whenever this section's logic is even consulted — the bar that
sets `ResumeAttemptBarIndex` never itself reaches this section, since
that bar's one allowed transition was already consumed entering
`RESUME_ATTEMPT` (section 10). This condition is stated explicitly
anyway, as the normative requirement this document's structure is built
to satisfy, not merely an incidental consequence.

Evaluated only when the state active at bar entry is `RESUME_ATTEMPT`, no
crossover fired this bar, and (section 12) the resume boundary was **not**
lost this bar:

```text
Upward (Direction == UP):
    FastMA > SlowMA
    AND FastSlowDistanceATR >= N
    -> CONFIRMED

Downward (Direction == DOWN):
    FastMA < SlowMA
    AND FastSlowDistanceATR >= N
    -> CONFIRMED
```

**Simple V1 policy, deliberately**: no requirement that separation first
fall below `N`, no requirement to recross `N` a second time, no
requirement to exceed the resume-attempt bar's own values, and no
requirement to exceed the previous swing peak. The same confirmation test
as section 8, reused unchanged.

**Actions on confirmation:**
```text
WaveCount++
TrendConfirmedBarIndex = current bar
FailedResumeCount       = 0
```

If not met (boundary held, separation still `< N`): remains
`RESUME_ATTEMPT` unchanged, no transition this bar.

------------------------------------------------------------------------

# 12. Resume failure and chop

Evaluated only when the state active at bar entry is `RESUME_ATTEMPT`, no
crossover fired this bar. **This section's boundary-loss test runs
*before* section 11's confirmation test — see section 13 for the full
precedence resolution.**

```text
Upward resume failure:
    CandleClose < HighestMA

Downward resume failure:
    CandleClose > LowestMA
```

**Equality leaves `RESUME_ATTEMPT` unchanged this bar** — neither failure
nor (by itself) confirmation; section 11's confirmation test is still
checked afterward on an equality bar, since equality means the boundary
was *not lost*.

**If the boundary was lost** (strict inequality above):

```text
NewFailedResumeCount = FailedResumeCount + 1
FailedResumeCount     = NewFailedResumeCount

if NewFailedResumeCount >= Z:
    -> IN_CHOP
else:
    -> UNCERTAINTY
```

**This is one transition, not a failure transition followed by a second
chop transition** — `RESUME_ATTEMPT` goes directly to either `UNCERTAINTY`
or `IN_CHOP` in a single step, per section 14's "at most one transition
per bar" rule.

**A resume failure that returns to `UNCERTAINTY` does not increment
`PullbackCount`** — it is a continuation of the same pullback episode
(section 9), not a new one.

**On entry to `IN_CHOP`:**
```text
Direction           = NONE
WaveCount            = 0
PullbackCount         = 0
ResumeAttemptCount     = 0
FailedResumeCount      = (the terminal NewFailedResumeCount that caused entry — retained, not reset)
```

Section 7A provides a second entry route to `IN_CHOP`: reaching `Z`
cancelled countertrend candidates. That route applies the same public chop
state/counter clearing above, retains the lifecycle indices diagnostically,
and clears the private suspended-cycle fields.

**All four lifecycle indices (`CrossoverBarIndex`, `TrendConfirmedBarIndex`,
`CurrentPullbackBarIndex`, `ResumeAttemptBarIndex`) are retained
unchanged, diagnostically, while `IN_CHOP`.** They are reset when the next
cycle starts, either through a fresh crossover (section 7) or the direct
directional-separation recovery below — never on chop entry itself.

**Direct directional-separation recovery from `IN_CHOP`:** evaluated only
when the state active at bar entry is `IN_CHOP` and no fresh crossover fired
this bar:

```text
if FastMA > SlowMA AND FastSlowDistanceATR >= N:
    -> CONFIRMED, Direction = UP

if FastMA < SlowMA AND FastSlowDistanceATR >= N:
    -> CONFIRMED, Direction = DOWN
```

This is a new confirmed cycle rather than a continuation of the chopped
cycle:

```text
WaveCount                 = 1
PullbackCount             = 0
ResumeAttemptCount        = 0
FailedResumeCount         = 0
CrossoverBarIndex         = -1
TrendConfirmedBarIndex    = current bar
CurrentPullbackBarIndex   = -1
ResumeAttemptBarIndex     = -1
```

`CrossoverBarIndex = -1` is deliberate because no crossover started this
cycle. The transition is direct to `CONFIRMED`; it does not manufacture a
`PENDING_CONFIRMATION` step after separation already meets `N`.

------------------------------------------------------------------------

# 13. Resume failure versus confirmation precedence

**Resolved explicitly, not left to implementation discretion.** Within
`RESUME_ATTEMPT`, on a bar strictly after `ResumeAttemptBarIndex`, after
first confirming no opposite crossover fired this bar (section 6 always
takes precedence over everything below — see section 14):

1. Test loss of the resume boundary (section 12).
2. **If the boundary was lost, process failure/chop (section 12) — stop,
   this is the bar's one transition.**
3. Otherwise (boundary held, including exact equality), test later-bar
   continuation confirmation (section 11).
4. Equality with `HighestMA`/`LowestMA` leaves `RESUME_ATTEMPT` unchanged
   *by the boundary-loss test itself* — it does not independently trigger
   anything — but does not block step 3's confirmation test from
   evaluating and potentially firing on that same bar, since the boundary
   was not lost.

**Resume failure wins over MA-separation confirmation whenever both
conditions are met on the same bar** — if the boundary was lost this bar,
confirmation is never checked, full stop, regardless of what
`FastSlowDistanceATR` happens to be that same bar.

------------------------------------------------------------------------

# 14. Global transition discipline

**Closed bars only. At most one state transition per closed bar. Never
cascade through a newly entered state on the same processing pass.**

**Corrected in this revision: the no-op check (section 17, Case 0) must
run *before* per-bar dependency fetching and validation (section 3),
not after.** An earlier draft placed "invalid configuration/data
handling" first in a 3-item precedence list and left the no-op check
implicitly gated on "every *valid* bar" (section 17), which meant a
repeated call for an already-successfully-evaluated bar could still
re-fetch the Sensor, observe a transient `NOT_READY`, and publish
`NOT_READY` for a bar this engine had already correctly published —
directly contradicting both the no-op rule itself and this engine's own
"never revise an already-published closed bar" cadence. **The full
master ordering, authoritative and superseding any narrower "precedence
list" framing elsewhere in this document:**

1. **`Enabled`/configuration control** (section 3's configuration
   validity, checked once) and **full-recalculation control**
   (`sc.IsFullRecalculation`, section 21/22) — highest precedence. If
   disabled, `Status = DISABLED`, nothing below runs. If a
   recalculation is in progress or has just been forced (section 17
   Case B), section 21 governs entirely instead of the steps below.
2. **Determine the current closed-bar index** (`CurrentBarIndex`).
3. **Same-bar no-op check (section 17, Case 0)**:
   `if CurrentBarIndex == LastEvaluatedBarIndex: return` — pure no-op,
   **before** any Sensor read or per-bar validation. This is what makes
   the no-op genuinely free of side effects: a repeated call for an
   already-evaluated bar never touches the Sensor, never re-validates,
   and never republishes, regardless of what the Sensor's *current*
   transient status happens to be.
4. **Fetch and validate this bar's data** (section 3) — only reached for
   a bar not already covered by step 3.
5. **If invalid**: publish `NOT_READY`/`ERROR` (section 17), do **not**
   advance `LastEvaluatedBarIndex`, return. (This is section 17's Case A
   when a later retry lands on the same still-`LastEvaluatedBarIndex+1`
   bar, or contributes to a Case B gap if the chart advances past this
   bar before it's ever successfully evaluated.)
6. **Classify remaining sequential coverage** (section 17) and route
   explicitly — an implementation must not fall through from this step
   into step 7 without one of these three branches having decided that:

   ```text
   if LastEvaluatedBarIndex == -1:
       execute section 18 first-bar initialization;
       do not run crossover or state-transition logic;
       publish the initialized snapshot;
       return.

   if CurrentBarIndex == LastEvaluatedBarIndex + 1:
       continue to steps 7-9.

   if CurrentBarIndex > LastEvaluatedBarIndex + 1:
       force reconstruction (section 17 Case B) and return.
   ```

   **The first-bar branch is not a degenerate case of "ordinary
   next-bar"** — it has no prior baseline for step 7's crossover check to
   compare against, so it is routed to section 18 and returns before
   step 7, never falling through into crossover/state logic without a
   baseline already established.
7. **Run state logic** — reached only via step 6's own
   `CurrentBarIndex == LastEvaluatedBarIndex + 1` branch, never via the
   first-bar or gap branches, both of which already returned. In this
   exact order: **opposite directional crossover** (section 6) first,
   regardless of which state is currently active — if it fires, section 7
   or 7A runs and nothing else this bar; only if no crossover fired, **the
   logic belonging to the state active at bar entry** (sections 8-12, as
   applicable).
8. **Run the mandatory successful-bar epilogue** (below) — unconditional
   once steps 4-7 completed successfully for this bar, regardless of
   which branch of step 7 fired or whether any transition occurred at
   all.
9. **Publish**: payload fields first, `PublicationSeq` last (section 23).

**Consequence, stated explicitly**: a bar that enters `RESUME_ATTEMPT`
this bar (via section 10, within step 7) cannot also confirm the
continuation (section 11) or fail (section 12) on that same bar —
those sections only ever evaluate the state active *at bar entry*, and
`RESUME_ATTEMPT` was not active at entry of the bar that just created it.

**Mandatory successful-bar epilogue (step 8) — the commit step every
successful closed-bar evaluation ends with, whether or not a transition
fired.** Section 17's entire gap-detection correction depends on
`PreviousFastMA`/`PreviousSlowMA`/`LastEvaluatedBarIndex` genuinely
advancing after every successful evaluation; this is the one normative
statement that makes that happen, not merely something the
private-variable table describes in passing. **Reached only after steps
4-7 complete for a bar in step 6's ordinary-next-bar case** — never for
the step-3 no-op (which returns before this point entirely), never for a
step-5 invalid bar, and never for a step-6 forced-reconstruction gap
(which also returns before this point) — **unconditionally**:

```text
PreviousFastMA        = CurrentFastMA
PreviousSlowMA         = CurrentSlowMA
LastEvaluatedBarIndex  = CurrentBarIndex
```

This epilogue applies identically regardless of *which* branch of step 7
actually fired (a crossover, a confirmation, a pullback, a resume
attempt/confirmation/failure, or none of the above/no transition) — the
commit is unconditional on "did evaluation complete successfully," never
conditional on "did something interesting happen."

------------------------------------------------------------------------

# 15. Counter definitions

- **`WaveCount`** — number of confirmed legs in the current directional
  cycle. Initial confirmation (section 8) sets it to `1`; each confirmed
  continuation (section 11) increments it. Reset to `0` on new-cycle
  initialization (section 7) and on chop entry (sections 7A/12), but
  retained while section 7A evaluates a suspended countertrend candidate.
- **`PullbackCount`** — number of distinct `CONFIRMED` → `UNCERTAINTY`
  episodes (section 9). A resume failure returning to `UNCERTAINTY`
  (section 12) is part of the same episode, not a new one.
- **`ResumeAttemptCount`** — number of `UNCERTAINTY` → `RESUME_ATTEMPT`
  transitions (section 10), including ones that later fail.
- **`FailedResumeCount`** — number of failed resume attempts since the
  last confirmed leg. Reset to `0` on a successful continuation
  (section 11) and on new-cycle initialization (section 7). **The
   terminal count that triggered chop entry is retained, unchanged, while
   `IN_CHOP`** (section 12) — it resets when the next cycle starts through
   either a fresh crossover or direct directional-separation recovery.
- **`CountertrendWhipsawCount`** — private count of countertrend candidates
  cancelled by a recross into the suspended direction before confirmation.
  It uses `Z` independently of `FailedResumeCount`, and resets on any
  confirmation or fresh-cycle initialization.

------------------------------------------------------------------------

# 16. Index definitions

- **`CrossoverBarIndex`** — bar containing the crossover that started the
  current (or, while `IN_CHOP`, most recent) directional cycle. While a
  section 7A candidate is pending it identifies the candidate crossover;
  cancellation restores the suspended cycle's original value.
- **`TrendConfirmedBarIndex`** — bar of the most recently confirmed leg
  (initial or resumed).
- **`CurrentPullbackBarIndex`** — first bar of the active (or, while
  `CONFIRMED`/`IN_CHOP`, most recent) pullback episode.
- **`ResumeAttemptBarIndex`** — bar that entered the active (or most
  recent) resume attempt.

**`-1` is the unset sentinel** for all four, used at new-cycle
initialization (section 7) for the three not immediately set that bar.
**All four are retained, unchanged, while `IN_CHOP`**, for diagnostics —
never cleared on chop entry itself. They are reset by the next fresh
crossover's initialization or by direct directional-separation recovery;
the latter uses `CrossoverBarIndex = -1` because no crossover occurred.

------------------------------------------------------------------------

# 17. Invalid-data gaps

**Corrected in this revision: retaining and republishing the pre-gap
state as `VALID` after simply re-baselining `PreviousFastMA`/
`PreviousSlowMA` is unsafe and is not this document's rule.** If an
actual crossover occurred *during* an invalid bar and was never observed,
silently resuming live evaluation from a re-primed baseline can leave the
engine reporting a stale, factually wrong `State`/`Direction` — including
indefinitely, if the true (unobserved) ordering never crosses back. The
correct rule distinguishes two genuinely different cases by their effect
on sequential bar coverage, not by simply "was the immediately preceding
bar invalid":

**An invalid bar must never:**
- Transition state.
- Change `Direction`.
- Increment or reset any counter (section 15).
- Change any lifecycle index (section 16).
- Invent or evaluate a crossover.
- **Advance `LastEvaluatedBarIndex`** (the corrected rule — see below).

It publishes the appropriate `NOT_READY` or `ERROR` status (section 23),
with every payload field neutral/stale-marked, leaving
`State`/`Direction`/counters/indices/`PreviousFastMA`/`PreviousSlowMA`
completely untouched internally — a private, ongoing condition, separate
from the publicly invalid envelope snapshot published for that bar.

**`LastEvaluatedBarIndex` is the last closed bar this engine successfully
completed ordinary evaluation for** (section 14's full transition check
ran to completion, whether or not a transition actually fired). It is the
single fact this section's correctness depends on.

**Checked in two stages, per section 14's own corrected master
ordering — the `Case 0` no-op comes first and is deliberately
*not* gated on validity, since checking it requires nothing more than
comparing two bar indices, no Sensor read or per-bar validation
involved:**

```text
Stage 1 (section 14 step 3, before any data fetch/validation):
  if CurrentBarIndex == LastEvaluatedBarIndex:
      -> already evaluated successfully — Case 0, pure no-op, return
         immediately; do not fetch or validate this bar's data at all

Stage 2 (section 14 steps 4-6, only reached if Case 0 did not apply):
  fetch and validate this bar's data (section 3) first;
  if invalid -> section 17's invalid-bar handling above, return
  if valid, classify sequential coverage:
    if LastEvaluatedBarIndex == -1:
        -> first evaluable bar (section 18) — not a gap
    else if CurrentBarIndex == LastEvaluatedBarIndex + 1:
        -> Case A, ordinary — proceed with section 14 steps 7-9,
           ending in the mandatory successful-bar epilogue
    else if CurrentBarIndex > LastEvaluatedBarIndex + 1:
        -> Case B, genuine gap — forced reconstruction, below
```

**This ordering is what makes the no-op genuinely free of side effects**
— a repeated call for an already-evaluated bar never reaches Stage 2 at
all, so a transient Sensor hiccup on a bar this engine has already
successfully published can never cause that bar to be republished as
`NOT_READY`. The corrected mnemonic: **check "have I already done this
bar" before asking "is this bar's data any good," not after.**

(`CurrentBarIndex < LastEvaluatedBarIndex` does not occur here — bar
indices are monotonic within one chart; a chart-level rebuild is
`sc.IsFullRecalculation`, handled entirely by sections 21-22, not this
section.)

**Case 0 — already evaluated successfully (`CurrentBarIndex ==
LastEvaluatedBarIndex`), the ordinary repeated-call case, checked in
Stage 1 above.** The study function is invoked again for a bar this
engine already completed evaluation for (e.g. an intrabar update call
that doesn't advance the closed-bar index, or any other repeated call
before the next bar closes). **Pure no-op, and checked *before* any
Sensor read for this bar**: no data fetch, no validation, no crossover
check, no state-specific logic (sections 6-12), no counter/index change,
no baseline update, and no republication — the already-published
snapshot for this bar remains current as-is, immune to whatever the
Sensor's transient status happens to be on this particular call. This is
distinct from Case A (which re-attempts a bar that was *not yet*
successfully evaluated, and does fetch/validate) and must not be
confused with it.

**Case A — transient, same-bar retry (no gap, the ordinary case section
17 exists to make safe).** A bar is invalid, then becomes valid on a
later call **while `CurrentBarIndex` is still exactly
`LastEvaluatedBarIndex + 1`** (no closed bar advanced past it in the
meantime — an ordinary calculation-precedence timing case, e.g. the
Sensor had not yet evaluated this exact bar). Because
`LastEvaluatedBarIndex` was never advanced while invalid, this bar is
simply retried: ordinary section 14 evaluation runs using the
**unmodified, still-correct** `PreviousFastMA`/`PreviousSlowMA` from the
last successfully evaluated bar — **no re-baselining is needed or
performed**, because no bar's true data was ever actually missed. On
completion, section 14's mandatory epilogue commits
`PreviousFastMA`/`PreviousSlowMA`/`LastEvaluatedBarIndex` for this bar,
same as any other successful evaluation.

**Case B — genuine gap (`CurrentBarIndex > LastEvaluatedBarIndex + 1`):
at least one closed bar was never successfully evaluated and can never
be revisited live.** This is the case that can hide a real crossover
(the scenario this correction exists to close). **Required handling:**

1. **Do not publish `VALID` from the retained pre-gap state.** Continue
   publishing `NOT_READY`, `ReasonCode = GAP_RECONSTRUCTION_PENDING`
   (section 24), for this bar.
2. **Force a full reconstruction.** Concrete ACSIL mechanism:
   ```cpp
   sc.FlagFullRecalculate = 1;
   return;
   ```
   requesting the platform re-invoke this study with
   `sc.IsFullRecalculation` set, which drives section 21's own replay
   (including its own clean initialization) — reading the underlying
   MA/ATR/Close arrays directly, sequentially, from the earliest usable
   historical bar, never the live Sensor's possibly-glitchy snapshot for
   the missed bar. This correctly recovers whatever crossover/transition
   truly occurred during the gap, because reconstruction reads the
   authoritative historical arrays, not the live path that failed.
3. **Continue publishing `NOT_READY` for every evaluation between setting
   `sc.FlagFullRecalculate` and the recalculation sweep's own final
   reconstructed snapshot** (section 21) — never a partial or
   intermediate `VALID` while reconstruction is pending or in progress.
4. **Only once reconstruction's own final snapshot publishes** does this
   engine resume ordinary live evaluation, from the corrected state
   reconstruction produced — exactly as section 21/22 already govern for
   any other forced reconstruction.

------------------------------------------------------------------------

# 18. First evaluable bar

`LastEvaluatedBarIndex == -1` — no prior successful evaluation exists at
all (section 17). This covers both true startup and, identically, the
state immediately after a reconstruction's own clean initialization
(section 21):

```text
State     = NEUTRAL
Direction = NONE
```

Store `FastMA`/`SlowMA` as the initial `PreviousFastMA`/`PreviousSlowMA`
baseline, set `LastEvaluatedBarIndex = CurrentBarIndex`. **Do not infer a
crossover from the current MA ordering** — e.g. `FastMA` already above
`SlowMA` on this very first bar is not treated as a bullish crossover; a
crossover requires an actual bar-to-bar transition to detect, which by
definition cannot exist on the first bar evaluated. This is not a
"gap" — there is no prior successfully-evaluated bar for it to be a gap
relative to.

**Publish this initialized snapshot** (`Status = VALID`, payload fields
first, `PublicationSeq` last, section 23) **and return** — per section
14 step 6's explicit routing, this bar never reaches step 7 (crossover/
state logic) or step 8 (the ordinary-next-bar epilogue); the baseline
assignments above are this bar's own complete, self-contained commit.

------------------------------------------------------------------------

# 19. Continuous session scope

**No automatic reset at trading-day boundaries, overnight periods, or
session transitions of any kind.** A cycle continues across all of these
until the state model itself changes or resets it (sections 6, 7, 12) —
never on a calendar/session boundary. **Any future session-reset mode is
new scope, not designed here, and must default off** if ever added.

------------------------------------------------------------------------

# 20. Live operation

The MCtx consumes the latest valid raw Sensor PV snapshot once per newly
closed bar (the state pattern — re-checked every evaluation, never an
event-style one-shot consume; `SourceSensorPublicationSeq`, published in
this context's own payload for provenance, is a performance/traceability
aid only, exactly as established in `Chakra_MCtx_Candle_MA_Relation_V1.md`
section 13 — never an event-dedup gate).

The raw Sensor (`Chakra_Sensor_Candle_MA_Raw_V1`, Revision 2) publishes
the required `CandleClose` at Float PV 7, so this document's live path has
no outstanding Sensor-payload dependency.

**This Market Context must verify, every evaluation:**
- Envelope/schema identity (`SchemaVersion`/`ComponentKind`/
  `ProducerTypeID`/`PayloadSchemaVersion`), in full, before reading any
  payload field.
- Sensor `Status == VALID`.
- Exact closed-bar freshness: the Sensor's `EvalBarIndex` equals this
  context's own current evaluation bar index (same-chart wiring, section
  2 — the same exact-match freshness rule established in
  `Chakra_MCtx_Candle_MA_Relation_V1.md` section 3).
- Every required payload field finite (section 3).
- `ATR > 0` (section 3).

If any check fails: section 17 governs (an invalid bar, not a crash or a
best-effort read).

------------------------------------------------------------------------

# 21. Historical reconstruction

**Keeps the Sensor raw and PV-only** — this document's reconstruction
path does not read the Sensor at all, live or historical.

**Explicit clean initialization is mandatory before replay begins —
reconstruction must never resume on top of whatever state happened to be
retained from live operation.** At the start of every forced
reconstruction (whether triggered by `sc.IsFullRecalculation`, section
22's configuration-change triggers, or section 17's gap-detected forced
reconstruction), reset every piece of state this engine owns to the exact
same values a brand-new, never-run instance would start with:

```text
State                   = NEUTRAL
Direction               = NONE

WaveCount               = 0
PullbackCount           = 0
ResumeAttemptCount      = 0
FailedResumeCount       = 0

CrossoverBarIndex       = -1
TrendConfirmedBarIndex  = -1
CurrentPullbackBarIndex = -1
ResumeAttemptBarIndex   = -1

SuspendedDirection          = NONE
SuspendedCrossoverBarIndex  = -1
CountertrendWhipsawCount    = 0

LastEvaluatedBarIndex   = -1
PreviousFastMA          = (neutral/unset)
PreviousSlowMA          = (neutral/unset)
```

**Then replay from the earliest usable historical bar forward**, using
the ordinary section 14-18 evaluation logic unmodified for every bar in
sequence — the first bar of the replay is itself handled by section 18
(`LastEvaluatedBarIndex == -1`), exactly as a live instance's true first
bar would be. Skipping this initialization would let a reconstruction
silently inherit stale counters/indices/baseline values from before the
reconstruction was triggered, defeating the entire purpose of forcing one.

**During full recalculation, this Market Context reads directly:**
- The configured Fast MA study array.
- The configured Mid MA study array.
- The configured Slow MA study array.
- The configured ATR study array.
- This chart's own High/Low/Close arrays (`sc.BaseDataIn`, or equivalent)
  — reconstruction reads these directly from the chart, independently of
  the Sensor payload.

**Runs exactly the same state-transition function, sequentially, bar by
bar, as live operation** (sections 6-17) — the same code path, not a
second, approximate reconstruction algorithm. Each historical closed bar
is evaluated in order, exactly as if it were arriving live, so the same
crossover/confirmation/pullback/resume/chop logic produces the same
result a true sequential live run would have produced.

**Publishes only the final reconstructed PV snapshot** — no per-bar
publication during the sweep (the same low-churn discipline established
throughout this framework: `NOT_READY` on recalc entry, no intermediate
churn, one final snapshot for the latest properly evaluated closed bar,
anchored explicitly to `sc.ArraySize - 2`, matching
`Chakra_Sensor_Candle_MA_Raw_V1.md` section 4's own anchoring rule and
its `LastEvaluatedBarIndex`-bypass safety note, applied here identically).
**No public or hidden historical output subgraphs are required or
produced** — the sequential internal replay exists to compute the one
correct final state, not to expose every intermediate historical state.

**Reconstruction source wiring must exactly match the Sensor's own live
source wiring** — the Fast/Mid/Slow MA and ATR study/SG inputs configured
on this Market Context (section 2) must point at the identical concrete
studies/subgraphs the source Sensor itself is configured to read.
**This is documented here as an operational/configuration invariant, not
enforced by any runtime arbitration or cross-validation machinery** — no
code compares this context's own MA/ATR wiring against the Sensor's and
flags a mismatch; getting this wrong is an operator configuration error,
the same posture this framework already takes toward other manual-wiring
invariants it doesn't runtime-enforce (e.g. the Mode Baton Signal
Publisher's own same-chart requirement).

------------------------------------------------------------------------

# 22. Reconstruction triggers

**Force a full reconstruction whenever any transition-affecting
configuration changes**, including at minimum:

```text
N
Z
Fast MA study/SG
Mid MA study/SG
Slow MA study/SG
ATR study/SG
```

**Do not continue existing counters/state across such a change** — a
changed input invalidates every previously computed classification,
since the same historical bars could now produce different states.
Recalculation (section 21) rebuilds from scratch, using the new
configuration for the entire sequential replay, not just from the point
of the change forward.

------------------------------------------------------------------------

# 23. Required published payload

Common-envelope fields used exactly as baselined — never duplicated into
this context's own payload. State-style: no `EventSeq`; `Int64 PV 10`
stays unused.

## Common-envelope fields

| Field | Namespace/PV | Value for this Market Context |
|---|---|---|
| `SchemaVersion` | Int32 PV 1 | `K_CHAKRA_ENVELOPE_SCHEMA_VERSION_CURRENT` |
| `Status` | Int32 PV 2 | `NOT_READY` / `VALID` / `ERROR` / `DISABLED` — sections 3, 17, 20 |
| `ComponentKind` | Int32 PV 3 | `K_CHAKRA_KIND_MCTX` |
| `ProducerTypeID` | Int32 PV 4 | `K_CHAKRA_MCTX_MATRENDWAVE_TYPE_V1` |
| `EvalBarIndex` | Int32 PV 5 | This context's own chart's bar index for the evaluated closed candle |
| `DecayClass` | Int32 PV 6 | **`FAST`**, always — an ordinary closed-bar classification |
| `ReasonCode` | Int32 PV 7 | Section 24 |
| `PublicationSeq` | Int64 PV 1 | Commit marker — advances on publish-on-material-change, written last. **State-style: a cache/traceability hint, never an event-dedup sequence.** |
| `EvalDateTime` | Datetime PV 1 | The evaluated closed bar's own datetime |
| `ValidUntilDateTime` | Datetime PV 2 | `0` (sentinel) — never `STATIC_SESSION` |

## Layer payload

| Field | Namespace/PV | Purpose |
|---|---|---|
| `PayloadSchemaVersion` | Int32 PV 20 | Envelope convention |
| `TrendWaveState` | Int32 PV 21 | Section 5, `0`-`5` |
| `TrendDirection` | Int32 PV 22 | Section 5, `0`-`2` |
| `WaveCount` | Int32 PV 23 | Section 15 |
| `PullbackCount` | Int32 PV 24 | Section 15 |
| `ResumeAttemptCount` | Int32 PV 25 | Section 15 |
| `FailedResumeCount` | Int32 PV 26 | Section 15 |
| `CrossoverBarIndex` | Int32 PV 27 | Section 16, `-1` sentinel |
| `TrendConfirmedBarIndex` | Int32 PV 28 | Section 16, `-1` sentinel |
| `CurrentPullbackBarIndex` | Int32 PV 29 | Section 16, `-1` sentinel |
| `ResumeAttemptBarIndex` | Int32 PV 30 | Section 16, `-1` sentinel |
| `SourceSensorChartNumber` | Int32 PV 31 | Provenance — the *configured* Sensor chart number (section 2's input), published regardless of whether the Sensor was actually read this pass |
| `SourceSensorStudyID` | Int32 PV 32 | Provenance — the *configured* Sensor study ID, same reasoning |
| `SourceSensorPublicationSeq` | Int64 PV 11 | The Sensor's own `PublicationSeq` at the moment this context last read it — traceability only, never a dedupe gate (section 20). **During reconstruction (section 21), set to `0`** — no live Sensor read occurred this pass, and `0` is never a value the Sensor's own monotonic-from-startup sequence produces during ordinary operation, so it is unambiguous as "not read this pass" rather than a stale carried-forward number. |
| `FastSlowDistanceATR` | Float PV 1 | Section 4 |
| `HighestMA` | Float PV 2 | Section 4 |
| `LowestMA` | Float PV 3 | Section 4 |

**No collision with the common envelope, and no field duplicated from
it** — `Status`/`ReasonCode`/`EvalBarIndex`/`EvalDateTime`/`PublicationSeq`
are read from the envelope only, never re-published in this table.

## Private persistent variables (never read cross-study)

| Field | Namespace/PV | Purpose |
|---|---|---|
| `LastEvaluatedBarIndex` | Int32, private range | The last closed bar this engine successfully completed evaluation for. Section 17's sequential-coverage/gap-detection key (`-1` sentinel = never evaluated) — **not merely a same-bar reprocessing guard**, its correctness is what section 17's whole invalid-gap-recovery correction depends on. |
| `SuspendedDirection` | Int32, private range | Established pullback-cycle direction retained while a section 7A countertrend candidate is pending; `NONE` means no suspension. |
| `SuspendedCrossoverBarIndex` | Int32, private range | Original cycle's crossover index restored when a countertrend candidate is cancelled. |
| `CountertrendWhipsawCount` | Int32, private range | Section 7A cancellation count; independently compared with `Z`. |
| `PreviousFastMA` | Float, private range | Section 6 crossover baseline. Left untouched by an invalid bar (section 17) — only ever updated by a successfully completed evaluation. |
| `PreviousSlowMA` | Float, private range | Section 6 crossover baseline, same discipline. |

**No output subgraphs of any kind, public or hidden** (section 21).

------------------------------------------------------------------------

# 24. Diagnostics

```cpp
enum K_CHAKRA_MCTX_MATRENDWAVE_REASON_CODE
{
    K_CHAKRA_MCTX_MATRENDWAVE_REASON_NONE                 = 0,   // = K_CHAKRA_REASON_NONE
    K_CHAKRA_MCTX_MATRENDWAVE_REASON_CONFIGURATION_INVALID = 1,
    K_CHAKRA_MCTX_MATRENDWAVE_REASON_SENSOR_UNWIRED         = 2,
    K_CHAKRA_MCTX_MATRENDWAVE_REASON_SENSOR_NOT_READY       = 3,
    K_CHAKRA_MCTX_MATRENDWAVE_REASON_SENSOR_STALE           = 4,
    K_CHAKRA_MCTX_MATRENDWAVE_REASON_INVALID_MEASUREMENT    = 5,   // non-finite payload/derived value, or ATR <= 0
    K_CHAKRA_MCTX_MATRENDWAVE_REASON_RECONSTRUCTION_SOURCE_INVALID = 6,
    K_CHAKRA_MCTX_MATRENDWAVE_REASON_GAP_RECONSTRUCTION_PENDING = 7,   // section 17 Case B — awaiting forced reconstruction's final snapshot
    K_CHAKRA_MCTX_MATRENDWAVE_REASON_CONTEXT_PUBLISHED      = 8
    // extend per future revision as needed, numbered above these common ones
};
```

**Written into the common-envelope `ReasonCode` field (Int32 PV 7,
section 23) — not a separate payload field.** Strictly self-referential:
describes only what this Market Context itself observed, never a
Trigger's disposition.

------------------------------------------------------------------------

# 25. Performance

- **One evaluation per newly closed bar**, live or reconstruction —
  never per study-update call.
- **The Sensor's snapshot is always valid state while fresh** (section
  20); `SourceSensorPublicationSeq` may be used to skip redundant
  recomputation, never to gate validity — the same rule established in
  `Chakra_MCtx_Candle_MA_Relation_V1.md` section 13.
- **Do not allocate arrays or containers** during live, per-bar
  evaluation. Reconstruction (section 21) necessarily reads study arrays,
  but does not allocate beyond that.
- **No subgraphs are created or cleared** (section 23).
- **Log only decisions/transitions when Debug Logging is enabled** — a
  `Status` change, `State`/`Direction` change, or `ReasonCode` change,
  never every evaluation.
- **Write all payload fields first; write `PublicationSeq` last.**

------------------------------------------------------------------------

# 26. Test matrix

Deterministic, bar-by-bar test cases. Each assumes `N` and `Z` are fixed,
reasonable values (e.g. `N = 1.0`, `Z = 2`) unless stated otherwise.

1. **First bullish crossover and confirmation.** Bar 10: `PreviousFastMA
   <= PreviousSlowMA`, `FastMA > SlowMA` this bar → `Direction=UP,
   State=PENDING_CONFIRMATION`, all counters `0`, `CrossoverBarIndex=10`,
   others `-1`. Bar 12: `FastMA>SlowMA`, `FastSlowDistanceATR >= N` →
   `CONFIRMED`, `WaveCount=1`, `TrendConfirmedBarIndex=12`.
2. **First bearish crossover and confirmation.** Symmetric to (1), using
   `Direction=DOWN` and the mirrored conditions throughout.
3. **Equality around crossover.** Bar `N-1`: `FastMA == SlowMA` exactly.
   Neither `PreviousFastMA <= PreviousSlowMA && CurrentFastMA >
   CurrentSlowMA` (bullish) nor the bearish mirror can fire *from* this
   bar as the "current" side while `Current` is itself the equal value —
   no crossover this bar; state unchanged. The equal bar's own
   `FastMA`/`SlowMA` still becomes the new `PreviousFastMA`/
   `PreviousSlowMA` baseline for the next bar's own check.
4. **Pullback counted once across several touching candles.** `CONFIRMED`
   (`UP`) at bar 20. Bar 21: `CandleLow <= SlowMA` → `UNCERTAINTY`,
   `PullbackCount=1`, `CurrentPullbackBarIndex=21`. Bars 22-24: each also
   has `CandleLow <= SlowMA`-equivalent conditions but state is already
   `UNCERTAINTY` (section 9 only fires from `CONFIRMED`) — no further
   `PullbackCount` increment; the engine instead checks section 10's
   resume-attempt condition each of those bars.
5. **Several resume attempts within one pullback episode.** Continuing
   (4): bar 25 `CandleClose > HighestMA` → `RESUME_ATTEMPT`,
   `ResumeAttemptCount=1`, `ResumeAttemptBarIndex=25`. Bar 26:
   `CandleClose < HighestMA` (boundary lost), `Z=2` so `1 < 2` →
   `UNCERTAINTY`, `FailedResumeCount=1`, `PullbackCount` still `1` (not
   incremented). Bar 28: `CandleClose > HighestMA` again →
   `RESUME_ATTEMPT`, `ResumeAttemptCount=2`, `ResumeAttemptBarIndex=28`.
6. **Successful later-bar continuation.** Continuing (5): bar 29 (`29 >
   28`): boundary held, `FastMA>SlowMA && FastSlowDistanceATR>=N` →
   `CONFIRMED`, `WaveCount=2`, `TrendConfirmedBarIndex=29`,
   `FailedResumeCount=0`. `PullbackCount` remains `1` (unaffected by
   confirmation).
7. **Failed resume below `Z`.** As in step 5's bar 26: `NewFailedResumeCount
   (1) < Z (2)` → `UNCERTAINTY`, not chop.
8. **Failed resume reaching `Z`, entering chop directly.** From
   `RESUME_ATTEMPT` with `FailedResumeCount` already `1` (`Z=2`): boundary
   lost again → `NewFailedResumeCount=2`, `2 >= Z(2)` → `IN_CHOP` directly
   (one transition, not `UNCERTAINTY` then `IN_CHOP`).
9. **Terminal failure count retained in chop.** Continuing (8):
   `FailedResumeCount` stays `2` while `IN_CHOP` persists — not reset to
   `0` on chop entry and not incremented further while chopped.
10. **Same-direction expansion exits chop.** While `IN_CHOP`
    (`Direction=NONE`), Fast remains above Slow after the prior upward cycle
    and no fresh crossover occurs. When `FastSlowDistanceATR >= N`, enter
    `CONFIRMED`/`UP` directly with `WaveCount=1`, all pullback/resume/failure
    counts reset, `CrossoverBarIndex=-1`, and
    `TrendConfirmedBarIndex=current bar`.
11. **Fresh crossover exiting chop.** While `IN_CHOP` (`Direction=NONE`):
    a bearish crossover occurs → section 6/7 fire exactly as from
    `NEUTRAL` — `Direction=DOWN`, `State=PENDING_CONFIRMATION`, all
    counters reset to `0` (including `FailedResumeCount`, overwriting the
    retained terminal count from step 9), all four indices reset
    (`CrossoverBarIndex=`current bar, others `-1`).
12. **Ordinary opposite crossover starts a fresh cycle.** From
    `CONFIRMED` or an unsuspended `PENDING_CONFIRMATION`, an opposite
    crossover takes precedence and starts section 7's fresh cycle with
    counters reset.
12A. **Pullback crossover suspends, then cancels.** `UNCERTAINTY/UP` at
    `WaveCount=2`; a bearish crossover enters `PENDING_CONFIRMATION/DOWN`
    while retaining the UP cycle's counters and indices. Before DOWN
    separation reaches `N`, a bullish recross increments private
    `CountertrendWhipsawCount` to `1` and restores `UNCERTAINTY/UP` with
    `WaveCount=2`. Even if that recross candle closes above `HighestMA`, it
    does not enter `RESUME_ATTEMPT` on the same bar.
12B. **Restored cycle uses ordinary resume logic.** On the next eligible
    bar after 12A, `CandleClose > HighestMA` enters `RESUME_ATTEMPT/UP`.
    A strictly later bar holding that boundary with `FastMA>SlowMA` and
    `FastSlowDistanceATR>=N` enters `CONFIRMED/UP`, increments
    `WaveCount` to `3`, and resets the private whipsaw count.
12C. **Countertrend candidate confirms.** From a suspended UP cycle, the
    pending DOWN candidate reaches `FastMA<SlowMA` and separation `>=N`
    before a bullish recross. It becomes `CONFIRMED/DOWN`, `WaveCount=1`,
    resets all pullback/resume/failure counters, and clears suspension and
    whipsaw state.
12D. **Countertrend cancellations reach chop.** With `Z=2`, a second
    countertrend candidate is cancelled before either direction confirms.
    `CountertrendWhipsawCount` reaches `2`, so the recross bar enters
    `IN_CHOP` directly and clears the public wave/pullback/resume counts.
13. **Simultaneous failure and separation confirmation.** `RESUME_ATTEMPT`
    (`UP`), bar where `CandleClose < HighestMA` (boundary lost) **and**
    `FastSlowDistanceATR >= N` both happen to hold. Per section 13:
    boundary-loss is tested first and fires; confirmation (section 11) is
    never reached this bar. Result: failure/chop processing (section 12),
    not confirmation, regardless of the separation value.
14. **No same-bar resume entry and confirmation.** The bar that transitions
    `UNCERTAINTY → RESUME_ATTEMPT` (section 10) cannot, on that same bar,
    also satisfy section 11's `current bar > ResumeAttemptBarIndex`
    requirement (they are numerically equal that bar) — confirmation is
    structurally impossible on the entry bar, verified directly from the
    inequality, not merely asserted.
15. **First valid bar.** The very first bar this engine ever evaluates,
    with e.g. `FastMA=105 > SlowMA=100` already true on that bar. Despite
    the ordering already favoring an uptrend, `State=NEUTRAL,
    Direction=NONE` — no crossover is inferred (section 18); only the
    `PreviousFastMA`/`PreviousSlowMA` baseline is set.
16. **Transient invalid bar, same-bar retry, no gap (Case A).**
    `CONFIRMED` (`UP`) at bar 50, `LastEvaluatedBarIndex=50`. Bar 51: this
    engine's first attempt finds the Sensor not yet `VALID` for bar 51
    (ordinary calculation-precedence timing) → `Status=NOT_READY`,
    `LastEvaluatedBarIndex` stays `50`. A later call this same bar: bar
    51 is now valid, and `51 == LastEvaluatedBarIndex(50)+1` — no gap.
    Ordinary section 14 evaluation proceeds using the **unmodified**
    `PreviousFastMA`/`PreviousSlowMA` from bar 50; no re-baselining
    occurs, none is needed. `LastEvaluatedBarIndex` advances to `51`.
17. **Genuine gap, forced reconstruction, correctly recovers a missed
    crossover (Case B — the defect this correction closes).**
    `CONFIRMED` (`UP`) at bar 50, `LastEvaluatedBarIndex=50`,
    `FastMA(50) > SlowMA(50)`. A true bearish crossover occurs on bar 51,
    but bar 51's data is invalid for this engine at the only time it is
    evaluated (`Status=NOT_READY`, `LastEvaluatedBarIndex` stays `50`) —
    it is never successfully re-evaluated before the chart advances to
    bar 52. Bar 52: valid, but `52 != LastEvaluatedBarIndex(50)+1(=51)`
    — **genuine gap detected.** Per section 17 Case B: this engine does
    **not** publish `VALID` from the retained `CONFIRMED`/`UP` state.
    Instead it forces a full reconstruction (section 21), which replays
    sequentially from the earliest usable bar using the direct MA/ATR/
    Close arrays — correctly observing bar 51's true bearish crossover
    (`PENDING_CONFIRMATION`/`DOWN`) and whatever bar 51 onward's true
    classification is, up through bar 52. Only after reconstruction
    completes does live publication resume, now correctly reflecting
    `DOWN` (or whatever state bar 51's real crossover actually produced)
    — never the stale, factually wrong `CONFIRMED`/`UP` the uncorrected
    design would have republished indefinitely.
18. **Continuous cycle across session boundaries.** A `CONFIRMED` cycle
    active at the last bar of one trading day remains `CONFIRMED`,
    unchanged, at the first bar of the next trading day (no session-based
    reset, section 19) — behaves identically to any other consecutive
    pair of closed bars.
19. **Reconstruction producing the same final state as sequential live
    processing.** A chart replayed from bar 0 through bar 500 via full
    recalculation (section 21, including its own clean initialization,
    reading MA/ATR/High/Low/Close arrays directly) must reach the
    identical `TrendWaveState`/`TrendDirection`/counters/indices at bar
    500 as a hypothetical bar-by-bar live run would have — verified by
    running the same state-transition function sequentially in both
    cases, per section 21's own "no second approximate algorithm"
    requirement.
20. **Reconstruction after changes to `N`, `Z`, or source wiring.**
    Changing `N` from `1.0` to `1.5` (or `Z`, or any MA/ATR study/SG
    input) forces a full reconstruction (section 22), starting from the
    same clean initialization as any other forced reconstruction (section
    21), that recomputes every bar's classification from scratch under
    the new configuration — a bar that confirmed under the old `N` may
    not confirm under the new one, and the final state reflects only the
    new configuration, with no carryover from the previous run's
    counters.

------------------------------------------------------------------------

# Dependencies

`Chakra_Sensor_Candle_MA_Raw_V1.md` Revision 2 now publishes
`CandleClose` at Float PV 7. The previously recorded live-path dependency
is therefore satisfied. Reconstruction (section 21) continues to read
chart `Close` data directly and never depends on the Sensor.
**No derived field (`HighestMA`, `LowestMA`, `FastSlowDistanceATR`) is
added to the Sensor** — all three remain exclusively this Market
Context's own computation (section 4), per explicit instruction.

------------------------------------------------------------------------

# Unresolved design questions

None of substance. `N=1.0`/
`Z=2` (section 2) are stated as reasonable first-test defaults, a tuning
choice rather than an architectural one — an operator is expected to
retune both per instrument, and neither default is claimed to be
validated against real market data.
