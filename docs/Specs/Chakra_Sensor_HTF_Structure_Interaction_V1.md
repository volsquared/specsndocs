# Chakra Sensor: HTF Structure Interaction V1

Status: **authoritative design specification; implementation not yet created.**

Planned ACSIL study:

`K_Chakra_Sensor_HTF_Structure_Interaction_V1`

## 1. Authority and Supersession

This specification converts the behavioural model in
`HTF_Structure_Interaction_Model_V1.md` and the subsequent design decisions
into one implementable V1 contract.

For this feature, it supersedes the parked draft
`Chakra_Sensor_HTF_HVN_Interaction_V1.md`. That older draft describes a
different component: one instance observing one current HVN, evaluating a
Tatva-style candle condition, and retaining no interaction episode. It is not
the implementation specification for this study.

The following remain authoritative upstream sources:

- `K_HTF_Volume_Structure_Producer_V1.cpp` owns HTF VAP processing, bucket
  construction, and HVN selection.
- `K_HTF_Volume_Structure_Consumer_V1.cpp` defines the current effective HVN
  boundary presentation and completed-profile timing semantics.
- `HTF_Structure_Interaction_Model_V1.md` defines side, breach, excursion,
  significant-move, chop, and episode behaviour.

If this document conflicts with the older parked Sensor draft, this document
wins for `K_Chakra_Sensor_HTF_Structure_Interaction_V1`.

## 2. Purpose

Build one lean, deterministic execution-chart Sensor that tracks how price
interacts with a deliberately small set of completed HTF HVN zones.

The Sensor publishes objective structure-relative facts. It does not decide:

- entries or reversals;
- whether a setup is permitted;
- stop-loss price;
- profit target or scale-out quantity;
- global market regime;
- whether a previously tested structure is likely to hold or fail.

Those interpretations belong to later Trigger, EntryGate, stop-policy,
target-policy, and Market Context components.

## 3. Intended Downstream Uses

The same interaction facts must support four broad uses without embedding any
of them in the Sensor:

1. Continuation-entry qualification and structural stop placement.
2. Reversal qualification and structural stop placement.
3. Profit taking at a destination HTF structure.
4. Broader multi-timeframe market context / market theory.

The broader Market Context study is deliberately last in the delivery order.
Its timed Daily-HVN touch and breach history is deferred by section 18.

## 4. Instance and Tracked Structure Model

One Sensor instance consumes one same-chart
`K_HTF_Volume_Structure_Consumer_V1` instance and therefore observes one HTF
profile stream only.

Normal deployment uses three independent Sensor instances:

```text
Daily Sensor instance -> Daily Consumer study
4H Sensor instance    -> 4H Consumer study
1H Sensor instance    -> 1H Consumer study
```

The Sensor does not need a timeframe input. The configured Consumer instance
defines the source. Downstream components select the appropriate Sensor study
instance explicitly.

Each Sensor instance tracks at most three HVNs:

| Profile recency | Retained zones | Maximum |
|---|---:|---:|
| latest completed profile | HVN1 and HVN2 | 2 |
| completed profile immediately before the latest | HVN1 only, when enabled | 1 |
| | **Total maximum per instance** | **3** |

For the Daily instance, `Retain Prior Profile HVN1` is configured `No`, giving
a maximum of two zones. For the 4H and 1H instances it is configured `Yes`,
giving a maximum of three zones each.

“Latest completed profile” means the profile currently projected by the
Consumer after its source HTF bar has completed. The optional older profile is
exactly one profile earlier, not every profile from the current day.

If a profile has no valid HVN2, its HVN2 slot is invalid. There is no fallback
to HVN1 and no artificial zone.

No additional historical zones are active in V1. This bound is intentional:
retaining too many historical nodes would manufacture a dense set of ceilings
and floors and make ordinary price movement appear to be chop.

## 5. Structure Identity

Every valid tracked structure has stable identity:

```text
SourceConsumerStudyID
ProfileID
HVNRank
```

The Consumer is required to be on the Sensor's chart, so chart number is
implicit and need not be duplicated in each identity. `ProfileRecency` and the
physical storage slot are not identity. A latest HVN1 that rolls into the
older HVN1 slot remains the same structure because its identity is unchanged.

Two spatially overlapping HVNs are not merged in V1. Two successive profiles
with identical numeric boundaries are still distinct structures.

## 6. Upstream Acquisition and Effective Boundaries

The Sensor reads one configured same-chart
`K_HTF_Volume_Structure_Consumer_V1` through its study ID and explicit
subgraph IDs. It does not access a Producer store and must not recompute VAP,
buckets, peaks, ranks, effective boundaries, or HVN candidacy.

The Consumer already owns Producer access, completed-profile selection,
HTF-to-execution-chart alignment, and effective HVN boundary presentation.

For each execution bar, the Sensor reads the configured Consumer arrays at the
same bar index. The Consumer's `Active Profile ID` identifies the projected
completed profile. On a profile-ID transition, the immediately preceding
profile's HVN1 values are available in earlier Consumer-array elements and may
be retained when `Retain Prior Profile HVN1 = Yes`.

The configured Active Profile Flags subgraph is the authoritative validity and
HVN-existence source. Boundary zero is not used as an existence heuristic.

ACSIL cannot reliably prove that an arbitrary configured study ID is a
specific concrete Consumer type. The Sensor validates array availability,
coverage, profile ID, flags, and boundaries; correct study wiring remains an
operator configuration responsibility.

## 7. Inputs

Minimum V1 inputs:

| Input | Purpose |
|---|---|
| `Enabled` | Enable or disable evaluation and publication. |
| `HTF Consumer Study ID` | Same-chart Consumer instance. |
| `HVN1 Low SG ID` | Consumer HVN1 lower boundary. Default `0`. |
| `HVN1 High SG ID` | Consumer HVN1 upper boundary. Default `1`. |
| `HVN2 Low SG ID` | Consumer HVN2 lower boundary. Default `2`. |
| `HVN2 High SG ID` | Consumer HVN2 upper boundary. Default `3`. |
| `Active Profile ID SG ID` | Consumer completed-profile identity. Default `8`. |
| `Active Profile Flags SG ID` | Consumer validity/existence flags. Default `9`. |
| `Retain Prior Profile HVN1` | Retain one-earlier profile HVN1. Daily `No`; 4H/1H `Yes`. |
| `ATR Study/Subgraph Reference` | Normal ATR on the execution chart. |
| `Chop Breach Threshold N` | Alternating breaches required for `IN_CHOP`. Default `4`. |
| `Significant Move ATR Multiple X` | Required boundary-relative displacement. Default `1.5`. |
| `Debug Logging` | Log validation and lifecycle changes. Default No. |

V1 defaults are:

```text
N = 4 alternating breaches
X = 1.5 ATR
```

Both remain configurable inputs for empirical tuning.

### 7.1 Latest HVN1-to-HVN2 gap

When both latest-profile HVNs and ATR are valid, publish the clear
nearest-boundary gap normalized by the current execution-bar ATR:

```text
HVN1 below HVN2: Gap = HVN2.ZoneLow - HVN1.ZoneHigh
HVN2 below HVN1: Gap = HVN1.ZoneLow - HVN2.ZoneHigh
Zones overlap:   Gap = 0

HVN1ToHVN2GapATR = Gap / ATR
```

The midpoint distance is not used because it includes zone width rather than
measuring the clear tradable space between the structures. Publish a separate
validity fact; the gap is unavailable when either latest-profile HVN or ATR is
invalid. The older retained HVN1 does not participate in this measurement.

## 8. Execution Cadence and Processing Order

Use manual looping and low calculation precedence because the study consumes
cross-study state and reconstructs multi-bar episodes.

Only completed execution-chart bars may alter interaction state. Intrabar
wicks and the forming bar must not confirm a touch, external side, breach,
significant move, or chop state.

For each newly processable closed execution bar:

1. Fetch each configured Consumer array once for the call and validate coverage.
2. Build the bounded desired structure set from the Consumer snapshot.
3. Reconcile identities and profile rollover before evaluating price.
4. Read the execution bar's OHLC and ATR once.
5. Evaluate valid structures in fixed slot order.
6. Commit the complete publication after all slots are evaluated.

Fixed slot order:

```text
0 latest profile HVN1
1 latest profile HVN2
2 older profile HVN1, when enabled
```

Slot order is for deterministic publication only and does not imply strategic
priority.

## 9. Price Position

For every valid tracked structure and completed execution bar:

```text
Close > ZoneHigh  -> ABOVE
Close < ZoneLow   -> BELOW
otherwise         -> INSIDE
```

Equality with either boundary is `INSIDE`.

`CurrentSide` is the current bar's classification. `LastExternalSide` is the
most recent `ABOVE` or `BELOW` and is preserved while bars close `INSIDE`.

## 10. Touch Fact

A completed-bar touch is a factual contact with the zone:

```text
High >= ZoneLow AND Low <= ZoneHigh
```

This includes a wick into or through the zone and any close inside it. Touch
does not change `LastExternalSide`, does not increment `BreachCount`, and is
not itself a breach.

V1 may publish `TouchThisBar` and `LastTouchDateTime` because these are useful
objective facts. It must not classify repeated touches as weakening support or
resistance; that interpretation belongs to the later Market Context study.

## 11. Establishment and Episode Initialization

A newly tracked structure begins:

```text
InteractionState = UNTESTED
CurrentSide = classification of the current closed bar
LastExternalSide = NONE
BreachCount = 0
SignificantMoveOccurred = false
```

Initialization rules:

- If the first evaluated close is `ABOVE`, establish `LastExternalSide = ABOVE`.
- If the first evaluated close is `BELOW`, establish `LastExternalSide = BELOW`.
- If the first evaluated close is `INSIDE`, retain `LastExternalSide = NONE`
  until a later close establishes an external side.
- Initial establishment is not a breach.
- A first touch without a breach may move the interaction from `UNTESTED` to
  `ACTIVE`; `BreachCount` remains zero.
- A structure may also become `ACTIVE` on its first breach.

An interaction episode exists for one stable structure identity. Profile
rollover does not rewrite the identity of a retained structure.

## 12. Breach Definition

A breach is a completed close beyond the complete zone after price was
previously established on the opposite external side.

Upward breach:

```text
LastExternalSide == BELOW
AND CurrentSide == ABOVE
```

Downward breach:

```text
LastExternalSide == ABOVE
AND CurrentSide == BELOW
```

Bars closing `INSIDE` preserve `LastExternalSide`, so:

```text
BELOW -> INSIDE -> ABOVE
```

is one upward breach.

After a breach, update `LastExternalSide` to the new external side. Repeated
closes on that same side do not create additional breaches. A wick, touch, or
close inside the zone is never a breach.

No candle-colour, body-percentage, Tatva proximity, or gap filter applies.

## 13. Breach Count and Direction

`BreachCount` is episode-local and counts alternating completed breaches
through one structure:

```text
BELOW -> ABOVE  = 1, direction UP
ABOVE -> BELOW  = 2, direction DOWN
BELOW -> ABOVE  = 3, direction UP
```

Because a breach always changes the established external side, valid breaches
naturally alternate. Publish at least:

```text
BreachThisBar
LastBreachDirection
LastBreachBarIndex
LastBreachDateTime
BreachCount
```

## 14. Significant Move

After an upward breach:

```text
MaxPriceSinceBreach = maximum completed-bar High since the breach bar
UpExcursion = MaxPriceSinceBreach - ZoneHigh
```

After a downward breach:

```text
MinPriceSinceBreach = minimum completed-bar Low since the breach bar
DownExcursion = ZoneLow - MinPriceSinceBreach
```

Use the normal execution-chart ATR value for each bar being evaluated:

```text
SignificantMoveThisBar = Excursion >= X * ATR[bar]
```

V1 does not freeze ATR at the breach. Excursion is measured from the breached
outer boundary, never the zone midpoint.

The breach bar's completed High or Low participates in excursion measurement.

## 15. Structure-Local Chop

Set:

```text
InteractionState = IN_CHOP
```

when:

```text
BreachCount >= N
AND SignificantMoveOccurred == false
```

This is chop around one specific structure. It is not global market chop and
does not prohibit trading by itself.

`IN_CHOP` does not freeze or complete the episode. While the structure remains
`IN_CHOP`:

- continue updating `CurrentSide` and `LastExternalSide`;
- continue counting every subsequent alternating completed breach;
- continue measuring excursion from the latest breach;
- continue testing for `X * ATR` significant displacement.

`BreachCount` is not capped at `N`. `N` is only the threshold at which the
still-active oscillation episode becomes classified as `IN_CHOP`.

If the significant-move threshold is first satisfied on the same completed
bar that would otherwise satisfy the chop threshold, significant move takes
precedence: transition directly to `DISPLACED` and do not publish `IN_CHOP`
for that bar.

## 16. Significant Move and Retest Lifecycle

When the required significant move occurs while the episode is either
`ACTIVE` or `IN_CHOP`:

1. Mark `SignificantMoveOccurred = true` for the completed episode.
2. Record completion bar index, datetime, direction, and maximum excursion.
3. Set `InteractionState = DISPLACED`.
4. End the current oscillation sequence.
5. Do not count a later return as another breach in the completed sequence.

`DISPLACED` is the persistent post-completion phase. While it applies:

- preserve the completed episode's `BreachCount` and completion facts;
- do not increment `BreachCount`;
- do not start another breach sequence;
- wait for a later return to the same retained zone.

A later contact with the same retained HVN begins a new interaction episode.
The return is detected on the first later completed bar whose range overlaps
the zone:

```text
High >= ZoneLow AND Low <= ZoneHigh
```

The close does not need to be inside the zone. A forming-bar touch does not
qualify. A gap completely across the zone whose completed range does not
overlap it is not a return under this rule.

On that return:

```text
EpisodeId += 1
BreachCount = 0
InteractionState = ACTIVE
SignificantMoveOccurred = false
```

The completed episode's historical count may be copied to diagnostics before
reset, but `BreachCount` in the new episode is zero.

Initialize `LastExternalSide` from the side established immediately before the
return so a subsequent full traversal can be recognized deterministically.

The fact that a good breakout later returned may be interpreted downstream as
failed acceptance or reduced breakout quality. The Sensor records the return;
it does not make that judgment.

## 17. Profile Rollover and Expiry

When a new profile completes:

1. If `Retain Prior Profile HVN1 = Yes`, the former latest profile's HVN1
   becomes the older retained HVN1 and keeps its identity and full interaction
   state.
2. The former latest profile's HVN2 expires.
3. The former older HVN1 expires.
4. The new latest profile contributes new HVN1 and optional HVN2 identities
   with fresh interaction state.

If `Retain Prior Profile HVN1 = No`, both structures from the former latest
profile expire and only the new profile's HVN1 and optional HVN2 are tracked.
This is the intended Daily-instance configuration. Historical Daily event
retention belongs to the later Market Context study.

State must follow identity, not slot number. Moving HVN1 from the latest slot
to the older slot must not reset its episode.

Expired structures are removed from the active publication. V1 does not retain
their full event history internally beyond what is needed to publish the
current bounded set.

## 18. Deferred Daily Timed History and Market Context

The final broader Market Context study will need to know that, for example, a
Daily HVN was first tested at 05:00 and revisited at 14:00.

That future study must distinguish at least:

- touch from above or below;
- upward and downward completed breach;
- event datetime;
- episode identity;
- whether significant displacement followed;
- whether price later returned;
- elapsed time since the prior test.

This V1 Sensor therefore publishes current/last touch and breach datetimes and
episode facts, but it does **not** implement an unbounded Daily event history,
weakening-level score, breakout-quality judgment, or macro classification.

The timed Daily history and its MCtx interpretation will be specified and
built last, after interaction behaviour is verified.

## 19. Recalculation and Reconstruction

Persistent runtime state alone is insufficient because chart reloads and full
recalculations must reproduce the same result.

On a full recalculation:

1. Clear all interaction state owned by this Sensor.
2. Fetch and validate the configured Consumer arrays.
3. Replay closed execution bars in chronological order, using each bar's
   Consumer profile ID, flags, and boundary values to reconstruct the structure
   set eligible at that time.
4. Apply the same rollover, side, breach, excursion, chop, completion, and
   retest rules used live.
5. Publish the final reconstructed bounded state after the sweep.

The implementation must not evaluate today's latest structures against bars
that occurred before those structures became completed and available.

Consumer array coverage may limit how far reconstruction can go. V1 must
reconstruct deterministically from the array history actually available and
report a diagnostic readiness/coverage fact rather than invent missing
history.

## 20. Invalid Data Behaviour

### Invalid structure record

Mark only the affected structure slot invalid. Do not substitute another rank
or retain stale boundaries under a new profile identity.

### Missing or invalid Consumer reference

Publish the Sensor instance as unavailable and preserve no newly inferred
structure state. Distinguish an unreachable study/array from an ordinary valid
profile that does not contain HVN2.

### Invalid ATR

Side, touch, and breach facts can still be determined without ATR, but
significant-move and chop progression depend on ATR. For a completed bar with
invalid/non-positive ATR:

- do not declare significant move;
- do not newly declare `IN_CHOP` on that bar;
- do not reset valid existing state;
- publish `ATRValid = false` and the relevant diagnostic.

When ATR becomes valid, normal evaluation resumes. This avoids fabricating a
volatility-normalized conclusion while preserving objective side history.

### Invalid OHLC

Do not evaluate any structure for that execution bar. Preserve prior state and
publish a diagnostic.

## 21. Publication Contract

The Sensor must publish one atomic, versioned, same-chart shared state record
for the maximum three structure slots. Use a persistent shared-store contract
with explicit magic number and schema version so downstream ACSIL components
can validate it before reading.

The shared record must use fixed-capacity storage; no per-update allocation is
required after initialization.

Top-level fields should include:

```text
MagicNumber
SchemaVersion
SourceSensorChartNumber
SourceSensorStudyID
LastEvaluatedBarIndex
LastEvaluatedDateTime
PublicationSequence
ATRUsed
ATRValid
HVN1ToHVN2GapATR
HVN1ToHVN2GapValid
SourceConsumerStudyID
SourceStatus
RetainPriorProfileHVN1
ValidStructureCount
Structures[3]
```

Each structure record must include at least:

```text
Valid
SourceConsumerStudyID
ProfileID
ProfileRecency
HVNRank
ZoneLow
ZoneHigh
CurrentSide
LastExternalSide
TouchThisBar
LastTouchBarIndex
LastTouchDateTime
BreachThisBar
BreachCount
LastBreachDirection
LastBreachBarIndex
LastBreachDateTime
MaxExcursionSinceBreach
MaxExcursionSinceBreachATR
SignificantMoveThisBar
SignificantMoveOccurred
EpisodeID
InteractionState
EpisodeCompletionBarIndex
EpisodeCompletionDateTime
ReturnAfterSignificantMove
```

Write all payload fields before incrementing and publishing
`PublicationSequence` as the final commit marker.

The implementation contract shared between producer and downstream consumers
must live in one common header or otherwise have one authoritative binary
layout. Do not maintain several hand-copied, drifting structure definitions.

Optional subgraphs may display zones or diagnostics, but subgraphs are not the
authoritative multi-structure publication channel.

## 22. Formal Enums

```text
PriceSide:
    NONE   = 0
    ABOVE  = 1
    BELOW  = 2
    INSIDE = 3

BreachDirection:
    NONE = 0
    UP   = 1
    DOWN = 2

InteractionState:
    UNTESTED  = 0
    ACTIVE    = 1
    IN_CHOP   = 2
    DISPLACED = 3
```

No `ACCEPTED`, `REJECTED`, `RECLAIMED`, `LOST`, `FAILED_BREAKOUT`, or
`DOUBLE_INVALIDATED` state is introduced in V1.

## 23. Performance Requirements

- Maximum three active structure records per Sensor instance.
- Fixed-capacity storage and deterministic slot traversal.
- No VAP calculation or profile detection in this Sensor.
- Fetch each configured Consumer array at most once per study call and reuse it.
- Read execution OHLC and ATR once per evaluated bar.
- No chart drawings unless explicitly enabled for diagnostics.
- No unbounded event vectors or historical-zone registry.
- Process only newly closed bars during ordinary updates.

## 24. Acceptance Scenarios

### External-side establishment

First observed close below a new HVN establishes `BELOW`; it does not count as
a downward breach.

### Inside traversal

`BELOW -> INSIDE -> INSIDE -> ABOVE` produces exactly one upward breach.

### Same-side repetition

Several closes above after an upward breach do not increase `BreachCount`.

### Alternation into chop

With `N = 4`, four alternating breaches without an `X * ATR` significant move
set only that structure to `IN_CHOP`. Further alternating breaches continue
incrementing `BreachCount` while it remains `IN_CHOP`.

### Significant displacement

An upward breach followed by a completed-bar high at least `X * ATR` above
`ZoneHigh` transitions either `ACTIVE` or `IN_CHOP` to `DISPLACED` and
completes the episode.

### Escape from chop

An `IN_CHOP` structure with `BreachCount = 5` subsequently achieves the
required displacement from its latest breach. It transitions to `DISPLACED`,
retains the completed episode's count of 5, and stops accumulating breaches
until a later return begins a new episode.

### Later retest

From `DISPLACED`, a later touch of the same retained HVN starts a new `ACTIVE`
episode with a reset episode-local breach count.

### Latest-to-older rollover

With prior-HVN1 retention enabled, on a new Consumer profile the former latest
HVN1 moves to the older slot without losing state; its HVN2 expires; the prior
older HVN1 expires.

### Missing HVN2

A latest profile without HVN2 leaves that slot invalid and does not duplicate
HVN1.

### Bounded structure set

At no time does one Sensor instance publish more than three zones, regardless
of how much historical data exists in the Consumer arrays.

### Timed Daily facts

A Daily HVN touched at 05:00 and again at 14:00 publishes the latest touch time
and current episode facts. Long-term event counting and weakening-level
interpretation remain deferred to the later Market Context study.

## 25. Delivery Order

1. Implement and verify the bounded Interaction Sensor.
2. Build continuation/reversal consumers as required.
3. Build structural stop and destination-target policies as required.
4. Specify and implement the broader timed Daily-HVN Market Context study
   last.
