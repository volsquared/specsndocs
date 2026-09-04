# Chakra — Trade Policy: StopManager Interface Contract (Initial Risk Geometry, Pre-Entry) (Draft, Revision 10)

Status: **design only, no ACSIL implementation yet.**

------------------------------------------------------------------------

# Changes from Revision 9 (normative cleanup — Entry Authority/post-fill/PositionSizer machinery removed from the v1 live path, 2026-07-25)

**A direct correction, not a review round: Revision 9 added a clarifying
note about where `PV54`/`PV71` land, but left the entire normative body of
the document — section 9's pre-fill/post-fill reconciliation regimes and
hard gate, the "Pipeline position" section's PositionSizer framing, the
worked examples — still written as though the greenfield Entry Authority
and PositionSizer both exist and this document depends on them. They
don't, for the v1 live path. This pass rewrites the affected sections so
the document matches reality, not merely adds a pointer past the parts
that don't.**

**For v1: StopManager computes initial stop geometry once, from an
indicative reference price, and publishes it. That is the entire
scope.** No downstream component reconciles StopManager's reference price
against an actual fill — the superseded Order Management design would
have; it isn't being built. Mode Baton executes and manages the trade
using its own existing, tested behavior (which, per the source audit,
performs no reference-vs-fill reconciliation of its own either) and Candle
Baton owns everything after that. This document does not fill that gap
and does not claim anything downstream fills it either.

**Removed or retired, not merely annotated:**

- **Section 9's pre-fill/post-fill reconciliation regimes and its hard
  gate are removed outright.** There is no downstream policy for
  StopManager's geometry to wait on — the "must not become part of an
  executable entry until that downstream policy exists" gate had a
  precondition (that a downstream policy would eventually exist) that is
  no longer true, and the gate is retired along with it, not left pointing
  at a policy that was never built and isn't going to be.
- **"Pipeline position" no longer frames StopManager alongside
  PositionSizer at all.** PositionSizer is not part of the v1 live path
  (`Chakra_TradePol_PositionSizer_Interface_Contract.md`'s own superseded
  banner); StopManager's only consumer is the Entry Composer, full stop.
- **Every claim that a "complete action" requires candidate + size +
  geometry is corrected to candidate + geometry.** Quantity is Mode
  Baton's own concern for mode 25 (source-audit finding), not something
  StopManager's output is assembled alongside.
- **Open question 2 (reference-vs-fill reconciliation ownership) is
  retired, not "resolved."** It pointed at the superseded Order Management
  Handoff design; that design is not being built, so the question is not
  answered — it no longer applies, because this document no longer claims
  a reconciliation obligation exists for anything to own.

**Not affected**: geometry computation (section 6), risk-safe tick
rounding (section 7), validation (section 8, apart from removing the now-
inapplicable "instrument identity" framing already gone since Revision 6),
the architecture decision (StopManager = initial geometry only), and the
composite-correlation-identity delivery record (section 10) — all
unchanged in substance.

------------------------------------------------------------------------

# Changes from Revision 8 (Mode Baton integration clarification, 2026-07-25)

**Purely additive — no existing section 1-14 content changed.** A source
audit of `K_Util_ModeBaton_V1.cpp` found this document's own delivery
record (section 10) is directly reusable, unmodified, by
`Chakra_ModeBatonSignalPublisher_Interface_Contract.md` (new, Revision 1)
for two of Mode Baton's mode-25 fields, once converted:

```text
PV54 (stop distance, price-unit) = StopGeometryDistanceTicks * StopGeometryTickSize
PV71 (absolute stop reference price) = StopGeometryStopPrice
```

`StopGeometryStopPrice` is already risk-safe tick-rounded per section 7
(floor for `LONG`, ceiling for `SHORT`) — the publisher does not re-round
or re-derive it. No new field, no new section, no behavioral change to
this contract: this note exists only so a reader of this document alone
sees where its output actually lands, since the greenfield Order
Management/Entry Authority consumer this document originally anticipated
is superseded (see `Chakra_OrderManagement_Handoff_Interface_Contract.md`'s
own banner).

------------------------------------------------------------------------

Builds directly on:
- `Algo_Overlay_Framework_Architecture.md`, including the master decision
  cycle, `POLICY_DECISION`, `PROPOSED_ACTION`, and bounded `DecisionFrame`
  rules.
- `Chakra_Common_Interface_Envelope_Contract.md` (approved, Revision 8).
- `Chakra_Sensor_Interface_Contract.md` (Revision 4).
- `Chakra_MCtx_Interface_Contract.md` (Revision 5).
- `Chakra_TriggerStr_Interface_Contract.md` (Revision 3).
- `Chakra_TradePol_EntryGate_Interface_Contract.md` (Revision 3).

**`Chakra_TradePol_PositionSizer_Interface_Contract.md` is not a
dependency of this document as of Revision 10.** It never was, since
Revision 5 (see that revision's own "Changes" entry, below) — Revision 10
only removes the last vestiges of PositionSizer-adjacent framing from the
"Pipeline position" section and the worked examples (see "Changes from
Revision 9," above).

**This document designs only StopManager's pre-entry, initial-risk-geometry
role.** Open-position stop management (protective ratcheting once a
position exists) has materially different inputs, safety rules, and
order-state concerns and is explicitly **not** designed here — and, for
the v1 live path, not needed anywhere in Chakra at all (the existing
Candle Baton owns it) — see the architecture decision, below.

------------------------------------------------------------------------

# Changes from Revision 1

Three material issues and one minor consistency issue, all from external
review of Revision 1. Nothing else in Revision 1 was reopened — the
initial-geometry ownership decision, the audit/delivery sequence
separation, the expiry chain, the latest-wins overwrite policy, and the
prohibition on order actions all stand unchanged.

**Superseded in part by Revision 5 (below): item 1's "circularity
resolution" it refers to no longer exists — PositionSizer no longer
consumes `StopGeometrySeq` at all, for any reason. Preserved as an
accurate record of what was true and fixed at the time.**

1. **Section 12 corrected — no longer mischaracterizes or overclaims the
   circularity resolution.** Revision 1 described PositionSizer's
   consumption of `StopGeometrySeq` as "the same state-pattern-style read
   used for any other required dependency" and called the result "a real,
   working resolution for the common case." Both were wrong. `StopGeometrySeq`
   is consumed via the **event pattern** (sequence-gated, candidate-
   correlated), not the state pattern — a plain state-pattern read would
   let StopManager's independent, parallel evaluation silently miss a
   candidate that PositionSizer observed before StopManager's matching
   geometry was published, permanently losing it under PositionSizer's own
   default consume-and-drop rule. The actual operational fix — retry via
   non-advancement of `LastConsumedCandidateSeq`, with mandatory
   `StopGeometrySourceEntryGateCandidateSeq` correlation, scoped to
   stop-dependent sizing methods only — was designed and lived in
   `Chakra_TradePol_PositionSizer_Interface_Contract.md`, Revision 3,
   section 6, until that document's own Revision 6 removed stop-dependent
   sizing entirely.
2. **Section 7's rounding math corrected.** Revision 1's worked example
   rounded a fractional *tick count* directly (12.3 ticks "rounds up to
   12.5 ticks"), which is meaningless — 12.5 is not a whole-tick count
   either. The corrected operation order: compute the ideal stop *price*,
   round that *price* to the instrument's tick grid (floor for `LONG`,
   ceiling for `SHORT` — away from entry), then **recompute** the actual
   whole-tick distance from the rounded price. The worked example
   (section 14, example 2) is corrected to match. **This fix is unaffected
   by Revision 5 — it is pure price-grid geometry, not risk sizing.**
3. **Section 9 rewritten to distinguish pre-fill from post-fill
   reconciliation, with an explicit hard gate.** Revision 1 described the
   fill-discrepancy problem as a single, flat "revalidation or rejection"
   choice. That's only coherent before a fill exists. Once Order Management
   has actually filled the entry, quantity is already committed — rejection
   is no longer an available remedy; the real remedies (retain the stop as
   computed, adjust its price, reduce quantity, or emergency-exit) are
   qualitatively different from the pre-fill case and belong to policy that
   does not exist yet. Section 9 now states this distinction directly and
   adds an explicit hard-gate requirement: StopManager's geometry must not
   become part of an executable entry until that downstream policy exists.
4. **Minor — section 8's wrong-side-of-entry mapping corrected for internal
   consistency.** Sections 3 and 13 already mapped a wrong-side-of-entry
   construction failure to `REJECT_DISTANCE_INVALID`
   (`WRONG_SIDE_OF_ENTRY`). Section 8 independently said `SKIP_UNDECIDABLE`
   for the same failure. `REJECT_DISTANCE_INVALID` is kept — it fits the
   existing enum (a rejected geometry, not an inability to decide) — and
   section 8 is corrected to match sections 3 and 13.

Open question 3 (same-cycle coordination tightness) was updated in
Revision 2 to reflect that PositionSizer's then-existing Revision 3 retry
rule already removed the correctness blocker; **it is retired outright in
Revision 5 below, since the coordination it discussed no longer exists.**

------------------------------------------------------------------------

# Changes from Revision 2

One upstream correction, surfaced by the Entry Composer's own contract
review rather than by a direct review of this document — flagged
explicitly and fixed at the smallest necessary scope:

1. **`StopGeometrySourceEntryGateCandidateSeq` alone is not a globally
   safe correlation key.** The same issue PositionSizer's own Revision 4
   fixed for `SizedCandidate` applies identically here: a downstream
   consumer (the Entry Composer) needs to verify that a `StopGeometry`
   record came from the *same* EntryGate candidate as a correlated
   `EntryCandidate` — but a bare sequence-number comparison can't do that
   safely, since two different EntryGate instances can independently
   arrive at the same numeric `CandidateSeq` after a rewiring event or
   producer reconstruction. Fixed: the stop-geometry delivery record
   (section 10) now also carries the composite EntryGate identity —
   `StopGeometrySourceEntryGateChartNumber`/`...StudyID`/
   `...ProducerTypeID` alongside the existing
   `StopGeometrySourceEntryGateCandidateSeq`. Nothing about this
   document's own coordination behavior, geometry computation, rounding,
   or validation changes — this is purely about what StopManager
   *republishes* for a downstream consumer. **This fix is unaffected by
   Revision 5 — the correlation problem exists for the Entry Composer's
   consumption of `StopGeometry` regardless of whether PositionSizer ever
   also consumed it.**

------------------------------------------------------------------------

# Changes from Revision 3 (user-directed scope correction, part 1, 2026-07-19 — see Revision 5 below for part 2)

**Not a review round — the same project-wide scope boundary applied
identically across every upstream TradePol contract in this pass (see
`Chakra_OrderManagement_Handoff_Interface_Contract.md`'s "Changes from
Revision 3" for the full statement).** Chakra does not own portfolio or
account-level risk management; the user configures the chart, instrument,
and trade account manually, and StopManager operates within that context.
This cancels the previously planned
`Chakra_ExecutionContext_AccountInstrument_Interface_Contract.md`.
**Open question 4 ("what is the authoritative account/instrument identity
source?") is closed, not carried forward** — StopManager's geometry
computation (tick size, reference price) already comes from the
manually-configured instrument the wiring chart/study belongs to, per
ordinary ACSIL instrument-metadata access; no separate identity service
was ever actually required by anything in section 7's geometry math.
Section 10's "instrument/account identity required downstream" note was
corrected to describe this directly rather than deferring to a shared open
question. **Revision 5 (below) goes further and removes the currency-value-
per-tick fields this revision's own text still referenced — read that
section, not this one, for the currently accurate field list.**

------------------------------------------------------------------------

# Changes from Revision 7 (verification pass, 2026-07-19)

Two small wording issues found during final verification, both in section
9:

1. **Medium — stale boundary ambiguity.** The pre-fill regime's first
   bullet still said the discrepancy check was performed by "the Entry
   Composer, or Order Management, depending on how that boundary is
   eventually drawn" — a hedge left over from before that boundary was
   actually settled. It was settled: both perform an independent check, at
   two different moments (Composer at composition time, Entry Authority
   at submission time), the same "independent re-check at every
   consumption stage" discipline used throughout this framework. Fixed to
   state this directly instead of describing it as still open.
2. **Low — inaccurate "already-implemented," and an imprecise "match...
   exactly."** The post-fill paragraph called Order Management's remedy
   set "already-implemented" — it isn't; that contract remains
   design-only, no ACSIL exists for either document. Fixed to
   "already-defined and contractually resolved... though not yet
   implemented in ACSIL by either document." Separately, the pre-fill
   remedy paragraph claimed its own remedies "match Order Management's
   actual post-fill set exactly" — they don't: pre-fill is retain/
   recompute-tighten/decline, post-fill is retain/tighten-replace/
   emergency-exit, not the same set. Fixed to describe them as the same
   *kind* of choice, one stage earlier, explicitly noting the two sets
   aren't identical. (The post-fill-to-post-fill comparison in the same
   section, StopManager's own post-fill remedies against Order
   Management's, genuinely is the same three options — that "matches
   exactly" wording is accurate and unchanged.)

------------------------------------------------------------------------

# Changes from Revision 6 (independent review-correction pass, 2026-07-19)

Two findings from a further review pass:

1. **An impossible instrument-identity comparison, same shape as
   EntryGate's own Revision 7 fix.** Section 8's validation list required
   "instrument identity — matches the candidate's own instrument," and
   section 10 called instrument identity "required downstream" — but
   `EntryCandidate` publishes no instrument field to compare against, and
   `StopGeometry` never needed one either. Fixed: both removed. Tick size
   is read directly from this StopManager instance's own manual wiring
   (section 1); what's actually required downstream is the self-contained
   tick size and geometry fields already in the delivery record, not a
   separate identity to check.
2. **Obsolete quantity-remediation language in section 9's pre-fill and
   post-fill remedy lists.** "Resize" (pre-fill) and "reduce quantity"
   (post-fill) both depended on the risk-based sizing methods
   PositionSizer's own Revision 6 removed — there is no longer a
   mechanism to resize or partially unwind a position based on a risk
   budget. Fixed: both remedy lists now match Order Management's own
   actual, already-implemented post-fill set exactly (retain / tighten-
   or-recompute / decline-or-emergency-exit) — pre-fill: retain,
   recompute/tighten, or decline; post-fill: retain, tighten/replace, or
   emergency-exit.

**Nothing about the fill-reconciliation hard gate's own structure, the
pre-fill/post-fill regime split, or the "rejection is no longer coherent
post-fill" rule changes — only the specific remedy options listed within
each regime.**

------------------------------------------------------------------------

# Changes from Revision 5 (independent review-correction pass, 2026-07-19)

Two independent findings from a review of the Revision 5 scope-correction
pass:

1. **A real gap, mirroring EntryGate's own "Changes from Revision 6"
   fix.** Section 5 described "pending entry" as part of the
   platform-sourced lifecycle check — but the platform has no concept of
   an unsubmitted Chakra-internal proposal, so this was never actually
   checkable that way. Fixed: the bullet is removed; an in-flight
   submission conflict is exclusively the Entry Authority's own
   idempotency check, never visible upstream. **Nothing about the
   corrected source (platform, not the candle manager) from Revision 5 is
   reopened — this fix only narrows what that platform-sourced check
   claims to cover.**
2. **Open question 2 (reference-vs-actual-fill reconciliation ownership
   and tolerance) was stale, not actually open** — `Chakra_OrderManagement_Handoff_Interface_Contract.md`
   section 13 already resolves this (its own Revision 4/6 rewrite), but
   this document's own open-questions list still described it as flagged
   and undesigned. Fixed: marked resolved in place, with the resolution
   and its source stated directly.

------------------------------------------------------------------------

# Changes from Revision 4 (user-directed scope correction, part 2, 2026-07-19)

**A direct user correction — the first scope-correction pass narrowed
account/instrument-identity framing but left dollar-risk plumbing in
place that should have been removed at the same time.** The user
restated the boundary literally: Chakra does not perform monetary,
account, or portfolio risk management. `PositionSizer`'s own Revision 6
(companion pass) removed every sizing method that read a stop distance to
compute a dollar risk figure — **this document's own currency-conversion
fields existed for no other purpose**, and are removed here to match.

**Removed outright:**

1. **`StopDistanceCurrencyPerContract`** (decision record, was Float PV 4)
   and **`TickValue`** (decision record, was Float PV 6) — both existed
   solely to compute a dollar-risk-per-contract figure for PositionSizer's
   now-deleted risk-based sizing methods. Section 7's price-grid rounding
   math never used `TickValue` — only `TickSize` — so its removal does not
   touch geometry computation at all.
2. **`StopGeometryDistanceCurrencyPerContract`** (delivery record, was
   Float PV 9) and **`StopGeometryTickValue`** (delivery record, was Float
   PV 11) — same reasoning, delivery-record side.
3. **Section 8's "Tick size / tick value" validation bullet** narrowed to
   "Tick size" only — tick *value* (currency per tick) is no longer read
   or validated anywhere in this document.
4. **`K_CHAKRA_STOPMANAGER_REASON_TICK_VALUE_SIZE_INVALID` renamed to
   `..._TICK_SIZE_INVALID`** — only tick size is validated now.
5. **Section 9's post-fill remedy list reworded** to drop "risk budget"
   framing (`"reduce quantity to bring realized risk back within budget"`)
   in favor of plain price-distance-from-reference language — there is no
   longer a dollar risk budget concept anywhere upstream of this document
   for a post-fill remedy to be measured against. The remedies themselves
   (retain, adjust stop price, reduce quantity, emergency-exit) are
   unchanged; only the currency framing is removed.

**A larger, structural correction — StopManager and PositionSizer no
longer coordinate at all, because there is nothing left for them to
coordinate about:**

6. **The entire premise of the "Pipeline position" section and section 12
   — that PositionSizer consumes `StopGeometrySeq` for stop-dependent
   sizing methods — is no longer true.** `PositionSizer`'s Revision 6
   removed every sizing method that depended on stop geometry
   (`FIXED_MONETARY_RISK`/`PERCENT_EQUITY_RISK`/`VOLATILITY_RISK_UNIT`/
   `ACCOUNT_TIER`); its three retained methods (`FIXED_QUANTITY`/
   `CONTEXT_MULTIPLIER`/`CAPPED_QUANTITY`) never read `StopGeometrySeq` at
   all. **`StopGeometry` is now consumed by exactly one component: the
   Entry Composer**, which needs the stop *price* (not a dollar figure) to
   assemble a complete `PROPOSED_ACTION`. StopManager and PositionSizer are
   now fully independent parallel producers with no dependency on each
   other in either direction — "Pipeline position" is rewritten below to
   say this plainly, and section 12 (the former coordination-mechanism
   pointer) is retired in place, mirroring how PositionSizer retired its
   own former section 6.
7. **Former open question 3 (same-cycle coordination tightness) is
   retired outright** — there is no coordination left to be tight or loose
   about. Removed from the open-questions list rather than marked
   resolved.
8. **Former worked example 8 (PositionSizer's retry mechanism in action)
   is removed** — it describes a `PERCENT_EQUITY_RISK` PositionSizer and a
   coordination mechanism that both no longer exist. Worked examples
   renumbered 1-11 with no gap.

**Nothing about geometry computation, price-grid rounding (section 7),
validation (section 8, apart from the tick-value narrowing above), the
fill-reconciliation hard gate's actual structure (section 9), the
composite-correlation-identity fix (Revision 2), or the audit/delivery
split (section 10) changes beyond the field removals above.**

------------------------------------------------------------------------

# Architecture decision: `StopManager` is the initial-geometry role, not a new component

**Decided, not left open**: this contract uses the already-reserved
`K_CHAKRA_TRADEPOL_CATEGORY_STOP_MANAGER` (EntryGate contract, section 1)
for the pre-entry initial-risk-geometry role — **no new `RiskGeometry`
`PolicyCategory` is introduced.**

This is grounded in the base architecture document itself, not invented
here: `Algo_Overlay_Framework_Architecture.md`'s own "Trade Policy
(Decomposed)" examples section already treats **"Initial Stop"** and
**"Trailing Stop"** as two separate concerns (`## Initial Stop` / `##
Trailing Stop`), matching the already-reserved `STOP_MANAGER` (3) and
`TRAILING_MANAGER` (4) `PolicyCategory` values exactly. The base
architecture already implicitly decided this split before any Trade
Policy sub-component had its own contract pass — this document makes it
explicit rather than reinventing it: **`StopManager` = initial risk
geometry (this document); `TrailingManager` = ongoing trail/protection
(future, separate contract pass).**

**Do not blur pre-entry stop calculation with later trailing/protection
behavior in this or any future document.** The two have different inputs
(a candidate with no position yet, versus an existing position with live
P&L), different safety rules (geometry validation versus ratchet-only-
forward guards), and different order-state concerns (nothing submitted yet
versus an active stop order). **For the v1 live path, this is moot rather
than merely undecided: Chakra has no post-entry stop-management role at
all — the existing, unmodified Candle Baton owns everything once a trade
is live.** Whether `StopManager` itself ever grows a second phase, or
whether some future `TrailingManager` would take on that role instead, is
new scope for a future initiative to decide, not a question this contract
leaves open (see "Unresolved design questions," below).

------------------------------------------------------------------------

# Core responsibility

**Frozen:**

> StopManager (initial risk geometry) consumes one accepted `EntryCandidate`
> and proposes the initial stop's geometry — side, an indicative reference
> price, stop price, and stop distance — using only its declared
> risk-geometry rules and required Sensor/MCtx facts, before quantity is
> known. It does not size the position, manage the stop after entry, or
> submit, modify, or cancel any order.

**StopManager must not:**

- Determine position quantity — not `PositionSizer`'s job either, for the
  v1 live path: Mode Baton owns quantity for mode 25, and no component in
  this chain computes or publishes one (source-audit finding; see
  `Chakra_TradePol_PositionSizer_Interface_Contract.md`'s own superseded
  banner).
- Choose an entry price, or invent/promise an actual fill price.
- Choose profit targets. (The Mode Baton Signal Publisher computes target
  *prices* for `PV56`/`PV57` from this document's own reference price plus
  a configured tick offset — that is the publisher's own responsibility,
  one layer downstream, not a `ProfitManager`-style policy and not this
  document's concern.)
- Submit, modify, or cancel any order.
- Manage an existing position, or perform any trailing/ratcheting
  behavior.
- Create a complete manager-ready entry action.
- Revisit EntryGate's accept/reject judgment.
- **Compute or publish a dollar/currency risk figure** — tick size is
  retained solely for placing a valid stop price on the instrument's tick
  grid; nothing in this document needs a currency conversion.

Naming: `K_Chakra_TradePol_StopManager_<DescriptiveName>_V<n>`.

**StopManager is event/decision-style**, exactly like EntryGate, and needs
the same two-record separation (audit decision trail vs. correctness-
critical delivery record) — see section 10.

------------------------------------------------------------------------

# Pipeline position — StopManager feeds the Entry Composer directly, nothing else

**Rewritten in Revision 10 — PositionSizer removed from this section
entirely** (see "Changes from Revision 9"; PositionSizer was already not
a coordination partner as of Revision 5, but this section's diagram and
prose still framed StopManager alongside it):

```text
EntryGate candidate
    -> initial stop geometry (StopManager, this document)
    -> Entry Composer -> PROPOSED_ACTION -> Mode Baton Signal Publisher
```

**StopManager consumes EntryGate's candidate record directly.** It needs
no quantity, no size, and no other producer's output to compute a stop
distance — `StopGeometry` is a pure function of the candidate and this
document's own configured risk-geometry rules. The Entry Composer
correlates `StopGeometry` against the same source `EntryCandidate` (its
own contract, section 3) to assemble a `PROPOSED_ACTION`, which the Mode
Baton Signal Publisher then converts into Mode Baton's own signal record.

------------------------------------------------------------------------

# 1. Naming and identity

- **`ComponentKind`** is always `K_CHAKRA_KIND_TRADEPOL`.
- **`PolicyCategory`** is `K_CHAKRA_TRADEPOL_CATEGORY_STOP_MANAGER` (int32
  PV 21, the TradePol-layer-wide slot already reserved by the EntryGate
  contract).
- **`ProducerTypeID`** is scoped to StopManager specifically:
  ```cpp
  enum K_CHAKRA_TRADEPOL_STOPMANAGER_TYPE
  {
      K_CHAKRA_TRADEPOL_STOPMANAGER_TYPE_NONE                = 0,
      K_CHAKRA_TRADEPOL_STOPMANAGER_TYPE_FIXEDTICKS_ORBREAKOUT = 1,
      // one member per concrete named StopManager policy, assigned as each is built
  };
  ```
- **`PayloadSchemaVersion`** (int32 PV 20) scoped per `ProducerTypeID`.
- **One StopManager instance implements one coherent initial-geometry
  policy for one expected EntryGate candidate type** — same "one concept
  per instance" rule as every other layer in this framework.

**Identity validation order** (TradePol layer, unchanged):

```text
SchemaVersion -> ComponentKind -> PolicyCategory -> ProducerTypeID
  -> PayloadSchemaVersion -> payload
```

------------------------------------------------------------------------

# 2. Decision-cycle applicability and generic outcomes

Relevant when: a new active EntryGate `CandidateSeq` is available; the
candidate is unexpired; authoritative lifecycle state still permits
proposing geometry; the master decision cycle is active.

**Generic Trade Policy outcome** (same set EntryGate uses, same reasoning):

- **`NOT_APPLICABLE`** — nothing consumed this cycle at all.
- **`NO_ACTION`** — eligible, but no new candidate pending.
- **`POLICY_DECISION`** — a candidate was consumed and a geometry decision
  was produced: approved geometry, distance-invalid rejection, expired
  candidate, lifecycle-stale candidate, or undecidable (section 3). A
  rejected/invalid-distance geometry decision is `POLICY_DECISION`, never
  `NO_ACTION` — the same "a real decision about a consumed candidate isn't
  a no-op" rule already established elsewhere in this framework.

**`NO_CHANGE` is unused** (geometry decisions are discrete per candidate).
**`PROPOSED_ACTION` is never emitted by StopManager** — its output remains
incomplete until the Entry Composer assembles it with the matching
`EntryCandidate`.

------------------------------------------------------------------------

# 3. StopManager-specific decisions

```cpp
enum K_CHAKRA_STOPMANAGER_GEOMETRY_DECISION
{
    K_CHAKRA_STOPMANAGER_GEOMETRY_DECISION_NONE               = 0,
    K_CHAKRA_STOPMANAGER_GEOMETRY_DECISION_GEOMETRY_APPROVED  = 1,
    K_CHAKRA_STOPMANAGER_GEOMETRY_DECISION_REJECT_DISTANCE_INVALID = 2,
    K_CHAKRA_STOPMANAGER_GEOMETRY_DECISION_SKIP_EXPIRED       = 3,
    K_CHAKRA_STOPMANAGER_GEOMETRY_DECISION_SKIP_LIFECYCLE     = 4,
    K_CHAKRA_STOPMANAGER_GEOMETRY_DECISION_SKIP_UNDECIDABLE   = 5
};
```

| Case | `DecisionType` |
|---|---|
| Valid geometry computed | `GEOMETRY_APPROVED` |
| Distance below minimum | `REJECT_DISTANCE_INVALID` |
| Distance above maximum | `REJECT_DISTANCE_INVALID` — disambiguated by `ReasonCode` |
| Computed stop lands on the wrong side of entry (construction failure) | `REJECT_DISTANCE_INVALID` — disambiguated by `ReasonCode` |
| Candidate expired | `SKIP_EXPIRED` |
| Lifecycle changed since EntryGate accepted | `SKIP_LIFECYCLE` |
| Required dependency unavailable/stale | `SKIP_UNDECIDABLE` |
| Invalid instrument tick metadata | `SKIP_UNDECIDABLE` |
| Nonfinite arithmetic | `SKIP_UNDECIDABLE` |
| Unhandled context combination | `SKIP_UNDECIDABLE` |

------------------------------------------------------------------------

# 4. `EntryCandidate` consumption

Standard event-pattern candidate consumption, the same discipline every
TradePol sub-component in this framework applies:

- **Gate**: `CandidateStatus != NONE && CandidateSeq !=
  LastConsumedCandidateSeq`.
- **Identity validation first**: `SchemaVersion -> ComponentKind =
  TRADEPOL -> PolicyCategory = ENTRY_GATE -> expected EntryGate
  ProducerTypeID -> PayloadSchemaVersion -> candidate record`.
- **Lifecycle baselining, all four boundaries** (attachment, rewiring,
  StopManager's own recalc, StopManager's own disable/re-enable) — never
  consume on the same evaluation used to baseline.
- **Default consume-and-drop.**
- **Gap detection captures the prior sequence before mutation**, branching
  forward-gap-logged versus backward-movement-logged-as-probable-storage-
  reset.
- **Candidate source and Trigger event provenance retained**, carried
  forward into StopManager's own decision record (section 10).
- **`CandidateResolvedSide` must be exactly `LONG` or `SHORT`** — preserved,
  never re-resolved.
- **`CandidateValidUntilDateTime` checked against the master cycle's own
  evaluation time.**

**A consumed candidate always produces exactly one deterministic geometry
decision record, including every failure case.**

------------------------------------------------------------------------

# 5. Lifecycle revalidation

**StopManager independently revalidates authoritative lifecycle state
immediately before proposing geometry — it does not trust EntryGate's
earlier check**, the same "independent re-check at every stage" rule this
framework applies throughout. **Source of flat/working-entry state:
platform order state, read directly, never `K_Util_Candle_Baton_Manager_V1`**,
which has no pre-fill visibility at all. **"Pending" (already composed
and in flight further downstream) is explicitly not one of these checks —
the platform has no concept of an unsubmitted Chakra-internal proposal, so
there is nothing to query. There is no idempotency check anywhere in the
v1 live path** (no Entry Authority exists) — Mode Baton's own existing,
inherited lifecycle/duplicate-signal handling, not designed here, governs
once the Mode Baton Signal Publisher commits.

- Still flat, where required.
- No working entry order.
- No opposing exposure.
- Trade synchronization state is usable.
- The candidate remains unexpired.

```text
consume candidate
  -> POLICY_DECISION
  -> SKIP_LIFECYCLE or SKIP_EXPIRED
  -> no geometry output
  -> log reason
```

**The Entry Composer must independently revalidate again**, immediately
before its own use — not a reuse of StopManager's own revalidation result.
This chain of independent re-checks, one per consumer, is the same
discipline already required throughout this pipeline (EntryGate section
11).

------------------------------------------------------------------------

# 6. Indicative reference price — never an invented fill price

**StopManager never knows, invents, or promises an actual entry fill
price.** Before entry, no fill exists yet, and no Chakra component in the
v1 live path ever learns the real fill after the fact either (see
"Changes from Revision 9" and section 9, below) — StopManager's reference
price is purely indicative, at the moment of its own evaluation, with no
downstream reconciliation obligation resting on it.

- **`ReferencePrice`** is an **indicative** price used *as of the moment
  StopManager evaluates* (e.g. current market price, or a price the
  Trigger itself carries) — explicitly not a promise of what the eventual
  fill will be. It exists solely to give the geometry calculation a
  concrete anchor to measure distance from.
- **`StopPrice`** is computed as `ReferencePrice` offset by the stop
  distance, in the direction appropriate to the side (below for `LONG`,
  above for `SHORT`), then risk-safe rounded (section 7).
- **What happens when the actual fill differs from `ReferencePrice`** is
  addressed explicitly in section 9 — not ignored, and not solved by
  StopManager pretending its reference was exact.

------------------------------------------------------------------------

# 7. Risk-safe tick rounding

**Long stops and short stops must round in the direction that does not
understate risk — always away from entry, never toward it.**

**Rounding operates on the stop *price*, never on a fractional tick count
directly.** A fractional tick count (e.g. "12.3 ticks") has no valid
whole-tick rounding target of its own — rounding it to another fractional
value (e.g. "12.5 ticks") produces a number that still doesn't correspond
to any tick on the instrument's grid. The only quantity that can actually
be rounded to the tick grid is a *price*. The required operation order:

1. Compute the ideal (unrounded) stop price from `ReferencePrice` and the
   ideal (unrounded, possibly fractional) stop distance.
2. Round that **price** to the instrument's tick grid, in the direction
   that widens the stop, away from entry: **floor** for `LONG` (push the
   price further below entry), **ceiling** for `SHORT` (push the price
   further above entry).
3. **Recompute** the actual whole-tick `StopDistanceTicks` from the
   rounded price (`|RoundedStopPrice - ReferencePrice| / TickSize`,
   which is now guaranteed to be a whole number). This recomputed,
   whole-tick distance — never the pre-rounding fractional distance — is
   what downstream consumers (the Entry Composer) use.

Concretely, by side:

- **`LONG`** (stop below entry): a *larger* distance means a *lower* stop
  price. Round the stop price **down** (further below entry, away from
  entry) — never up (which would tighten the stop and understate distance).
- **`SHORT`** (stop above entry): a *larger* distance means a *higher*
  stop price. Round the stop price **up** (further above entry, away from
  entry) — never down.

**Worked example**: reference price 4500.00, ideal (unrounded) stop
distance 12.3 ticks on a 0.25-tick instrument, `SHORT` position. Ideal stop
price = 4500.00 + (12.3 x 0.25) = 4503.075. Round that **price** up
(section 7, away from entry) to the next valid tick at or above 4503.075,
which is **4503.25**. Recompute the whole-tick distance from the rounded
price: (4503.25 - 4500.00) / 0.25 = **13 ticks**. `StopPrice = 4503.25`,
`StopDistanceTicks = 13` — a whole number, and never less than the ideal
12.3 ticks. The equivalent `LONG` computation floors the price instead.

**This is pure price-grid geometry — no currency conversion of any kind is
involved anywhere in this section**, and nothing about it changed in
Revision 5's removal of currency-value-per-tick fields (those fields were
never used here; see "Changes from Revision 4").

------------------------------------------------------------------------

# 8. Validation

Before a geometry decision may be `GEOMETRY_APPROVED`, all of the
following are checked, each mapping to a distinct `SKIP_UNDECIDABLE`/
`REJECT_DISTANCE_INVALID` reason if it fails (section 13):

- **Minimum stop distance** — computed distance must be at or above a
  documented, code-versioned minimum.
- **Maximum stop distance** — computed distance must be at or below a
  documented, code-versioned maximum.
- **Finite arithmetic** — no NaN/Inf anywhere in the calculation.
- **Correct side of entry** — a `LONG` stop must land strictly below
  `ReferencePrice`; a `SHORT` stop must land strictly above it. A
  construction bug that produces a stop on the wrong side must never be
  silently accepted — this is a sanity check, not a business rule, and a
  failure here means `REJECT_DISTANCE_INVALID` (`ReasonCode =
  WRONG_SIDE_OF_ENTRY`, consistent with sections 3 and 13), not a quiet
  clamp and not `SKIP_UNDECIDABLE` — the geometry was constructible and
  evaluated, it just failed validation.
- **Tick size** (narrowed in Revision 5 — tick *value*/currency-per-tick is
  no longer read or validated anywhere in this document, see "Changes
  from Revision 4") — instrument tick-size metadata must be present and
  finite/positive before it's used in any calculation, read directly from
  the manually configured chart/instrument this StopManager instance is
  wired to.

**Not "instrument identity matches the candidate's own instrument"
(corrected on review) — `EntryCandidate` publishes no instrument field to
match against.** StopManager's tick size comes from its own manually
configured wiring (section 1), the same instrument the EntryGate feeding
it is also wired to by construction — there is nothing to compare at
runtime, only a self-contained value to read and use.

------------------------------------------------------------------------

# 9. Reference-vs-actual-fill reconciliation — not designed, not required, for v1

**Retired to a short, honest statement in Revision 10** (see "Changes from
Revision 9"). Revisions 1 through 9 built an increasingly detailed
pre-fill/post-fill reconciliation policy and a hard gate requiring that
policy to exist before StopManager's geometry could become part of an
executable entry. **That entire design assumed a greenfield Order
Management/Entry Authority component would eventually implement it. It
is not being built** (`Chakra_OrderManagement_Handoff_Interface_Contract.md`'s
own superseded banner) — carrying the policy forward as "resolved
elsewhere" would misdescribe the v1 live path.

**What actually happens in the v1 live path**: StopManager publishes an
indicative-reference-based geometry once (section 6); the Entry Composer
performs its own pre-publish drift check
(`Chakra_TradePol_EntryComposer_Interface_Contract.md` section 7) before
forwarding to the Mode Baton Signal Publisher; the publisher writes Mode
Baton's signal record; **Mode Baton then executes and manages the trade
using its own existing, tested behavior, which — per direct source audit
of `K_Util_ModeBaton_V1.cpp` — performs no reconciliation between its own
entry reference and the actual fill.** No Chakra component reconciles a
fill against `ReferencePrice`/`StopPrice` after the fact. This is a
deliberately inherited gap, not an oversight this document is failing to
close — see the Mode Baton Signal Publisher contract's own non-goals list.

**There is no hard gate.** StopManager's geometry becomes usable by the
Entry Composer as soon as it is `GEOMETRY_APPROVED` and correlated
(section 3, section 10) — nothing downstream is required to exist first.

**StopManager's own responsibility remains exactly what section 6 already
states**: publish an honest, indicative-reference-based geometry. There is
no reconciliation obligation for anything to own, downstream or otherwise,
in the v1 live path.

------------------------------------------------------------------------

# 10. Two separate output records

**The same two-record separation established for EntryGate, applied
here.**

## Decision record

**`GeometryDecisionSeq`** increments for every geometry decision —
approved, rejected, or any skip alike. Audit/`DecisionFrame` record, not
the delivery key. **Revised in Revision 5**: `StopDistanceCurrencyPerContract`
and `TickValue` are removed — both existed solely for PositionSizer's
now-deleted risk-based sizing methods.

| Field | Namespace/PV | Purpose |
|---|---|---|
| `PayloadSchemaVersion` | Int32 PV 20 | Envelope convention |
| `PolicyCategory` | Int32 PV 21 | Section 1 |
| `DecisionType` | Int32 PV 22 | Section 3 |
| `ResolvedSide` | Int32 PV 23 | Preserved from the candidate |
| `SourceEntryGateChartNumber` | Int32 PV 24 | Provenance |
| `SourceEntryGateStudyID` | Int32 PV 25 | Provenance |
| `SourceEntryGateProducerTypeID` | Int32 PV 26 | Provenance |
| `SourceTriggerEventType` | Int32 PV 27 | Carried forward |
| `StopDistanceTicks` | Int32 PV 28 | After risk-safe rounding (section 7), whole ticks |
| `GeometryDecisionSeq` | Int64 PV 10 | The audit sequence |
| `SourceCandidateSeq` | Int64 PV 11 | Which EntryGate candidate |
| `SourceTriggerEventSeq` | Int64 PV 12 | Provenance, carried forward |
| `MasterCycleSeq` | Int64 PV 13 | Which decision cycle |
| `SourceCandidateValidUntilDateTime` | Datetime PV 10 | Propagated |
| `SourceTriggerAvailabilityDateTime` | Datetime PV 11 | Provenance |
| `RawStopDistanceTicks` | Float PV 1 | Pre-rounding, may be fractional |
| `ReferencePrice` | Float PV 2 | Section 6 — indicative only |
| `StopPrice` | Float PV 3 | Post-rounding |
| `TickSize` | Float PV 5 | Instrument metadata — price-grid geometry only |

## Stop-geometry delivery record

**`StopGeometrySeq` increments only when `GEOMETRY_APPROVED`** — never on
`REJECT_DISTANCE_INVALID`/`SKIP_*`. This is the correctness-critical
delivery interface the **Entry Composer** consumes — its sole consumer —
structurally identical to EntryGate's `CandidateSeq` — **not
`PublicationSeq`, not `GeometryDecisionSeq`.** A shared sequence would let
a later rejection/skip for a *different* candidate silently erase an
already-approved, not-yet-consumed geometry.

Self-contained sticky record. **Revised in Revision 5**:
`StopGeometryDistanceCurrencyPerContract` and `StopGeometryTickValue` are
removed.

| Field | Namespace/PV | Purpose |
|---|---|---|
| `StopGeometryStatus` | Int32 PV 29 | `NONE = 0` / `ACTIVE = 1` |
| `StopGeometryResolvedSide` | Int32 PV 30 | Self-contained copy |
| `StopGeometryDistanceTicks` | Int32 PV 31 | Self-contained copy, post-rounding |
| `StopGeometrySourceEntryGateChartNumber` | Int32 PV 32 | Self-contained copy — part of the composite correlation identity |
| `StopGeometrySourceEntryGateStudyID` | Int32 PV 33 | Self-contained copy — composite correlation identity |
| `StopGeometrySourceEntryGateProducerTypeID` | Int32 PV 34 | Self-contained copy — composite correlation identity |
| `StopGeometrySeq` | Int64 PV 14 | The delivery sequence |
| `StopGeometrySourceDecisionSeq` | Int64 PV 15 | Links back to the `GeometryDecisionSeq` that produced it |
| `StopGeometrySourceEntryGateCandidateSeq` | Int64 PV 16 | Self-contained copy |
| `StopGeometryValidUntilDateTime` | Datetime PV 12 | Section 11 |
| `StopGeometryReferencePrice` | Float PV 7 | Self-contained copy |
| `StopGeometryStopPrice` | Float PV 8 | Self-contained copy |
| `StopGeometryTickSize` | Float PV 10 | Self-contained copy — price-grid geometry only |

**Why the composite EntryGate identity is required here, not just the bare
`StopGeometrySourceEntryGateCandidateSeq`:** `CandidateSeq` alone is not a
globally safe correlation key, since two different EntryGate instances
can independently arrive at the
same numeric value after a rewiring event or producer reconstruction. The
Entry Composer, correlating `StopGeometry` against a candidate, needs the
full composite identity — chart/study/`ProducerTypeID` plus
`CandidateSeq` — and that identity must live in this self-contained
delivery record, not only in the decision record's own, overwriteable
provenance fields (`SourceEntryGateChartNumber`/`StudyID`/
`ProducerTypeID`, section 10's decision-record table), which may already
describe a later, unrelated decision by the time a downstream consumer
reads them.

**What is actually required downstream is the self-contained tick size and
the resulting geometry (`StopGeometryTickSize`, `StopGeometryDistanceTicks`,
`StopGeometryStopPrice`, all already in the table above) — not a separate
instrument identity field (corrected on review, matching EntryGate's own
identical correction).** `StopGeometry` never claimed and does not need an
instrument-identity field to compare against anything; the value is read
directly from the manually configured chart/instrument this StopManager
instance is wired to (ordinary ACSIL instrument-metadata access), and a
downstream consumer needing to confirm consistency relies on that same
manual wiring, not a runtime comparison.

**Overwrite policy**: a later rejection/skip never overwrites an earlier
unconsumed approved geometry; a later *approval* (for a different
candidate) does overwrite, latest-approved-wins, with the downstream
consumer detecting a missed intermediate approval via sequence-gap
logging. No queued delivery.

**Downstream gate:**

```text
identityValid
  AND StopGeometryStatus == ACTIVE
  AND StopGeometrySeq != LastConsumedStopGeometrySeq
```

Lifecycle baselining, consume-and-drop, sequence-gap handling, expiry, and
storage-reset handling all apply, unmodified.

------------------------------------------------------------------------

# 11. Geometry expiry

**The expiry chain extends transitively:**

```text
StopGeometryValidUntilDateTime <= EntryCandidateValidUntilDateTime <= Trigger.EventValidUntilDateTime
```

StopManager cannot extend upstream actionability. If downstream processing
does not complete before `StopGeometryValidUntilDateTime`: consume/drop the
geometry, do not form a complete proposed action, log the expiry using the
detecting component's own reason code (section 13's self-referential
rule).

------------------------------------------------------------------------

# 12. [RETIRED in Revision 5] Former PositionSizer coordination note

**This section is retired, not renumbered away, so any existing
cross-reference lands somewhere meaningful rather than on a missing
section.** Its entire prior content — describing how StopManager's
parallel-with-PositionSizer timing contributed to (but did not solve)
PositionSizer's stop-distance/risk-sizing circularity, and pointing to
PositionSizer's own retry mechanism for the actual fix — is **deleted**,
not narrowed, because the premise no longer exists: PositionSizer's
Revision 6 removed every sizing method that depended on stop geometry, so
PositionSizer no longer consumes `StopGeometrySeq` at all, and there is
nothing left to coordinate.

**Current state, stated plainly (see "Pipeline position," above, for the
full picture):** StopManager and PositionSizer are independent, parallel
consumers of EntryGate's candidate with no dependency on each other in
either direction — PositionSizer is not part of the v1 live path at all
(see "Changes from Revision 9"). `StopGeometry` is consumed by the Entry
Composer alone, which correlates it against `EntryCandidate` by identity
(chart/study/`ProducerTypeID`/`CandidateSeq`), not by timing — there is no
retry, no coordination window, and no risk of a candidate being
permanently lost the way the old mechanism was built to prevent, because
nothing here gates on anything else's arrival.

------------------------------------------------------------------------

# 13. Diagnostics

**Revised in Revision 5**: `TICK_VALUE_SIZE_INVALID` renamed to
`TICK_SIZE_INVALID` (only tick size is validated now, see "Changes from
Revision 4").

```cpp
enum K_CHAKRA_STOPMANAGER_REASON_CODE
{
    K_CHAKRA_STOPMANAGER_REASON_NONE                       = 0,   // = K_CHAKRA_REASON_NONE
    K_CHAKRA_STOPMANAGER_REASON_NO_NEW_CANDIDATE            = 1,
    K_CHAKRA_STOPMANAGER_REASON_CANDIDATE_EXPIRED           = 2,
    K_CHAKRA_STOPMANAGER_REASON_CANDIDATE_SEQUENCE_GAP      = 3,
    K_CHAKRA_STOPMANAGER_REASON_CANDIDATE_STORAGE_RESET     = 4,
    K_CHAKRA_STOPMANAGER_REASON_ENTRYGATE_SCHEMA_MISMATCH   = 5,
    K_CHAKRA_STOPMANAGER_REASON_INVALID_SIDE                = 6,
    K_CHAKRA_STOPMANAGER_REASON_LIFECYCLE_STALE             = 7,
    K_CHAKRA_STOPMANAGER_REASON_ENTRY_ALREADY_PENDING       = 8,
    K_CHAKRA_STOPMANAGER_REASON_SENSOR_UNAVAILABLE_STALE    = 9,
    K_CHAKRA_STOPMANAGER_REASON_MCTX_UNAVAILABLE_STALE      = 10,
    K_CHAKRA_STOPMANAGER_REASON_INSTRUMENT_METADATA_INVALID = 11,
    K_CHAKRA_STOPMANAGER_REASON_TICK_SIZE_INVALID           = 12,
    K_CHAKRA_STOPMANAGER_REASON_DISTANCE_BELOW_MINIMUM      = 13,
    K_CHAKRA_STOPMANAGER_REASON_DISTANCE_ABOVE_MAXIMUM      = 14,
    K_CHAKRA_STOPMANAGER_REASON_WRONG_SIDE_OF_ENTRY         = 15,
    K_CHAKRA_STOPMANAGER_REASON_INVALID_NONFINITE_ARITHMETIC = 16,
    K_CHAKRA_STOPMANAGER_REASON_UNHANDLED_CONTEXT_COMBINATION = 17,
    K_CHAKRA_STOPMANAGER_REASON_GEOMETRY_APPROVED           = 18
    // extend per concrete StopManager policy as needed, numbered above these common ones
};
```

`..._INSTRUMENT_METADATA_INVALID`, `..._TICK_SIZE_INVALID`,
`..._SENSOR_UNAVAILABLE_STALE`, `..._MCTX_UNAVAILABLE_STALE`, and
`..._UNHANDLED_CONTEXT_COMBINATION` resolve to `DecisionType =
SKIP_UNDECIDABLE`. `..._DISTANCE_BELOW_MINIMUM`, `..._ABOVE_MAXIMUM`, and
`..._WRONG_SIDE_OF_ENTRY` resolve to `DecisionType =
REJECT_DISTANCE_INVALID`. **Log on decisions/transitions, not every master
bar.**

**Reference-vs-fill discrepancy handling (section 9) is diagnosed by the
detecting downstream component — never a member of this enum**, per the
self-referential `ReasonCode` principle already established (TriggerStr
section 18; EntryGate section 11's Revision 3 fix, which removed an
equivalent misplaced entry from EntryGate's own enum for exactly this
reason).

------------------------------------------------------------------------

# 14. Worked examples

**Renumbered in Revision 5** — the example describing PositionSizer's
retry mechanism (former example 8) is removed; nothing in this document
describes that flow anymore (see "Changes from Revision 4"). Eleven
examples remain, renumbered 1-11 with no gaps.

1. **Approved geometry, `LONG`.** Reference price 4500.00, computed
   distance 12 ticks (0.25 tick size, no rounding needed), stop price
   4497.00. `DecisionType = GEOMETRY_APPROVED`, `StopGeometrySeq`
   increments.
2. **Risk-safe rounding, `SHORT`.** Reference price 4500.00, ideal
   (unrounded) distance 12.3 ticks, 0.25-tick instrument. Ideal stop price
   = 4503.075. Round the **price** up (away from entry, section 7) to the
   next valid tick at or above it: `StopPrice = 4503.25`. Recompute the
   whole-tick distance from the rounded price:
   `(4503.25 - 4500.00) / 0.25 = 13 ticks` -> `StopDistanceTicks = 13`.
   The placed distance (13 ticks) is never less than the ideal 12.3 ticks.
3. **Reject — distance below minimum.** Computed distance 2 ticks against
   a documented minimum of 5. `DecisionType = REJECT_DISTANCE_INVALID`,
   `ReasonCode = DISTANCE_BELOW_MINIMUM`. No geometry delivered.
4. **Reject — distance above maximum.** Computed distance 400 ticks
   against a documented maximum of 100 (a sanity bound, not a risk
   preference). Same `DecisionType`, `ReasonCode =
   DISTANCE_ABOVE_MAXIMUM`.
5. **Candidate expired before geometry computed.**
   `consumer evaluation time > CandidateValidUntilDateTime`. `DecisionType
   = SKIP_EXPIRED`.
6. **Lifecycle stale.** A working entry order from a different path now
   exists (per section 5's corrected source: detected via direct platform
   order-state query, not the candle baton manager). `DecisionType =
   SKIP_LIFECYCLE`, `ReasonCode = ENTRY_ALREADY_PENDING`.
7. **Required dependency unavailable.** Instrument tick-size metadata
   isn't currently reachable. `DecisionType = SKIP_UNDECIDABLE`,
   `ReasonCode = TICK_SIZE_INVALID`. Never guessed.
8. **Reference-vs-fill discrepancy — not reconciled anywhere.** The
   actual fill lands materially away from `ReferencePrice`. StopManager's
   own published geometry record is untouched; per section 9, no Chakra
   component reconciles this — it is a deliberately inherited gap, not a
   remediation this document or any downstream one performs.
9. **Invalid — order action.** A StopManager that submits, modifies, or
   cancels an order, or that computes a position size. Either is a
   different component's job wearing StopManager's name.
10. **Overwrite policy.** A second `GEOMETRY_APPROVED` decision (for a
    different, later candidate) overwrites the sticky `StopGeometrySeq`
    record before the Entry Composer consumed the first — expected,
    latest-approved-wins, detected via sequence-gap logging on the
    consumer side.
11. **Geometry passes onward, still incomplete.** Example 1's approved
    geometry reaches the Entry Composer directly (see "Pipeline position,"
    above) — at no point does anything reach Mode Baton on its own; only
    the composed action (candidate + geometry), forwarded through the Mode
    Baton Signal Publisher, does.

------------------------------------------------------------------------

# Unresolved design questions — deliberately left open

**Renumbered in Revision 10** — former question 2 (reference-vs-fill
reconciliation ownership) is retired outright, not carried forward as
"resolved" (see "Changes from Revision 9" — it pointed at the superseded
Order Management design). **Former question 1 (post-entry/`TrailingManager`
ownership) is removed from this list entirely, not merely marked
"not a v1 blocker" — see the direct statement below**, which replaces it.
Only questions that affect real implementation. Envelope, Sensor, MCtx,
TriggerStr, EntryGate, and base-architecture decisions already settled are
not reopened here.

**Chakra v1 has no post-entry stop-management role, stated directly rather
than left as an open question.** The existing, unmodified Candle Baton
(`K_Util_Candle_Baton_Manager_V1`) owns everything once a trade is live —
trailing, break-even, protective ratcheting, all of it, exactly as it
already does today for Bhaskar-originated trades. This document's own
scope stops at initial, pre-entry geometry (the architecture decision,
above) and nothing in the v1 live path needs it to do anything else. If a
future initiative ever gives Chakra an open-position role — a `StopManager`
second phase, a `TrailingManager`, or otherwise — that is new scope
requiring its own design pass, not an unresolved question sitting in this
contract today.

1. **Is one StopManager instance restricted to one EntryGate source?**
   Mirrors EntryGate's own open question about single-source restriction —
   not stress-tested here either.

This remains genuinely open, and does not block the v1 live path. It is
not silently resolved by giving StopManager responsibilities belonging to
the Entry Composer or account/portfolio risk management.
