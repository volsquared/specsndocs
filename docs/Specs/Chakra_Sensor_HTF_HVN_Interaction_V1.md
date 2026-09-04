# Chakra — Sensor: HTF HVN Interaction (Draft, Revision 1)

> **PARKED for a later vertical slice — not Chakra's first walking
> skeleton.** See the status note at the end of this document. This is
> documentation only: no ACSIL source is created or modified by this
> document, no baselined Chakra contract is changed, and the Sensor
> described here is not implemented.

Status: **design only, parked, no ACSIL implementation.**

Builds directly on:
- `Chakra_Sensor_Interface_Contract.md` — the generic Sensor contract this
  document's identity, freshness, historical-validity, and diagnostics
  sections follow exactly.
- `Chakra_Common_Interface_Envelope_Contract.md` (approved, Revision 8) —
  `ComponentKind`, `K_CHAKRA_STATUS`, `PublicationSeq`, and PV-ownership
  ranges, unmodified.
- `K_HTF_Volume_Structure_Producer_V1_FINAL_SRS_v1_3.md` and
  `K_HTF_Volume_Structure_Producer_V1.cpp` — read for background only; this
  Sensor does not compute HVN structure and does not depend on the
  Producer's own internal ranking algorithm, only on what the Consumer
  already exports.
- `K_HTF_Volume_Structure_Consumer_V1.cpp` — the exact, existing,
  unmodified source of the HVN boundary/profile fields this Sensor reads.
  Field/subgraph names below are taken directly from this file, not
  assumed.
- `K_Alert_Tatva_CandleStick_V1.cpp` — the exact, existing, unmodified
  source of the breach-condition formulas this Sensor's own breach logic
  is modeled on (Signal Mode specifically). Formulas are expressed in this
  document; no large source blocks are quoted.

------------------------------------------------------------------------

# Purpose

Specify a generic Chakra Sensor that measures how each newly closed LTF
(lower-timeframe) candle interacts with **one** configured HVN
(High-Volume-Node) zone, as exported by the existing, unmodified
`K_HTF_Volume_Structure_Consumer_V1`. **This Sensor does not calculate an
HVN.** HTF volume-profile computation belongs entirely to
`K_HTF_Volume_Structure_Producer_V1`; boundary projection onto the LTF
chart belongs entirely to `K_HTF_Volume_Structure_Consumer_V1`. This
Sensor's own job starts and ends at reading those already-projected
boundaries and measuring one closed candle against them.

------------------------------------------------------------------------

# Non-goals — explicitly excluded

- HTF VAP or HVN calculation of any kind (the Producer's job).
- Combining multiple HVNs into one measurement or one envelope (see
  "Generic instance model," below).
- Choosing which HVN is strategically important, or any ranking/selection
  logic beyond reading a single, operator-configured rank.
- Trigger-style edge detection or deduplication of any kind.
- Entry permission or trade-decision logic of any kind.
- Stop or target price calculation.
- Mode Baton publication, or any Chakra-to-Mode-Baton signal handoff.
- Position sizing.
- Order execution.
- Post-entry management.
- Rejection/retest/failed-breakout interpretation — this Sensor reports
  what one closed candle did, nothing about what happens afterward.

------------------------------------------------------------------------

# Generic instance model

**One Sensor instance observes exactly one configured HVN rank.** HVN1 and
HVN2 are distinct volume areas with independent existence, independent
boundaries, and independent breach conditions — they are never combined
into a single envelope or a single measurement record.

- Sensor instance A, configured for HVN1, observes only HVN1.
- Sensor instance B, configured for HVN2, observes only HVN2.

Each instance's own delivery record includes both the configured HVN rank
and the active HTF producer profile ID it evaluated against (section 10) —
never left implicit, since the same instance's meaning changes entirely if
either changes.

**Source-grounded, deterministic rank mapping — fixed for v1, not
independently configurable**: `K_HTF_Volume_Structure_Consumer_V1` exports
exactly two fixed HVN slots — `Subgraph_HVN1Low`/`Subgraph_HVN1High`
(`sc.Subgraph[0]`/`[1]`, zone index 0) and `Subgraph_HVN2Low`/
`Subgraph_HVN2High` (`sc.Subgraph[2]`/`[3]`, zone index 1)
(`K_HTF_Volume_Structure_Consumer_V1.cpp:385-394`, `537-540`). The
"Configured HVN Rank" input deterministically selects the matching pair —
there is no independent low/high subgraph override:

```text
Rank 1 -> Consumer Subgraph[0]/Subgraph[1] (HVN1Low/HVN1High) and PROFILE_FLAG_HVN1_FOUND
Rank 2 -> Consumer Subgraph[2]/Subgraph[3] (HVN2Low/HVN2High) and PROFILE_FLAG_HVN2_FOUND
```

**Deliberately closed, not left as an open question**: allowing the low/
high subgraph pair to be configured independently of the rank would permit
contradictory wiring — e.g. a "Rank 1" instance whose existence check
reads `PROFILE_FLAG_HVN1_FOUND` while its boundaries are wired to
`Subgraph[2]`/`[3]` (HVN2's own values). The rank alone determines both
the boundary subgraphs and the existence flag, with no way to wire them
apart. More than two ranks (should a future Consumer revision export
them) is future scope for a new document revision, not a v1 configuration
option.

------------------------------------------------------------------------

# 1. Naming and identity

- **`ComponentKind`** is always `K_CHAKRA_KIND_SENSOR`.
- **`ProducerTypeID`** is scoped to this Sensor family specifically:
  ```cpp
  enum K_CHAKRA_SENSOR_HTFHVNINTERACTION_TYPE
  {
      K_CHAKRA_SENSOR_HTFHVNINTERACTION_TYPE_NONE = 0,
      K_CHAKRA_SENSOR_HTFHVNINTERACTION_TYPE_TATVASTYLE_V1 = 1,
      // one member per concrete named measurement policy, assigned as each is built
  };
  ```
- **`PayloadSchemaVersion`** (int32 PV 20) scoped per `ProducerTypeID`,
  standard envelope convention.
- **One Sensor instance = one measurement family**, per the base Sensor
  contract's own hard rule: this Sensor's family is "how one closed LTF
  candle relates to one configured HVN boundary pair" — direction (above/
  below/inside), distance, and Tatva-style breach qualification are all
  facets of that one family, published together; a different HVN rank is
  a different instance, not a second output of the same one (see "Generic
  instance model," above).

Naming: `K_Chakra_Sensor_HTFHVNInteraction_<DescriptiveName>_V<n>`.

**Cadence declaration** (required by the base Sensor contract's own §5):
**closed-bar cadence on this Sensor's own LTF chart** — this Sensor
publishes once per newly closed bar on the chart it is attached to, never
intrabar, never speculatively on the forming bar. It is *not* an
HTF-boundary-cadence Sensor itself — the HTF cadence belongs to the
Producer/Consumer pair it reads from; this Sensor's own cadence is
whatever LTF chart it is placed on.

------------------------------------------------------------------------

# 2. Inputs

Modeled directly on `K_Alert_Tatva_CandleStick_V1.cpp`'s own Signal Mode
inputs, renamed for HVN-interaction clarity per instruction — same
semantic role, new names:

**Default values for the Tatva-derived inputs are adopted directly from
`K_Alert_Tatva_CandleStick_V1.cpp`'s own proven Signal Mode defaults** —
a starting point, not an architectural commitment; all four remain
ordinary configurable inputs.

| Input | Renamed from (Tatva) | Default | Purpose |
|---|---|---|---|
| `Enabled` | — | Yes | Standard enable/disable. |
| `HTF Volume Structure Consumer Study ID` | — (new; Tatva used `periodOHLCStudy`) | — | Study ID only, not chart+study — **for v1, this Sensor and its configured Consumer must run on the same LTF chart** (section 3; item 11). No chart reference input, and no cross-chart index mapping anywhere in this Sensor. |
| `Configured HVN Rank` | — (new) | — | `1` or `2`. Deterministically selects the Consumer's fixed `Subgraph[0]`/`[1]` (rank 1) or `Subgraph[2]`/`[3]` (rank 2) boundary pair and the matching `PROFILE_FLAG_HVN1_FOUND`/`HVN2_FOUND` existence flag — no independent subgraph override (see "Generic instance model"). |
| `Active Profile ID Subgraph` | — (new) | `Subgraph[8]` | Defaults to the Consumer's "Active Profile ID". |
| `Active Profile Flags Subgraph` | — (new) | `Subgraph[9]` | Defaults to the Consumer's "Active Profile Flags". |
| `ATR Study` / `ATR Study SG` | `ATRStudy` / `ATRStudySG` | — | Unchanged role — volatility input for the ATR-fallback proximity rule and distance-in-ATR-units output. |
| `Min Body % Beyond Boundary` | `percentBodyAboveBelowFinish` | `20` | Minimum percent of candle body beyond the applicable HVN boundary (section 7/8 formula) — Tatva's own default. |
| `Ignore Candle Extreme Relation To Boundary` | `ignoreCandleLowHighRelationWithRange` | No | If Yes, skips the near-side candle-extreme proximity check entirely (section 7/8). |
| `Max Ticks Outside Boundary` | `signalModeMaxLengthTicksOutsideRange` | `8` | Fixed-tick proximity bound; `0` selects the ATR-fallback rule instead (section 7/8) — Tatva's own default. |
| `Near-Boundary ATR Multiplier` | `boCandleNearRangeATRMultipleier` | `1.0` | ATR-fallback near-side multiplier — Tatva's own default. |
| `Far-Close ATR Multiplier` | `boCandleFarRangeATRMultipleier` | `2.0` | ATR-fallback far-side (close-distance) multiplier — Tatva's own default. |
| `Debug Logging` | — | No | Standard debug-log toggle. |

**Explicitly not carried over from Tatva**: `Tatva Mode` (Entry mode is out
of scope — this Sensor only ever measures, per "Sensor versus Market
Context boundary," below), parent-Tatva inclusion, NB-Aug augmentation,
benchmark-line mode (OR/Close alternatives — the boundary here is always
the Consumer's HVN projection, never a substitute range source), and
drawing-tolerance inputs (this Sensor is a measurement producer, not a
chart-drawing study).

------------------------------------------------------------------------

# 3. Upstream source validation

**Same-chart requirement, v1 (item 11): this Sensor and its configured
`K_HTF_Volume_Structure_Consumer_V1` instance must run on the same LTF
chart.** The Producer performs HTF volume-structure computation on its own
HTF chart; the existing, unmodified Consumer already performs the
HTF-to-LTF projection and index alignment. This Sensor reads the
Consumer's already-aligned arrays directly, by bar index, on that shared
chart — **no cross-chart index mapping of any kind is performed inside
this Sensor.** The Consumer reference input is a Study ID only (section 2)
— there is no chart-selection input, because there is only ever one valid
chart to read from.

Before producing a measurement, all of the following are checked. Each
failure maps to one of the common envelope's own `Status` values (`Int32
PV 2`, `K_CHAKRA_STATUS`) — `NOT_READY`, `ERROR`, or (if the Sensor itself
is turned off) `DISABLED` — never a partial or best-guess measurement.
There is no Sensor-specific status field; `Status` is the common-envelope
field, not duplicated in this Sensor's own payload (section 10).

1. **Configured study reachable and structurally usable.** The configured
   Study ID (on this Sensor's own chart, per the same-chart requirement
   above) and its required subgraphs/arrays are reachable and
   structurally valid (correct array coverage, no null/invalid pointer).
   **This document does not claim ACSIL can verify that the configured
   study is specifically a `K_HTF_Volume_Structure_Consumer_V1`
   instance** — ACSIL has no reliable mechanism to prove a concrete study
   *type* from its subgraph outputs alone; this Sensor can only verify
   that *something* reachable at the configured Study ID exposes arrays
   of the expected shape. Getting this wrong is a configuration error the
   operator is responsible for, not something this Sensor detects.
   **`Status`**: `ERROR` if a previously-reachable reference stops
   resolving or the required arrays/flags are structurally absent;
   `NOT_READY` if the Sensor's own evaluation index is still within its
   own warmup window (insufficient bars) or the configured study's own
   arrays don't yet cover the evaluation index.
2. **Active profile flags, `PROFILE_FLAG_VALID` (bit 0)** — read from the
   Active Profile Flags subgraph. If this bit is not set, the Consumer's
   own current profile is not itself valid — ordinarily because the
   Producer has not yet produced its first valid profile (early in a
   session). **`Status`**: `NOT_READY` — verified against
   `K_HTF_Volume_Structure_Producer_V1_FINAL_SRS_v1_3.md` section 7.3,
   `PROFILE_FLAG_VALID = 1 << 0`.
3. **Selected-HVN existence — `PROFILE_FLAG_HVN1_FOUND` (bit 5) /
   `PROFILE_FLAG_HVN2_FOUND` (bit 6), matching this instance's configured
   rank (section "Generic instance model").** This is the authoritative
   existence check, not a boundary-value heuristic. **If the selected HVN
   does not exist for the current, otherwise-valid active profile — for
   example, a Rank-2 instance against a profile that only produced HVN1 —
   this Sensor does not fall back to reading the other rank's
   boundaries.** **`Status`**: `NOT_READY` — a missing HVN2 (or HVN1) on a
   given profile is an ordinary, expected outcome of the Producer's own
   selection rules (`K_HTF_Volume_Structure_Producer_V1_FINAL_SRS_v1_3.md`
   section 12.7 — HVN2 is only accepted if it clears an additional
   minimum-relative-to-HVN1 bar), not an error condition.
4. **Boundary validity** — `HVNLow` and `HVNHigh` both finite, and
   `HVNHigh > HVNLow`. **No positivity requirement** — the profile flags
   (step 3) are what establish that the selected zone genuinely exists;
   requiring both prices to be strictly positive on top of that would be
   redundant with the flag check and would incorrectly reject a
   legitimate low-priced instrument. **`Status`**: `ERROR` if this check
   fails despite the corresponding `FOUND` flag being set (step 3) — an
   inconsistency between "the Producer says this zone exists" and "the
   boundary values are usable" that should not occur once dependencies
   are available, and is worth surfacing distinctly from an ordinary
   not-yet-ready state.
5. **Active profile ID present and unchanged mid-measurement** — read once
   per evaluation, from the Active Profile ID subgraph, and carried
   through the entire evaluation as a single snapshot (never re-read
   mid-calculation). **`Status`**: `NOT_READY` if the array does not yet
   cover the evaluation index.
6. **ATR available and finite/positive** — required whenever the ATR
   fallback path is used (`Max Ticks Outside Boundary == 0`) or whenever
   `DistanceBeyondBoundaryATR` is computed (section 6, always). **`Status`**:
   `NOT_READY` if still within the ATR study's own warmup window;
   `ERROR` if invalid despite being past it (a dependency that should be
   available but isn't). Either way, an invalid ATR fails the whole
   measurement, not just the ATR-dependent output field — this Sensor
   never publishes a partially-valid record.

**Profile freshness is not a separate check, and no elapsed-LTF-bar
timeout exists — closed, not left open.** An HTF profile does not become
stale merely because many LTF bars have elapsed since it became active.
It remains the currently active, usable profile — and this Sensor keeps
measuring against it — for as long as the Consumer itself continues to
expose it as the active profile (steps 1, 2, and 5 above: valid flags,
current profile ID, arrays covering the evaluation index). **A new active
HTF profile is evaluated normally, with no historical memory requirement
and no crossing precondition.** If the Consumer's Active Profile ID
changes between one closed LTF candle and the next, this Sensor simply
evaluates the new profile's own current HVN boundaries against the
current candle — it does **not** require any prior crossing of the
previous profile's boundary, and carries no state across a profile-ID
change beyond the ordinary "read fresh, validate fresh" cycle described
above.

------------------------------------------------------------------------

# 4. Evaluation cadence

- **Evaluate newly closed LTF candles only** (`sc.GetBarHasClosedStatus(Index)
  == BHCS_BAR_HAS_CLOSED`, the same gate `K_Alert_Tatva_CandleStick_V1.cpp:219-221`
  uses). **The forming candle is never treated as confirmed** — no
  intrabar evaluation, no speculative measurement of an in-progress bar.
- **Full-recalculation/historical-download behavior — aligned exactly to
  the base Sensor contract's own §8 rule, not merely "no live
  measurement":**
  1. **On recalculation entry, publish `Status = NOT_READY` once** —
     envelope + identity fields written, payload neutralized, no
     measurement claimed yet.
  2. **No intermediate publication during the historical sweep** — this
     Sensor evaluates every historical closed bar internally (for
     correctness of the final result) but does not commit a persistent
     publication for each one; no churn, no misleading intermediate
     `VALID` snapshots during warmup.
  3. **On recalculation completion, publish exactly one final snapshot**
     — `Status = VALID` (or the appropriate `NOT_READY`/`ERROR` if the
     latest evaluated closed bar itself fails section 3's checks),
     representing the **latest properly evaluated closed bar**, using the
     same commit discipline as ordinary live publication (payload first,
     `PublicationSeq` last).
  4. **Ordinary change-based publication resumes immediately afterward**
     — each subsequent newly closed live bar is evaluated and published
     exactly as described throughout this document, with no further
     recalculation-specific handling.
- **This Sensor publishes through PVs only** — no output subgraphs are
  specified in this document, so there is no historical-diagnostic-
  subgraph mechanism to describe.
- Upstream validation (section 3) runs fresh on every closed-candle
  evaluation — no caching of a prior validation result across bars.

------------------------------------------------------------------------

# 5. Raw close relation

For the closed LTF candle's `Close`, against the currently valid `HVNLow`/
`HVNHigh` (section 3):

```text
NONE       = 0   (default/reset value only — see below)
ABOVE_HVN  = 1   when Close > HVNHigh
BELOW_HVN  = 2   when Close < HVNLow
INSIDE_HVN = 3   otherwise, including exact equality with either boundary
```

**No `UNAVAILABLE` member exists in this enum.** `RawCloseRelation` is
**readable only when the common-envelope `Status == VALID`** (section 10)
— when `Status != VALID`, the field carries no defined meaning, and this
document does not support or encourage a consumer reading it without
first checking `Status`. This is a deliberate correction: an earlier
draft added a `RawCloseRelation = UNAVAILABLE` category specifically so a
consumer *could* skip the `Status` check, which directly contradicts the
common envelope's own atomic-validity rule (section 10) — every payload
field, `RawCloseRelation` included, is only meaningful conditional on
`Status == VALID`, with no per-field escape hatch.

`INSIDE_HVN` is the inclusive default — a close exactly on `HVNHigh` or
`HVNLow` is `INSIDE_HVN`, never `ABOVE_HVN`/`BELOW_HVN` by a strict-vs-
non-strict inequality ambiguity.

------------------------------------------------------------------------

# 6. Distances

```text
Above (Close > HVNHigh):
    DistanceBeyondBoundaryPrice = Close - HVNHigh
    DistanceBeyondBoundaryATR   = DistanceBeyondBoundaryPrice / ATR

Below (Close < HVNLow):
    DistanceBeyondBoundaryPrice = HVNLow - Close
    DistanceBeyondBoundaryATR   = DistanceBeyondBoundaryPrice / ATR

Inside:
    DistanceBeyondBoundaryPrice = 0
    DistanceBeyondBoundaryATR   = 0
```

**No internal-zone percentile is added in v1** — `INSIDE_HVN` reports a
flat zero distance regardless of where inside the zone the close actually
landed. A future revision could add one; not designed here.

**`NearSideCandleExtremeDistance` — a raw measurement, always computed
when `Status == VALID`, never conditioned on any input:**

```text
Above (RawCloseRelation == ABOVE_HVN):
    NearSideCandleExtremeDistance = CandleLow - HVNHigh

Below (RawCloseRelation == BELOW_HVN):
    NearSideCandleExtremeDistance = HVNLow - CandleHigh

Inside (RawCloseRelation == INSIDE_HVN):
    NearSideCandleExtremeDistance = 0
```

**`Ignore Candle Extreme Relation To Boundary` (section 2) affects only
whether this measurement participates in the Tatva-style breach gate
(sections 7-8) — it never zeroes, suppresses, or otherwise alters this raw
value.** A value of exactly `0` is itself a legitimate measurement (the
candle's near-side extreme touched the boundary precisely) and must never
be confused with "not computed" or "check disabled" — this field carries
no sentinel meaning distinct from its literal price-distance value at any
time `Status == VALID`.

------------------------------------------------------------------------

# 7. Tatva-style upper breach condition

**`HVNHigh` plays the role of Tatva's `rangeHigh`.** Formula grounded
directly in `K_Alert_Tatva_CandleStick_V1.cpp`'s own Signal Mode long-side
logic (lines 300-370), translated field-for-field:

A closed candle qualifies (`UpperBreach = true`) only if **all** of the
following hold:

1. **Close above boundary**: `Close > HVNHigh`.
2. **Green candle**: `Close > Open` (`isCandleGreen`, line 15-17).
3. **Minimum body-beyond-boundary percentage**:
   ```text
   BodyPercentBeyondBoundary = (|HVNHigh - Close| / |Open - Close|) * 100
                                (0 if |Open - Close| == 0)
   ```
   must be `>= Min Body % Beyond Boundary` (`bodyRatioAboveBelowRange`,
   lines 19-26, with `rangeHighLow = HVNHigh`). **This value can exceed
   `100`** — if `Open` itself already sits beyond `HVNHigh`, the numerator
   (`|HVNHigh - Close|`) can be larger than the denominator (`|Open -
   Close|`), matching Tatva's own formula exactly (not clamped there
   either); this is not a bug to guard against, and the `>= threshold`
   comparison remains correct unclamped.
4. **Near-side candle-extreme relation to the boundary**, using
   `NearSideCandleExtremeDistance` as already defined in section 6 above
   (`CandleLow - HVNHigh` for this side) — **`Ignore Candle Extreme
   Relation To Boundary` controls only whether this check participates in
   the breach gate; it never affects the measurement itself.** If `Ignore
   Candle Extreme Relation To Boundary = Yes`, this gate is skipped
   (always passes) — the measurement is still computed and published
   exactly as section 6 defines it, only its use as a breach precondition
   is bypassed. Otherwise, exactly one of:
   - **Fixed-tick rule** (`Max Ticks Outside Boundary > 0`): passes if
     `NearSideCandleExtremeDistance <= MaxTicksOutsideBoundary * TickSize`.
   - **ATR-fallback rule** (`Max Ticks Outside Boundary == 0`): passes if
     `NearSideCandleExtremeDistance <= ATR * NearBoundaryATRMultiplier`
     **and** `DistanceBeyondBoundaryPrice >= 0` **and**
     `DistanceBeyondBoundaryPrice <= ATR * FarCloseATRMultiplier`
     (mirrors lines 311-329 exactly, including the redundant-but-source-
     faithful `>= 0` check, which is always true here since step 1 already
     established `Close > HVNHigh`).

**Both requirements (fixed-tick or ATR-fallback) govern the *same*
`NearSideCandleExtremeDistance`/`DistanceBeyondBoundaryPrice` fields
already defined in sections 5-6 above** — the breach condition does not
introduce a second, independent distance concept.

------------------------------------------------------------------------

# 8. Tatva-style lower breach condition

**Mirrors section 7 exactly, using `HVNLow`** (`rangeLow`), grounded in
the same file's short-side logic (lines 373-437):

A closed candle qualifies (`LowerBreach = true`) only if **all** hold:

1. **Close below boundary**: `Close < HVNLow`.
2. **Red candle**: `Close < Open` (`isCandleRed`, lines 11-13).
3. **Minimum body-beyond-boundary percentage**:
   ```text
   BodyPercentBeyondBoundary = (|HVNLow - Close| / |Open - Close|) * 100
                                (0 if |Open - Close| == 0)
   ```
   must be `>= Min Body % Beyond Boundary`. **Can exceed `100`**, same
   reasoning as section 7 (an `Open` already beyond `HVNLow`) — not
   clamped there either.
4. **Near-side candle-extreme relation**, using
   `NearSideCandleExtremeDistance` as already defined in section 6 above
   (`HVNLow - CandleHigh` for this side) — same rule as section 7: the
   `Ignore` input only bypasses this as a breach precondition and never
   affects the measurement itself. Skipped as a gate (always passes) if
   `Ignore Candle Extreme Relation To Boundary = Yes`; otherwise exactly
   one of:
   - **Fixed-tick rule**: passes if `NearSideCandleExtremeDistance <=
     MaxTicksOutsideBoundary * TickSize`.
   - **ATR-fallback rule**: passes if `NearSideCandleExtremeDistance <=
     ATR * NearBoundaryATRMultiplier` **and**
     `DistanceBeyondBoundaryPrice >= 0` **and**
     `DistanceBeyondBoundaryPrice <= ATR * FarCloseATRMultiplier`
     (mirrors lines 384-401).

**`UpperBreach` and `LowerBreach` are mutually exclusive for one evaluated
closed candle** — structurally guaranteed, since step 1 of each requires
`Close` on strictly opposite sides of two different boundaries
(`HVNHigh`/`HVNLow`, with `HVNHigh > HVNLow` already validated in section
3) and steps 2 require opposite candle color; no candle can satisfy both
conditions' first two requirements simultaneously.

------------------------------------------------------------------------

# 9. Gap candles

**No special gap-candle rule is added.** A gap candle (one whose `Open`
lands beyond the HVN boundary before any intrabar price action) passes or
fails through the ordinary section 5-8 rules exactly like any other
candle — the body-percentage, near-side-extreme, and ATR-proximity checks
already govern how far outside the boundary a qualifying candle may sit,
with no separate gap-detection logic layered on top.

------------------------------------------------------------------------

# 10. Required Sensor output

**Self-contained measurement/delivery record**, per the base Sensor
contract's state-consumption pattern — the common envelope's own fields
used exactly as defined, never duplicated into this Sensor's own payload.
This Sensor has no event-style `EventSeq` (state-style, not event-style —
base Sensor contract §5, §9) and therefore **`Int64 PV 10` (the
event-style layer's conventional `EventSeq` slot) is never assigned by
this Sensor and stays unused.**

**No `MasterCycleSeq`-equivalent field** — the base Sensor and Envelope
contracts define no such concept anywhere; not invented here.

## Common-envelope fields (used exactly as baselined — not redefined, not duplicated)

| Field | Namespace/PV | Value for this Sensor |
|---|---|---|
| `SchemaVersion` | Int32 PV 1 | `K_CHAKRA_ENVELOPE_SCHEMA_VERSION_CURRENT` |
| `Status` | Int32 PV 2 | `K_CHAKRA_STATUS` — `NOT_READY`/`VALID`/`ERROR`/`DISABLED` (section 3) |
| `ComponentKind` | Int32 PV 3 | `K_CHAKRA_KIND_SENSOR` (section 1) |
| `ProducerTypeID` | Int32 PV 4 | `K_CHAKRA_SENSOR_HTFHVNINTERACTION_TYPE_*` (section 1) |
| `EvalBarIndex` | Int32 PV 5 | This Sensor's own chart's bar index for the evaluated closed candle |
| `DecayClass` | Int32 PV 6 | **`FAST`**, always — this is an ordinary closed-bar measurement, never `STATIC_SESSION`/`SLOW` |
| `ReasonCode` | Int32 PV 7 | This Sensor's own `K_CHAKRA_SENSOR_HTFHVNINTERACTION_REASON_CODE` value (section 11) |
| `PublicationSeq` | **Int64 PV 1** | Snapshot commit marker — advances on publish-on-material-change (base Sensor contract semantics), written last |
| `EvalDateTime` | Datetime PV 1 | The evaluated closed candle's own datetime |
| `ValidUntilDateTime` | Datetime PV 2 | **`0`** (sentinel) — only meaningful when `DecayClass == STATIC_SESSION`, which this Sensor never sets; freshness tolerance is entirely the downstream consumer's own concern (`Age = now - EvalDateTime`, compared against the consumer's own tolerance for `FAST`) |

## Layer payload (this Sensor's own fields only — genuinely new information, nothing already carried by the envelope)

| Field | Namespace/PV | Purpose |
|---|---|---|
| `PayloadSchemaVersion` | Int32 PV 20 | Envelope convention (section 1) |
| `ConfiguredHVNRank` | Int32 PV 21 | `1` or `2` — this instance's own configuration (section 1) |
| `ActiveHTFProfileID` | Int32 PV 22 | Snapshot read from the Consumer's Active Profile ID subgraph (section 3) |
| `SourceConsumerStudyID` | Int32 PV 23 | Provenance — the chart number is never carried separately, since the same-chart requirement (section 3, item 11) makes it always equal to this Sensor's own chart |
| `RawCloseRelation` | Int32 PV 24 | `0 = NONE` / `1 = ABOVE_HVN` / `2 = BELOW_HVN` / `3 = INSIDE_HVN` (section 5) — **readable only when `Status == VALID`; carries no defined meaning otherwise, and no `UNAVAILABLE` member exists** |
| `UpperBreachCondition` | Int32 PV 25 | `0`/`1` — section 7, mutually exclusive with `LowerBreachCondition` |
| `LowerBreachCondition` | Int32 PV 26 | `0`/`1` — section 8 |
| `HVNLow` | Float PV 1 | Section 3 |
| `HVNHigh` | Float PV 2 | Section 3 |
| `CandleOpen` | Float PV 3 | |
| `CandleHigh` | Float PV 4 | |
| `CandleLow` | Float PV 5 | |
| `CandleClose` | Float PV 6 | |
| `ATRUsed` | Float PV 7 | The exact ATR value this evaluation used |
| `DistanceBeyondBoundaryPrice` | Float PV 8 | Section 6 |
| `DistanceBeyondBoundaryATR` | Float PV 9 | Section 6 |
| `BodyPercentBeyondBoundary` | Float PV 10 | Section 7/8 — computed against whichever boundary is applicable (`HVNHigh` for an above-close, `HVNLow` for a below-close; `0` when `INSIDE_HVN`). **Can exceed `100`** — not clamped, matching Tatva's own formula. |
| `NearSideCandleExtremeDistance` | Float PV 11 | Section 6 — a raw measurement, always computed from `CandleLow`/`CandleHigh` whenever `Status == VALID` (`CandleLow - HVNHigh` above, `HVNLow - CandleHigh` below, `0` when `INSIDE_HVN`). **Never zeroed or suppressed by `Ignore Candle Extreme Relation To Boundary`** — that input affects only sections 7-8's breach gate, never this field. A published `0` means the candle's near-side extreme touched the boundary exactly, a legitimate measurement, not "unavailable" or "check skipped." |

**Explicitly not duplicated in this payload, because the common envelope
already carries them**: `Status`, `ReasonCode`, `EvalBarIndex` (was
`EvaluatedCandleIndex`), `EvalDateTime` (was `EvaluatedCandleDateTime`),
`PublicationSeq` (was misplaced at Int64 PV 10 in an earlier draft — that
slot is reserved for event-style layers' `EventSeq` only, which this
Sensor, being state-style, never uses).

**`Status != VALID` means every payload field above is neutral/
stale-marked, never a partial measurement** — the same "atomic
publication" discipline every other Chakra layer in this framework
already follows. A consumer must gate on the common-envelope `Status`
field before reading any payload field, `RawCloseRelation` included —
this document does not support or encourage bypassing that gate.

------------------------------------------------------------------------

# 11. Diagnostics

**These are the concrete values this Sensor writes into the
common-envelope `ReasonCode` field (Int32 PV 7, section 10) — not a
separate payload field of their own.**

```cpp
enum K_CHAKRA_SENSOR_HTFHVNINTERACTION_REASON_CODE
{
    K_CHAKRA_SENSOR_HTFHVNINTERACTION_REASON_NONE                    = 0,   // = K_CHAKRA_REASON_NONE
    K_CHAKRA_SENSOR_HTFHVNINTERACTION_REASON_CONSUMER_UNREACHABLE     = 1,
    K_CHAKRA_SENSOR_HTFHVNINTERACTION_REASON_PROFILE_INVALID          = 2,
    K_CHAKRA_SENSOR_HTFHVNINTERACTION_REASON_SELECTED_HVN_NOT_FOUND   = 3,
    K_CHAKRA_SENSOR_HTFHVNINTERACTION_REASON_BOUNDARY_INVALID         = 4,
    K_CHAKRA_SENSOR_HTFHVNINTERACTION_REASON_ATR_UNAVAILABLE          = 5,
    K_CHAKRA_SENSOR_HTFHVNINTERACTION_REASON_MEASUREMENT_PUBLISHED    = 6
    // extend per concrete instance as needed, numbered above these common ones
};
```

**Strictly self-referential** — this enum describes only what this Sensor
itself observed (an unreachable Consumer, an invalid or non-existent
selected HVN, an invalid boundary or ATR), never a downstream consumer's
judgment about the measurement. No Trigger/EntryGate/Market Context
disposition is ever described here, per the envelope's own self-
referential `ReasonCode` principle.

------------------------------------------------------------------------

# 12. Intended downstream use (non-binding)

**Not specified here — recorded only as context for why this Sensor's
output is shaped the way it is.** A future Market Context component could
consume this Sensor's measurement and separately publish:

1. **Current Tatva-style breach context** — upper breach / lower breach /
   no breach / unavailable, derived from `UpperBreachCondition`/
   `LowerBreachCondition` and the common-envelope `Status`.
2. **Continuing location state** — above HVN / inside HVN / below HVN /
   unavailable, derived from `RawCloseRelation` and `Status`.

**This Sensor records the current closed candle's condition only — it
does not decide whether that condition is a fresh transition.** A future
Trigger Strategy owns edge detection and deduplication (e.g. "this is the
*first* candle to qualify as an upper breach after N candles that
didn't"), exactly as every other Sensor in this framework leaves edge
detection to TriggerStr. This document does not specify the Market
Context contract, its PV layout, or its own edge-detection rules — that
is a separate, future design pass.

------------------------------------------------------------------------

# 13. Worked examples

1. **Qualifying upper breach.** `HVNHigh = 4500.00`, `HVNLow = 4480.00`,
   candle `Open=4498.00, High=4503.00, Low=4497.50, Close=4502.00`
   (green). `Close > HVNHigh` ✓. `BodyPercentBeyondBoundary = (|4500-4502|
   / |4498-4502|) * 100 = 50%` — passes a configured 20% minimum.
   `NearSideCandleExtremeDistance = 4497.50 - 4500.00 = -2.50` (candle low
   is inside the boundary) — passes either the fixed-tick or ATR rule
   trivially. `UpperBreachCondition = 1`, `LowerBreachCondition = 0`,
   `RawCloseRelation = ABOVE_HVN`.
2. **Qualifying lower breach.** Mirrored: `HVNLow = 4480.00`, candle
   `Open=4482.00, High=4482.50, Low=4477.00, Close=4478.00` (red).
   `Close < HVNLow` ✓, body percentage and near-side checks pass
   symmetrically. `LowerBreachCondition = 1`, `UpperBreachCondition = 0`,
   `RawCloseRelation = BELOW_HVN`.
3. **Close above HVN but Tatva breach false.** Same boundaries as example
   1, candle `Open=4501.50, High=4503.00, Low=4501.00, Close=4502.00`
   (green, close above `HVNHigh`). `BodyPercentBeyondBoundary =
   (|4500-4502|/|4501.5-4502|)*100 = 400%` — passes body threshold, but
   `NearSideCandleExtremeDistance = 4501.00 - 4500.00 = 1.00`, larger than
   the configured `MaxTicksOutsideBoundary` (say, 2 ticks = 0.50 at a
   0.25 tick size) — fails the proximity rule.
   `RawCloseRelation = ABOVE_HVN`, `UpperBreachCondition = 0`. This is the
   case the raw-close-relation field exists to distinguish from a
   qualifying breach.
4. **Close inside the HVN.** `Close = 4490.00`, between `HVNLow=4480.00`
   and `HVNHigh=4500.00`. `RawCloseRelation = INSIDE_HVN`,
   `UpperBreachCondition = 0`, `LowerBreachCondition = 0`,
   `DistanceBeyondBoundaryPrice = 0`, `DistanceBeyondBoundaryATR = 0`.
5. **HVN2 requested but unavailable.** This instance is configured for
   `ConfiguredHVNRank = 2`; the active profile's own Active Profile Flags
   value does not have `PROFILE_FLAG_HVN2_FOUND` (bit 6) set (a profile
   that only produced an HVN1 cluster). Common-envelope `Status =
   NOT_READY` (section 3, step 3 — a missing HVN2 on an otherwise-valid
   profile is ordinary, not an error), `ReasonCode =
   SELECTED_HVN_NOT_FOUND`. `RawCloseRelation` carries no defined meaning
   this publication (section 5). **Not** silently substituted with HVN1's
   boundaries.
6. **Invalid ATR.** `Max Ticks Outside Boundary = 0` (ATR-fallback
   selected) but the configured ATR study/subgraph returns a non-finite
   or non-positive value this evaluation, past the Sensor's own warmup
   window. Common-envelope `Status = ERROR` (section 3, step 6 — a
   dependency that should be available but isn't), `ReasonCode =
   ATR_UNAVAILABLE` — the entire measurement is withheld, not just the
   ATR-dependent proximity check.
7. **Active profile ID changing.** On closed candle N, `ActiveHTFProfileID
   = 7`, HVN1 boundaries at `[4480, 4500]`. On closed candle N+1, the
   Consumer's Active Profile ID subgraph now reads `8` (a new HTF period
   closed and became active) with HVN1 boundaries at `[4510, 4530]`. This
   Sensor evaluates candle N+1 against profile 8's own boundaries
   immediately — no requirement that candle N+1 (or any candle) first
   cross profile 7's old boundary; `ActiveHTFProfileID = 8` is simply
   published as part of the normal measurement.
8. **Repeated qualifying candles — each is a measurement, not an event.**
   Closed candles N, N+1, and N+2 all independently satisfy the upper
   breach condition against the same, unchanged HVN boundaries.
   `PublicationSeq` advances on each of the three evaluations (each
   produces a materially complete `VALID` measurement, republished per
   the base Sensor contract's own publish-on-material-change semantics),
   and `UpperBreachCondition = 1` on all three. **This Sensor does not
   report "a new breach occurred" versus "the breach condition continues
   to hold"** — that distinction belongs to a future Trigger Strategy's
   own edge-detection logic (section 12), consuming three ordinary,
   independently-valid measurements from this Sensor.

------------------------------------------------------------------------

# Unresolved design questions

**None remain open as of this correction pass.** The four questions
originally listed here are all now closed by direct decision elsewhere in
this document, not left pending:

1. Tatva-derived input defaults — adopted directly (section 2: `20`/`8`/
   `1.0`/`2.0`), remaining ordinary configurable inputs, not an
   architectural commitment.
2. Fixed HVN1/HVN2 subgraph mapping vs. a generic subgraph-index pair —
   fixed deterministically for v1 (section "Generic instance model");
   more than two ranks is future scope for a future document revision,
   not a v1 configuration choice.
3. Freshness tolerance between the active HTF profile and this Sensor's
   own evaluation — closed directly in section 3: no elapsed-LTF-bar
   timeout exists; a profile remains usable for as long as the Consumer
   itself continues to expose it as active.
4. Whole-tick distance export — not added; no consumer requires it, and
   price-unit plus ATR-unit distance (section 6) are sufficient.

If a genuine new implementation question surfaces in a future revision of
this document, it belongs here — this section is not deleted, only
currently empty.

------------------------------------------------------------------------

**Parked for a later vertical slice. The first Chakra walking skeleton
will use a simpler local-chart Sensor. This document preserves the agreed
HVN Sensor semantics so the design discussion does not need to be
reconstructed later.**
