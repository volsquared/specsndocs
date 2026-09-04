# Chakra — Market Context: Candle/MA Relation (Draft, Revision 2)

Status: **design only, no ACSIL implementation yet.**

Builds directly on:
- `Chakra_Common_Interface_Envelope_Contract.md` (approved, Revision 8) —
  `ComponentKind`, `K_CHAKRA_STATUS`, `PublicationSeq`, and PV-ownership
  ranges, unmodified.
- `Chakra_Sensor_Interface_Contract.md` — this document's Sensor-
  consumption rules (state pattern, atomic validity) follow it exactly.
- `Chakra_MCtx_Interface_Contract.md` (Revision 5) — this document's
  identity, classification, freshness, transition, recalculation, and
  write-last-sequence rules follow it exactly; nothing here redefines or
  loosens any of them.
- `Chakra_Sensor_Candle_MA_Raw_V1.md` (Revision 2) — the **sole, required**
  Sensor dependency this Market Context consumes.
- `K_Alert_CandleAndAverageRelation_V1.cpp` — a **behavioral reference for
  the classification formulas only**, not code to copy. That study also
  computes candle/MA close classifications, breach/touch history, and
  richer banner behavior — **none of that is carried into this Market
  Context** (see "Non-goals"). Only the six-state MA relationship and the
  two ATR-normalized distance formulas are preserved, exactly as that
  study computes them.

------------------------------------------------------------------------

# Purpose

A lightweight Market Context that consumes exactly one
`Chakra_Sensor_Candle_MA_Raw_V1` instance and publishes three descriptive
facts, with no additional interpretation:

1. The established six-state MA relationship.
2. Fast-to-slow MA distance, in ATR units.
3. Directional candle-extreme-to-fast-MA distance, in ATR units.

------------------------------------------------------------------------

# Non-goals — explicitly excluded

- Close-relation-to-each-MA classification (above/below, body-percentage
  filtered or otherwise).
- Body/range percentage filters of any kind.
- Touch history (candles-since-MA-touch).
- Crossover history (candles-since-fast/slow-crossover).
- Crossover events of any kind.
- Trading direction or signal.
- Trigger behavior.
- EntryGate behavior.
- Stops, targets, sizing, or orders.
- Any change to `K_Alert_CandleAndAverageRelation_V1.cpp`.
- Cross-chart Sensor wiring (decided against for v1, see "Future scope").
- Historical backfill/full-recalculation reconstruction of past states
  (decided against for v1, see "Future scope" and section 11).

------------------------------------------------------------------------

# 1. Identity and naming

- **`ComponentKind`** is always `K_CHAKRA_KIND_MCTX`.
- **`ProducerTypeID`**:
  ```cpp
  enum K_CHAKRA_MCTX_CANDLEMARELATION_TYPE
  {
      K_CHAKRA_MCTX_CANDLEMARELATION_TYPE_NONE = 0,
      K_CHAKRA_MCTX_CANDLEMARELATION_TYPE_V1   = 1,
  };
  ```
- **`PayloadSchemaVersion`** (int32 PV 20) scoped per `ProducerTypeID`,
  standard envelope convention.
- **One study instance owns this one context concept** — the six-state
  relationship and its two supporting ATR-distance measurements, published
  together as one coherent classification act (per the base MCtx
  contract's own "one coherent context concept" rule), not bundled with
  any unrelated context.

Naming: `K_Chakra_MCtx_CandleMARelation_V1`.

------------------------------------------------------------------------

# 2. Inputs

| Input | Default | Purpose |
|---|---|---|
| `Enabled` | Yes | Standard enable/disable. |
| `Source Sensor Chart/Study` | — | Standard chart+study reference (`Input.SetChartStudyValues`) to a `Chakra_Sensor_Candle_MA_Raw_V1` instance. **For this first implementation, only same-chart wiring (chart number 0, "this chart") is intended** — cross-chart deployment is not designed here (see "Future scope"). |
| `Enable Display Banner` | **No** | Diagnostic-only text banner (section 8). Never affects PV calculation or publication. |
| `Banner Horizontal Position` | 30 | Standard chart-relative horizontal position, matching `K_Alert_CandleAndAverageRelation_V1.cpp`'s own input pattern. |
| `Banner Vertical Position` | 100 | Same, vertical. |
| `Debug Logging` | No | Standard debug-log toggle; logs decisions/transitions only (section 10). |

**No classification thresholds are exposed as inputs** — the six-state
relationship and both ATR-distance formulas are fixed, code-versioned
logic (section 3-5), per the base MCtx contract's own binding rule that
classification thresholds are never runtime-configurable.

**Cadence declaration** (required by the base MCtx contract's section 6):
**closed-bar cadence only.** This context classifies once per newly
closed bar on its own chart (matching its required Sensor's own closed-bar
cadence) and never revises a classification intrabar — there is no
forming-bar evaluation and no concept of a classification "locking in"
versus revising, since it is never published before the bar it describes
has closed.

------------------------------------------------------------------------

# 3. Sensor dependency

- **Required**: exactly one `Chakra_Sensor_Candle_MA_Raw_V1` instance.
  This context cannot function at all without it — there is no optional
  secondary Sensor.
- **Expected identity**: `ComponentKind == K_CHAKRA_KIND_SENSOR`,
  `ProducerTypeID == K_CHAKRA_SENSOR_CANDLEMARAW_TYPE_V1`, plus the
  expected `SchemaVersion`/`PayloadSchemaVersion` — validated in full,
  per the envelope's identity order, before any payload field is read.
- **Fields read**: `FastMA`, `MidMA`, `SlowMA`, `ATRUsed`, `CandleHigh`,
  `CandleLow` (all Float payload fields, section 5 of the Sensor
  document) — nothing else from that Sensor's payload.
- **Freshness tolerance**: the Sensor's `DecayClass` is `FAST` (an
  ordinary closed-bar measurement). For same-chart wiring (section 2),
  this Market Context requires the Sensor's `EvalBarIndex` to equal its
  own current evaluation bar index — i.e. the Sensor must have already
  evaluated *this exact* closed bar, not an older one. This is a stricter,
  simpler tolerance than a generic `Age` comparison, and is only valid
  because same-chart wiring is required for v1 (section 2).
- **Live-versus-historical access**: **this Market Context has no output
  subgraphs, and its Sensor dependency has no output subgraphs either**
  (`Chakra_Sensor_Candle_MA_Raw_V1.md` section 7). Consequently, neither
  component reconstructs historical classification during a full
  recalculation sweep — see section 11 for exactly what this does and
  does not mean, and why the resulting behavior remains compliant with
  the base MCtx contract's own latest-state PV interface even though
  historical-subgraph reconstruction is deliberately excluded.

------------------------------------------------------------------------

# 4. Six-state MA relationship

**Preserved exactly from `K_Alert_CandleAndAverageRelation_V1.cpp`'s own
`MA_RELATION` logic, including its ordered if/else-if evaluation — this
is proven, tested behavior being carried forward unmodified, not
re-derived.**

```cpp
enum K_CHAKRA_MCTX_CANDLEMARELATION_STATE
{
    NONE                            = 0,  // equality, or no arrangement below matches
    PERFECT_UP                      = 1,  // FastMA > MidMA && MidMA > SlowMA
    PERFECT_DOWN                    = 2,  // FastMA < MidMA && MidMA < SlowMA
    FAST_BELOW_MID_BOTH_ABOVE_SLOW  = 3,  // FastMA < MidMA && MidMA > SlowMA && FastMA > SlowMA
    FAST_ABOVE_MID_BOTH_BELOW_SLOW  = 4,  // FastMA > MidMA && MidMA < SlowMA && FastMA < SlowMA
    FAST_BELOW_SLOW_MID_ABOVE_SLOW  = 5,  // FastMA < SlowMA && MidMA > SlowMA
    FAST_ABOVE_SLOW_MID_BELOW_SLOW  = 6   // FastMA > SlowMA && MidMA < SlowMA
};
```

**Evaluated in this exact order, first match wins** (matching source
lines 336-356 precisely):

```text
if      FastMA > MidMA && MidMA > SlowMA                             -> PERFECT_UP
else if FastMA < MidMA && MidMA < SlowMA                              -> PERFECT_DOWN
else if FastMA < MidMA && MidMA > SlowMA && FastMA > SlowMA           -> FAST_BELOW_MID_BOTH_ABOVE_SLOW
else if FastMA > MidMA && MidMA < SlowMA && FastMA < SlowMA           -> FAST_ABOVE_MID_BOTH_BELOW_SLOW
else if FastMA < SlowMA && MidMA > SlowMA                             -> FAST_BELOW_SLOW_MID_ABOVE_SLOW
else if FastMA > SlowMA && MidMA < SlowMA                             -> FAST_ABOVE_SLOW_MID_BELOW_SLOW
else                                                                   -> NONE
```

**This exact order must be preserved** — states 3 and 4 are checked
before states 5 and 6, matching the proven source exactly. This document
does not independently re-derive or claim to have proven which pairs can
or cannot co-occur; the ordering is carried forward as-is because it is
what the tested reference does, not because this document has verified an
alternate order would misbehave.

**No additional bullish/bearish/mixed summary enum is created.** This one
enum, with these seven values, is the complete state space.

------------------------------------------------------------------------

# 5. Numeric values

**`FastSlowDistanceATR`** — fast-to-slow MA distance, in ATR units:

```text
FastSlowDistanceATR = abs(FastMA - SlowMA) / ATR
```

Published as a float. Not categorized, not thresholded, not converted
into a state of its own.

**`CandleExtremeFastDistanceATR`** — directional candle-extreme-to-fast-MA
distance, in ATR units, preserving the source's own directional choice
exactly (source lines 507-524):

```text
if FastMA >= SlowMA:
    CandleExtremeFastDistanceATR = abs(CandleHigh - FastMA) / ATR
else:
    CandleExtremeFastDistanceATR = abs(FastMA - CandleLow) / ATR
```

Published as a float. Not categorized. **Note the asymmetry preserved
from source**: when `FastMA >= SlowMA`, the candle's *high* is compared
against `FastMA`; when `FastMA < SlowMA`, the candle's *low* is compared
instead — this is the proven reference's own directional convention
(the candle extreme on the side away from the slow MA), not something
this document re-derives independently.

------------------------------------------------------------------------

# 6. Availability

The complete context record requires **all** of the following; if any
fails, the record is unavailable (section 7):

- An active, fresh Sensor measurement (section 3: identity valid,
  `Status == VALID`, `EvalBarIndex` matches this context's own evaluation
  bar).
- Finite `FastMA`/`MidMA`/`SlowMA`.
- Finite `ATRUsed`, strictly greater than zero (required as the divisor
  for both numeric values, section 5).
- Finite `CandleHigh`/`CandleLow`.
- Valid source identity and `PublicationSeq` (section 3).

------------------------------------------------------------------------

# 7. Input validity and status

Reuses `K_CHAKRA_STATUS`, per the base MCtx contract's own meanings:

- **`NOT_READY`** — the required Sensor is not yet `VALID` (its own
  warmup), or its `EvalBarIndex` does not yet match this context's
  current evaluation bar (same-chart wiring means this should resolve
  within one evaluation, not linger).
- **`VALID`** — all of section 6's conditions hold; `MARelationState`,
  `FastSlowDistanceATR`, and `CandleExtremeFastDistanceATR` are complete
  and trustworthy.
- **`DISABLED`** — this context's own `Enabled` input is off.
- **`ERROR`** — the Sensor is reachable and `VALID` but publishes a
  non-finite MA/ATR/candle value (should not happen given the Sensor's
  own validation, but checked independently here per this framework's
  "never trust a single validation pass" discipline), or `ATRUsed <= 0`.

**If unavailable** (any status other than `VALID`):
- Publish the appropriate status/reason (this section; section 9).
- Set `MARelationState = NONE (0)`.
- **Do not retain `FastSlowDistanceATR`/`CandleExtremeFastDistanceATR`
  from a prior valid evaluation as though they were current** — both are
  zeroed/neutralized alongside `MARelationState`. A consumer must gate on
  `Status == VALID` before reading any of the three; nothing in this
  payload carries meaning otherwise.
- Identity/schema fields survive unchanged, per the envelope's
  identity-survives-invalid-status rule.

------------------------------------------------------------------------

# 8. State transitions

**This Market Context is state-style, not event-style** — `PublicationSeq`
is the only commit marker, publishing on material change; no
payload-level `EventSeq` is ever defined. If a future consumer needs to
detect a transition (e.g. entering `PERFECT_UP`), it does so privately, on
its own initiative, with its own lifecycle-baselined "last observed
`MARelationState`" tracker — exactly the discipline the base MCtx contract
requires (section 7 of that document) and does not repeat here.

------------------------------------------------------------------------

# 9. Diagnostics

```cpp
enum K_CHAKRA_MCTX_CANDLEMARELATION_REASON_CODE
{
    K_CHAKRA_MCTX_CANDLEMARELATION_REASON_NONE               = 0,   // = K_CHAKRA_REASON_NONE
    K_CHAKRA_MCTX_CANDLEMARELATION_REASON_SENSOR_UNWIRED       = 1,
    K_CHAKRA_MCTX_CANDLEMARELATION_REASON_SENSOR_SCHEMA_MISMATCH = 2,
    K_CHAKRA_MCTX_CANDLEMARELATION_REASON_SENSOR_NOT_READY     = 3,
    K_CHAKRA_MCTX_CANDLEMARELATION_REASON_SENSOR_STALE          = 4,
    K_CHAKRA_MCTX_CANDLEMARELATION_REASON_INVALID_MEASUREMENT   = 5,
    K_CHAKRA_MCTX_CANDLEMARELATION_REASON_ATR_INVALID           = 6,
    K_CHAKRA_MCTX_CANDLEMARELATION_REASON_CONTEXT_PUBLISHED     = 7
    // extend per future revision as needed, numbered above these common ones
};
```

**Written into the common-envelope `ReasonCode` field (section 10) — not
a separate payload field.** Strictly self-referential: describes only
what this Market Context itself observed, never a Trigger/EntryGate
disposition.

------------------------------------------------------------------------

# 10. Persistent-variable output

Common-envelope fields used exactly as baselined — never duplicated into
this context's own payload. State-style: no `EventSeq`; `Int64 PV 10`
stays unused.

## Common-envelope fields

| Field | Namespace/PV | Value for this Market Context |
|---|---|---|
| `SchemaVersion` | Int32 PV 1 | `K_CHAKRA_ENVELOPE_SCHEMA_VERSION_CURRENT` |
| `Status` | Int32 PV 2 | Section 7 |
| `ComponentKind` | Int32 PV 3 | `K_CHAKRA_KIND_MCTX` |
| `ProducerTypeID` | Int32 PV 4 | `K_CHAKRA_MCTX_CANDLEMARELATION_TYPE_V1` |
| `EvalBarIndex` | Int32 PV 5 | This context's own chart's bar index for the evaluated closed candle |
| `DecayClass` | Int32 PV 6 | **`FAST`**, always — this classification is only ever as fresh as the same closed bar's Sensor measurement |
| `ReasonCode` | Int32 PV 7 | Section 9 |
| `PublicationSeq` | Int64 PV 1 | Commit marker, written last |
| `EvalDateTime` | Datetime PV 1 | The evaluated closed bar's own datetime |
| `ValidUntilDateTime` | Datetime PV 2 | `0` (sentinel) — never `STATIC_SESSION` |

## Layer payload

| Field | Namespace/PV | Purpose |
|---|---|---|
| `PayloadSchemaVersion` | Int32 PV 20 | Envelope convention |
| `MARelationState` | Int32 PV 21 | Section 4, `0`-`6` |
| `SourceSensorChartNumber` | Int32 PV 22 | Provenance (section 2/3) |
| `SourceSensorStudyID` | Int32 PV 23 | Provenance |
| `SourceSensorPublicationSeq` | Int64 PV 11 | The Sensor's own `PublicationSeq` at the moment this context read it — provenance/traceability only, not a dedupe key (state pattern, re-read every cycle regardless) |
| `FastSlowDistanceATR` | Float PV 1 | Section 5 |
| `CandleExtremeFastDistanceATR` | Float PV 2 | Section 5 |

**No separate `SourceSensorEvalDateTime`/`SourceSensorDataAsOfDateTime`
field is published, deliberately** — the base MCtx contract's own worked
example propagates one, but that's for a cross-chart Sensor whose
freshness anchor can genuinely differ from the consuming context's own
`EvalDateTime`. Here, the same-chart/exact-`EvalBarIndex`-match design
(section 3) means the Sensor's `EvalDateTime` is always identical to this
context's own envelope `EvalDateTime` — publishing both would be a pure
duplicate, not new information. If cross-chart wiring is ever added
(open question 1), this field would need to be added at that point.

**No output subgraphs of any kind** (section 3, section 11).

------------------------------------------------------------------------

# 11. Recalculation

**Compliant with the base MCtx contract's own latest-state PV interface
and low-churn publication pattern, while deliberately excluding
historical reconstruction** — not a claim of full compliance with every
recalculation-related rule in that contract, since the rules concerning
*historical subgraph* rebuilding do not apply here: neither this Market
Context nor its Sensor dependency has one.

- **On recalculation entry, publish `Status = NOT_READY` once.**
- **No intermediate publication during the historical sweep.** This
  Market Context does not attempt to reconstruct historical
  classification for any bar — there is no Sensor historical subgraph to
  read (per the base MCtx contract's own anti-lookahead rule, reading the
  Sensor's *persistent* payload for a historical bar would misclassify
  that bar using today's data, which is exactly why this document does
  not do that).
- **On recalculation completion, publish exactly one final snapshot,
  anchored explicitly to the latest properly evaluated *closed* bar —
  never the forming bar.** Concretely: `EvalBarIndex = sc.ArraySize - 2`
  under ordinary conditions (the last fully closed bar; `sc.ArraySize - 1`
  is the still-forming bar and must never be used as the final snapshot's
  anchor). **This Market Context does not independently derive that
  index** — it reads whatever `EvalBarIndex` the Sensor itself reports for
  its own final post-recalculation snapshot (`Chakra_Sensor_Candle_MA_Raw_V1.md`
  section 4) and copies it, publishing its own final snapshot for that
  same bar. This is legitimately valid for that specific closed bar — "the
  Sensor's final post-recalculation snapshot" and "the last properly
  closed bar" refer to the same evaluation point, never the forming one.
- **Ordinary change-based publication resumes immediately afterward,**
  starting from the next bar to close live.

**What "no historical subgraphs" does and does not mean, stated
precisely rather than left to be inferred:**

- **Static, full-recalculation reconstruction of every past bar's state
  is out of scope** — this Market Context never rebuilds or exposes a
  classification for any bar other than the single final one described
  above during a recalculation sweep. There is no mechanism to query "what
  was `MARelationState` 200 bars ago" after a reload.
- **This does not prevent an ordinary chart replay from producing correct
  states progressively as bars close.** Replay that advances one closed
  bar at a time (not a single full-recalculation sweep) drives this
  Market Context through its ordinary, ongoing change-based publication
  path (section 8) exactly as live operation does — each newly closed
  replay bar produces its own current, valid snapshot in turn. What is
  excluded is only the *backfill* case: instantaneously reconstructing
  every historical bar's classification in one full-recalculation pass
  with no per-bar subgraph record of the result.

------------------------------------------------------------------------

# 12. Display banner

**Diagnostic only — never affects PV calculation or publication in either
direction.** Patterned after `K_Alert_CandleAndAverageRelation_V1.cpp`'s
own `drawTextAtLineKey` (an `s_UseTool` `DRAWING_TEXT` drawing, line
number persisted in a private PV, `UTAM_ADD_OR_ADJUST`).

Suggested display: `MA:1  FS:0.42  CF:0.18` (relation state, then both
ATR-distance values rounded to two decimal places).

- **Protected by `Enable Display Banner`, default `No`** (section 2).
- **When disabled: do no ordinary banner-update work.** No text
  formatting, no `s_UseTool` construction, on any ordinary evaluation.
- **If a banner is already drawn at the moment `Enable Display Banner`
  transitions to `No`, remove/blank it exactly once** (one `s_UseTool`
  call with empty text, matching the source's own disable-transition
  pattern) — not on every subsequent evaluation while it stays disabled.
- **When enabled: update only when the displayed state changes** — the
  relation state itself, or either ATR-distance value after rounding to
  the displayed precision. A private tracker (`LastBannerDisplayedText`
  or equivalent, private PV range) holds the last-drawn text; redraw only
  when the freshly formatted text differs from it. **Do not redraw on
  every closed bar if nothing displayed would actually change.**
- **The drawing line number is persisted in a private PV** (matching
  source's own `r_LineNumber` pattern) and reused via `UTAM_ADD_OR_ADJUST`
  — **never create a new drawing on every update.**
- **Banner horizontal/vertical position are configurable** (section 2).
- **Color**: the existing study's familiar yellow (`RGB(255, 255, 0)`,
  matching source's `COLOR::YELLOW`). **No state-dependent color** — the
  banner's color never changes based on `MARelationState` or either
  numeric value.

------------------------------------------------------------------------

# 13. Performance

**One rule, stated once, governing every bullet below and superseding any
looser phrasing elsewhere in this document: the Sensor's current
persistent snapshot is always valid state, for as long as
`Status == VALID` and it passes section 3's freshness check. Reading it
again on a later evaluation is never wrong. `SourceSensorPublicationSeq`
is a performance/traceability aid — it may be used to skip *redundant
recomputation or republication* when nothing has changed — but it is
never an event-dedup gate, and an unchanged `PublicationSeq` never makes
the Sensor's state stale or unusable.**

- **One evaluation per newly closed bar, not per study-update call** —
  this context still checks the Sensor's current snapshot on every
  evaluation (the state pattern, unmodified); "once per closed bar" bounds
  *how often this context itself evaluates*, not how it treats the
  Sensor's snapshot once read.
- **`SourceSensorPublicationSeq` may be compared against its own
  previously-observed value purely to skip redundant work** — if
  unchanged, this context's own classification would recompute to the
  identical result, so it may skip recomputation and republication as a
  performance optimization. **This is never a validity check** — even
  without this optimization, reading the same unchanged Sensor snapshot
  again is always correct, just redundant.
- **Do not loop over history.** No historical reconstruction (section
  11).
- **Do not allocate arrays or containers.**
- **No subgraphs are created or cleared** (section 10, section 11).
- **Log only decisions/transitions when Debug Logging is enabled** — a
  `Status` change or `ReasonCode` change, never every evaluation.
- **Write all payload fields first; write `PublicationSeq` last.**

------------------------------------------------------------------------

# 14. Worked examples

1. **`PERFECT_UP`.** `FastMA=110, MidMA=105, SlowMA=100`. First condition
   (`FastMA > MidMA && MidMA > SlowMA`) is true. `MARelationState =
   PERFECT_UP (1)`.
2. **`PERFECT_DOWN`.** `FastMA=90, MidMA=95, SlowMA=100`. Second condition
   true. `MARelationState = PERFECT_DOWN (2)`.
3. **State 3 reached via the ordered evaluation.**
   `FastMA=95, MidMA=105, SlowMA=90`. Checked in order: condition 1
   (`95>105`) false; condition 2 (`95<105 && 105<90`) false (`105<90` is
   false); condition 3 (`95<105 && 105>90 && 95>90`) — all three true.
   `MARelationState = FAST_BELOW_MID_BOTH_ABOVE_SLOW (3)`. Per section 4,
   this evaluation stops here — conditions 5/6 are never even checked for
   this triple, by construction of the ordered if/else-if chain.
4. **State 4 reached via the ordered evaluation.**
   `FastMA=105, MidMA=95, SlowMA=110`. Conditions 1-3 false; condition 4
   (`105>95 && 95<110 && 105<110`) — all three true. `MARelationState =
   FAST_ABOVE_MID_BOTH_BELOW_SLOW (4)`.
5. **Equality resulting in state 0.** `FastMA=100, MidMA=100, SlowMA=90`.
   Condition 1 (`100>100`) false (equal, not strict); condition 2 false;
   condition 3 (`100<100`) false; condition 4 (`100>100`) false;
   condition 5 (`100<90`) false; condition 6 (`100>90 && 100<90`) — second
   half false. No condition matches. `MARelationState = NONE (0)`.
6. **Both numeric ATR calculations.** `FastMA=102, SlowMA=98, ATRUsed=4`,
   candle `High=105, Low=99`. `FastSlowDistanceATR = |102-98|/4 = 1.00`.
   Since `FastMA(102) >= SlowMA(98)`: `CandleExtremeFastDistanceATR =
   |105-102|/4 = 0.75`.
7. **Invalid/zero ATR.** Sensor publishes `Status = VALID` with
   `ATRUsed = 0` (a genuine but unusable value — should not happen given
   the Sensor's own validation, checked independently here regardless).
   `Status = ERROR`, `ReasonCode = ATR_INVALID`, `MARelationState = NONE`,
   both numeric fields neutralized.
8. **Stale versus repeated Sensor measurement — two different cases, not
   one.** *Stale*: this context's own `EvalBarIndex` is 500; the Sensor's
   `EvalBarIndex` is still 499 (the Sensor has not yet evaluated the
   current bar on this pass, an ordinary calculation-precedence timing
   case). `Status = NOT_READY`, `ReasonCode = SENSOR_NOT_READY` — the
   snapshot fails section 3's freshness check outright. *Repeated*: the
   Sensor's `EvalBarIndex` matches and `PublicationSeq` is unchanged from
   the last time this context read it (nothing changed since the last
   evaluation). This is **not** stale and **not** an error — the snapshot
   remains valid state per section 13's rule; this context may skip
   recomputing/republishing (same classification would result) or may
   simply recompute it again, both are correct. Reading an unchanged
   `PublicationSeq` never means "discard this measurement."
9. **Banner enabled and disabled.** With `Enable Display Banner = Yes`,
   a closed bar produces `MARelationState=1, FastSlowDistanceATR=0.42,
   CandleExtremeFastDistanceATR=0.18` — text `"MA:1  FS:0.42  CF:0.18"`
   differs from the last-drawn text, so it redraws once via
   `UTAM_ADD_OR_ADJUST` on the persisted line number. The next several
   closed bars produce the identical rounded text — no redraw occurs
   until the displayed text next changes. With `Enable Display Banner =
   No` from the start, none of this work happens at all; if the input is
   flipped from `Yes` to `No` mid-session, the existing banner is blanked
   exactly once, then no further banner work occurs while disabled.

------------------------------------------------------------------------

# Future scope — not v1 blockers, decided against for this design

Not open questions: the current design already decides against both of
these for v1, deliberately, not by omission. Recorded here only so a
future revision that genuinely needs either knows where to start, not
because either is unresolved today.

1. **Cross-chart Sensor wiring.** Out of scope — section 2/3 require
   same-chart wiring for v1, and the Sensor itself is explicitly
   local-chart-only (`Chakra_Sensor_Candle_MA_Raw_V1.md` section 1). A
   future context needing this would add the standard Sensor-contract
   cross-chart provenance/freshness fields (`SourceChartNumber`,
   `SourceDataAsOfDateTime`, etc.) — new scope, not a gap in this design.
2. **Historical backfill / full-recalculation reconstruction of past
   states.** Out of scope — section 11 excludes it by design, while still
   supporting ordinary progressive replay (section 11). A future need for
   backtest-accurate historical reconstruction would add hidden historical
   subgraphs to both documents, following the base Sensor/MCtx contracts'
   own documented pattern for that — new scope, not a gap in this design.
