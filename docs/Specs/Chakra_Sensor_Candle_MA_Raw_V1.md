# Chakra — Sensor: Candle/MA Raw Measurement (Draft, Revision 2)

Status: **design only, no ACSIL implementation yet.**

Builds directly on:
- `Chakra_Common_Interface_Envelope_Contract.md` (approved, Revision 8) —
  `ComponentKind`, `K_CHAKRA_STATUS`, `PublicationSeq`, and PV-ownership
  ranges, unmodified.
- `Chakra_Sensor_Interface_Contract.md` — this document's identity,
  freshness, historical-validity, and diagnostics sections follow it
  exactly.
- `K_Alert_CandleAndAverageRelation_V1.cpp` — a **behavioral reference for
  what raw inputs exist**, not code to copy. That study also computes MA
  relationships, candle/MA classifications, breach/touch history, and a
  display banner — **none of that belongs in this Sensor** (see
  "Non-goals"). Only its Fast/Mid/Slow MA and ATR study/subgraph inputs
  are preserved as this Sensor's own inputs.

------------------------------------------------------------------------

# Purpose

A lightweight, local-chart Sensor that reads three configured moving-
average subgraphs and one ATR subgraph, once per newly closed chart bar,
and publishes the raw values. **This Sensor publishes raw measurements
only** — it does not classify the MA relationship, does not compute any
derived ratio, and does not calculate the final Market Context values.
That work belongs entirely to `Chakra_MCtx_Candle_MA_Relation_V1.md`, the
one intended consumer this Sensor is designed alongside.

------------------------------------------------------------------------

# Non-goals — explicitly excluded

- The six-state MA relationship (fast/mid/slow arrangement).
- The fast/slow ATR ratio.
- The candle-extreme/fast-MA ATR ratio.
- Candle-close/MA classifications (above/below, body-percentage filtered
  or otherwise).
- Crossover direction detection.
- Bars-since-crossover tracking.
- Bars-since-MA-touch tracking.
- Trading direction or signal of any kind.
- A display banner or any other visual output (this Sensor has no
  subgraphs at all — see "Storage and performance").

All of the above are present in `K_Alert_CandleAndAverageRelation_V1.cpp`
and are deliberately not carried into this Sensor. The six-state
relationship and the two ATR ratios are computed by
`Chakra_MCtx_Candle_MA_Relation_V1.md` instead, from this Sensor's raw
output. Everything else in the list above (touch/crossover history,
signal direction) has no consumer in this pair of documents at all and is
not designed here.

------------------------------------------------------------------------

# 1. Naming and identity

- **`ComponentKind`** is always `K_CHAKRA_KIND_SENSOR`.
- **`ProducerTypeID`** is scoped to this Sensor specifically:
  ```cpp
  enum K_CHAKRA_SENSOR_CANDLEMARAW_TYPE
  {
      K_CHAKRA_SENSOR_CANDLEMARAW_TYPE_NONE = 0,
      K_CHAKRA_SENSOR_CANDLEMARAW_TYPE_V1   = 1,
  };
  ```
- **`PayloadSchemaVersion`** (int32 PV 20) scoped per `ProducerTypeID`,
  standard envelope convention.
- **One Sensor instance = one measurement family**: the fast/mid/slow MA
  values, ATR, and closed-candle high/low/close read together, in one pass,
  for one closed bar — a single coherent measurement act (per the base Sensor
  contract's own "coherent family" test), not three unrelated
  measurements bundled for convenience.
- **Local-chart only, no cross-chart mapping.** This Sensor reads its own
  chart's bar data and study arrays wired on that same chart
  (`GetStudyArrayUsingID`, never `GetStudyArrayFromChartUsingID`). None of
  the Sensor contract's cross-chart source-anchor fields (`SourceChartNumber`,
  `SourcePeriodStartDateTime`, `SourceDataAsOfDateTime`, etc.) apply or
  are published here.

Naming: `K_Chakra_Sensor_CandleMARaw_V1`.

**Cadence declaration**: closed-bar cadence on this Sensor's own chart —
publishes once per newly closed bar, never intrabar, never on the forming
bar.

------------------------------------------------------------------------

# 2. Inputs

| Input | Default | Purpose |
|---|---|---|
| `Enabled` | Yes | Standard enable/disable. |
| `Fast MA Study` / `Fast MA SG` | Study ID 3, SG 0 | Local chart/study/subgraph reference — matches `K_Alert_CandleAndAverageRelation_V1.cpp`'s own default wiring. |
| `Mid MA Study` / `Mid MA SG` | Study ID 3, SG 1 | Same. |
| `Slow MA Study` / `Slow MA SG` | Study ID 3, SG 2 | Same. |
| `ATR Study` / `ATR Study SG` | Study ID 9, SG 0 | Same. |
| `Debug Logging` | No | Standard debug-log toggle; logs decisions/transitions, never every update call (section 7). |

No other inputs. No cross-chart reference input — see section 1.

------------------------------------------------------------------------

# 3. Upstream dependency validation

Checked fresh on every newly closed bar, before publishing. Each failure
maps to the common-envelope `Status` (Int32 PV 2) — never a partial or
best-guess measurement:

1. **Array coverage** — the Fast/Mid/Slow MA arrays and the ATR array all
   cover the evaluated closed-bar index. **`Status`**: `NOT_READY` if the
   evaluation index is still within warmup (insufficient bars for the
   configured studies to have produced a value yet).
2. **MA values finite** — Fast/Mid/Slow MA values are all finite (no
   NaN/Inf). **`Status`**: `ERROR` if non-finite despite array coverage
   (a dependency that should be usable but isn't) — this is not an
   ordinary warmup condition once the array covers the index.
3. **ATR finite and greater than zero** — required for this Sensor's own
   output (`ATRUsed`, section 5) even though this Sensor performs no ATR
   division itself (that belongs to the consuming MCtx). **`Status`**:
   `NOT_READY` if still within the ATR study's own warmup window; `ERROR`
   if non-finite or `<= 0` despite being past it.
4. **`Enabled = No`** — **`Status`**: `DISABLED`, checked first, before any
   of the above.

If all checks pass: **`Status`**: `VALID`.

**No positivity requirement on the MA values themselves** — a moving
average can legitimately be any finite price level; only finiteness is
checked, matching this Sensor's own "raw measurement, no interpretation"
scope.

------------------------------------------------------------------------

# 4. Evaluation cadence

- **Evaluate newly closed bars only** (`sc.GetBarHasClosedStatus(Index) ==
  BHCS_BAR_HAS_CLOSED`). **The forming bar is never treated as a
  confirmed measurement** — no intrabar evaluation.
- **Do not repeatedly process the same closed bar.** A private tracker
  (`LastEvaluatedBarIndex`, int32, private range) records the last closed
  bar index this Sensor actually evaluated; a subsequent call observing
  the same index (e.g. a chart update that doesn't advance the bar) is a
  no-op — no re-fetch of dependent arrays, no re-validation, no
  republication.
- **Dependent arrays are fetched only when a new closed bar actually
  requires evaluation** — never fetched speculatively on every study
  update call (section 7).
- **Full-recalculation/historical-download behavior, aligned to the base
  Sensor contract's own low-churn rule:**
  1. On recalculation entry, publish `Status = NOT_READY` once.
  2. No intermediate publication during the historical sweep — every
     historical closed bar may be evaluated internally for correctness,
     but no persistent publication is committed per historical bar.
  3. On recalculation completion, publish exactly one final snapshot,
     anchored explicitly to the latest properly evaluated **closed** bar
     — normally `sc.ArraySize - 2`, never the still-forming
     `sc.ArraySize - 1` — `Status = VALID` (or the applicable
     `NOT_READY`/`ERROR`) for that bar, payload first, `PublicationSeq`
     last.
  4. Ordinary change-based publication resumes immediately afterward for
     each subsequent newly closed live bar.

  **Implementation-safety note**: the same-bar guard (`LastEvaluatedBarIndex`,
  above) exists to suppress *redundant* processing of an already-published
  bar during ordinary live operation — it must not be allowed to suppress
  step 3's *mandatory* final publication. During a recalculation sweep,
  `LastEvaluatedBarIndex` may already equal the final closed-bar index by
  the time step 3 is reached (the internal, non-publishing evaluation
  pass in step 2 can legitimately advance it), and an overly literal
  implementation could read that as "already processed, skip" and
  silently drop the required final publication. Recalculation must either
  bypass the same-bar guard entirely for step 3's own publication, or
  explicitly reset `LastEvaluatedBarIndex` to an unset/sentinel value at
  recalculation entry (step 1) before the sweep begins, so step 3's
  publication is never conditional on what the guard last recorded.
- **This Sensor has no output subgraphs** (section 6) — there is no
  historical-diagnostic-subgraph mechanism to describe, and none is
  needed: this Sensor never claims to expose reconstructed historical
  measurements, only its current closed-bar snapshot.

------------------------------------------------------------------------

# 5. Required Sensor output

Self-contained measurement/delivery record, state-consumption pattern —
common-envelope fields used exactly as baselined, never duplicated into
this Sensor's own payload. No event-style `EventSeq`; `Int64 PV 10` (the
event-style layer's conventional slot) stays unused.

## Common-envelope fields

| Field | Namespace/PV | Value for this Sensor |
|---|---|---|
| `SchemaVersion` | Int32 PV 1 | `K_CHAKRA_ENVELOPE_SCHEMA_VERSION_CURRENT` |
| `Status` | Int32 PV 2 | Section 3 |
| `ComponentKind` | Int32 PV 3 | `K_CHAKRA_KIND_SENSOR` |
| `ProducerTypeID` | Int32 PV 4 | `K_CHAKRA_SENSOR_CANDLEMARAW_TYPE_V1` |
| `EvalBarIndex` | Int32 PV 5 | This Sensor's own chart's bar index for the evaluated closed candle |
| `DecayClass` | Int32 PV 6 | **`FAST`**, always — an ordinary closed-bar chart measurement |
| `ReasonCode` | Int32 PV 7 | Section 6 |
| `PublicationSeq` | Int64 PV 1 | The delivery/measurement sequence — advances on publish-on-material-change, written last |
| `EvalDateTime` | Datetime PV 1 | The evaluated closed bar's own datetime |
| `ValidUntilDateTime` | Datetime PV 2 | `0` (sentinel) — never `STATIC_SESSION` |

**This is the "required source/master identity" the base Sensor contract
requires** — `SchemaVersion`/`ComponentKind`/`ProducerTypeID`/
`PayloadSchemaVersion`, all common-envelope or envelope-convention fields,
satisfy it; no additional identity field is invented.

## Layer payload

| Field | Namespace/PV | Purpose |
|---|---|---|
| `PayloadSchemaVersion` | Int32 PV 20 | Envelope convention |
| `FastMA` | Float PV 1 | |
| `MidMA` | Float PV 2 | |
| `SlowMA` | Float PV 3 | |
| `ATRUsed` | Float PV 4 | |
| `CandleHigh` | Float PV 5 | The evaluated closed candle's high |
| `CandleLow` | Float PV 6 | The evaluated closed candle's low |
| `CandleClose` | Float PV 7 | The evaluated closed candle's close |

No int32/int64/datetime payload fields beyond `PayloadSchemaVersion` — the
common envelope already carries everything else this Sensor needs to
publish (identity, bar index, timestamp, status, reason, sequence).

**`Status != VALID` means every payload field above is neutral/
stale-marked, never a partial measurement** — the same atomic-publication
discipline every Chakra layer in this framework follows.

------------------------------------------------------------------------

# 6. Diagnostics

```cpp
enum K_CHAKRA_SENSOR_CANDLEMARAW_REASON_CODE
{
    K_CHAKRA_SENSOR_CANDLEMARAW_REASON_NONE                = 0,   // = K_CHAKRA_REASON_NONE
    K_CHAKRA_SENSOR_CANDLEMARAW_REASON_ARRAYS_NOT_READY     = 1,
    K_CHAKRA_SENSOR_CANDLEMARAW_REASON_MA_VALUE_INVALID     = 2,
    K_CHAKRA_SENSOR_CANDLEMARAW_REASON_ATR_NOT_READY        = 3,
    K_CHAKRA_SENSOR_CANDLEMARAW_REASON_ATR_INVALID          = 4,
    K_CHAKRA_SENSOR_CANDLEMARAW_REASON_MEASUREMENT_PUBLISHED = 5
    // extend per future revision as needed, numbered above these common ones
};
```

**Written into the common-envelope `ReasonCode` field (Int32 PV 7,
section 5) — not a separate payload field.** Strictly self-referential:
describes only what this Sensor itself observed, never a downstream
consumer's judgment.

------------------------------------------------------------------------

# 7. Storage and performance

- **All public output uses persistent variables** (section 5). **No
  plotted or hidden data subgraphs of any kind.**
- **No historical subgraph clearing or backfilling** — there is nothing
  to clear or backfill, since none exist.
- **Dependent arrays fetched only when a new closed bar requires
  evaluation** (section 4) — never on every study update call.
- **Write every payload field first; write `PublicationSeq` last** — the
  standard commit-order discipline.
- **Debug logging defaults off, and logs decisions/transitions** (a
  `Status` change, a `ReasonCode` change) **— never every update call.**

------------------------------------------------------------------------

# 8. Worked examples

1. **Ordinary valid measurement.** Bar closes; Fast/Mid/Slow MA arrays and
   ATR array all cover the index; all four values finite; ATR `> 0`.
   `Status = VALID`, `ReasonCode = MEASUREMENT_PUBLISHED`, payload
   populated with the four values plus `CandleHigh`/`CandleLow`/
   `CandleClose`,
   `PublicationSeq` incremented last.
2. **Still warming up.** Evaluation index is bar 12; the configured Slow
   MA study needs 20 bars before producing a value; its array does not
   yet cover index 12. `Status = NOT_READY`, `ReasonCode =
   ARRAYS_NOT_READY`.
3. **Non-finite MA value.** Arrays cover the index, but the Mid MA study
   returns `NaN` this bar (an unrelated dependency of that study briefly
   broke). `Status = ERROR`, `ReasonCode = MA_VALUE_INVALID` — not treated
   as ordinary warmup, since array coverage already establishes the
   study should be producing real values by now.
4. **Invalid ATR.** All three MA values are fine; the ATR study's array
   covers the index but returns `0`. `Status = ERROR`, `ReasonCode =
   ATR_INVALID`.
5. **Same closed bar observed twice.** A chart update calls this study
   again without the bar having advanced (`LastEvaluatedBarIndex` already
   equals the current closed-bar index). No array re-fetch, no
   re-validation, no publication — a pure no-op, per section 4.
6. **Full recalculation.** On recalc entry, `Status = NOT_READY` publishes
   once and `LastEvaluatedBarIndex` resets to its unset sentinel. No
   further publication occurs until the sweep reaches its final closed
   bar (`sc.ArraySize - 2`), at which point exactly one snapshot publishes
   for that bar's own properly-evaluated state — unconditionally, not
   gated on `LastEvaluatedBarIndex`, even though the internal (non-
   publishing) sweep pass may have already advanced it to that same index.
   Ordinary per-closed-bar publication resumes from the next live bar
   forward.

------------------------------------------------------------------------

# Unresolved design questions

None of substance for this narrow first implementation. The default
Fast/Mid/Slow MA and ATR study/subgraph wiring (section 2) mirrors
`K_Alert_CandleAndAverageRelation_V1.cpp`'s own defaults as a reasonable
starting point, not an architectural commitment — an operator is expected
to rewire these to whatever concrete MA/ATR studies a given chart actually
uses.
