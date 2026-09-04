# Chakra — Sensor Interface Contract (Draft, Revision 4)

Status: **design only, no ACSIL implementation yet.**

Builds directly on the approved `Chakra_Common_Interface_Envelope_Contract.md`
(Revision 8). This document does not redefine anything from that contract —
it specifies how the Sensor layer specifically fills in the envelope, and
what a Sensor payload must and must not contain. Read the envelope contract
first; this document assumes familiarity with `Status`, `PublicationSeq`,
the state consumption pattern, the PV ownership ranges (including the
Float/Double namespaces added in envelope Revision 8), and the identity
validation order.

**Definition (from `Algo_Overlay_Framework_Architecture.md`).** A Sensor
measures an objective property of the market. It never classifies market
state, never recommends action, never detects a setup, and never knows who
consumes it. Everything in this document exists to make that boundary
enforceable, not just aspirational.

Naming: `K_Chakra_Sensor_<DescriptiveName>_V<n>`.

------------------------------------------------------------------------

# Changes from Revision 1

Three practical fixes from a review pass, none requiring a redesign:

1. **Zero does not need a separate validity flag under atomic validity.**
   Revision 1 required a dedicated validity flag or sentinel for any field
   where zero is a legitimate measurement. Unnecessary given the atomic
   validity model this contract already defaults to (section 3): envelope
   `Status` gates the *entire* snapshot, so whatever a field reads while
   `Status != VALID` is irrelevant — nobody may read it without checking
   `Status` first. `ATR = 0.0` while `Status = NOT_READY` is safely ignored,
   and `Delta = 0.0` while `Status = VALID` is safely a real zero, with no
   extra field needed to tell them apart. Per-field validity flags are only
   needed *if* a sensor later opts into per-field partial validity (open
   question 1) — not as a default requirement for every field.
2. **Cross-chart/HTF sensors need a source-time anchor, not just a producer-chart
   anchor.** The rule that `EvalBarIndex`/`EvalDateTime` always describe the
   producer's own chart is still correct for the common envelope, but it
   silently loses the *factual* source anchor for anything cross-chart: a
   Weekly ATR sensor running on a 1-minute chart would show `EvalDateTime`
   as the 1-minute bar's timestamp, not the weekly bar/period the
   measurement actually describes — making the value look continuously
   fresh regardless of whether the underlying weekly data actually updated.
   Fixed by requiring explicit source-provenance payload fields for any
   cross-chart or higher-timeframe sensor — see section 4.
3. **Full recalculation could churn the persistent interface unnecessarily.**
   Revision 1's "publish only on change" rule, applied naively across an
   entire AutoLoop historical sweep, would publish the persistent
   envelope/payload on every historical bar where the measurement changed —
   for something like ATR, essentially every bar, producing thousands of
   pointless publications when only the *final* state is ever meaningful to
   the latest-state interface (section 9). Fixed by requiring persistent
   publication to be suppressed through the middle of a recalculation sweep
   and issued once, at the end, representing the final bar — see section 8.

------------------------------------------------------------------------

# Changes from Revision 2

One gap found while writing the MCtx Interface Contract (which consumes
Sensor history for its own recalculation): **array coverage alone does not
prove a historical measurement is valid.** A Sensor's measurement subgraph
can be populated at a given historical bar while holding a warmup/default
zero, a deliberately-cleared value, or a nonfinite result — "the array has a
number there" and "that number was a trustworthy measurement" are different
claims, and only the Sensor itself knows which is true at each historical
bar. Fixed by requiring a historical validity signal alongside the
measurement subgraph(s) — see section 9.

------------------------------------------------------------------------

# Changes from Revision 3

Two material anomalies found on a further deep review, both in bar/time
anchoring:

1. **Retrospective anchoring created a lookahead-bias risk.** Revision 3
   said a retrospective measurement's `EvalBarIndex`/`EvalDateTime` should
   be the *earlier* bar the fact pertains to (matching the visual Renko
   ring/dot convention). That's correct for chart display, but unsafe as
   the *only* historical timestamp: a Renko ring confirmed on bar 105 but
   retrospectively placed on bar 103 would, under the Revision 3 rule, have
   its historical fact written at index 103 — meaning a replay/backtest
   consumer evaluating bar 103 would see information that wasn't actually
   knowable until bar 105, producing unrealistically good backtest results.
   Fixed by splitting the concept into two distinct anchors: envelope
   `EvalBarIndex`/`EvalDateTime` reverts to its plain, safe meaning
   (when the fact became available/knowable — which, for any sensor, is
   simply whichever bar/call is currently being processed at publication
   time, no special case needed), and a **new** payload-level
   `PertainsToBarIndex`/`PertainsToDateTime` pair carries the earlier,
   semantic/visual anchor. Machine-readable historical consumption uses the
   availability anchor; visual display may still use the pertains-to
   anchor. See section 4.
2. **Cross-chart `SourceDateTime` conflated period start with data
   freshness.** For a weekly measurement, a "weekly bar's own timestamp" is
   normally the period's *start* (Monday). But a continuously-updating
   measurement like `WeeklyRangeCovered` changes throughout the week — using
   the period-start timestamp as the freshness anchor would make a value
   freshly recomputed on Wednesday look several days stale. Fixed by
   splitting `SourceDateTime` into `SourcePeriodStartDateTime` (+ optional
   `SourcePeriodEndDateTime`, which period is measured) and
   `SourceDataAsOfDateTime` (the actual freshness anchor — latest
   market-data time incorporated into the current measurement). Consumers
   must compute freshness from `SourceDataAsOfDateTime`, never period start.
   See section 4.

------------------------------------------------------------------------

# 1. Sensor identity and schema

- **`ComponentKind`** is always `K_CHAKRA_KIND_SENSOR` for anything using
  this contract.
- **`ProducerTypeID`** is a Sensor-layer-local enum, one member per distinct
  sensor type, no central cross-layer registry (same reasoning as the
  envelope's `ProducerTypeID` rule). Include an explicit `NONE = 0` member
  for the same reason `K_CHAKRA_KIND_NONE` exists — a zero read should have
  defined meaning, not be an accident of an unwired dependency.
  ```cpp
  enum K_CHAKRA_SENSOR_TYPE
  {
      K_CHAKRA_SENSOR_TYPE_NONE       = 0,
      K_CHAKRA_SENSOR_TYPE_WEEKLY_ATR = 1,
      // one member per concrete sensor, assigned as each is built
  };
  ```
- **`PayloadSchemaVersion`** (int32 PV 20, per the envelope convention) is
  scoped **per `ProducerTypeID`**, not shared across all sensors — different
  sensor types have completely unrelated payload layouts, so there is no
  meaningful single "sensor payload version" number. A consumer only
  validates `PayloadSchemaVersion` after it has already confirmed
  `ProducerTypeID` matches the specific sensor it expects (this is already
  the envelope's mandated order — restated here because it matters more for
  Sensors than most layers, given how varied sensor payloads are).
- **File-version suffix (`_V<n>`) is not the same number as
  `PayloadSchemaVersion`.** A file might bump `_V2` for an unrelated
  bugfix, performance change, or input reorganization that doesn't touch the
  payload's field layout at all — `PayloadSchemaVersion` only changes when
  the actual field set/types/meaning changes in a way that would break an
  existing consumer. Track them independently; do not assume they move
  together.
- **One study instance represents one clearly defined sensor.** This is a
  hard rule, not a guideline.
- **Multi-measurement outputs are allowed only when they form one coherent
  measurement family** — different facets of a *single measurement act*,
  computed together from the same inputs in the same pass, where splitting
  them into separate studies would be artificial. Litmus test: are these
  numbers different *views* of one measurement, or are they independent
  measurement acts that merely share a category label? Weekly ATR, its
  range-covered, and its percent-consumed are one family — the latter two
  are arithmetically derived from the same underlying computation and don't
  exist independently of it. ATR and MACD are not a family — they are two
  unrelated computations that happen to both be "technical indicators."
  Bundling them to reduce study count is exactly the kind of premature
  consolidation this rule exists to block.

------------------------------------------------------------------------

# 2. Measurement payload

No generic `Value1`/`Value2`/`Value3` fields, ever. Every payload field is
named for what it specifically measures. For **every** field, a sensor's own
contract must state, explicitly and in writing:

| Aspect | Requirement |
|---|---|
| **Value type** | `float` / `int` / `bool` / `enum` — enum only when the value is itself an objective measurement (e.g. "which session is active" is a measured fact; "is this bullish" is not a value type problem, it's a layer-boundary violation — see section 11). |
| **Units** | Ticks, price, contracts, percent, seconds, bars, ratio, currency — stated explicitly per field. Whether units are a shared runtime enum or a documentation-only convention is an open question (section 12) — not resolved here, but every field must have *some* explicit, written unit regardless of mechanism. |
| **Scale** | E.g. is a percent field `40.0` (meaning 40%) or `0.40` (fractional)? Default recommendation, matching the base architecture spec's own `PercentConsumed: 40` example: percentages as `0-100`, not `0-1`. A sensor may deviate, but must say so explicitly in its own field documentation if it does. |
| **Sign convention** | E.g. does "EMA distance" grow more positive above the EMA or below it? Must be stated, not inferred. |
| **Valid range** | Explicit min/max where one exists (may be open-ended, e.g. `PercentConsumed` can exceed 100 in an expansion week — state whether it's bounded or not). |
| **Neutral/unset representation** | What the field reads while `Status != VALID` is unspecified and irrelevant under the atomic validity model (section 3) — no consumer may read payload without checking `Status == VALID` first, so no field needs a dedicated sentinel to disambiguate "not available" from a real value. This only becomes a real design question for a field on a sensor that has explicitly opted into per-field partial validity (open question 1) — the default case needs no answer here at all. |
| **Is zero a valid measurement?** | State explicitly, per field, for documentation clarity — but under atomic validity this has no safety consequence (see above): `0.0` while `Status != VALID` is exactly as meaningless as any other stray value in an untrusted snapshot. Only matters if a field ever needs a per-field validity flag under the (currently undecided) partial-validity mode. |
| **Precision / rounding** | E.g. "rounded to the nearest 0.25 tick" or "full float precision, no rounding." Downstream threshold/equality comparisons depend on knowing this. |

**PV namespace for Sensor payload.** Sensor measurements routinely need
float precision (price, ATR, ratios) that the common envelope itself never
needed. Use the Float/Double persistent-variable namespaces defined in the
envelope contract's "Public versus private persistent-variable ownership"
section (added in envelope Revision 8 specifically because this Sensor
contract needed them) — framework-wide payload ranges, not a Sensor-specific
convention, even though Sensors are what first required them. Int32 PV 20
stays reserved for `PayloadSchemaVersion` (envelope convention); int32 PVs
21-49 remain available for any genuinely integer-typed measurement (e.g. a
bar count or contract count).

------------------------------------------------------------------------

# 3. Validity

Reuses the envelope's `K_CHAKRA_STATUS` exactly, with Sensor-specific
meaning:

- **`NOT_READY`** — insufficient warmup/data. Expected, normal, temporary;
  resolves itself as more bars arrive. Not a fault.
- **`VALID`** — a complete, trustworthy measurement. See atomicity rule
  below.
- **`ERROR`** — evaluation was attempted but required data was unavailable
  or invalid. Distinct from `NOT_READY` on purpose: `NOT_READY` is "keep
  waiting," `ERROR` is "something is wrong and warrants attention" (e.g. a
  configured dependency disappeared mid-session).
- **`DISABLED`** — the sensor's own `Enable` input is off.

**Atomic validity is the default and only currently-defined mode.** A
multi-field sensor publishes `VALID` only when **every** required field in
its snapshot is valid together — never a partial-valid snapshot where some
fields are trustworthy and others aren't, unless that specific sensor's own
payload schema explicitly adds per-field validity flags as a deliberate,
documented opt-in (this should be rare — see open question in section 12
before building one). Default assumption for any new sensor: one `Status`
governs the whole snapshot.

Because `Status` gates the whole snapshot atomically, a field where zero is
a legitimate measurement does **not** need its own validity flag or sentinel
— `Status != VALID` already means nothing in the payload may be trusted,
regardless of what bit pattern happens to sit in any individual field. A
dedicated per-field sentinel or flag is only relevant if a specific sensor
explicitly adopts per-field partial validity (see open question 1) — it is
not a default requirement, and adding one where atomic validity already
applies adds a field and sentinel-handling logic without improving safety.

------------------------------------------------------------------------

# 4. Bar and time anchoring

- **`EvalBarIndex`/`EvalDateTime` always mean "when this fact became
  available/knowable" — the bar/call currently being processed at
  publication time.** This is the plain, safe, envelope-level meaning, with
  **no special case for retrospective sensors.** (Corrected from Revision 3
  — see "Changes from Revision 3" above for why the earlier rule was
  unsafe.) `sc.BaseDateTimeIn[EvalBarIndex]` for bar-based live evaluation,
  matching the envelope's "Age and 'now'" historical-evaluation rule
  exactly.
- **Retrospective measurements need a second, distinct anchor —
  `PertainsToBarIndex`/`PertainsToDateTime` — carried in the Sensor's own
  payload, never conflated with `EvalBarIndex`/`EvalDateTime`.** A
  retrospective fact (e.g. a Renko ring, confirmed only after two
  opposite-color bricks per this project's ring/dot convention in
  `reference_acsil_skill.md`) genuinely describes an *earlier* bar — bar
  103, say — but only becomes knowable at a *later* bar — bar 105, once
  confirmed. Both facts matter and must never be collapsed into one field:
  - `EvalBarIndex`/`EvalDateTime` (envelope) = **availability** — bar 105,
    when the fact could first have been acted on. This is what freshness/
    `Age` and all the generic Chakra machinery use.
  - `PertainsToBarIndex`/`PertainsToDateTime` (payload) = **semantic/visual
    anchor** — bar 103, where the fact actually belongs. Used for chart
    display and for a consumer that specifically needs to know what the
    fact is *about*, not when it could have been used.
  - **A visual subgraph may plot at the `PertainsTo` index** (matching the
    existing, purely-visual Renko ring/dot convention exactly as before —
    that convention is not wrong, it was simply never designed with an
    automated replay consumer in mind).
  - **The machine-readable historical subgraph (section 9) — the one a
    downstream consumer like MCtx reads for its own historical
    recalculation — must be indexed at the `EvalBarIndex`/availability
    anchor (bar 105), never at `PertainsTo` (bar 103).** This is what makes
    historical/replay consumption availability-safe by construction: a
    consumer processing up through bar N can never see a retrospective fact
    that wasn't actually knowable by bar N, because the machine array
    simply doesn't have it there yet.
  - A non-retrospective sensor has no need for `PertainsTo*` fields at all —
    for it, availability and pertains-to are always the same bar by
    construction, and the plain envelope fields already say so.
- **Intrabar sensors must state explicitly, in their own documentation,
  whether the still-open/forming bar may be published at all**, and if so,
  what `EvalBarIndex`/`EvalDateTime` mean for a value that may still change
  before that bar closes (see cadence, section 5, and the provisional-value
  open question in section 12).
- **Cross-chart sensors must state which chart's bar index/time owns the
  result.** Rule: envelope `EvalBarIndex`/`EvalDateTime` always describe the
  *producer study's own chart* — the chart the sensor is instantiated on and
  publishes its own subgraph/output against — regardless of which chart(s)
  it reads input data from. A sensor reading Daily-chart data while running
  on a 1-minute chart still anchors its envelope fields to its own
  (1-minute) chart's index. This keeps the generic Chakra machinery
  (recalc-entry detection, the general "Age and 'now'" rules) working
  identically regardless of layer or chart topology.
- **Cross-chart and higher-timeframe sensors must additionally publish a
  source-time anchor in their own payload — the producer-chart envelope
  fields alone are not enough.** Republishing on every 1-minute bar makes
  `EvalDateTime` look continuously fresh even when the underlying weekly (or
  other source-chart) data hasn't actually changed, which misleads any
  consumer computing `Age` against it. Required payload fields for any
  sensor whose measurement is computed from a different chart than the one
  it's instantiated on:
  - `SourceChartNumber` (int) — which chart the measurement actually comes from.
  - `SourceBarIndex` (int) — the bar index on *that* chart corresponding to
    `SourceDataAsOfDateTime` below (not necessarily the period's first bar).
  - `SourcePeriodStartDateTime` (SCDateTime) — **which period** is being
    measured (e.g. this week's Monday open). Identification only.
  - `SourcePeriodEndDateTime` (SCDateTime, optional, sentinel = 0 if
    open/ongoing) — the period's end, where applicable/known.
  - **`SourceDataAsOfDateTime` (SCDateTime) — the latest market-data time
    actually incorporated into the current measurement. This, not
    `SourcePeriodStartDateTime`, is the field consumers must use for
    freshness.** (Corrected from Revision 3, which used a single
    `SourceDateTime` field that conflated these two — see "Changes from
    Revision 3.") A weekly measurement's period-start timestamp is normally
    the week's Monday open; a continuously-updating value like
    `WeeklyRangeCovered` changes throughout the week, and using period-start
    for freshness would make a value freshly recomputed on Wednesday look
    several days stale. Do not conflate "which period" with "how current."
  - Optionally `SourceStudyID`/`SourceSubgraphIndex` (int) if the sensor
    reads from another *study* on the source chart rather than raw price
    data, for full traceability.

  These live in the sensor's own payload ranges (int32 21-49 for the chart/
  bar-index fields, SCDateTime payload range 10-29 for the three
  `Source*DateTime` fields), not the common envelope. A same-chart,
  same-timeframe sensor does not need these — its envelope `EvalDateTime`
  already *is* the true source time, so duplicating it into payload fields
  would be redundant.
- Reinforcing the envelope's own warning: the bar index in any Chakra
  envelope is local to the *producer's* chart. A consumer on a different
  chart must never assume it can use that index directly against its own
  chart's arrays — and for cross-chart sensors specifically, a consumer
  caring about true freshness should compute `Age` from the payload's
  `SourceDataAsOfDateTime`, never `SourcePeriodStartDateTime` and never the
  envelope's `EvalDateTime` (which only reflects when *this chart*
  republished, not when the source data was last updated).

------------------------------------------------------------------------

# 5. Publication cadence

Every sensor declares exactly **one** cadence (an input may let the user
choose among supported modes, but at any given time the active mode must be
singular and unambiguous, never an implicit blend):

- **Closed-bar** — publishes only once a bar closes.
- **Intrabar / on-update** — republishes as the current (open) bar's
  underlying data changes.
- **Event-driven** — publishes only when triggered by something other than
  a fixed per-bar cadence (e.g. a ring/dot index change).
- **Session boundary** — publishes once per session (e.g. an opening-range
  sensor).
- **Higher-timeframe boundary** — publishes once per HTF bar close while
  running on a lower-timeframe chart.

**Publish only when the complete public snapshot actually changes** —
directly reuses the envelope's publication-model rule; re-evaluating and
landing on identical values is a no-op, not a republish.

**`PublicationSeq` remains the commit marker, never an event counter — this
is non-negotiable for Sensors.** A Sensor uses the envelope's **state
consumption pattern exclusively** (`identityValid && Status == VALID &&
freshnessValid` — see the envelope contract). It never defines a
payload-level `EventSeq`, `EventType`, or anything from the event pattern —
if a component needs that machinery, it has stopped being a Sensor and
belongs in TriggerStr instead (see section 11).

Whether an intrabar publication is provisional (subject to being overwritten
by a later, more authoritative closed-bar value for the same bar) is left
as an open decision — see section 12.

------------------------------------------------------------------------

# 6. Decay and freshness

The Sensor declares its factual `DecayClass`; the **consumer**, not the
Sensor, decides what age it will tolerate — this is the base architecture
spec's "consumer decides staleness" principle, unchanged.

- **`FAST`** — normally current bar/tick. Decays to staleness within the
  same bar/tick cycle it was published in, for most consumers.
- **`SLOW`** — a valid measurement that may simply be too old for some
  *particular* consumer's purpose. The Sensor does not get an opinion on
  this — only `Age` (via `EvalDateTime`) and the consumer's own tolerance
  matter.
- **`STATIC_SESSION`** — has a hard, known validity boundary. This is the
  only class where `ValidUntilDateTime` is meaningful.
- **`ValidUntilDateTime` production rule for Sensors:** a `STATIC_SESSION`
  sensor must **pre-resolve** its own boundary (e.g. "next Thursday's
  close") to a concrete `SCDateTime` at publish time — never a symbolic or
  lazily-resolved boundary. A Sensor's whole purpose is to already know
  facts about the market; deferring boundary resolution to the consumer
  would push interpretation work onto a layer that's supposed to stay pure
  measurement.
- Live and paced-replay evaluation use `sc.GetCurrentDateTime()`;
  historical-bar evaluation uses `sc.BaseDateTimeIn[EvalIndex]` — the
  envelope's "Age and 'now'" rules, unchanged for Sensors.
- **Never declare a measurement invalid merely because one particular
  consumer might consider it stale.** `Status` describes producer-side
  trustworthiness only. Staleness is entirely a consumer-side `Age`-vs-
  tolerance computation and must never leak backward into the Sensor's own
  `Status`.

------------------------------------------------------------------------

# 7. Dependencies and calculation order

For a derived sensor (one computed from other studies' output, not raw
market data):

- **List every required upstream study/array explicitly**, in that sensor's
  own documentation — no implicit or undocumented dependencies.
- **Validate complete array coverage at `EvalBarIndex`** before trusting any
  of it. Matches the hardening already applied to
  `K_CloseProtect_DeltaSMA_BodyPct_Decision` in this codebase: every
  dependent array must cover the evaluation index or the whole computation
  is treated as not-ready, deterministically — never a partial evaluation
  against whichever dependencies happen to be ready this call.
- **No partial calculation from whichever dependencies happen to be
  available.** This is the section-3 atomicity rule applied to dependency
  readiness specifically: if any required dependency doesn't cover
  `EvalBarIndex`, the entire publication for this call is `NOT_READY` (or
  `ERROR`, if the dependency should have been available and wasn't) — never
  a best-effort partial result.
- Use `sc.CalculationPrecedence = LOW_PREC_LEVEL` where the sensor depends
  on another study calculating first, matching existing codebase convention.
- Use manual looping (`sc.AutoLoop = 0`) and
  `sc.GetCalculationStartIndexForStudy()` when cross-study alignment
  requires it — per this project's ACSIL skill guidance.
- **Defer expensive cross-study fetches until cheap eligibility checks
  pass**, and **fetch each required dependency once per call, reused rather
  than re-fetched** — this is this user's established low-latency pattern
  (see `reference_acsil_skill.md`), applied to Sensors without exception.

------------------------------------------------------------------------

# 8. Full recalculation and replay

- Sensors have **no external actions, ever** — recalc or not, live or
  replay. This is stronger than the envelope's general "no consumer action
  during the consumer's own recalc" rule, which is about consumers with
  side effects; a Sensor by definition has none, at any time.
- **Historical recalculation must rebuild measurements deterministically.**
  Combined with the base architecture spec's versioning rule (code version +
  market data version determine the result, no separate calibration
  infrastructure): replay must produce the same measurement from the same
  market data and the same code version, every time.
- **Clear or overwrite stale historical subgraph output deliberately.** If a
  recalculation's effective window or logic changes (e.g. an input changed),
  old subgraph values from a since-invalidated computation must not linger
  as visual or machine-readable noise.
- **Do not publish a misleading intermediate `VALID` snapshot during
  warmup.** A rolling 20-bar measurement computed from only 5 available bars
  is not a smaller-but-valid measurement — it's `NOT_READY`. Never publish a
  value that *looks* complete but was computed from a truncated window.
- **The final public snapshot after recalculation represents the latest
  properly evaluated measurement** — not an intermediate warmup-era value
  accidentally left in persistent storage once the sweep concludes.
- **Do not churn the persistent interface during the sweep — publish it
  once, at the end, not once per historical bar.** Applied naively, "publish
  only on change" (envelope contract) would republish the persistent
  envelope/payload on every historical bar where the computed value
  differs — for a measurement like ATR, that's nearly every bar, producing
  thousands of pointless writes across a large chart when only the *final*
  state is ever meaningful to a latest-state interface (section 9).
  Required behavior:
  - **Subgraphs rebuild normally, every bar**, exactly as section 9 already
    requires — this rule only concerns the *persistent envelope/payload*,
    never the historical subgraph series.
  - **The persistent interface does not publish per intermediate historical
    bar.** A full recalculation sweep produces at most two persistent
    publications total, regardless of how many historical bars were
    processed: the recalc-entry `NOT_READY` publication (already required
    generally), and one final publication representing the last relevant
    bar's properly-evaluated state, issued once the sweep reaches that bar.
  - **AutoLoop producers:** detect the sweep's last call as
    `sc.IsFullRecalculation && sc.Index == sc.ArraySize - 1` and perform the
    persistent publication only on that call (assuming the measurement is
    actually valid there — otherwise the recalc-entry `NOT_READY`
    publication remains the only one, correctly). Every other call within
    the sweep updates subgraphs only, never the persistent payload.
  - **Manual-loop producers:** already receive exactly one call for the
    whole sweep per this codebase's convention, so this falls out naturally
    — whatever internal historical rebuild that single call performs, it
    still issues only one persistent publication at the end of it,
    representing the final bar.
  - Once the sweep concludes and normal (non-recalc) evaluation resumes,
    ordinary change-based publication applies again without restriction.

------------------------------------------------------------------------

# 9. Subgraphs versus persistent payload

**The common envelope is a latest-state interface, not a historical data
bus.** This distinction matters enough to restate plainly:

- **Subgraphs** carry the historical series — for chart visualization,
  study-on-study consumption via `GetStudyArrayUsingID`, and replay
  inspection.
- **The persistent payload** (envelope + Sensor payload fields) exposes only
  the *latest* contracted snapshot, generically, to any Chakra-aware
  consumer — matching the state pattern's design throughout the envelope
  contract.
- **Both representations must agree for the same evaluation anchor** — the
  subgraph value at `EvalBarIndex` and the persistent payload's current
  fields must be the same computation, not divergent logic or rounding.
- **Hidden subgraphs are acceptable** for machine-readable historical output
  a sensor wants other studies to read via array access — `DrawStyle =
  DRAWSTYLE_IGNORE`, matching the existing `InternalState` convention
  already used in `K_Template_CloseProtect_DecisionStudy`/`Aggregator`.
- **A consumer needing history reads subgraph arrays — it must never expect
  the persistent snapshot to contain history.** The envelope was never
  designed to be a queue or a series; it is single-slot/latest-only by
  design (the same reasoning that produced the event pattern's
  latest-event-wins delivery semantic applies here too, generalized to all
  Chakra payload).
- **Array coverage is not validity — every Sensor must also publish a
  historical validity signal, not just measurement subgraphs.** A bar being
  present in a measurement subgraph's array only proves the array is long
  enough to have *some* number there — it says nothing about whether that
  number was a warmup/default placeholder, a deliberately-cleared value
  from a since-invalidated computation, a nonfinite result, or a genuine
  measurement. Only the Sensor itself knows which, at each historical bar.
  **Standard/preferred approach:** publish a hidden historical `Status`
  subgraph (`DrawStyle = DRAWSTYLE_IGNORE`), one `K_CHAKRA_STATUS` integer
  value per bar, mirroring the same values the Sensor's own live envelope
  `Status` field takes — written in the same pass as the measurement
  subgraph(s), so the two are always in lockstep. A downstream historical
  consumer then requires all of: the mapped index is covered by the array,
  the historical `Status` value at that index equals `K_CHAKRA_STATUS_VALID`,
  and the measurement value itself is finite/in-range (defense in depth,
  since a bug could in principle write `VALID` alongside a bad number).
  A Sensor may use a documented sensor-specific alternative (e.g. a single
  combined subgraph encoding validity via a sentinel convention) only if it
  unambiguously establishes per-bar validity and is documented as precisely
  as the standard approach would be — silence or "assume it's fine if the
  array has a value" is never acceptable.

------------------------------------------------------------------------

# 10. Diagnostics

Deterministic, numeric `ReasonCode` values, Sensor-layer-owned (per the
envelope's per-layer reason-code convention):

```cpp
enum K_CHAKRA_SENSOR_REASON_CODE
{
    K_CHAKRA_SENSOR_REASON_NONE                    = 0,   // = K_CHAKRA_REASON_NONE
    K_CHAKRA_SENSOR_REASON_WARMUP_INCOMPLETE        = 1,
    K_CHAKRA_SENSOR_REASON_DEPENDENCY_UNWIRED       = 2,
    K_CHAKRA_SENSOR_REASON_DEPENDENCY_UNAVAILABLE   = 3,
    K_CHAKRA_SENSOR_REASON_ARRAY_TOO_SHORT          = 4,
    K_CHAKRA_SENSOR_REASON_INVALID_MEASUREMENT      = 5,   // NaN/Inf or otherwise non-finite
    K_CHAKRA_SENSOR_REASON_SESSION_BOUNDARY_UNAVAILABLE = 6,
    K_CHAKRA_SENSOR_REASON_CONFIGURATION_INVALID    = 7
    // extend per concrete sensor as needed, numbered above these common ones
};
```

**Avoid repetitive per-update logging.** Log on `Status` transitions or
first occurrence of a given `ReasonCode`, not every call — matching the
`DebugLogging` input convention already used throughout this codebase (e.g.
`K_Template_CloseProtect_Aggregator`). A verbose/debug-detail mode may be
offered as an opt-in input, off by default.

------------------------------------------------------------------------

# 11. Explicit prohibitions

A Sensor must **not**:

- Classify or interpret market state — no `BULLISH`, `BEARISH`, `TREND`,
  `EXTREME`, `UNDER_EXPANDED`, or any similar label. That's Market Context's
  job.
- Recommend entry or exit.
- Fire a trigger / detect a "setup."
- Know about positions or trades in any way.
- Perform sizing, stop, trailing, or profit-taking behavior.
- Apply a consumer-specific freshness threshold — that decision belongs
  entirely to the consumer (section 6).
- Publish a payload-level `EventSeq` — unless the component is genuinely an
  event producer, in which case it is not a Sensor at all and belongs in
  TriggerStr (`K_CHAKRA_KIND_TRIGGERSTR`, different naming, different
  contract).

**Litmus test:** *Could two different Trade Policy components legitimately
interpret this measurement differently, without the Sensor itself changing
at all?* If the answer is no — if there's only one "correct" interpretation
already baked into what the Sensor publishes — interpretation has leaked
into the Sensor and the design needs to move it up into Market Context or
Trade Policy instead.

------------------------------------------------------------------------

# 12. Examples

**Valid — single measurement, multiple units:**
A Weekly-ATR sensor publishing both `ATRPrice` (float, price units) and
`ATRTicks` (float, ticks) — same underlying measurement act, two unit
expressions of it. Legitimate single sensor.

**Valid — coherent family (worked example):**

Assuming this sensor runs on an intraday trading chart while its
measurement is computed from a separate weekly chart's data (the realistic
deployment — a sensor attached only to a weekly chart would rarely be
useful to trading-chart consumers), it is a cross-timeframe sensor per
section 4 and needs source-anchor fields alongside the measurement family:

| Field | Type | Namespace/PV | Units | Notes |
|---|---|---|---|---|
| `WeeklyATRPrice` | float | Float PV 1 | price | |
| `WeeklyRangeCoveredPrice` | float | Float PV 2 | price | Range covered so far this week — updates continuously, not just at week open |
| `WeeklyPercentConsumed` | float | Float PV 3 | percent, 0-100+ (open-ended) | Derived: `RangeCovered / ATR * 100` |
| `SourceChartNumber` | int | Int32 PV 21 | — | The weekly chart this is computed from |
| `SourceBarIndex` | int | Int32 PV 22 | — | Weekly chart's bar index for `SourceDataAsOfDateTime` below |
| `SourcePeriodStartDateTime` | SCDateTime | Datetime PV 10 | — | This week's Monday open — which period is measured, not freshness |
| `SourceDataAsOfDateTime` | SCDateTime | Datetime PV 11 | — | Latest weekly-chart data actually incorporated — **use this for `Age`, never `SourcePeriodStartDateTime`** |

(`SourcePeriodEndDateTime` omitted here — this example's week is still
ongoing, so it stays at its sentinel/unset value per section 4.)

The three measurement fields are facets of one measurement act (this week's
ATR consumption), computed together, in one pass, from the same inputs —
still one sensor, `K_Chakra_Sensor_WeeklyATRConsumption_V1`. The source
fields are not a second measurement family, just required provenance for a
cross-chart sensor per section 4.

**Invalid — interpretation leakage:** a sensor publishing `UNDER_EXPANDED`.
That's a classification of the `WeeklyPercentConsumed` measurement above,
not a measurement itself — it belongs in a Market Context component that
*consumes* this sensor, not in the sensor.

**Invalid — interpretation leakage:** a "Delta Sensor" publishing
`LONG_ENTRY_ALLOWED`. That's a Trade Policy (`EntryGate`) decision wearing a
Sensor's name. The delta measurement itself (signed value, contracts or
ticks) is a legitimate sensor payload; the allow/disallow judgment is not.

**Invalid — arbitrary bundling:** one producer publishing unrelated ATR,
MACD, and VWAP-distance measurements together merely to reduce the number of
studies on a chart. These are three independent measurement acts with no
shared computation or inherent relationship — three sensors, not one.

------------------------------------------------------------------------

# Unresolved design questions — deliberately left open

Only questions that affect real implementation:

1. **Multi-output atomic validity.** Is strict atomic (whole-snapshot)
   validity the *only* supported mode, or should per-field validity flags be
   a first-class, documented pattern for specific coherent families where
   sub-measurements can legitimately be independently ready? Section 3
   defaults to atomic-only; this needs a real decision before the first
   sensor that might benefit from partial validity gets built, not an
   assumption either way.
2. **Provisional intrabar publication.** For intrabar/on-update cadence
   sensors, is the latest intrabar value ever "final" for that bar, or does
   every bar's measurement only become authoritative once it closes? Does a
   consumer need an explicit way to distinguish "provisional" from "final"
   (a payload flag), or is `DecayClass = FAST` plus the normal
   closed-bar republish (which simply overwrites the provisional value)
   sufficient on its own?
3. **Do units need a shared runtime enum, or stay payload-specific /
   documentation-only?** Section 2 requires every field to have an explicit
   unit but doesn't mandate the mechanism. A shared `K_CHAKRA_UNIT` enum
   would let a generic consumer introspect units at runtime (useful for,
   say, a generic logging/debugging tool across all sensors); payload-specific
   documentation is simpler and avoids a central registry, consistent with
   this contract's general aversion to central registries elsewhere
   (`ProducerTypeID`, `ReasonCode`). Not resolved here.
