# Chakra — Market Context (MCtx) Interface Contract (Draft, Revision 5)

Status: **design only, no ACSIL implementation yet.**

Builds directly on:
- `Algo_Overlay_Framework_Architecture.md`
- `Chakra_Common_Interface_Envelope_Contract.md` (approved, Revision 8)
- `Chakra_Sensor_Interface_Contract.md` (Revision 4)

This document does not redefine anything from those — it specifies how the
Market Context layer fills in the envelope, how it consumes Sensors, and
what an MCtx payload must and must not contain. Read the envelope and Sensor
contracts first; this document assumes familiarity with `Status`,
`PublicationSeq`, the state consumption pattern, the PV ownership ranges
(int32/int64/SCDateTime/Float/Double), the identity validation order, and
the Sensor contract's atomic-validity and low-churn-recalc rules — all of
which MCtx reuses without modification.

**Definition (from `Algo_Overlay_Framework_Architecture.md`).** A Market
Context consumes Sensor outputs and converts them into reusable market
facts. It does not recommend actions. It does not know who consumes it. It
must never recommend, permit, reject, size, enter, exit, stop, or trail a
trade — that boundary is what this document exists to make enforceable.

Naming: `K_Chakra_MCtx_<DescriptiveName>_V<n>`.

------------------------------------------------------------------------

# Changes from Revision 1

Three material fixes from a review pass:

1. **Required Sensor unwired was wrongly treated as silent.** Revision 1
   imported the envelope's generic "missing dependency = skip silently"
   rule without distinguishing required from optional slots. Fixed: optional
   slots stay silent; a **required** Sensor being unwired now publishes
   `NOT_READY`, clears condition payload to neutral, sets `ReasonCode =
   K_CHAKRA_MCTX_REASON_SENSOR_UNWIRED`, and logs on the transition — the
   same treatment as a required Sensor going invalid/stale (section 5),
   not a different, silent case. See section 4.
2. **Classification threshold inputs quietly softened an already-settled
   architecture rule.** Revision 1 allowed thresholds as ACSIL `Input`s,
   mitigated by a "log/version them" note. That directly contradicted the
   base architecture spec's explicit "thresholds are code constants, no
   calibration infrastructure, no hot-editing without a commit" rule,
   without flagging the conflict — an unauthorized deviation, not a
   considered choice. Fixed: thresholds, hysteresis values, and confirmation
   constants are code constants, full stop; only operational wiring, display
   choices, and meaning-preserving cadence choices may remain inputs. See
   section 6. **Revision 5 supersedes that blanket prohibition with the
   architecture's now-explicit distinction between fixed behavioral rules
   and designated per-instrument tuning parameters.**
3. **Historical recalculation had no defined Sensor access path.** A
   Sensor's persistent payload is latest-only (Sensor contract, section 9)
   — using it during MCtx's own historical recalculation would classify
   every historical bar using the Sensor's *current* value, not what it was
   at that bar. Fixed: explicit dual access path — live evaluation reads
   the Sensor's persistent snapshot (state pattern, unchanged), historical
   recalculation reads the Sensor's documented historical subgraph arrays,
   with required chart/time alignment and array-coverage validation. See
   section 4 and section 11.

------------------------------------------------------------------------

# Changes from Revision 2

One material fix: **Revision 2 treated "the Sensor's historical array covers
the mapped index" as the historical equivalent of `Status == VALID`. Those
are not equivalent.** A Sensor subgraph can cover a bar while holding a
warmup/default zero, a deliberately-cleared value, a nonfinite result, or a
genuine measurement — array coverage only proves there's *a number* there,
not that the number is trustworthy. An MCtx checking coverage alone could
classify an invalid historical Sensor value as if it were real. Fixed by
requiring the Sensor's own historical validity signal (Sensor contract
Revision 3, section 9: a hidden historical `Status` subgraph, or a
documented sensor-specific equivalent) as part of the historical read, not
array coverage alone — see section 4.

------------------------------------------------------------------------

# Changes from Revision 3

Three material anomalies from a further deep review, spanning both this
contract and the Sensor contract:

1. **Retrospective-Sensor lookahead bias must be preserved through MCtx,
   not silently collapsed.** The Sensor contract (now Revision 4) splits
   `EvalBarIndex`/`EvalDateTime` (availability — when a fact became
   knowable) from a new payload-level `PertainsToBarIndex`/`PertainsToDateTime`
   (which earlier bar the fact is actually about) for retrospective
   sensors, specifically to prevent a replay/backtest consumer from seeing
   a fact before it was actually knowable. If an MCtx depends on a
   retrospective Sensor, it must read that Sensor's **availability-indexed**
   machine subgraph for its own historical recalculation — never the
   pertains-to-indexed visual one — or MCtx would silently reintroduce the
   exact lookahead bias the Sensor contract just closed. See section 4.
2. **Cross-chart freshness must propagate `SourceDataAsOfDateTime`, not
   period start.** The Sensor contract now splits a cross-chart Sensor's
   `SourceDateTime` into `SourcePeriodStartDateTime` (which period) and
   `SourceDataAsOfDateTime` (actual freshness anchor) — using period-start
   for freshness made a continuously-updating measurement look falsely
   stale. This contract's own provenance-propagation guidance and worked
   example used the old, now-superseded single-field name and must track
   the split. See section 9 and the worked example.
3. **Private transition detection had no lifecycle-baselining rule — the
   same bug class already found and fixed once in the envelope contract
   (Revision 6/7), reappearing in a new place.** Section 7 told a downstream
   consumer to detect `RANGE -> TREND` by privately comparing current vs.
   previous MCtx state, but never said how to baseline that private state.
   A consumer attaching while MCtx is already `TREND`, with its private
   "previous state" defaulting to `UNKNOWN`, would misread
   `UNKNOWN -> TREND` as a fresh transition on its very first evaluation —
   exactly the "first attachment is not inert by default" mistake already
   made and corrected once for `LastConsumedEventSeq`, now recurring for a
   business-logic value instead of a sequence number. Fixed by requiring
   the same lifecycle-baselining discipline here: baseline on attachment/
   rewiring/recalc/disable-re-enable, never fire a transition on the
   baselining evaluation itself, and handle MCtx's own invalid periods
    explicitly rather than accidentally. See section 7.

------------------------------------------------------------------------

# Changes from Revision 4

One normative architecture alignment:

1. **The blanket prohibition on classification parameters as ACSIL
   `Input`s was too broad for the intended operating model.** Concrete MCtx
   specifications may now explicitly designate genuine per-instrument tuning
   values as inputs, provided their units, defaults, validity bounds, and
   deterministic change/reconstruction lifecycle are fully specified.
   Classification formulas, state-machine structure, transition precedence,
   and all non-designated behavioral rules remain fixed and code-versioned.
   Manual ACSIL configuration is the current path; future Bheshma JSON
   population is acknowledged but not designed here. See section 6.

------------------------------------------------------------------------

# 1. Identity and naming

- **`ComponentKind`** is always `K_CHAKRA_KIND_MCTX`.
- **`ProducerTypeID`** is an MCtx-layer-local enum, one member per distinct
  context concept, no central cross-layer registry — same reasoning as the
  Sensor layer. Explicit `NONE = 0`.
  ```cpp
  enum K_CHAKRA_MCTX_TYPE
  {
      K_CHAKRA_MCTX_TYPE_NONE             = 0,
      K_CHAKRA_MCTX_TYPE_WEEKLY_EXPANSION = 1,
      K_CHAKRA_MCTX_TYPE_TREND            = 2,
      K_CHAKRA_MCTX_TYPE_AUCTION_STATE    = 3,
      // one member per concrete context, assigned as each is built
  };
  ```
- **`PayloadSchemaVersion`** (int32 PV 20) is scoped **per `ProducerTypeID`**,
  exactly as for Sensors — each context type has its own unrelated state
  space and payload shape, so there is no single shared "context payload
  version."
- **One study instance owns one coherent context concept.** `WeeklyExpansion`,
  `Trend`, and `AuctionState` are three separate studies — they are
  unrelated state spaces (`UNDER_EXPANDED` has nothing in common with
  `FAILED_LOWER_AUCTION`), not facets of one measurement the way a Sensor's
  coherent-family exception allows. **Do not build one giant "Market
  Context" study that publishes several unrelated contexts** — that
  reintroduces exactly the kind of bundling the Sensor contract already
  rejected for indicators, one layer up.

Examples: `K_Chakra_MCtx_WeeklyExpansion_V1`, `K_Chakra_MCtx_Trend_V1`,
`K_Chakra_MCtx_AuctionState_V1`.

------------------------------------------------------------------------

# 2. Context payload shape

Every context payload defines:

| Field | Required? | Notes |
|---|---|---|
| **`State`** | Always | Context-specific enum. **Every context owns its own `State` enum** — no shared/universal context enum, no generic `State1`/`Value1`. `UNDER_EXPANDED`, `FAILED_LOWER_AUCTION`, and `TREND` are unrelated state spaces and must never share one enum type. Explicit `UNKNOWN = 0` member required. |
| **`Direction`/`Bias`** | Optional | Descriptive orientation (e.g. bullish/bearish/neutral) — see section 8. Explicit `NONE = 0` if present. |
| **`Strength`** | Optional | Magnitude of the observed context — see section 8. Explicit `NONE = 0` if present, and only included where the context can define its thresholds meaningfully and deterministically (section 6). |
| **`Confidence`/`Quality`** | Optional, rare | Only if objectively and deterministically calculated from the classification inputs — never a subjective or heuristic "gut feel" number. |
| **Contributing Sensor provenance** | Always, if the context depends on any Sensor | Which Sensor(s) fed this classification — see section 4/9. |
| **Numeric measurements explaining the classification** | Always where applicable | E.g. `PercentConsumed = 42` alongside `State = UNDER_EXPANDED` — the raw number that justifies the label, so a consumer isn't forced to re-derive or guess why a state was assigned. |

`EvalDateTime`, `DecayClass`, and `ValidUntilDateTime` are envelope fields
(section 9 covers MCtx-specific selection rules for them) — not re-defined
in payload.

No generic fields. No universal context enum. Each context's `State` (and
`Direction`/`Strength` where present) is a distinct type, defined in that
context's own contract.

------------------------------------------------------------------------

# 3. Description versus recommendation (the hard boundary)

A context describes what is true. Trade Policy decides what that fact means
for a trade. This boundary is not a style preference — it is the single
most important rule in this document.

**Valid MCtx output:**
```
WeeklyExpansionState = UNDER_EXPANDED
PercentConsumed = 42
```

**Invalid MCtx output:**
```
AllowBreakoutEntry = true
HoldRunnerLonger = true
AvoidShorts = true
```

**Litmus test:** *Could two different Trade Policy components respond
differently to this context without either one being wrong?* If yes, it's a
valid descriptive context — different policies are allowed to use the same
fact differently (exactly the base architecture spec's "no context knows who
consumes it" principle). If the payload already dictates one specific
action, policy has leaked into MCtx and the field needs to move out.

------------------------------------------------------------------------

# 4. Sensor dependency contract

Every MCtx must declare, per Sensor dependency:

- **Whether the dependency is required or optional.** This determines its
  unwired behavior — see below. Most MCtx contexts have exactly one required
  Sensor (the one the classification is fundamentally about); a context that
  additionally reads a supplementary/secondary Sensor may treat that second
  one as optional if the classification still means something without it.
- The exact expected `ProducerTypeID` (which specific Sensor).
- The exact expected envelope `SchemaVersion` and Sensor
  `PayloadSchemaVersion` — validated per the envelope's identity order
  (`SchemaVersion` -> `ComponentKind` -> `ProducerTypeID` ->
  `PayloadSchemaVersion` -> only then `Status`/payload). **Validate complete
  dependency identity before reading any measurement** — no reading payload
  fields "optimistically" ahead of full identity validation.
- The chart/study input used to wire the Sensor (`Input.SetChartStudyValues`,
  same pattern already used throughout this codebase).
- **Which specific Sensor payload fields it reads** — not "everything the
  Sensor happens to publish." An MCtx that only needs `PercentConsumed`
  should say so explicitly, not implicitly depend on the Sensor's entire
  payload shape.
- The allowed freshness tolerance for that dependency (an `Age`-vs-`DecayClass`
  tolerance the MCtx itself chooses — per the envelope's "consumer decides
  staleness" rule, now MCtx is the consumer for this purpose).
- **The live-vs-historical access mapping** — see the dedicated subsection
  below. This is not optional documentation; an implementation that skips it
  is the most likely way to silently classify history using today's data.

## Unwired behavior: required vs. optional (correction from Revision 1)

Revision 1 imported the envelope's generic "missing dependency = skip
silently" rule wholesale, without distinguishing required from optional
slots. That's correct for an optional slot, but wrong for a context's
required input — if `WeeklyExpansion`'s Weekly ATR Sensor is unwired, the
context cannot function at all, and silently doing nothing would leave a
consumer with either no signal or, worse, a stale previous state with no
indication anything is broken.

- **Optional Sensor slot unwired** (`(0,0)`, deliberately unused) — skip
  silently, no log. This is the envelope's normal generic rule, unchanged,
  and correct here because the slot being empty is a legitimate
  configuration, not a fault.
- **Required Sensor unwired** — **not** silent. Publish `NOT_READY`, clear
  the condition payload to neutral (`State -> UNKNOWN`, `Direction`/
  `Strength`/`Confidence` cleared if present), set `ReasonCode =
  K_CHAKRA_MCTX_REASON_SENSOR_UNWIRED`, and log **on the transition into
  this condition** (not every call, per section 13's general logging rule).
  This is functionally identical to section 5's already-defined "required
  Sensor becomes invalid" handling — an unwired required Sensor is just
  another way for the required dependency to be unusable, not a
  fundamentally different case deserving silence.
- **Required Sensor invalid or stale** — immediately invalidate the MCtx
  exactly as section 5 already defines. No change from Revision 1 here;
  this bullet exists only to make clear it sits alongside the unwired case
  above under one coherent "required dependency unusable" umbrella, rather
  than being handled differently from it.

## Live versus historical access path (new in Revision 2)

**A Sensor's persistent payload holds only its latest snapshot** (Sensor
contract, section 9 — "the common envelope is a latest-state interface, not
a historical data bus"). That is correct and sufficient for live
evaluation, but it **cannot** be used to rebuild MCtx's own historical
subgraphs during full recalculation: reading the Sensor's persistent payload
at every historical bar would classify every single historical bar using
whatever the Sensor's value happens to be *right now* — effectively judging
the past using the future. This must be prevented explicitly, not left to
be discovered as a bug.

Two distinct access paths, used in two distinct contexts:

- **Live/latest evaluation** (normal, non-recalc operation): read the
  Sensor's validated **persistent snapshot**, via the state consumption
  pattern exclusively (`identityValid && Status == VALID && freshnessValid`
  — see the envelope contract). The current valid state may be read
  repeatedly, on every evaluation, with no event-style deduplication —
  Sensors are state-style producers and there is no `EventSeq` to track on
  that side. MCtx must not invent one.
- **Historical recalculation** (rebuilding MCtx's own historical
  subgraphs): read the Sensor's **documented historical subgraph arrays**
  at each evaluation bar (`GetStudyArrayUsingID`/
  `GetStudyArrayFromChartUsingID` against the Sensor's own subgraph
  indices — hidden subgraphs are an acceptable and expected source here,
  per the Sensor contract's own hidden-subgraph allowance), never the
  persistent payload.

Because the Sensor contract already requires "both representations must
agree for the same evaluation anchor" (identical measurement semantics
between a Sensor's subgraph and its persistent payload), switching access
path based on live-vs-historical context introduces no semantic
inconsistency — it is purely a difference in *where* the same fact is read
from.

Each Sensor dependency declaration must therefore also map:

- **Persistent payload fields used for live/latest state** (already
  required above).
- **The corresponding Sensor subgraph index(es) used for history** — which
  specific subgraph on the Sensor study exposes the same fact for
  historical bars.
- **Chart/time alignment rules** — how an MCtx recalculation bar index maps
  to the corresponding index in the Sensor's array, especially where the
  Sensor is cross-chart/HTF (Sensor contract section 4) and the two charts'
  bar indices don't correspond 1:1.
- **The Sensor's historical validity signal** — per Sensor contract section
  9, every Sensor publishes a historical `Status` signal (standard approach:
  a hidden `K_CHAKRA_STATUS`-valued subgraph, or a documented
  sensor-specific equivalent) alongside its measurement subgraph(s). This
  declaration must name which subgraph/mechanism supplies it for *this*
  dependency — there is no default, and there is no substitute for it.

**Historical evaluation at a given bar requires all three of the following
— coverage alone is not validity, and none of the three may be skipped:**

```
mapped source index exists (array coverage)
AND historical Sensor Status at that index == K_CHAKRA_STATUS_VALID
AND the required measurement value(s) are finite and in documented range
```

Array coverage only proves the array is long enough to have *some* number at
that index — it says nothing about whether that number is a warmup/default
placeholder, a deliberately-cleared value, a nonfinite result, or a genuine
measurement. Only the Sensor's own historical validity signal answers that.
The finiteness/range check is defense in depth on top of the `Status` check,
not a substitute for it — a bug could in principle write `VALID` alongside a
bad number, and this catches that case too. If any of the three conditions
fails at a given historical bar, that bar's classification is `NOT_READY`,
never a partial or best-effort read — same atomicity discipline as every
other validity gate in this framework.

`sc.DataStartIndex` can help define a Sensor's own warmup boundary, but it
is not sufficient on its own: it says nothing about intermittent invalidity
occurring *later* in history (e.g. a dependency of the Sensor itself
becoming briefly unavailable mid-session on some historical date) — only a
genuine per-bar historical validity signal covers that.

- **`LOW_PREC_LEVEL` and manual-loop + `GetCalculationStartIndexForStudy()`
  handling**, where cross-study historical alignment requires it — the same
  requirement already stated for Sensors themselves, restated here
  specifically for the recalculation-time historical read path.

**If the Sensor dependency is retrospective** (publishes
`PertainsToBarIndex`/`PertainsToDateTime` per the Sensor contract), the
mapped subgraph index used for MCtx's own historical recalculation **must be
the Sensor's availability-indexed machine subgraph — the one written at
`EvalBarIndex`, i.e. when the fact became knowable — never the
pertains-to-indexed visual subgraph.** Reading the visual subgraph here
would reintroduce lookahead bias into MCtx's own historical recalculation
even though the Sensor contract already prevents it at the source: MCtx
would see the retrospective fact placed at the earlier (pertains-to) bar and
classify history as if that information had been available then, when it
was not. If MCtx's own payload wants to expose *what the fact was actually
about* (as opposed to when it became usable), it propagates the Sensor's
`PertainsToBarIndex`/`PertainsToDateTime` as its own provenance fields,
using the same naming convention, clearly distinct from its own
`EvalBarIndex`/`EvalDateTime`.

------------------------------------------------------------------------

# 5. Input validity and status

Reuses `K_CHAKRA_STATUS`, MCtx-specific meaning:

- **`NOT_READY`** — the required Sensor is warming up, **or** the Sensor is
  valid but the MCtx's own classification logic cannot yet determine an
  initial state (e.g. a minimum-duration confirmation window, section 6,
  hasn't yet elapsed). Both are normal, expected, temporary.
- **`VALID`** — the context's `State` (and any present `Direction`/
  `Strength`/`Confidence`) is complete and trustworthy.
- **`DISABLED`** — the MCtx's own `Enable` input is off.
- **`ERROR`** — configuration is invalid, or a required dependency failed
  *unexpectedly* (distinct from the Sensor merely being `NOT_READY`, which
  propagates as MCtx `NOT_READY`, not `ERROR` — `ERROR` means something is
  actually wrong, not just "still warming up").

**When the required Sensor becomes invalid or stale:**
- Immediately invalidate any cached context — no exceptions by default.
- Publish `NOT_READY` or `ERROR` (whichever applies) with a deterministic
  `ReasonCode`.
- **Do not retain the previous context as if it were still current.** This
  is the same "cached state becomes immediately unusable" rule from the
  envelope contract, applied to MCtx's *own* published state, not just to
  what MCtx reads from its Sensor.
- Identity/schema fields (`SchemaVersion`, `ComponentKind`, `ProducerTypeID`,
  `PayloadSchemaVersion`) are preserved through all of this, per the
  envelope's identity-survives-invalid-status rule.

**Graceful carry-forward, if wanted at all, must be an explicit,
context-specific, bounded-age rule** — e.g. "continue reporting the last
classification for up to 5 minutes if the Sensor goes briefly stale" is a
legitimate design choice, but only if it's a deliberate, documented input
with a stated reason and a bound. It must never be the accidental behavior
of code that simply forgot to check freshness before reusing a cached value.

------------------------------------------------------------------------

# 6. Classification rules

Thresholds and state transitions must be deterministic and explicit. Fixed
behavioral rules are code-versioned; concrete contexts may expose only
specifically documented per-instrument tuning parameters as ACSIL `Input`s,
matching the base architecture's versioning boundary.

Every context must document, in writing:

- **Exact inputs and units** — which Sensor field(s), in what units.
- **Thresholds** — concrete numbers (e.g. "40-50% = `UNDER_EXPANDED`," per
  the base architecture spec's own Weekly Expansion example).
- **Inclusive/exclusive boundary behavior** — is exactly 40% `UNDER_EXPANDED`
  or `NORMAL`? State `>=` vs `>` explicitly; never leave a boundary
  ambiguous.
- **Precedence if several state conditions match** — deterministic ordering,
  not "whichever `if` happens to come first in the code, undocumented."
- **`UNKNOWN` fallback** — what happens when inputs don't cleanly map to any
  defined state. Must default to `UNKNOWN = 0`, never silently picked as the
  "closest" wrong bucket.
- **Hysteresis, if used** — must be documented explicitly (e.g. "requires a
  2-point buffer past the original threshold to switch back"), to prevent
  undocumented flapping near a boundary.
- **Minimum-duration/confirmation requirements, if used** — e.g. "state must
  persist for N bars before being published as confirmed." This mirrors the
  existing Renko ring/dot two-brick confirmation pattern already established
  in this codebase (`reference_acsil_skill.md`): a probe is not an accepted
  state change, only a confirmed one is.
- **Closed-bar or intrabar classification** — stated explicitly, matching
  the Sensor contract's cadence-declaration requirement.
- **Whether a state may revise before bar close** — can an intrabar
  classification flip back and forth before the bar closes, or does it lock
  in once set for that bar?

**Correction from Revision 1:** Revision 1 allowed classification thresholds
to remain user-configurable ACSIL `Input`s, mitigated only by requiring
replay/live configurations to match and be logged. That directly softened
the base architecture spec's already-settled rule — "no separate
calibration/config infrastructure... threshold constants live as code
constants... no hot-editing a threshold without committing it, or replay
runs against historical data won't match what live actually did" — without
flagging the conflict. That was a quiet, unauthorized deviation, not a
considered design choice, and is reverted here. The boundary is:

- **Classification thresholds, hysteresis values, and confirmation
  constants are code constants**, exactly per the base architecture spec.
  Not ACSIL `Input`s. A threshold change is a code change and a commit —
  full stop, no exception for MCtx.
- **Operational wiring** (which chart/study a Sensor dependency points at)
  **and display choices** (subgraph colors, draw styles, labels) may remain
  ACSIL `Input`s — these don't affect classification meaning, only plumbing
  and cosmetics.
- **Cadence may be configurable only where it does not change the meaning
  of the classification.** If choosing closed-bar vs. intrabar evaluation
  would itself alter which state gets classified (not just when it's
  reported), that choice is not merely operational and must not be exposed
  as a runtime input.
- **Changing a classification rule — a threshold, a hysteresis value, a
  confirmation duration — requires a code change and a commit.** If tunable
  threshold inputs are ever genuinely wanted, that is a conscious change to
  the base architecture itself, to be made there first and explicitly — not
  introduced quietly inside one layer's contract.

**Revision 5 amendment — supersedes the blanket input prohibition above:**

- **Classification formulas, state-machine structure, transition precedence,
  and non-designated behavioral constants remain code constants.** Changing
  those rules requires a code change and commit.
- **A concrete MCtx may explicitly designate a genuine per-instrument tuning
  value as an ACSIL `Input`.** Its specification must define the parameter's
  meaning, units, default, valid range, and exact effect on classification.
  Being numerically threshold-like does not by itself authorize an input;
  the concrete specification must name it as a tuning parameter.
- **A tuning-input change must have deterministic lifecycle behavior.** If
  the parameter can change historical state or counters, the MCtx must
  discard the prior derived state and reconstruct from clean initialization
  before publishing `VALID` under the new value. Continuing old state under
  the new configuration is prohibited.
- **Live/replay equivalence requires the same code, market data, and tuning
  values.** Manual ACSIL inputs are the current configuration path. Future
  Bheshma-supplied JSON may populate designated inputs, but its schema,
  loading, and versioning are outside this contract and are not anticipated
  with interim machinery here.
- **Operational wiring, display choices, and meaning-preserving cadence
  choices remain permitted inputs** under the existing rules above.

------------------------------------------------------------------------

# 7. State transitions and publication

**MCtx is state-style, not event-style.** This is non-negotiable, same as
for Sensors:

- Uses `PublicationSeq` as the general commit marker.
- **Never defines a payload-level `EventSeq`.** If a component needs
  event-dedupe semantics, it has stopped being MCtx and belongs in
  TriggerStr instead.
- Publishes when the complete public context snapshot (envelope + payload)
  actually changes — the envelope's publish-on-change rule, unmodified.
- The current valid state remains reusable by any consumer until stale or
  invalid — the state pattern, not gated on a fresh publication having just
  arrived.
- State transitions may be logged for diagnostics, but **are not executable
  events.**

**If a downstream component needs to detect a transition (e.g. `RANGE ->
TREND`), it compares current and previous context state privately, on its
own initiative.** The consuming component keeps its own private "last
observed `State`" (a private PV in its own 50-89 range — a business-logic
value, not to be confused with `LastObservedSeq`, which is a sequence-cache
hint) and compares it to the current `State` each evaluation. **MCtx must
never become a TriggerStr merely to announce a transition** — that would be
event-pattern machinery leaking into a state-style layer, exactly the
boundary section 5 of the envelope contract and this document's own rules
above exist to prevent.

**This private tracking needs the same lifecycle-baselining discipline
already established for the envelope's `LastConsumedEventSeq` (Revision 6/7)
— this is that exact bug class recurring in a new place, and the fix is the
same shape.** Without it: a consumer attaching while MCtx is already
`TREND`, with its private previous-state defaulting to `UNKNOWN` (the
required zero-default enum value), would compare `UNKNOWN` against `TREND`
on its very first evaluation and misread it as a fresh transition — even
though nothing just happened; MCtx may have been sitting in `TREND` for
hours. Any consumer doing private transition detection against an MCtx
must:

- **Explicitly baseline its private previous-state to the current valid
  MCtx `State`** — not leave it at its `UNKNOWN` default — at all of: initial
  attachment, dependency rewiring, the consumer's own full recalculation, and
  the consumer's own disable -> re-enable. Same four boundaries as the
  envelope's own consumer-lifecycle rule, applied here to a business-logic
  value instead of a sequence number.
- **Never fire/report a transition on the evaluation where baselining
  happens.** Baselining and transition-detection are never the same call —
  same discipline as the envelope's "baselining and consuming never happen
  in the same evaluation" rule.
- **Compare only after a subsequent valid MCtx publication/state change**
  beyond the baseline.
- **If the MCtx becomes invalid (`Status != VALID`), invalidate the
  consumer's baseline**, and re-baseline (silently, no transition fired) once
  MCtx becomes valid again — **unless** the consumer explicitly defines
  "recovery from invalid" as itself a transition worth reporting, which must
  be a deliberate, documented, per-consumer choice, never an accidental
  default.

This matters most once TriggerStr exists: a TriggerStr using an MCtx
transition as part of a setup condition is exactly the case where an
unbaselined comparison would manufacture a false setup the instant it
attaches — the TriggerStr contract pass must incorporate this rule, not
rediscover it.

------------------------------------------------------------------------

# 8. Direction, strength, and confidence

Keep these three concepts strictly separate — they answer different
questions and must never be conflated into one field or into each other:

- **Direction/Bias** — descriptive orientation (bullish/bearish/neutral).
  Answers "which way does this fact point," purely as description.
- **Strength** — magnitude of the observed context. Answers "how big is
  this effect."
- **Confidence/Quality** — reliability of the classification itself.
  Answers "how sure is this classification," which is a different question
  from both of the above — a context can be low-magnitude but
  high-confidence, or high-magnitude but low-confidence (e.g. based on
  thin/noisy underlying data).

**Do not treat `BULLISH` as "go long."** Direction is a fact about the
market, never an instruction. This is section 3's boundary applied
specifically to this field, since `Direction` is the field most tempting to
quietly turn into a recommendation.

**Do not make `Strength`/`Confidence` mandatory universal fields.** Include
them only where a specific context can define their thresholds meaningfully
and deterministically (section 6) — a context with no principled way to
compute confidence should omit the field entirely, not publish a fabricated
number to satisfy a perceived contract requirement.

**Every enum value must have documented, precise semantics.** Avoid a label
like `STRONG` unless the exact threshold that produces `STRONG` is written
down (section 6) — an undocumented `STRONG` is just as much a boundary
violation as an undocumented `UNDER_EXPANDED` would be.

------------------------------------------------------------------------

# 9. Decay and freshness

**The MCtx producer selects its context's `DecayClass` based on how long the
classified *fact* remains meaningful — not by copying its Sensor's
`DecayClass`.** These can differ in either direction:

- A **`FAST`**-class Sensor can produce a **`SLOW`** context, if the
  classified fact (e.g. "recent aggressive buying pressure detected") stays
  meaningful longer than the raw tick-level measurement that triggered it.
- A **`STATIC_SESSION`** weekly context can have a hard weekly boundary even
  though its underlying Sensor readings update far more often.
- A context must never remain `VALID` past a known hard boundary — once
  `ValidUntilDateTime` passes for a `STATIC_SESSION` context, it must
  transition out of `VALID`, not linger.

Consumers still choose their own maximum tolerated age — MCtx declaring a
factual `DecayClass` does not override a consumer's own tolerance decision,
same principle as the Sensor and envelope contracts.

Fields:

- **`EvalDateTime`** — when/as-of what market time *this classification*
  was made. This can differ from the Sensor's own measurement time: if the
  MCtx applies a minimum-duration confirmation window (section 6), the
  classification's `EvalDateTime` may lag the raw Sensor reading that
  ultimately triggered it.
- **`ValidUntilDateTime`** — concrete hard boundary for `STATIC_SESSION`
  context, pre-resolved at publish time (never symbolic/lazy), exactly as
  required for Sensors.
- **Source Sensor measurement time, tracked separately where useful** — an
  MCtx that wants to expose *both* "when I classified this" and "when the
  underlying Sensor data was actually measured" needs a distinct payload
  field for the latter, since these are not guaranteed to be the same
  instant once confirmation windows are involved. This is part of the
  "contributing Sensor provenance" payload requirement from section 2. **If
  the Sensor is cross-chart, propagate its `SourceDataAsOfDateTime`
  specifically — never its `SourcePeriodStartDateTime`** (Sensor contract
  section 4): freshness must always be computed from the latest incorporated
  data, not from which period is being measured. A field named
  `SourceSensorDataAsOfDateTime` (not `...EvalDateTime`) makes this
  unambiguous in MCtx's own payload.
- **Cross-chart provenance** — if the MCtx's *own* Sensor dependency is
  itself a cross-chart/HTF sensor (per the Sensor contract's
  `SourceChartNumber`/`SourceBarIndex`/`SourcePeriodStartDateTime`/
  `SourceDataAsOfDateTime` fields), the MCtx should propagate those as part
  of its own provenance payload rather than re-deriving them — propagating
  the `DataAsOf` field specifically for freshness, as above, not just
  whichever `Source*DateTime` happens to be convenient. If the MCtx itself
  runs on a different chart than its Sensor (an additional layer of
  indirection beyond the Sensor's own cross-chart handling), it needs its
  own analogous `Source*` fields, following the same pattern the Sensor
  contract already defines. **If the Sensor is retrospective, this
  provenance is separate from and must not be confused with the
  `PertainsToBarIndex`/`PertainsToDateTime` propagation described in
  section 4** — one is about *where the source data comes from*, the other
  about *what an already-available fact is actually about*.

------------------------------------------------------------------------

# 10. Conflict handling

Restating the base architecture spec's already-resolved principle, because
it is easy to accidentally violate in an implementation even when everyone
agrees with it in the abstract:

- **Contexts do not arbitrate against other contexts.** They are
  independent, purely descriptive facts.
- **Contradictory-looking facts may coexist.** `WeeklyExpansion =
  UNDER_EXPANDED` and `TrendStretch = EXTREME` are not in tension — both can
  be simultaneously true and simultaneously valid.
- **No global conflict table.** Nothing in this framework maintains a
  shared registry of "conflicting context pairs."
- **An MCtx must never suppress its own publication because another context
  says something different.** Each context evaluates and publishes entirely
  on its own inputs, with zero awareness that other contexts even exist.
- **Reconciling multiple simultaneous truths into one action is a Trade
  Policy problem, solved locally inside the consuming component** — not an
  MCtx problem, and not a framework-level arbitration problem.

------------------------------------------------------------------------

# 11. Recalculation and replay

Follows the Sensor contract's rules directly (MCtx is also state-style,
also zero-side-effect):

- **Historical subgraphs rebuild deterministically, every bar, by reading
  each Sensor dependency's historical subgraph arrays — never its
  persistent payload.** This is section 4's live-vs-historical access path
  rule, restated here because recalculation is exactly where getting it
  wrong is easiest: reading a Sensor's persistent (latest-only) snapshot
  while looping through history would classify every historical bar using
  today's Sensor value.
- **No external actions, ever** — recalc or not, live or replay.
- **No event publication** — reinforces section 7; recalculation is not an
  excuse to introduce event-style behavior.
- **Persistent latest-state publication follows the Sensor contract's
  low-churn pattern exactly**: at most two persistent publications per full
  recalculation sweep (recalc-entry `NOT_READY`, then one final publication
  representing the last relevant bar's properly-evaluated state).
  `sc.IsFullRecalculation && sc.Index == sc.ArraySize - 1` for AutoLoop
  producers; falls out naturally for manual-loop's one-call-per-sweep
  convention.
- **The final snapshot represents the latest properly evaluated context** —
  not an intermediate warmup-era or mid-confirmation-window value.
- **Replay produces the same state transitions from identical data and
  configuration** — determinism, tied to section 6's no-hidden-threshold
  rule.
- **Stale historical subgraph states are cleared deliberately**, same as
  Sensors — a changed threshold or logic revision must not leave old,
  now-invalid classification markers lingering.

------------------------------------------------------------------------

# 12. Subgraphs versus persistent payload

Same shape as the Sensor contract, applied to context data:

- **Subgraphs** carry historical context state, `Direction`, `Strength`,
  and supporting measurements — for visualization, study-on-study
  consumption, and replay inspection. A `State` subgraph plots the integer
  enum value per bar so other studies/tools can read historical
  classification via `GetStudyArrayUsingID`.
- **Persistent payload** exposes only the latest generic context snapshot.
- **Both representations must use identical enum values and identical
  classification logic** — a subgraph must never show an approximation
  while the payload holds the "real" answer, or vice versa.
- **Hidden machine-readable subgraphs are acceptable** where visualization
  is undesired (`DrawStyle = DRAWSTYLE_IGNORE`, same established convention).
- **A consumer needing historical context reads subgraphs.** Persistent
  storage remains latest-state only, never a historical bus.

------------------------------------------------------------------------

# 13. Diagnostics

```cpp
enum K_CHAKRA_MCTX_REASON_CODE
{
    K_CHAKRA_MCTX_REASON_NONE                     = 0,   // = K_CHAKRA_REASON_NONE
    K_CHAKRA_MCTX_REASON_SENSOR_UNWIRED            = 1,
    K_CHAKRA_MCTX_REASON_SENSOR_SCHEMA_MISMATCH    = 2,
    K_CHAKRA_MCTX_REASON_SENSOR_UNAVAILABLE        = 3,
    K_CHAKRA_MCTX_REASON_SENSOR_STALE              = 4,
    K_CHAKRA_MCTX_REASON_WARMUP_INCOMPLETE         = 5,
    K_CHAKRA_MCTX_REASON_CLASSIFICATION_UNAVAILABLE = 6,
    K_CHAKRA_MCTX_REASON_CONFIGURATION_INVALID     = 7,
    K_CHAKRA_MCTX_REASON_INVALID_MEASUREMENT       = 8,   // NaN/Inf or otherwise non-finite
    K_CHAKRA_MCTX_REASON_SOURCE_PROVENANCE_INVALID = 9
    // extend per concrete context as needed, numbered above these common ones
};
```

**Log transitions or first occurrence of a given `ReasonCode`, not every
update** — same convention as Sensors, matching the `DebugLogging` input
pattern already used throughout this codebase.

------------------------------------------------------------------------

# 14. Worked examples

**Weekly Expansion MCtx consuming Weekly ATR Consumption Sensor** — direct
continuation of the Sensor contract's own worked example:

| Field | Type | Namespace/PV | Notes |
|---|---|---|---|
| `State` | int (`K_CHAKRA_MCTX_WEEKEXP_STATE`) | Int32 PV 21 | `UNKNOWN=0` / `UNDER_EXPANDED=1` / `NORMAL=2` / `OVER_EXPANDED=3` |
| `PercentConsumed` | float | Float PV 1 | Copied through from the Sensor's own field — the number that justifies `State` |
| `SourceSensorChartNumber` | int | Int32 PV 22 | Provenance: which chart the `K_Chakra_Sensor_WeeklyATRConsumption_V1` instance is on |
| `SourceSensorStudyID` | int | Int32 PV 23 | Provenance: which study instance |
| `SourceSensorDataAsOfDateTime` | SCDateTime | Datetime PV 10 | Propagated from the Sensor's own `SourceDataAsOfDateTime` (not its `SourcePeriodStartDateTime`) — use this for freshness, may differ from this context's own `EvalDateTime` if confirmation windows apply |

This example's Sensor is not retrospective, so no `PertainsTo*` propagation
is needed here — a context consuming a retrospective Sensor (e.g. one built
on a Renko ring/dot detection) would additionally propagate
`PertainsToBarIndex`/`PertainsToDateTime` per section 4.

Classification bands are illustrative only — per the base architecture
spec's own example, `40-50%` consumed by Thursday close is specifically the
`UNDER_EXPANDED` condition (the statistically interesting case, not a
"normal middle band"). This table shows field *shape*; a concrete context's
own contract must state its exact bands and inclusivity per section 6.

**Trend MCtx** consuming an objective trend-measurement Sensor (e.g. a
directional-movement or higher-high/higher-low measurement) — publishes
`State` (`RANGE` / `TREND_UP` / `TREND_DOWN`, `UNKNOWN=0`), optionally
`Direction` and `Strength`, with documented thresholds and (if used)
hysteresis and minimum-duration confirmation per section 6.

**Auction MCtx** consuming a breach/reclaim measurement Sensor — publishes
`State` including `FAILED_LOWER_AUCTION` / `FAILED_UPPER_AUCTION` /
`NONE`/`UNKNOWN`, matching the base architecture spec's own Auction
Asymmetry example (prior low breached, then reclaimed, sellers failed).

**Invalid — policy leakage:** a "Trend" context publishing
`AllowBreakoutEntry = true`. That is an `EntryGate` decision wearing a
context's name — the objective trend state (`TREND_UP`, with its supporting
measurements) is the legitimate payload; the allow/reject judgment belongs
in Trade Policy.

**Valid — coexistence without arbitration:** `WeeklyExpansion.State =
UNDER_EXPANDED` and `TrendStretch.State = EXTREME` published simultaneously
by two independent MCtx studies, neither aware of the other, neither
suppressing itself. A consuming `TrailingManager` might use both together
(e.g. "trail loosely because under-expanded, unless stretch is extreme") —
that reconciliation is the `TrailingManager`'s own local rule (per section
10), not something either context participates in.

------------------------------------------------------------------------

# Unresolved design questions — deliberately left open

Only questions that affect real implementation. Envelope and Sensor
decisions already settled are not reopened here.

1. **Does hysteresis metadata need a common payload field?** E.g. a
   standardized `BarsSinceLastTransition` or `IsNearBoundary` field usable
   generically by any consumer, or does hysteresis stay entirely internal/
   private to each context's own classification logic with no public
   visibility? Section 6 requires hysteresis to be *documented*, but doesn't
   currently require it to be *exposed* as a field.
2. **Is source Sensor provenance standardized or context-specific?** The
   worked example above uses ad hoc `SourceSensor*` field names; should
   there be a common convention (fixed PV slots, fixed field names) across
   every MCtx that depends on a Sensor, the way `PayloadSchemaVersion` has a
   fixed slot (PV 20) in the envelope convention? Not resolved here.
3. **Should `Confidence` remain purely context-specific**, as section 8
   currently states, or does the framework eventually need a common
   confidence scale/definition once enough contexts implement one and
   consumers start wanting to compare across contexts?
4. **How does provisional intrabar Sensor data affect context finality?**
   This directly extends the Sensor contract's own open question 2
   (provisional intrabar publication). If an MCtx's input Sensor is
   currently publishing a provisional (not-yet-final) intrabar value, should
   the MCtx's own classification also be considered provisional/subject to
   revision, or does MCtx simply classify whatever it currently reads at
   face value, regardless of the Sensor's own provisional status? Not
   resolved here — depends on how Sensor open question 2 is eventually
   settled.
