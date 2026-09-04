# Chakra Lean Entry Baton V1

Status: **authoritative implementation specification**

Planned implementation:
`ACS_Source/K_Chakra_Lean_Entry_Baton_V1.cpp`

## 1. Purpose

`K_Chakra_Lean_Entry_Baton_V1` is the single-trade entry executor between a
completed timed-chart trade signal and
`K_Util_Candle_Baton_Manager_V1`.

The study runs on a very small Renko chart, normally a 2-tick NQ or ES chart.
It consumes one approved signal from one configured timed-chart study, waits
for live Renko price to breach the signal candle's directional activation
level, submits the entry with initial attached orders, and publishes the
existing master-trade handoff expected by the Candle Baton Manager.

This component is deliberately lean. It is an entry baton, not a signal
detector and not a post-entry manager.

## 2. Responsibility boundary

### 2.1 In scope

- Consume one sticky, sequenced approved-signal record from one configured
  `Main Trade Chart/Study`.
- Hold at most one armed signal.
- Replace an armed signal when a newer approved signal arrives.
- Apply optional timed-chart bar expiry.
- Apply optional lower and upper close-based invalidation.
- Detect a strict live Renko price breach of the directional activation level.
- Support a user-selected position size from 1 through 5 contracts.
- Submit a market entry in the implemented entry mode.
- Create one ordinary attached stop and one ordinary attached target per
  contract/leg.
- Publish attached-order IDs and executed-trade metadata through the exact PV
  contract already consumed by `K_Util_Candle_Baton_Manager_V1`.
- Mark and log material entry lifecycle events.

### 2.2 Out of scope

- Candlevana, Setup A, MA, Wave, HVN, candle-anatomy, or other setup logic.
- Entry permission, confluence, risk scoring, or trade selection.
- More than one signal source.
- HL, HHHL, ring, dot, or other Renko analytical-study dependencies.
- Adding to or reversing an open position.
- Break-even, trailing, target movement, partial reduction, exits, or any
  other post-entry management.
- Stop-limit submission/status/cancellation logic in V1.
- Post-fill slippage/risk remediation in V1.
- Stop/target recovery after a platform or broker failure beyond reporting
  the synchronous entry rejection.

Post-entry stop and target ownership belongs to
`K_Util_Candle_Baton_Manager_V1` immediately after a successful handoff.

## 3. Architectural position

```text
Candlevana / Setup A raw events
        -> TriggerStr
        -> EntryGate
        -> EntryComposer / timed-chart signal publisher
        -> Chakra Lean Entry Baton on low-tick Renko
        -> K_Util_Candle_Baton_Manager_V1
```

The Entry Baton consumes only the final approved signal. It must not reach
back into raw pattern, Sensor, MCtx, EntryGate, or TradePol studies.

## 4. Proven V2 patterns retained

The implementation may reuse small, verified ACSIL patterns from
`K_Util_ModeBaton_V2.cpp` for:

- `s_SCNewOrder` market submission;
- separate attached Stop 1-5 and Target 1-5 construction;
- `sc.BuyEntry` / `sc.SellEntry` result handling;
- capture of attached internal order IDs;
- global simulation/live routing;
- entry markers and concise transition logging;
- the existing master/follower PV handoff.

No V2 strategy, HL/ring lookup, multiple-source arbitration, break-even,
trailing, target-management, ramp-down, or exit code is to be copied.

## 5. Inputs

| Input | Default | Rule |
|---|---:|---|
| Enabled | Yes | When No, do not arm or submit; clear private armed state. Never alter an open position or cancel post-entry orders. |
| Main Trade Chart/Study | unset | The single timed-chart approved-signal publisher. |
| Entry Mode | Market On Activation | Dropdown defined in section 6. |
| Position Size | 1 | Integer 1-5, owned by this Entry Baton. The signal publisher does not size the trade. |
| Debug Logging | No | Log material transitions and errors only. |

The study must not expose separate long/short enable inputs. The single
`Enabled` input is sufficient.

The study must set `sc.MaximumPositionAllowed` consistently with configured
Position Size and must use ordinary Sierra Chart safety controls that prevent
unintended multiple entries.

## 6. Entry Mode dropdown

Stable values:

```text
0 = Market On Activation
1 = Stop-Limit - Reserved / Not Implemented
```

### 6.1 Market On Activation

Fully implemented in V1. The baton arms an approved signal and submits a
market order only after the directional Renko activation breach.

### 6.2 Stop-Limit

Reserved for a later revision. If selected in V1:

- publish no order;
- do not arm a signal;
- never fall back to Market mode;
- log the unsupported selection once per configuration/lifecycle transition,
  not every chart update.

Future stop-limit work will require working-order ID persistence, status/fill
queries, partial-fill ownership, time/price cancellation, cancellation
dedupe, restart recovery, and handoff timing. None is implemented speculatively
in V1.

## 7. Timed-chart approved-signal input contract

The configured Main Trade study publishes one sticky record. All fields are
written first and `SignalSeq` is written last as the commit marker.

`SignalSeq` is monotonically increasing in ordinary operation and changes
only for a new approved trade signal. The Entry Baton compares by inequality,
not ordering, and consumes each sequence at most once.

### 7.1 Required fields

| Accessor / PV | Field | Meaning |
|---|---|---|
| Int PV 51 | `SignalSeq` | Positive unique signal identity; written last. |
| Int PV 52 | `Direction` | `1 = LONG`, `2 = SHORT`; other values invalid. |
| Int PV 66 | `SignalCandleIndex` | Timed-chart candle T index, diagnostic/provenance only. |
| Int PV 72 | `SignalCandleMaxIndex` | `0` when time expiry is disabled; otherwise `T + ExpiryBars`, diagnostic only. |
| Int PV 78 | `ExpiryBars` | `0` disables time expiry; positive N permits activation during T+1 through T+N. |
| Int PV 79 | `SourceMask` | Bit mask of all same-direction trigger sources combined into this one approved signal. |
| Float PV 54 | `StructuralStopPrice` | Absolute price where the trade idea is invalidated; must be on the protective side at signal creation. |
| Float PV 55 | `InitialTargetDistanceTicks` | Positive offset in ticks, applied identically to Target 1-N from actual entry. |
| Float PV 61 | `LowerInvalidationPrice` | `0` disables lower invalidation. |
| Float PV 63 | `UpperInvalidationPrice` | `0` disables upper invalidation. |
| Float PV 73 | `LongActivationPrice` | For LONG, normally T High; must be positive. Zero for SHORT. |
| Float PV 74 | `ShortActivationPrice` | For SHORT, normally T Low; must be positive. Zero for LONG. |
| Datetime PV 1 | `SignalAvailabilityDateTime` | When the completed timed candle's approved signal became available. |
| Datetime PV 2 | `SignalValidUntilDateTime` | `0` when expiry is disabled; otherwise the start of T+N+1. |

PV numbers intentionally preserve the familiar V2/Narayan signal range where
the same concept already existed. Newly required fields use unused accessor
slots in that producer. Accessor namespaces are independent.

### 7.2 Directional defaults

Unless the approved signal explicitly supplies different optional
invalidation levels:

```text
LONG:
    activation = T High
    lower invalidation = T Low
    upper invalidation = 0 (disabled)

SHORT:
    activation = T Low
    upper invalidation = T High
    lower invalidation = 0 (disabled)
```

The structural stop and pre-entry invalidation may be the same price but are
separate concepts:

- invalidation decides whether an unfilled signal remains eligible;
- structural stop protects an entered position.

### 7.3 Same-bar source combination and conflicts

Upstream publication rules, recorded here for the consumer contract:

- Multiple same-direction patterns on candle T produce one signal and combine
  their bits in `SourceMask`.
- Opposing LONG and SHORT patterns on the same T are logged upstream and
  produce no approved actionable signal.
- The Entry Baton does not reconstruct or reinterpret source patterns.

## 8. Signal acceptance and replacement

On observing a new `SignalSeq`, snapshot and validate the complete record.

Accept only when:

- Enabled is Yes;
- Entry Mode is implemented Market mode;
- the study is in normal live processing;
- no position is open for this symbol/account;
- Direction is LONG or SHORT;
- the appropriate activation price is finite and positive;
- StructuralStopPrice is finite and positive;
- InitialTargetDistanceTicks is finite and strictly positive;
- enabled invalidation prices are finite and positive;
- the signal has not expired.

If a position is already open, log the new signal once and forget it. Do not
queue it, add, reverse, or pass it to post-entry policy. Whether an open trade
should react to a new signal is a future Candle Baton Manager/TradePol design.

If another accepted signal arrives while one is ARMED, the new signal replaces
the old signal regardless of direction. Log the replacement once. The old
signal can never reactivate without a new sequence.

Malformed, unsupported, expired-at-first-read, or otherwise unusable signals
are consumed and dropped. They are never retried under the same `SignalSeq`.

## 9. Time expiry

Time expiry is optional.

```text
ExpiryBars = 0:
    no time expiry

ExpiryBars = N > 0:
    valid during timed bars T+1 through T+N
    expired from the start of T+N+1
```

Example:

```text
T index = 100
ExpiryBars = 2
SignalCandleMaxIndex = 102
Valid during bars 101 and 102
Expires at the start of bar 103
```

The Renko baton must use `SignalValidUntilDateTime`, not compare its own Renko
index to the timed-chart indices. Cross-chart bar indices are not comparable.

On the first Renko evaluation after current chart time is later than the
nonzero validity time, clear the armed signal and log expiry once. No order is
submitted.

## 10. Pre-entry price invalidation

Invalidation is evaluated from the **completed Renko bar Close**, not intrabar
High/Low.

Strict breach rules:

```text
if LowerInvalidationPrice > 0 and completed Renko Close < lower level:
    invalidate

if UpperInvalidationPrice > 0 and completed Renko Close > upper level:
    invalidate
```

A touch or equal close does not invalidate.

Invalidation is checked before activation on a call that processes a newly
completed Renko bar. On invalidation, clear the armed signal, log once, and do
nothing else with that signal.

No elaborate same-Renko-bar ambiguity state is required. The intended chart is
a very small Renko chart (normally 2 ticks), making simultaneous traversal of
widely separated activation and invalidation levels practically negligible.
Simple deterministic invalidation-first ordering is sufficient.

## 11. Activation

Activation is live/intrabar and uses the current Renko bar's developing
High/Low:

```text
LONG:  current Renko High > LongActivationPrice
SHORT: current Renko Low  < ShortActivationPrice
```

The breach is strict. A touch is not enough.

If the baton first observes a newly accepted signal when Renko price is already
beyond the activation level, it may submit immediately, provided the signal is
not expired or invalidated and all final safety gates pass.

Activation does not require the Renko candle to have a particular colour and
does not reference an HL/ring/dot study.

## 12. Market-order construction

For Market mode, build one `s_SCNewOrder`:

- `OrderType = SCT_ORDERTYPE_MARKET`;
- `TimeInForce = SCT_TIF_GOOD_TILL_CANCELED`, matching the proven V2 pattern;
- `OrderQuantity = configured Position Size`;
- LONG uses `sc.BuyEntry`;
- SHORT uses `sc.SellEntry`.

### 12.1 Initial targets

Create Target 1 through Target N for quantity N:

- ordinary `SCT_ORDERTYPE_LIMIT` attached targets;
- each target uses the same positive `InitialTargetDistanceTicks` offset from
  actual entry;
- no target is created for unused legs N+1 through 5;
- no target chasing or movement occurs in this study.

The signal publisher owns the initial target-distance value. The Entry Baton
does not calculate a target from ATR or market structure.

### 12.2 Initial stops

Create Stop 1 through Stop N for quantity N:

- ordinary `SCT_ORDERTYPE_STOP` attached stops;
- separate stop per leg, matching the working V2 custom-trailing order shape;
- all stops initially represent the same absolute StructuralStopPrice;
- `StopAll` is not used in V1;
- no trailing or break-even instruction is attached by this study.

ACSIL entry construction uses stop offsets. Immediately before submission,
derive a positive provisional price offset from the current Renko Close and the
absolute StructuralStopPrice:

```text
LONG offset  = CurrentRenkoClose - StructuralStopPrice
SHORT offset = StructuralStopPrice - CurrentRenkoClose
```

The resulting offset must be finite and positive. If it is not, submission is
invalid and the signal is dropped. Sierra Chart may also reject invalid order
geometry; no retry follows a rejection.

The structural price remains the semantic stop anchor. A later revision may
inspect the actual fill and reconcile/check realized risk. V1 does not delay
the handoff for that deferred enhancement.

## 13. Submission eligibility and safety

Immediately before submission, revalidate:

- Enabled remains Yes;
- Market mode remains selected;
- signal identity still matches the armed signal;
- signal is not expired;
- no completed-bar invalidation has cleared it;
- position is flat;
- no incompatible entry submission is already in progress;
- configured Position Size is 1-5;
- target distance and stop offset are positive;
- this is normal real-time processing.

Never submit during:

- `sc.IsFullRecalculation`;
- historical data download;
- historical reconstruction/replay calculation that is not authorized live
  order processing.

Follow Sierra Chart's global simulation/live mode as in V2:

```cpp
sc.SendOrdersToTradeService = !sc.GlobalTradeSimulationIsOn;
```

The implementation must set the standard ACSIL order-safety flags deliberately
and prevent duplicate submission of one `SignalSeq`.

## 14. Submission result

### 14.1 Rejected submission

If `sc.BuyEntry` / `sc.SellEntry` returns a non-positive result:

- log the rejection/result once;
- clear the armed signal;
- mark its `SignalSeq` consumed;
- do not retry;
- do not change PV43 or publish a manager handoff.

A fresh attempt requires a new approved `SignalSeq`.

### 14.2 Successful submission

On a positive ACSIL submission result:

- capture Target 1-5 and Stop 1-5 internal order IDs from `s_SCNewOrder`;
- clear unused order-ID slots to zero;
- clear StopAll ID to zero;
- publish the master handoff in section 15;
- mark the entry Subgraph;
- clear ARMED state;
- do not perform any subsequent stop/target management.

The handoff occurs immediately on the successful entry submission/order-ID
creation path. It is not delayed for the future post-fill risk check.

## 15. Candle Baton Manager handoff

This publication is intentionally the legacy PV block consumed directly by
`K_Util_Candle_Baton_Manager_V1`; it is not a Chakra common-envelope record.

Write in this exact order:

| Order | PV | Type | Value |
|---:|---:|---|---|
| 1 | 20 | Int | `StopAllOrderID = 0` |
| 2 | 21-25 | Int | Target 1-N internal IDs; unused slots zero |
| 3 | 34-38 | Int | Stop 1-N internal IDs; unused slots zero |
| 4 | 52 | Int | `1 = LONG`, `2 = SHORT` |
| 5 | 53 | Int | Configured/submitted quantity N |
| 6 | 54 | Double | Initial stop distance in price units used for attached-order construction |
| 7 | 43 | Int | Incremented executed-trade leader ID, written **last** |

PV43 is the commit/edge observed by the Candle Baton Manager. Never advance it
for arming, invalidation, expiry, unsupported Stop-Limit mode, or rejected
submission.

The manager already:

- detects a new positive PV43;
- copies PV20-25 and PV34-38;
- copies direction and quantity from PV52/53;
- consumes PV54 as `syncedMasterStopOffset`;
- refreshes initially missing order IDs from the same master study.

The Entry Baton must remain compatible with that behavior without modifying
`K_Util_Candle_Baton_Manager_V1`.

## 16. Private state machine

Use the smallest practical private state model:

```text
IDLE
ARMED
```

An additional one-call submission guard/last-consumed sequence is private
state, not a public state-machine phase. Market submission is synchronous for
V1's required decision path; Stop-Limit pending-order states are deferred.

### 16.1 IDLE

- Observe/baseline the configured signal publisher.
- Accept a genuinely new valid signal and enter ARMED.
- If a position is open, consume/log/forget new signals.

### 16.2 ARMED

Per evaluation precedence:

1. consume and replace with a newer valid signal, if present;
2. expire by nonzero validity datetime;
3. on a newly completed Renko bar, apply close-based lower/upper invalidation;
4. validate Market mode and flat-position safety;
5. detect strict intrabar activation breach;
6. submit once;
7. publish successful handoff or consume/drop rejection;
8. otherwise remain ARMED.

## 17. Signal delivery, lifecycle, and recalculation

- On first attachment or dependency rewiring, baseline `LastConsumedSignalSeq`
  to the publisher's current sequence without acting on the sticky historical
  signal.
- On the Entry Baton's own full recalculation, perform no trading action and
  do not mutate a compatible existing ARMED snapshot or its
  `LastConsumedSignalSeq`. An unrelated input change such as Debug Logging or
  Position Size therefore does not silently discard a live armed signal.
- After ordinary live processing resumes, revalidate the preserved signal
  against current expiry, position, mode, dependency, and record-validity
  rules before permitting invalidation or activation. Preservation never
  bypasses an ordinary safety gate.
- Recalculation must not manufacture an entry from a historical signal or
  replace the preserved snapshot from historical reconstruction.
- If the recalculated configuration is incompatible with the armed snapshot,
  clear it and baseline delivery. Incompatible changes are: Enabled becoming
  No, Main Trade Chart/Study rewiring, or Entry Mode becoming the reserved
  Stop-Limit mode. Returning later to a compatible configuration cannot revive
  the discarded signal without a new `SignalSeq`.
- A forming Renko update may evaluate only the cheap live activation path and
  signal-sequence observation needed for low-latency entry.
- Completed-bar invalidation runs once per completed Renko bar.
- Disabling clears private ARMED state and baselines delivery so re-enable does
  not replay an old signal.
- Dependency rewiring discards the old armed signal and baselines the newly
  configured producer.
- `SignalSeq` inequality is the dedupe gate; the same signal is never submitted
  twice after ordinary updates.

No V1 guarantee is made for reconstructing an already-submitted live order
after Sierra Chart/process restart. Stop-limit restart recovery belongs to its
later lifecycle design. Existing broker-side attached orders remain broker/
platform facts and must never be cancelled merely because this analytical
study recalculates.

## 18. Subgraphs

Event markers only; zero otherwise:

| SG | Name | Meaning | Placement |
|---:|---|---|---|
| 0 | Signal Armed Long | A LONG approved signal was armed or replaced the prior signal | Below Renko bar |
| 1 | Signal Armed Short | A SHORT approved signal was armed or replaced the prior signal | Above Renko bar |
| 2 | Market Entry Long | LONG market submission succeeded and handoff published | Below Renko bar |
| 3 | Market Entry Short | SHORT market submission succeeded and handoff published | Above Renko bar |
| 4 | Signal Invalidated | Armed signal invalidated by completed Renko Close | Direction-aware opposite side |
| 5 | Signal Expired | Armed signal expired by time | Direction-aware opposite side |
| 6 | Entry Rejected | ACSIL submission rejected or record failed final geometry | Direction-aware opposite side |

Provide one non-negative `Marker Offset In Ticks` input if these markers are
implemented. Marker Subgraphs are diagnostic only and never form the manager
handoff.

## 19. Logging

With Debug Logging enabled, log once per material identity/transition:

- first dependency baseline and rewiring baseline;
- signal armed, including sequence, direction, T index, activation, structural
  stop, target ticks, expiry, invalidation levels, and source mask;
- armed signal replacement;
- open-position signal ignored;
- expiry;
- lower/upper invalidation;
- unsupported Stop-Limit selection;
- market submission attempt;
- synchronous rejection/result code;
- successful handoff, including PV43, direction, quantity, stop/target IDs,
  and stop distance.
- preserved ARMED state resuming after compatible recalculation, or an ARMED
  state being discarded because recalculated configuration is incompatible.

Do not log on every Renko update while merely waiting for activation.

## 20. Deferred enhancements

Explicitly deferred, not partially implemented:

### 20.1 Stop-Limit mode

- place stop-limit parent order;
- persist/query entry-order ID and status;
- cancel on timed expiry or price invalidation;
- handle fill, rejection, cancellation, and partial fill;
- recover an in-flight order after recalculation/restart;
- publish manager handoff at the correct filled-trade point.

### 20.2 Post-fill execution-risk check

The structural stop remains fixed while a market fill may slip. Example:

```text
expected entry = 100
structural stop = 75
expected distance = 25
actual fill = 125
actual structural risk distance = 50
```

A later revision may compare actual fill-to-structural-stop risk against a
configured threshold and immediately flatten/cancel if excessive. That check
must happen after the ordinary handoff; it must not delay PV43 or Candle Baton
Manager synchronization. Exact threshold ownership and remediation mechanics
remain unspecified.

### 20.3 New signals during an open trade

V1 logs and forgets them. Future TradePol/Candle Baton Manager work may decide
whether a new signal means HOLD, REDUCE, ADD, EXIT, or TIGHTEN_STOP. The Entry
Baton must not anticipate that policy.

## 21. Acceptance scenarios

1. A new LONG signal arms from one configured timed-chart producer; no order is
   submitted at a touch of T High.
2. Renko High strictly exceeds LONG activation while valid and flat; one market
   entry is submitted with N separate stops and N same-offset targets.
3. SHORT behaves symmetrically using strict Renko Low breach.
4. Completed Renko Close strictly below an enabled lower invalidation clears
   the signal; a touch does not.
5. Completed Renko Close strictly above an enabled upper invalidation clears
   the signal; a touch does not.
6. `ExpiryBars=2` permits T+1/T+2 and expires from T+3 using datetime, not Renko
   index.
7. `ExpiryBars=0` and validity datetime 0 never expire by time.
8. Lower or upper invalidation price 0 disables only that side.
9. A newer signal replaces an armed signal in either direction.
10. A signal arriving while a position is open is logged and forgotten.
11. A positive submission captures order IDs, writes payload, then increments
    PV43 last; the Candle Baton Manager can sync without modification.
12. A rejected submission is logged and discarded with no retry and no PV43
    advance.
13. Selecting reserved Stop-Limit mode submits nothing and never falls back to
    Market.
14. Full recalculation, historical download, first attachment, rewiring, and
    re-enable never replay an old signal into a live order.
15. One `SignalSeq` can produce at most one submission attempt.
16. No HL/ring study is read anywhere in the implementation.
17. After successful handoff, no stop/target modification is performed by the
    Entry Baton.
18. Given a valid ARMED signal, a compatible full recalculation such as a Debug
    Logging or Position Size change submits no order during recalculation,
    preserves the snapshot/consumed sequence, and resumes only after ordinary
    live revalidation.
19. Given an ARMED signal, disabling, rewiring Main Trade Chart/Study, or
    selecting reserved Stop-Limit during recalculation clears the signal and
    baselines delivery; returning to the prior configuration does not revive
    it without a new `SignalSeq`.

## 22. Relationship to older draft contracts

This concrete user-authorized specification supersedes conflicting generic
design assumptions in
`Chakra_OrderManagement_Handoff_Interface_Contract.md` for this component,
specifically:

- one signal source rather than multiple Composer slots;
- Entry Baton input owns quantity 1-5;
- separate Stop 1-N and Target 1-N are attached at entry;
- one shared positive target distance comes from the timed signal;
- activation is delayed to a strict Renko breach;
- price invalidation is completed-Renko-close based;
- the legacy handoff is published immediately on successful market submission,
  without waiting for the deferred post-fill risk check;
- Stop-Limit is a reserved dropdown value only.

The existing manager-facing PV numbers and PV43-last commit ordering remain
authoritative and compatible.
