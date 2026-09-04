# Chakra — Trade Policy: PositionSizer Interface Contract (Draft, Revision 7)

> **SUPERSEDED/DEFERRED for the version-one live path (2026-07-25).** A
> source audit of `K_Util_ModeBaton_V1.cpp` confirmed that Mode Baton's
> `justTakeTheTrade == 25` ("Simple Baton") entry path — the mode Chakra's
> Mode Baton Signal Publisher targets — never reads the published quantity
> field (`PV53`); position size is controlled entirely by the consuming
> Mode Baton instance's own local configuration
> (`effectPositionSizePV`/its own "Position Size" input). **This document
> is not an implementation blocker for the version-one Chakra-to-Mode-Baton
> integration** — see `Chakra_ModeBatonSignalPublisher_Interface_Contract.md`
> section 8, which writes `PV53 = 0` explicitly rather than publish a
> quantity Mode Baton will ignore. This document's own design is preserved
> unmodified below; it becomes relevant again only if a future integration
> path is built where the downstream consumer actually honors a published
> quantity.

Status: **design only, no ACSIL implementation yet.**

Builds directly on:
- `Algo_Overlay_Framework_Architecture.md`, including the master decision
  cycle, `POLICY_DECISION`, `PROPOSED_ACTION`, and bounded `DecisionFrame`
  rules.
- `Chakra_Common_Interface_Envelope_Contract.md` (approved, Revision 8).
- `Chakra_Sensor_Interface_Contract.md` (Revision 4).
- `Chakra_MCtx_Interface_Contract.md` (Revision 5).
- `Chakra_TriggerStr_Interface_Contract.md` (Revision 3).
- `Chakra_TradePol_EntryGate_Interface_Contract.md` (Revision 3).

**This document designs only PositionSizer.** `StopManager`,
`ProfitManager`, `TrailingManager`, the Entry Composer's own
implementation, and physical `DecisionFrame` storage are explicitly out of
scope.

**Core responsibility, frozen (reworded in Revision 6 — see "Changes from
Revision 5" — to drop "account" now that no retained method reads account
state):**

> PositionSizer consumes one valid accepted `EntryCandidate` and determines
> the permitted entry quantity for that candidate, using only its declared
> sizing rules, current lifecycle state, and required Sensor/MCtx facts.

**PositionSizer must not:**

- Accept or reject the original trigger setup on market-quality grounds
  (that already happened — EntryGate).
- Change entry direction.
- Choose entry price.
- Choose stop or target prices.
- Submit orders.
- Manage an existing position.
- Create a complete manager-ready entry action.
- Arbitrate among unrelated accepted candidates, unless a later
  architecture decision explicitly assigns that responsibility (open
  question 2).
- **Perform monetary or account-level risk calculations** — read account
  equity, compute margin or buying power, compute current or historical
  monetary exposure, or reconstruct historical equity (new in Revision 6).
  **Chakra does not own that responsibility; it does not exist inside this
  component, or anywhere else in Chakra.** See "Changes from Revision 5."

Naming: `K_Chakra_TradePol_PositionSizer_<DescriptiveName>_V<n>`.

**PositionSizer is event/decision-style, like EntryGate and TriggerStr.**
It reuses the same machinery structurally, extended one layer further:
where EntryGate needed two separate sequences (decision audit trail vs.
candidate delivery), PositionSizer needs the same two-record separation
applied to *its own* output — see section 11.

------------------------------------------------------------------------

# Changes from Revision 1

Three material issues from a review pass — none required reopening the
stop-distance circularity, the audit/delivery separation, lifecycle
revalidation, or commit ordering, all of which were confirmed correct:

1. **The sticky `SizedCandidate` slot had an unstated overwrite policy for
   the approval-after-approval case.** Revision 1 explicitly protected
   against a later rejection overwriting an earlier unconsumed approved
   candidate, but never said what happens when a *second approval* arrives
   before the first is consumed — the sticky-single-slot mechanism would
   silently overwrite it, with no stated policy and no worked example
   testing that actual case (worked example 8 only tested
   rejection-after-approval, the easier case). Fixed: **latest-approved-
   candidate-wins is the explicit default**, with the downstream consumer
   detecting a missed intermediate approval via the same sequence-gap
   logging already established for every other event-style sequence in
   this framework — matching this framework's established semantics
   rather than leaving it implicit. A specific policy may instead choose
   to refuse/defer a new approval while one remains active, but only as an
   explicit, documented exception; queued delivery remains out of scope as
   unneeded complexity. See section 11 and worked example 6.
2. **Fixed replay capital was described as satisfying determinism in a way
   that blurred into "reproduces live sizing," which it does not.** A
   constant equity assumption is deterministic and reproducible run-to-run,
   but it models a *hypothetical* constant-equity scenario, not what live
   sizing actually would have done with genuinely fluctuating equity —
   those are different claims, and Revision 1 didn't clearly separate
   them. **Fully obsolete as of Revision 6** — no retained sizing method
   reads equity at all, so historical/replay sizing is deterministic by
   construction and this three-way distinction no longer applies. See
   "Changes from Revision 5" and section 14.
3. **Risk-specific fields had no neutralization rule, risking stale sticky
   values.** `RiskBudget`/`ComputedRisk` (decision record) and
   `SizedCandidateRiskBudget`/`SizedCandidateComputedRisk` (delivery
   record) were described only as present "where applicable" — with no
   stated value when not. **Fully obsolete as of Revision 6** — these
   fields, and the sizing methods that populated them, no longer exist.
   See "Changes from Revision 5" and section 11.

------------------------------------------------------------------------

# Changes from Revision 2

**Fully obsolete as of Revision 6 — see "Changes from Revision 5."** This
revision fixed a real operational bug in the stop-distance/risk-sizing
coordination mechanism (the "parallel pipeline" silently lost every
stop-dependent candidate). Preserved below only as an accurate historical
record of a bug that was real *at the time*, against machinery that no
longer exists — the underlying stop-dependent sizing methods, and the
coordination mechanism this revision fixed, were deleted in Revision 6
because no remaining PositionSizer method depends on stop geometry.

One material, architecture-level issue from a review of the paired
StopManager contract, which exposed a real operational bug in this
document rather than in StopManager's own:

1. **The "parallel pipeline" as specified in Revision 2 silently loses
   every stop-dependent candidate — a real bug, not a hypothetical.**
   Section 6 said a stop-dependent PositionSizer reports
   `SKIP_UNDECIDABLE`/`STOP_DISTANCE_UNAVAILABLE` when no geometry exists
   yet, and section 9's generic dependency-failure rule is
   consume-and-drop. Applied literally to *this* case, the candidate is
   permanently gone the instant PositionSizer evaluates even a moment
   before `StopManager` publishes matching geometry — which, given the two
   run independently against the same candidate, is close to the *typical*
   case, not a rare race. A stop-dependent PositionSizer configured this
   way would essentially never successfully size anything. Section 12 of
   the StopManager contract overclaimed this as "a real, working
   resolution for the common case" — it was not.

   **Fixed by choosing a real coordination rule** (not both options
   offered — one, deliberately): **PositionSizer retains an unconsumed
   candidate in an implicit pending state — by simply not yet advancing
   `LastConsumedCandidateSeq` — until matching geometry arrives or the
   candidate itself expires**, for stop-dependent sizing methods
   specifically. This is the pre-existing, already-established exception
   to the general consume-and-drop default (envelope contract: "a specific
   consumer's own contract pass may deliberately choose retry semantics
   instead, but must do so explicitly and for a stated reason") — applied
   here because dropping this candidate isn't protecting against replaying
   something stale, it is actively destroying the only chance to size a
   still-valid one. The alternative option (`StopManager` guaranteed to run
   before PositionSizer within the same master cycle, e.g. via ACSIL
   `CalculationPrecedence`) was considered and *not* chosen — it would
   still leave a gap for the case where `StopManager` itself doesn't
   succeed on the first attempt, and it would require asserting an
   architecture-level same-cycle ordering guarantee this framework has
   deliberately avoided committing to elsewhere.

   **Also fixed, folded into the same section**: PositionSizer must
   correlate geometry to the *specific* candidate it is sizing —
   `StopGeometrySourceEntryGateCandidateSeq == the CandidateSeq currently
   being sized` — never accept "any current geometry" merely because it's
   freshly published, since `StopGeometrySeq`'s single-slot latest-wins
   design means the sticky record could belong to a *different*,
   unrelated candidate by the time PositionSizer reads it. And: consuming
   `StopGeometrySeq` is an **event-pattern** read (sequence-gated,
   candidate-correlated), not a state-pattern read the way Sensor/MCtx
   facts are.

------------------------------------------------------------------------

# Changes from Revision 3

One upstream correction, surfaced by the Entry Composer's own contract
review rather than by a review of this document directly — flagged
explicitly and fixed at the smallest necessary scope, not silently worked
around downstream:

1. **`SizedCandidateSourceEntryGateCandidateSeq` alone is not a globally
   safe correlation key.** The Entry Composer needs to verify that a
   `SizedCandidate` and a `StopGeometry` record actually came from the
   *same* EntryGate candidate before assembling a proposal from them — but
   a bare sequence-number comparison can't do that safely: after a
   rewiring event or a producer reconstruction, two different EntryGate
   instances can independently arrive at the same numeric `CandidateSeq`.
   Comparing only the numbers risks silently combining records that came
   from different producers. Fixed: the sized-candidate delivery record
   (section 11) now also carries the composite EntryGate identity —
   `SizedCandidateSourceEntryGateChartNumber`/`...StudyID`/
   `...ProducerTypeID` alongside the existing
   `SizedCandidateSourceEntryGateCandidateSeq` — so a downstream consumer
   can correlate on the full identity, not the bare sequence number. This
   fix is about correlating *which candidate* a sized output came from,
   independent of which sizing method produced it, and is unaffected by
   Revision 6's removal of stop-dependent sizing.

------------------------------------------------------------------------

# Changes from Revision 4 (user-directed scope correction, part 1, 2026-07-19 — SUPERSEDED, see Revision 5 below)

**Superseded.** This revision's own text claimed `FIXED_MONETARY_RISK`/
`PERCENT_EQUITY_RISK` were "proposal-local sizing, not account-wide risk
management" and preserved them. **The user corrected this directly in a
second cleanup pass: reading account equity for any sizing decision,
proposal-local or not, is still an account-risk calculation, and Chakra
does not perform those.** Revision 5 (below) removes both methods (plus
`ACCOUNT_TIER`/`VOLATILITY_RISK_UNIT`) outright. Preserved below only as
an accurate record of what Revision 4 actually said and why it needed
correcting — do not read the "What this explicitly does NOT change"
section below as still true.

**Not a review round — the same project-wide scope boundary applied
identically across every upstream TradePol contract in this pass (see
`Chakra_OrderManagement_Handoff_Interface_Contract.md`'s "Changes from
Revision 3" for the full statement).** Chakra is a trade-decision
framework; it does not own portfolio or account-level risk management.
The user configures the chart, instrument, and trade account manually, and
every Chakra component — including PositionSizer — operates within that
manually configured context. This cancels the previously planned
`Chakra_ExecutionContext_AccountInstrument_Interface_Contract.md`.

**What this narrowed:**
1. **Account/instrument *identity*** (section 9, section 11) no longer
   waits on a separate Chakra-level identity abstraction — it is
   established by the manually-configured chart/study this PositionSizer
   instance is wired to, the same wiring convention every other Chakra
   layer already uses. No new field shape is invented, and none is needed.
2. **Open question 5's "portfolio-level concern" framing is corrected** —
   a portfolio-level arbitration layer above independently-configured
   instances is excluded by this project's scope boundary, not an
   undesigned future extension; the concrete case that matters (multiple
   Composer sources feeding one Entry Authority instance) is now resolved
   in `Chakra_OrderManagement_Handoff_Interface_Contract.md` section 6.
   **(This point stands; it is not superseded.)**

**What this claimed it did NOT change (INCORRECT — see Revision 5):**
- ~~Every existing sizing method, including `FIXED_MONETARY_RISK` and
  `PERCENT_EQUITY_RISK`~~ — **wrong; removed in Revision 6.**
- The stop-distance-before-sizing circularity (section 6, as it existed at
  the time) — **also retired in Revision 6, since its premise (a
  stop-dependent sizing method existing at all) no longer holds.**

------------------------------------------------------------------------

# Changes from Revision 6 (independent review-correction pass, 2026-07-19)

**A real gap found on review, mirroring EntryGate's own "Changes from
Revision 6" fix.** Section 5's lifecycle revalidation listed "no
incompatible pending entry action (from this or another candidate path)"
as a platform-sourced check — but the platform has no concept of an
unsubmitted Chakra-internal proposal, so this was never actually
checkable the way it was described. Fixed: the bullet is removed;
overwrite-in-place (this document's own section 11 latest-wins policy)
already handles a later candidate superseding an earlier unconsumed one,
and an in-flight submission conflict is exclusively the Entry Authority's
own idempotency check (`Chakra_OrderManagement_Handoff_Interface_Contract.md`
section 8/10), never visible upstream. **Nothing about the corrected
source (platform, not the candle manager) from Revision 6 is reopened —
this fix only narrows what that platform-sourced check claims to cover.**

------------------------------------------------------------------------

# Changes from Revision 5 (user-directed scope correction, part 2, 2026-07-19)

**A direct user correction to Revision 4/5's own reasoning above — the
first pass did not go far enough.** The user restated the boundary
literally: Chakra does not perform monetary, account, or portfolio risk
management, full stop — including reading account equity to size one
trade. Manual chart/instrument/account/order configuration remains
acceptable, potentially permanently.

**Removed outright, not merely narrowed:**

1. **`K_CHAKRA_POSITIONSIZER_METHOD_FIXED_MONETARY_RISK`,
   `..._PERCENT_EQUITY_RISK`, `..._VOLATILITY_RISK_UNIT`** — all three
   were "stop-dependent" methods computing quantity from a risk budget
   (dollar amount or percent-of-equity) divided by a currency-converted
   stop distance. Deleted, not deprecated — nothing in this document
   computes a risk budget or reads account equity/exposure anymore
   (section 6, retired).
2. **`K_CHAKRA_POSITIONSIZER_METHOD_ACCOUNT_TIER`** — also removed, though
   not explicitly named by the user. Its own definition ("a lookup table
   keyed by account size") requires reading current account size/equity
   to select a tier, which is exactly the class of account query being
   removed — keeping the name while forbidding the read it depends on
   would leave a method that cannot actually be implemented under this
   document's own rules. If a future strategy wants tiered sizing from a
   value the user sets directly (not read from the platform), that is
   just a documented constant/wiring input under `CONTEXT_MULTIPLIER` or
   `FIXED_QUANTITY`, not a distinct method.
3. **Account-equity queries, margin/buying-power calculations, current or
   historical monetary-exposure calculations, maximum-monetary-exposure
   limits, and historical-equity reconstruction** (former sections 7, 8,
   9, 14) — deleted wherever their only purpose was feeding the
   now-removed risk-based methods. Nothing in the retained methods needs
   any of this.
4. **`currency-value-per-tick`-based sizing** — removed from section 7's
   documentation checklist and from the sized-candidate computation path.
   PositionSizer no longer converts a stop distance into a currency risk
   figure at all; that was exclusively in service of the now-removed
   risk-based methods.
5. **Section 6 (stop-distance-before-sizing circularity and its Revision
   3 coordination/retry machinery) is retired as obsolete, per explicit
   user instruction, not preserved for its own sake.** The premise
   required a sizing method that needs the stop distance before it can
   compute a quantity — **no remaining method does.** `FIXED_QUANTITY`,
   `CONTEXT_MULTIPLIER`, and `CAPPED_QUANTITY` (the three retained
   methods, below) are all stop-independent by construction, exactly as
   they always were under the old "stop-independent" heading. The section
   number is kept (now a short retirement notice) so cross-references from
   `StopManager`'s own contract are not left dangling; its own body content
   — the four candidate ordering resolutions, the coordination rule, the
   retry-via-non-advancement mechanism, the `StopGeometrySeq` event-pattern
   consumption, the `CANDIDATE_EXPIRED_AWAITING_GEOMETRY`/
   `STOP_DISTANCE_UNAVAILABLE` reason codes — is deleted, not narrowed.
   **This also removes former open question 1 entirely** (no ordering
   problem remains to be open about).
6. **Former open question 2 (which ACSIL account-equity source/metric to
   read) is now moot, not merely resolved** — there is no longer a method
   that reads equity, so there is nothing left to document a source for.
   Removed from the open-questions list rather than marked resolved.
7. **Former open question 3 (are runtime hard-risk limits configurable)
   is also removed** — "hard risk limits" as a distinct category from
   ordinary per-order quantity bounds no longer exists (section 8,
   retitled).

**Retained, and why these are genuinely different in kind:**
- **`FIXED_QUANTITY`, `CONTEXT_MULTIPLIER`, `CAPPED_QUANTITY`** — none of
  these reads account state, computes currency risk, or depends on stop
  geometry. They compute a quantity from a configured base size, an
  optional Sensor/MCtx-driven multiplier, and a configured hard maximum —
  the same category of computation EntryGate and every other Chakra layer
  already performs against Sensor/MCtx facts, not a risk engine wearing a
  different name.
- **A manually configured per-order maximum quantity** (section 8) — a
  simple upper bound the user sets directly, exactly like every other
  wiring/safety input in this codebase (e.g. `MaxManagedContracts` on the
  existing candle baton manager). **This is explicitly not an
  account-wide capacity system** — it bounds one order, it is not derived
  from equity or exposure, and it does not coordinate across instances or
  strategies.

**Corrected core responsibility (top of document) and the "PositionSizer
must not" list both updated to state this boundary directly, not just
imply it through absence.**

------------------------------------------------------------------------

# 1. Naming and identity

- **`ComponentKind`** is always `K_CHAKRA_KIND_TRADEPOL`.
- **`PolicyCategory`** is `K_CHAKRA_TRADEPOL_CATEGORY_POSITION_SIZER`
  (int32 PV 21) — the enum itself was already defined by the EntryGate
  contract (section 1 there) as a TradePol-layer-wide convention; this
  document does not redefine it, only uses its already-reserved value.
- **`ProducerTypeID`** is scoped to PositionSizer specifically (not shared
  with any other `PolicyCategory`, same reasoning as EntryGate section 1):
  ```cpp
  enum K_CHAKRA_TRADEPOL_POSITIONSIZER_TYPE
  {
      K_CHAKRA_TRADEPOL_POSITIONSIZER_TYPE_NONE                   = 0,
      K_CHAKRA_TRADEPOL_POSITIONSIZER_TYPE_FIXEDQTY_ORBREAKOUT    = 1,
      // one member per concrete named PositionSizer policy, assigned as each is built
  };
  ```
- **`PayloadSchemaVersion`** (int32 PV 20) scoped per `ProducerTypeID`,
  unchanged convention.
- **One PositionSizer instance implements one coherent sizing policy for
  one expected EntryGate candidate type** — not a generic "sizes anything"
  component. It is built against a specific, named EntryGate
  `ProducerTypeID` (section 4) and a specific, documented sizing method
  (section 7).

**Identity validation order (TradePol layer, unchanged from EntryGate
section 1):**

```text
SchemaVersion
  -> ComponentKind
  -> PolicyCategory
  -> ProducerTypeID
  -> PayloadSchemaVersion
  -> payload
```

------------------------------------------------------------------------

# 2. Decision-cycle applicability and generic outcomes

PositionSizer is relevant when all four hold:

- A new active EntryGate `CandidateSeq` is available.
- The candidate is unexpired.
- Authoritative lifecycle state still permits sizing (section 5).
- The master decision cycle is active (`Algo_Overlay_Framework_Architecture.md`).

**Generic Trade Policy outcome** (the architecture's five-value set;
PositionSizer uses the same three EntryGate uses, for the same reasons):

- **`NOT_APPLICABLE`** — no candidate source is configured for this
  lifecycle, or sizing is structurally irrelevant this cycle (e.g. this
  PositionSizer's bound EntryGate is itself not relevant this cycle). A
  true no-op — nothing consumed.
- **`NO_ACTION`** — eligible, but no new candidate is pending.
- **`POLICY_DECISION`** — a candidate was consumed and a sizing decision
  was produced: approved size, zero-size rejection, expired candidate,
  lifecycle-stale candidate, or undecidable (section 3). **Do not call a
  zero-size rejection `NO_ACTION`** — it is a real sizing decision about a
  genuinely consumed candidate, exactly the same distinction EntryGate's
  own contract already drew for `REJECT` versus `NO_ACTION`.

**`NO_CHANGE` is normally unused** — sizing decisions are discrete per
candidate, not a continuously-maintained recommendation. **`PROPOSED_ACTION`
is never emitted by PositionSizer** — its output remains structurally
incomplete (no stop, no target) and must pass through the Entry Composer
before anything is manager-ready, exactly as EntryGate's own output does.

------------------------------------------------------------------------

# 3. PositionSizer-specific decisions

```cpp
enum K_CHAKRA_POSITION_SIZER_DECISION
{
    K_CHAKRA_POSITION_SIZER_DECISION_NONE              = 0,
    K_CHAKRA_POSITION_SIZER_DECISION_SIZE_APPROVED      = 1,
    K_CHAKRA_POSITION_SIZER_DECISION_REJECT_ZERO_SIZE   = 2,
    K_CHAKRA_POSITION_SIZER_DECISION_SKIP_EXPIRED       = 3,
    K_CHAKRA_POSITION_SIZER_DECISION_SKIP_LIFECYCLE     = 4,
    K_CHAKRA_POSITION_SIZER_DECISION_SKIP_UNDECIDABLE   = 5
};
```

Required cases and their mapping (several deliberately share a
`DecisionType`, disambiguated by `ReasonCode` — section 16). **Revised in
Revision 6**: the "hard risk limit" and "invalid account/instrument
metadata" rows are removed — neither concept exists in this document
anymore (see "Changes from Revision 5").

| Required case | `DecisionType` |
|---|---|
| Positive approved quantity | `SIZE_APPROVED` |
| Computed quantity below minimum (rounds to zero) | `REJECT_ZERO_SIZE` |
| Per-order maximum quantity clamps the result to zero | `REJECT_ZERO_SIZE` — distinguished from the row above by `ReasonCode` |
| Candidate expired | `SKIP_EXPIRED` |
| Lifecycle changed since EntryGate accepted | `SKIP_LIFECYCLE` |
| Required Sensor/MCtx dependency unavailable/stale | `SKIP_UNDECIDABLE` |
| Arithmetic or nonfinite calculation failure | `SKIP_UNDECIDABLE` |
| Unsupported sizing method (misconfiguration) | `SKIP_UNDECIDABLE` |

------------------------------------------------------------------------

# 4. `EntryCandidate` consumption

Reuses EntryGate's own consumption discipline for TriggerStr events
(EntryGate contract, section 5), one layer further downstream, applied to
EntryGate's own candidate record instead:

- **The gate**: `CandidateStatus != NONE && CandidateSeq !=
  LastConsumedCandidateSeq`.
- **Identity validation before any candidate field is read**:
  ```text
  SchemaVersion
    -> ComponentKind = K_CHAKRA_KIND_TRADEPOL
    -> PolicyCategory = K_CHAKRA_TRADEPOL_CATEGORY_ENTRY_GATE
    -> expected EntryGate ProducerTypeID
    -> PayloadSchemaVersion
    -> candidate record
  ```
- **Lifecycle baselining, all four boundaries**: initial attachment to that
  EntryGate, rewiring, PositionSizer's own `IsFullRecalculation`, and
  PositionSizer's own disable -> re-enable. **Never consume the candidate
  on the same evaluation used to baseline** — baselining and consuming are
  never the same call, per the unified mechanism established since the
  envelope contract's Revision 6/7.
- **Default failure policy is consume-and-drop** (unconditionally, for
  every case in this document — the one narrow retry-via-non-advancement
  exception this document used to carve out for stop-dependent sizing no
  longer applies; see "Changes from Revision 5").
- **Gap detection captures the prior sequence before mutating
  `LastConsumedCandidateSeq`**, branching forward-gap-logged versus
  backward-movement-logged-as-probable-storage-reset (envelope Revision 7's
  corrected pattern) — never a raw arithmetic difference.
- **Candidate source provenance is retained**, not discarded: the source
  `DecisionSeq` that produced this candidate, and the Trigger event
  provenance EntryGate itself carried forward, propagate into
  PositionSizer's own decision record (section 11).
- **`CandidateResolvedSide` must be exactly `LONG` or `SHORT`** — EntryGate
  already guarantees this in its own output (never `BOTH`/`NONE` reaches
  the candidate record); PositionSizer does not need to re-resolve it, only
  preserve it (section 10).
- **`CandidateValidUntilDateTime` is checked against the master cycle's own
  evaluation time**, using the same live/historical "now" rules established
  throughout this framework.

**A consumed candidate must always produce exactly one deterministic
sizing decision record, including every failure case.** There is no
"consumed but produced nothing" state.

------------------------------------------------------------------------

# 5. Lifecycle revalidation

**PositionSizer must independently revalidate authoritative lifecycle
state immediately before sizing — it does not trust EntryGate's earlier
lifecycle check.** This is EntryGate's own Revision 3 rule (its section
11), restated here because it applies with equal force one stage further
downstream, for the identical reason: time passes between EntryGate's
acceptance and PositionSizer's evaluation, and the world can change in
that window.

**Source of this state (corrected in Revision 6 — see EntryGate's own
"Changes from Revision 5" for the full correction this mirrors; narrowed
further in Revision 7 — see EntryGate's own "Changes from Revision 6" for
the matching correction): pre-fill flat/working-order state is platform
order state, read directly** (ACSIL position/working-order queries for
the manually configured symbol/account), **never
`K_Util_Candle_Baton_Manager_V1`**, which has no visibility into anything
before a fill is confirmed and handed off to it — it cannot be, and is
never cited here as, an authoritative source for a pre-fill check. This is
a **preliminary, best-effort check**, exactly the same standing every
other pre-Entry-Authority stage's own lifecycle check has; the Entry
Authority performs the final, authoritative revalidation immediately
before submission (`Chakra_OrderManagement_Handoff_Interface_Contract.md`
section 8), including against its own private in-flight-submission marker
that no upstream stage can see.

**"Pending" (already composed and in flight at the Entry Authority, or
awaiting composition further downstream) is explicitly not one of these
checks (corrected in Revision 6 — a real gap in the original fix, which
corrected the source for flat/working-order state but not what "pending"
actually is) — the platform has no concept of an unsubmitted
Chakra-internal proposal, so there is nothing to query.** A later approval
overwriting an earlier unconsumed one is handled by this document's own
overwrite policy (section 11), not by a lifecycle block here; an
incompatible submission already in flight is the Entry Authority's own
idempotency check, immediately before submission (the section 8 reference
above).

Checked, at minimum:

- Still flat, where the candidate's entry requires flat.
- No working entry order.
- No opposing exposure.
- Trade synchronization state is usable.
- The candidate remains unexpired (time-based check, re-applied here too —
  time keeps passing between stages).

**If any check fails:**

```text
consume candidate
  -> POLICY_DECISION
  -> SKIP_LIFECYCLE or SKIP_EXPIRED
  -> no sized-candidate output
  -> log reason
```

**Restated for the future: the Entry Composer must perform another,
independent lifecycle check immediately before producing a manager-ready
proposal** — not a reuse of PositionSizer's own revalidation result. This
chain of independent re-checks, one per stage, is deliberate — see section
12.

------------------------------------------------------------------------

# 6. [RETIRED in Revision 6] Stop-distance/risk-sizing dependency

**This section is retired, not renumbered away, so existing
cross-references from `StopManager`'s own contract land somewhere
meaningful rather than on a missing section.** Its entire prior content —
the four candidate resolutions for ordering PositionSizer against
`StopManager`, the Revision 3 coordination/retry-via-non-advancement
mechanism, the `StopGeometrySeq` event-pattern consumption rule, and the
`CANDIDATE_EXPIRED_AWAITING_GEOMETRY`/`STOP_DISTANCE_UNAVAILABLE` reason
codes — is **deleted**, not narrowed, per explicit user instruction: "if no
remaining sizing method depends on stop distance, delete that ordering
problem and all associated retry/coordination machinery as obsolete."

**Why the premise no longer holds:** the circularity existed only because
this document once offered stop-dependent sizing methods
(`FIXED_MONETARY_RISK`/`PERCENT_EQUITY_RISK`/`VOLATILITY_RISK_UNIT`, all
removed in Revision 6 — see "Changes from Revision 5"). The three retained
methods (section 7) are all stop-independent by construction — none of
them has ever needed the initial stop distance to compute a quantity, so
there is no ordering question left to resolve. `StopManager` and
PositionSizer remain independent, parallel consumers of EntryGate's
candidate (unchanged), but PositionSizer no longer consumes anything
StopManager publishes.

If a future PositionSizer method ever needs stop geometry again, it would
need to re-derive a coordination mechanism from first principles (or
consult this section's Revision 5 history, preserved above in "Changes
from Revision 2," for what was tried and why it needed a retry-based
fix) — nothing here is preserved as a ready-to-reuse mechanism.

------------------------------------------------------------------------

# 7. Sizing methods, inputs, and units

**Retained sizing methods (Revision 6 — see "Changes from Revision 5" for
what was removed and why):**

```cpp
enum K_CHAKRA_POSITIONSIZER_METHOD
{
    K_CHAKRA_POSITIONSIZER_METHOD_NONE               = 0,
    K_CHAKRA_POSITIONSIZER_METHOD_FIXED_QUANTITY     = 1,
    K_CHAKRA_POSITIONSIZER_METHOD_CONTEXT_MULTIPLIER = 2,
    K_CHAKRA_POSITIONSIZER_METHOD_CAPPED_QUANTITY    = 3
    // one member per concrete named PositionSizer method; extend here as new
    // stop-independent, account-free methods are designed. Do not add a
    // method that requires reading account/equity/exposure state or the
    // initial stop distance -- see the "PositionSizer must not" list above.
};
```

- **Fixed quantity** — a documented, code-versioned constant.
- **Context multiplier applied to a fixed base** (e.g. `Globex context ->
  50% of base size`) — the base is a constant; the multiplier comes from a
  required Sensor/MCtx dependency (section 9), never account state.
- **Capped quantity** — a fixed quantity subject only to the per-order
  maximum (section 8).

Every concrete sizing policy must document, in writing:

- Sizing method (`K_CHAKRA_POSITIONSIZER_METHOD`, above).
- Input fields and their units.
- Lot/contract increment.
- Minimum allowed quantity.
- Maximum allowed quantity (the per-order maximum, section 8).
- Rounding direction.
- Behavior when the computed size is fractional.
- Hard platform/broker quantity limits (a platform constraint, not an
  account-risk calculation).
- How context modifies the base size.
- Finite/range validation for every numeric input (no NaN/Inf silently
  propagating into a sizing decision).

**Quantity output must be an integer number of valid tradable
units/contracts, after rounding and clamping.** No fractional contracts
ever leave PositionSizer.

**Never round upward past the configured per-order maximum.** Default
behavior is round **down** to the valid lot increment — the same "never
let a safety margin erode by rounding the wrong direction" principle
already embedded in this framework's other hard-limit guards (e.g. the
candle baton manager's ratchet-only-forward stop guard: when in doubt, the
safer direction wins).

------------------------------------------------------------------------

# 8. Per-order quantity limits versus policy rules

**Retitled in Revision 6** (was "Risk limits versus policy rules") — there
is no longer a distinct "risk limit" category; there is only an ordinary
per-order quantity bound, exactly like `MaxManagedContracts` on the
existing candle baton manager.

Two distinct categories, kept separate:

- **A manually configured per-order maximum quantity** — a hard upper
  bound the user sets directly. **This is explicitly not an account-wide
  capacity system**: it bounds a single order, it is never derived from
  account equity or exposure, and it never coordinates across instances,
  strategies, or symbols.
- **Policy sizing rules**: base size, context multipliers.

**The per-order maximum may reduce or reject a policy-computed quantity.
It must never increase it.** A policy rule computing 10 contracts against
a configured maximum of 6 yields 6 (or a rejection, per that policy's own
configuration) — never does the maximum push a smaller policy-computed
number upward.

**Versioning discipline** (base architecture's rule, MCtx's corrected
wiring/display/meaning-preserving-cadence boundary, applied identically
here):

- Sizing formula constants and context multipliers are code-versioned
  constants — never ACSIL `Input`s.
- Wiring/display inputs, and the per-order maximum quantity itself, may
  remain ACSIL `Input`s — it is a safety wiring value, not a strategy
  parameter, but nothing about it requires code-constant treatment the way
  a sizing formula does.
- Do not hot-edit sizing policy constants without a code commit.

------------------------------------------------------------------------

# 9. Sensor and MCtx dependencies

**Retitled in Revision 6** (was "Sensor, MCtx, and account dependencies")
— PositionSizer no longer has any account dependency to declare.

Each dependency declares, reusing the discipline already established
(MCtx section 4, EntryGate section 8) without modification:

- Required versus optional.
- Exact producer/schema identity.
- Fields consumed, and their units.
- Freshness tolerance PositionSizer itself chooses.
- Live state access via the state pattern.
- Historical subgraph access, if historical sizing decisions are plotted
  (section 14).
- Cross-chart `SourceDataAsOfDateTime`, not `SourcePeriodStartDateTime`,
  for any cross-chart dependency's freshness.
- Behavior when unavailable, stale, or invalid — immediate consume-and-drop
  is the only policy (below); there is no longer a retry exception (the
  one that used to exist here, for `StopGeometrySeq`, was removed with
  section 6).

**Policy-specific rule (PositionSizer's own version of the "truth
condition" rule already established for TriggerStr and EntryGate):**

> Any dependency capable of changing the approved quantity is required for
> that PositionSizer policy. Optional dependencies may enrich audit/
> provenance only. They cannot change quantity.

For required dependency failure:

```text
consume candidate
  -> SKIP_UNDECIDABLE
  -> no sized candidate
  -> log exact reason
```

------------------------------------------------------------------------

# 10. Direction and context handling

- **PositionSizer preserves the candidate's resolved side unchanged.** It
  does not re-derive or reinterpret direction.
- **It may produce different quantities for long and short only when the
  sizing policy explicitly documents why** — asymmetric sizing must be a
  stated design choice, never an accidental artifact of the calculation.
- **MCtx may influence magnitude, never direction or the accept/reject
  judgment already made by EntryGate:**
  ```text
  Globex context       -> 50% of base size
  High-volatility context -> reduced size
  ```
- **If the computed permitted quantity is zero, that is a sizing
  rejection — not a reinterpretation of whether the setup was desirable.**
  PositionSizer must never revisit EntryGate's `ACCEPT` judgment; a
  `REJECT_ZERO_SIZE` decision says "the sizing math doesn't support any
  size right now," never "I disagree that this was a good setup."

**Context reconciliation remains local and declarative**, exactly the
principle already established (base architecture's Conflict Resolution,
EntryGate section 9) — its third implementation in this framework:

- Exact context states considered.
- Precedence, where more than one rule could apply.
- `UNLESS` exceptions.
- Default behavior for unhandled combinations.
- No global conflict table, ever.

```text
Unhandled combination
    -> consume candidate
    -> SKIP_UNDECIDABLE
    -> no size output
    -> log contexts, candidate, cycle, and reason
```

------------------------------------------------------------------------

# 11. Two separate output records

**The same separation learned in EntryGate is required here — a general
audit sequence is unsafe as a correctness-critical delivery key.**

## Decision record

**`SizingDecisionSeq`** increments for **every** sizing decision —
`SIZE_APPROVED`, `REJECT_ZERO_SIZE`, `SKIP_EXPIRED`, `SKIP_LIFECYCLE`,
`SKIP_UNDECIDABLE` alike. This is the audit/`DecisionFrame` record, **not**
the delivery key for successful size output.

**Revised in Revision 6**: `RiskBudget`/`ComputedRisk` fields and their
neutralization rule are removed — no retained sizing method produces a
risk figure.

| Field | Namespace/PV | Purpose |
|---|---|---|
| `PayloadSchemaVersion` | Int32 PV 20 | Envelope convention |
| `PolicyCategory` | Int32 PV 21 | Section 1 |
| `DecisionType` | Int32 PV 22 | Section 3 |
| `ResolvedSide` | Int32 PV 23 | Preserved from the candidate, unchanged |
| `SourceEntryGateChartNumber` | Int32 PV 24 | Provenance |
| `SourceEntryGateStudyID` | Int32 PV 25 | Provenance |
| `SourceEntryGateProducerTypeID` | Int32 PV 26 | Provenance |
| `SourceTriggerEventType` | Int32 PV 27 | Carried forward from the candidate |
| `RoundedQuantity` | Int32 PV 28 | After rounding direction (section 7), before per-order-maximum clamp |
| `FinalPermittedQuantity` | Int32 PV 29 | After per-order-maximum clamping (section 8) — the decision's real output |
| `SizingMethodIdentifier` | Int32 PV 34 | `K_CHAKRA_POSITIONSIZER_METHOD` (section 7) |
| `SizingDecisionSeq` | Int64 PV 10 | The audit sequence |
| `SourceCandidateSeq` | Int64 PV 11 | Which EntryGate candidate this decision is about |
| `SourceTriggerEventSeq` | Int64 PV 12 | Provenance, carried forward further |
| `MasterCycleSeq` | Int64 PV 13 | Which decision cycle |
| `SourceCandidateValidUntilDateTime` | Datetime PV 10 | Propagated from the candidate |
| `SourceTriggerAvailabilityDateTime` | Datetime PV 11 | Provenance, carried forward further |
| `RawComputedQuantity` | Float PV 1 | Pre-rounding, may be fractional |

Provenance for the Sensor/MCtx states actually used follows the same
documented-per-instance convention already established (EntryGate section
10) — publish each consulted dependency's `ProducerTypeID` and the
`PublicationSeq` it was read at.

## Sized-candidate delivery record

**`SizedCandidateSeq` increments only when a strictly positive, valid size
is approved** — never on `REJECT_ZERO_SIZE`/`SKIP_*`. This is the
correctness-critical delivery interface to the Entry Composer stage — the
exact reasoning that produced EntryGate's `CandidateSeq` (its Revision 2
fix) applies identically one layer deeper: a shared sequence would let a
later rejection silently erase an already-approved, not-yet-consumed sized
candidate.

Self-contained sticky record. **Revised in Revision 6**:
`SizedCandidateRiskBudget`/`SizedCandidateComputedRisk` and their
neutralization rule are removed.

| Field | Namespace/PV | Purpose |
|---|---|---|
| `SizedCandidateStatus` | Int32 PV 30 | `NONE = 0` / `ACTIVE = 1` |
| `SizedCandidateResolvedSide` | Int32 PV 31 | Self-contained copy |
| `SizingMethodIdentifier` | Int32 PV 32 | `K_CHAKRA_POSITIONSIZER_METHOD` (section 7) |
| `SizedCandidateFinalQuantity` | Int32 PV 33 | Self-contained copy of the approved integer quantity |
| `SizedCandidateSourceEntryGateChartNumber` | Int32 PV 35 | Self-contained copy — part of the composite correlation identity |
| `SizedCandidateSourceEntryGateStudyID` | Int32 PV 36 | Self-contained copy — composite correlation identity |
| `SizedCandidateSourceEntryGateProducerTypeID` | Int32 PV 37 | Self-contained copy — composite correlation identity |
| `SizedCandidateSeq` | Int64 PV 14 | The delivery sequence |
| `SizedCandidateSourceDecisionSeq` | Int64 PV 15 | Links back to the `SizingDecisionSeq` that produced it |
| `SizedCandidateSourceEntryGateCandidateSeq` | Int64 PV 16 | Self-contained copy — which EntryGate candidate this came from |
| `SizedCandidateValidUntilDateTime` | Datetime PV 12 | Section 12 |

**Why the composite EntryGate identity is required here, not just the bare
`SizedCandidateSourceEntryGateCandidateSeq`:** `CandidateSeq` alone is not
a globally safe correlation key. After a rewiring event or a producer
reconstruction, two different EntryGate instances can independently reach
the same numeric `CandidateSeq` value — a downstream consumer correlating
on the bare number alone (the Entry Composer, in particular) could
silently combine records that came from *different* EntryGate producers
merely because their sequence numbers happen to coincide. The full
composite identity — `SizedCandidateSourceEntryGateChartNumber`/
`...StudyID`/`...ProducerTypeID` plus
`SizedCandidateSourceEntryGateCandidateSeq` together — is what actually
identifies "the same candidate," and must live in this self-contained
delivery record, not only in the decision record's own provenance fields
(`SourceEntryGateChartNumber`/`StudyID`/`ProducerTypeID`, section 11's
decision-record table) — those are overwriteable audit fields that may
already describe a *later*, unrelated decision by the time a downstream
consumer reads them, exactly the reasoning that required a separate
delivery record in the first place.

**Overwrite policy — two distinct cases, both explicit:**

- **A later rejected/skip sizing decision must not overwrite or invalidate
  an earlier unconsumed sized candidate.** `SizingDecisionSeq` moving
  forward on its own never touches `SizedCandidateSeq` or the sticky
  sized-candidate record.
- **A later *approved* sizing decision — a second `ACCEPT`-equivalent —
  overwrites the sticky sized-candidate record with the newer candidate.
  Latest-approved-candidate-wins is the explicit default policy here**,
  matching this framework's established semantics for every other
  single-slot sticky event record (TriggerStr's `EventSeq`, EntryGate's
  own `CandidateSeq`). If a downstream consumer hasn't consumed the first
  approval before a second one publishes, the first is gone — the
  consumer detects this via the same sequence-gap logging already
  required for every other event-style sequence in this framework
  (capture the previous sequence before mutating, log a forward gap
  greater than 1 as a dropped intermediate approval). A specific
  PositionSizer policy may instead choose to refuse or defer producing a
  new approval while an existing sized candidate remains active and
  unconsumed — but that must be an explicit, documented choice for that
  policy, not the default, and it is a form of the lifecycle-ineligibility
  handling already established (section 5), not a new mechanism. Queued
  delivery (preserving every approval) remains explicitly out of scope —
  unneeded complexity for this pass.

**Downstream gate** (for the Entry Composer, structurally identical to
every event-pattern consumer in this framework):

```text
identityValid
  AND SizedCandidateStatus == ACTIVE
  AND SizedCandidateSeq != LastConsumedSizedCandidateSeq
```

Lifecycle baselining, consume-and-drop, sequence-gap handling, expiry, and
storage-reset handling all apply, unmodified from the pattern already
established.

------------------------------------------------------------------------

# 12. Sized-candidate expiry and downstream revalidation

**The expiry chain is transitively bounded — each stage's deadline can
only tighten, never extend, what came before it:**

```text
SizedCandidateValidUntilDateTime
    <= EntryCandidateValidUntilDateTime
    <= Trigger.EventValidUntilDateTime
```

**PositionSizer cannot extend upstream actionability.** A sizing decision
that took long enough to compute that little time remains before
`EntryCandidateValidUntilDateTime` must set a correspondingly tight (or
tighter) `SizedCandidateValidUntilDateTime` — never a later one.

**The Entry Composer must independently revalidate**, immediately before
its own action (section 5's rule, restated one stage further):

- Expiry.
- Lifecycle state (platform order state, per section 5's corrected source
  — never the candle baton manager pre-fill).
- Still-flat requirement.
- No incompatible working entry (platform-visible; an in-flight
  submission conflict is the Entry Authority's own idempotency check, not
  a check any upstream stage can perform — section 5).
- **Quantity remains within the *current* per-order maximum** — a
  quantity valid when computed may no longer respect a maximum that has
  since been reconfigured (e.g. the user lowered the configured per-order
  cap between sizing and composition).

If stale:

- Consume-and-drop the sized candidate.
- Do not form a complete proposed action.
- Use the *detecting* component's own reason code — never PositionSizer's
  `ReasonCode` (section 16's self-referential rule).

------------------------------------------------------------------------

# 13. Publication and commit discipline

PositionSizer is decision/event-style. **Three sequences stay strictly
distinct — never conflated, never used as a substitute for one another:**

- **`PublicationSeq`** — any public snapshot revision (envelope-level,
  general-purpose, never an event/decision dedupe key — the lesson this
  entire framework keeps re-deriving).
- **`SizingDecisionSeq`** — every sizing decision, the audit trail.
- **`SizedCandidateSeq`** — only successful, positive sizing outputs, the
  correctness-critical delivery key.

**Commit order for a successful sizing decision:**

```text
envelope + decision payload + sized-candidate payload
    -> SizingDecisionSeq
    -> SizedCandidateSeq
    -> PublicationSeq
```

**Commit order for reject/skip:**

```text
envelope + decision payload
    -> SizingDecisionSeq
    -> PublicationSeq
```

**Reject/skip must never touch `SizedCandidateSeq` or the sticky
sized-candidate record.**

**Delivery records are cleared only at explicit disable-entry/recalc-entry
boundaries** — consistent with TriggerStr and EntryGate; nothing else ever
clears them.

------------------------------------------------------------------------

# 14. Full recalculation and replay

**Substantially simplified in Revision 6** — the entire "historical
account/equity-dependent sizing" discussion (fixed replay capital vs. a
genuine historical equity timeline vs. neither available) is removed. No
retained sizing method reads account or equity state at any time, live or
historical, so replay is deterministic and account-independent by
construction — there is nothing left to reconstruct or approximate.

- **No live `SizingDecisionSeq` or `SizedCandidateSeq` increments during
  full recalculation**, under any circumstance.
- **No orders or external actions**, recalc or not, live or replay.
- **Historical sizing-decision subgraphs may record what would have
  happened** — the same hidden Occurrence/Status subgraph pair pattern
  already established (Sensor Revision 3, TriggerStr section 16, EntryGate
  section 12), entirely separate from the persistent sequences above.
- **Historical accepted sizes are never live-deliverable candidates.**
- **Historical dependency access uses availability-indexed arrays and the
  three-part historical-validity check** — never the persistent payload,
  never array coverage alone.
- **Private sequence and lifecycle trackers baseline at all four
  boundaries** (attachment, rewiring, PositionSizer's own recalc,
  PositionSizer's own disable/re-enable).
- **Replay's forward evaluation uses the normal master-cycle path** — no
  special-casing beyond following the rules above correctly.

------------------------------------------------------------------------

# 15. `DecisionFrame` participation and logging

Record only what was actually consumed:

- Master cycle identity.
- The `EntryCandidate` actually consumed.
- The lifecycle snapshot actually relevant.
- The required Sensor/MCtx snapshots actually used.
- The generic outcome (`NOT_APPLICABLE`/`NO_ACTION`/`POLICY_DECISION`).
- The sizing-specific `DecisionType`/`ReasonCode`.
- Calculation inputs and the intermediate result (raw, rounded, final).
- The approved sized candidate, if any.

**Meaningful `POLICY_DECISION` records qualify for sparse logging** — the
architecture's bounded-retention rule, amended during the EntryGate
Revision 2/3 passes to explicitly include this category:

- Approved quantity.
- Zero-size rejection.
- Candidate expired.
- Lifecycle stale.
- Unhandled context combination.

**Routine `NO_ACTION`/`NOT_APPLICABLE` cycles do not qualify.**

------------------------------------------------------------------------

# 16. Diagnostics

**Revised in Revision 6** — every reason code tied to account state,
instrument-metadata-for-risk, or the retired stop-distance coordination
mechanism is removed.

```cpp
enum K_CHAKRA_POSITIONSIZER_REASON_CODE
{
    K_CHAKRA_POSITIONSIZER_REASON_NONE                        = 0,   // = K_CHAKRA_REASON_NONE
    K_CHAKRA_POSITIONSIZER_REASON_NO_NEW_CANDIDATE             = 1,
    K_CHAKRA_POSITIONSIZER_REASON_CANDIDATE_EXPIRED            = 2,
    K_CHAKRA_POSITIONSIZER_REASON_CANDIDATE_SEQUENCE_GAP       = 3,
    K_CHAKRA_POSITIONSIZER_REASON_CANDIDATE_STORAGE_RESET      = 4,
    K_CHAKRA_POSITIONSIZER_REASON_ENTRYGATE_SCHEMA_MISMATCH    = 5,
    K_CHAKRA_POSITIONSIZER_REASON_INVALID_SIDE                 = 6,
    K_CHAKRA_POSITIONSIZER_REASON_LIFECYCLE_STALE              = 7,
    K_CHAKRA_POSITIONSIZER_REASON_ENTRY_ALREADY_PENDING        = 8,
    K_CHAKRA_POSITIONSIZER_REASON_SENSOR_UNAVAILABLE_STALE     = 9,
    K_CHAKRA_POSITIONSIZER_REASON_MCTX_UNAVAILABLE_STALE       = 10,
    K_CHAKRA_POSITIONSIZER_REASON_INVALID_NONFINITE_ARITHMETIC = 11,
    K_CHAKRA_POSITIONSIZER_REASON_COMPUTED_SIZE_BELOW_MINIMUM  = 12,
    K_CHAKRA_POSITIONSIZER_REASON_PER_ORDER_MAXIMUM_CLAMPED    = 13,
    K_CHAKRA_POSITIONSIZER_REASON_UNHANDLED_CONTEXT_COMBINATION = 14,
    K_CHAKRA_POSITIONSIZER_REASON_SIZE_APPROVED                = 15
    // extend per concrete PositionSizer policy as needed, numbered above these common ones
};
```

`..._SENSOR_UNAVAILABLE_STALE` and `..._MCTX_UNAVAILABLE_STALE` both
resolve to `DecisionType = SKIP_UNDECIDABLE`, disambiguated here.
`..._COMPUTED_SIZE_BELOW_MINIMUM` and `..._PER_ORDER_MAXIMUM_CLAMPED` both
resolve to `DecisionType = REJECT_ZERO_SIZE`, likewise disambiguated. **Log
on decisions/transitions, not repeatedly every master bar.**

**Downstream sized-candidate expiry/lifecycle-staleness belongs to the
detecting downstream component's own `ReasonCode`, not PositionSizer's** —
per section 12 and the self-referential `ReasonCode` principle already
established throughout this framework (TriggerStr section 18, EntryGate
section 11's Revision 3 fix, which removed an equivalent misplaced entry
from EntryGate's own enum). No member of this enum describes what the
Entry Composer later did with a sized candidate.

------------------------------------------------------------------------

# 17. Worked examples

**Renumbered in Revision 6** — the account/instrument-unavailable example,
the three-part stop-dependent coordination example, and the historical
risk-sizing example are removed (nothing left in this document that they
describe). Nine examples remain, renumbered 1-9 with no gaps.

1. **Fixed quantity approved.** A `FIXED_QUANTITY`-method PositionSizer
   consumes an active `LONG` candidate; base size is 2 contracts, no
   context modifier applies. `DecisionType = SIZE_APPROVED`,
   `FinalPermittedQuantity = 2`. `SizedCandidateSeq` increments.
2. **Context reduces size, rounds down.** Same policy, but `Globex`
   context is active: base size 4, `50%` multiplier -> raw `2.0` — no
   rounding needed here, but a base size of 3 with the same multiplier
   would yield raw `1.5`, rounded down to `1` per section 7's default.
   `DecisionType = SIZE_APPROVED`, `RawComputedQuantity = 1.5`,
   `FinalPermittedQuantity = 1`.
3. **Below minimum, zero-size rejection.** A context multiplier reduces
   computed size to `0.4`, rounding down to `0`, below the policy's
   minimum of `1`. `DecisionType = REJECT_ZERO_SIZE`, `ReasonCode =
   COMPUTED_SIZE_BELOW_MINIMUM`. No sized candidate.
4. **Candidate expired before sizing.** By the time this cycle evaluates a
   pending candidate, `consumer evaluation time >
   CandidateValidUntilDateTime`. `DecisionType = SKIP_EXPIRED`, `ReasonCode
   = CANDIDATE_EXPIRED`. Consumed and dropped.
5. **Lifecycle changed after EntryGate acceptance.** A working entry order
   from a different candidate path now exists (per section 5's corrected
   source: detected via direct platform order-state query, not the candle
   baton manager). `DecisionType = SKIP_LIFECYCLE`, `ReasonCode =
   ENTRY_ALREADY_PENDING`.
6. **Overwrite policy, both cases:**
   - **6a — rejection does not overwrite.** `SizingDecisionSeq` advances to
     a `REJECT_ZERO_SIZE` for a *different*, later candidate; the earlier
     `SizedCandidateSeq` record (still `ACTIVE`, still unexpired) is
     completely untouched — the Entry Composer can still consume it on a
     later evaluation.
   - **6b — a second approval *does* overwrite.** A second candidate is
     approved (`SizingDecisionSeq` advances with `DecisionType =
     SIZE_APPROVED`, `SizedCandidateSeq` increments again) before the
     Entry Composer consumed the first approval. The sticky sized-candidate
     record now holds the second candidate's data only — the first is
     gone. Per the latest-approved-wins policy (section 11), this is
     expected: the consumer's next evaluation sees `SizedCandidateSeq`
     jump by more than 1 versus its own `LastConsumedSizedCandidateSeq`,
     logs a dropped intermediate approval, and proceeds with the current
     (second) candidate.
7. **Invalid — side change.** A PositionSizer that flips `LONG` to `SHORT`
   (or vice versa) for any reason. Direction is EntryGate's authoritative
   output (EntryGate section 7); PositionSizer only preserves it.
8. **Invalid — order submission.** A PositionSizer that calls an
   order-placement function directly. That's Order Management wearing a
   PositionSizer's name — PositionSizer's only legitimate output is a
   sizing decision and, on approval, a bare sized candidate.
9. **Sized candidate passes onward, still incomplete.** Example 1's
   approved `SizedCandidateSeq` record reaches the Entry Composer (not yet
   designed against this document at the time of writing) — at this point
   there is still no stop, no target, and nothing has reached the candle
   baton manager. Only the Entry Composer, once wired, can produce a
   `PROPOSED_ACTION` from this.

------------------------------------------------------------------------

# Unresolved design questions — deliberately left open

Only questions that affect real implementation. Envelope, Sensor, MCtx,
TriggerStr, EntryGate, and base-architecture decisions already settled are
not reopened here. **Renumbered in Revision 6** — former open questions 1
(stop-distance ordering), 2 (account-equity source), 3 (runtime
risk-limit configurability), and 8 (historical capital model) are removed
outright as moot, not carried forward (see "Changes from Revision 5").

1. **Is one PositionSizer instance restricted to one EntryGate source?**
   Section 1 assumes yes, mirroring EntryGate's own assumption about
   TriggerStr — not stress-tested.
2. **How are simultaneous accepted candidates arbitrated** — across
   multiple EntryGate/PositionSizer paths in the same cycle? Not a context
   conflict in the section 10 sense. **Partially resolved**: a
   portfolio-level arbitration layer above independently-configured
   instances is excluded by this project's scope boundary, not an
   undesigned future extension; the concrete case that matters — multiple
   Composer sources feeding one Entry Authority instance — is resolved in
   `Chakra_OrderManagement_Handoff_Interface_Contract.md` section 6.
   Mirrors EntryGate's own open question 3.
3. **Which component owns the Entry Composer role?** Still just a reserved
   name (EntryGate section 4); its own identity, contract, and lifecycle
   remain undesigned.
4. **Does PositionSizer run only on the next master cycle after EntryGate
   accepts, or may staged TradePol components execute sequentially within
   the same cycle?** No longer tied to any stop-geometry coordination
   concern (that mechanism is retired, section 6) — this is now a purely
   general staged-pipeline timing question, affecting how tight the
   expiry chain (section 12) needs to be in practice, and whether
   same-cycle synchronous composition would ever be worth designing.
5. **Does every approved size need a separately durable `SizedCandidateSeq`,
   or will the eventual physical coordinator guarantee same-cycle
   consumption, making the separate sequence unnecessary overhead?**
   **Default to the separate sequence (section 11) until such a guarantee
   is designed and proven** — the same reasoning that produced
   `CandidateSeq` in EntryGate applies here with equal force, and assuming
   a guarantee that doesn't exist yet is exactly the mistake this framework
   keeps correcting.

None of these are silently resolved by giving PositionSizer
responsibilities belonging to `StopManager`, the Entry Composer, portfolio
arbitration, account/portfolio risk management, or Order Management.
