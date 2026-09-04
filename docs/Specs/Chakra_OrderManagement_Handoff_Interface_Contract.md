# Chakra — Order Management Handoff Interface Contract (Draft, Revision 8, APPROVED for design-stage status)

> **SUPERSEDED for the version-one live path (2026-07-25).** A direct
> source audit of `K_Strat_Bhaskar_V1.cpp`, `K_Strat_Bhaskar_V2.cpp`, and
> `K_Util_ModeBaton_V1.cpp` found that this document's entire premise —
> that a new Entry Authority component must be designed and built to
> submit orders, track fills, confirm attached stops, and handle
> emergency remediation — is unnecessary. `K_Util_ModeBaton_V1.cpp`
> already does all of that, live and proven, today. Chakra's actual
> integration boundary is one order of magnitude smaller: publish into
> Mode Baton's existing master-signal PV record (`PV51`-`PV74`) and stop.
> See `Chakra_ModeBatonSignalPublisher_Interface_Contract.md` (Revision 1)
> for the replacement design. **This document is preserved below,
> unmodified, as a historical record of the greenfield design and the
> review process that hardened it — not as active guidance.** None of its
> content is deleted because the correlation-identity, idempotency, and
> commit-order lessons captured here remain generally instructive even
> though the component they were designed for is not being built. Do not
> implement anything from this document without first checking whether
> `Chakra_ModeBatonSignalPublisher_Interface_Contract.md` already covers
> the same ground more simply.

Status: **design only, no ACSIL code created or modified for this contract.**
Two small, additive corrections were made to already-approved upstream
documents while writing this contract — see "Upstream corrections," below.
Revision 1 was published, then subjected to an independent external review
that found 12 issues (2 critical, 8 high, 2 medium) — all confirmed real
and fixed for Revision 2; see "Changes from Revision 1," immediately below.
Revision 2 was then reviewed again and found to have three further material
issues (a proposal-correlation loss in the execution record, a single
disposition slot unable to represent simultaneous arbitration results, and
a PV-43-not-written-last commit-order violation) — all confirmed real and
fixed for Revision 3; see "Changes from Revision 2," below that. A final
consistency pass then caught one small remaining inconsistency (the
`NO_INSTRUCTION` exact-meanings list still cited "arbitration lost" despite
section 6's own log-only rule for losers) — fixed in place, no revision
bump. The external reviewer's own closing verdict after that: **"OK for
design-stage status."** Revision 4 is a **user-directed scope correction**,
not a review round: it removes the previously-planned Execution Context /
Account-Instrument contract as a dependency and narrows this document's
account/instrument handling to manual Entry Authority configuration plus
basic pre-submission consistency checks, per a standing project decision
that Chakra does not own portfolio or account-level risk management — see
"Changes from Revision 3," below. None of the correlation, idempotency,
fill-handling, emergency-remediation, or PV-ordering rules established
through Revision 3 are weakened by this pass. Revisions 5-7 are a further
scope-correction pass (dollar-risk sizing/reconciliation removed) followed
by two independent review-correction passes that each found and fixed a
real bug in the prior revision's own text (Revision 6: a broken post-fill
tolerance comparison; Revision 7: that comparison's own directional blind
spot for a fill crossing through the stop) — see each revision's own
"Changes from Revision N" section. This does not resolve the remaining
open questions (section, below) — those still gate a concrete
implementation.

Builds directly on:
- `Algo_Overlay_Framework_Architecture.md` (amended by this pass — see
  "Upstream corrections"), including the master decision cycle,
  `POLICY_DECISION`/`PROPOSED_ACTION` outcomes, and the
  `NO_INSTRUCTION`/`EXECUTABLE_INSTRUCTION` Order Management disposition.
- `Chakra_Common_Interface_Envelope_Contract.md` (Revision 9 — amended by
  this pass, purely additively — see "Upstream corrections").
- `Chakra_TradePol_EntryComposer_Interface_Contract.md` (Revision 2) —
  this document is the consumer every section of that contract deferred to
  by name (its own section 12, and its open question 8).
- `Chakra_TradePol_StopManager_Interface_Contract.md` (Revision 3, section
  9's hard gate: geometry must not become executable until a downstream
  pre-fill/post-fill policy exists) — this document is that policy.
- The **actual implementation** of `K_Util_Candle_Baton_Manager_V1.cpp`,
  its own design contract (`K_Candle_Baton_Manager_V1_Design.md`), and its
  own decision-study contract (`Candle_Baton_Decision_Study_Contract_Draft.md`
  and `K_BatonDecisionContract.h`) — treated as evidence of current
  behavior, not silently reshaped around (see "The central finding,"
  immediately below).

------------------------------------------------------------------------

# The central finding — read this before anything else in this document

Every upstream Chakra document that mentions Order Management (the base
architecture doc's original "Order Management" section, the Entry
Composer's own framing of "how a Composer's proposal becomes a real order
via the candle baton manager") assumed **one** component, the existing
`K_Util_Candle_Baton_Manager_V1`, could both *decide/submit an entry* and
*manage it afterward*. **That assumption is false for the actual codebase,
verified directly against the manager's own source and its own two design
documents, not inferred:**

- `K_Candle_Baton_Manager_V1_Design.md`, written when this manager was
  first designed, states its own scope in its own words: *"Non-goals for
  V1: no order entry logic... This study is a slave/follower manager
  only."* And: *"The existing `K_Util_ModeBaton_V1.cpp` remains the entry
  authority and source of truth for: trade creation, initial attached
  orders, master trade sequencing, the persistent-variable handoff
  contract. The new candle manager must not replace this ownership."*
- `Candle_Baton_Decision_Study_Contract_Draft.md` restates the same
  boundary from the other side: *"Manager: `K_Util_Candle_Baton_Manager_V1`,
  the study that syncs to a master baton trade and owns order modification/
  flatten execution"* (never *creation*) — and its own "Non-Goals" list
  opens with *"No entry logic."*
- The actual `K_Util_Candle_Baton_Manager_V1.cpp` source confirms this
  exactly: it contains **zero** order-submission calls anywhere (no
  `s_SCNewOrder` used for a new position, no buy/sell entry). Every order
  action in the file is `sc.ModifyOrder(...)` against an **already-existing**
  `orderId`, or `sc.FlattenAndCancelAllOrders()`. It discovers that a trade
  exists at all via a leader/follower handoff read from whatever chart+study
  is wired into its own `Master Trade Chart/Study` input (`sc.Input[1]`):
  ```cpp
  const int currentTradeLeaderNumberAsIdFromMaster =
      sc.GetPersistentIntFromChartStudy(masterChartNumber, masterStudyId, 43);
  const bool shouldSyncNewTrade = currentTradeLeaderNumberAsIdFromMaster > 0
      && (currentTradeFollowerNumberAsId < currentTradeLeaderNumberAsIdFromMaster || ...);
  ```
  followed by reading order IDs (PV 20-25, 34-38), direction/quantity (PV
  52/53), and a stop distance (PV 54) from that same master instance. This
  is a **read-only, poll-for-a-leader-ID-bump** mechanism — there is no
  code path in this file that could turn a `PROPOSED_ACTION` into a
  submitted order, because the file was never designed to submit orders.

**Conclusion, stated plainly rather than smoothed over:** "Order
Management," precisely, is at least two physically separate components in
this codebase:

1. **An Entry Authority** — decides whether and how a `PROPOSED_ACTION`
   becomes a real submitted order, and is the component this document
   actually designs. It does not exist yet in a Chakra-aware form; the
   closest existing analog is the legacy `K_Util_ModeBaton_V1`, which
   already fills this role for non-Chakra strategies but has no awareness
   of `PROPOSED_ACTION`, `ProposedActionSeq`, or anything in this framework.
2. **`K_Util_Candle_Baton_Manager_V1`** — manages a trade after it is
   confirmed live (BE, trailing, targets, close/protect), completely
   unchanged by anything in this document. It needs **zero code changes**
   for a Chakra-originated trade to work, because it is already generic
   over whichever study is wired into `Master Trade Chart/Study` — it has
   no idea, and does not need to know, whether that producer is the legacy
   `K_Util_ModeBaton_V1` or a new Chakra-aware Entry Authority. This is the
   load-bearing fact that makes the rest of this design possible without
   touching a single line of already-working, already-tested management
   code.

This document therefore designs component (1) and the exact way it
crosses into component (2) — by **publishing the same, already-established,
already-consumed master-handoff persistent-variable contract** the candle
manager already knows how to read (see section 11). "Crossing into
`K_Util_Candle_Baton_Manager_V1`" means exactly this: an Entry Authority
becoming, from the candle manager's point of view, indistinguishable from
whatever it already treats as "the master."

------------------------------------------------------------------------

# Upstream corrections made by this document

Both purely additive, neither reopens anything already settled:

1. **`Algo_Overlay_Framework_Architecture.md`, "Order Management" section**
   — corrected the standing implication that one component ("the candle
   baton manager") covers all of Order Management including entry
   submission. Replaced with the accurate two-component picture above,
   cross-referenced to this document. Nothing about the rest of that
   section (or any other section of that document) is touched.
2. **`Chakra_Common_Interface_Envelope_Contract.md`, Revision 8 -> 9** —
   `K_CHAKRA_COMPONENT_KIND` never reserved a value for Order Management,
   since no prior contract needed one (every layer through Trade Policy
   stops at `PROPOSED_ACTION`). Added `K_CHAKRA_KIND_ORDERMGMT = 5`, purely
   additive, so this document's own disposition record (section 14) can use
   the same standard identity-validation chain as every other Chakra
   producer instead of a bespoke mechanism. This does **not** make
   `K_Util_Candle_Baton_Manager_V1` itself a Chakra producer — see section 2
   for why the disposition record and the legacy manager are necessarily
   different persistent-storage instances.

------------------------------------------------------------------------

# Changes from Revision 1

An independent external review of the published Revision 1 found 12
issues — 2 critical, 8 high, 2 medium — against every upstream contract and
the actual manager source. All 12 were confirmed real on verification (in
two cases by re-checking directly against the installed `sierrachart.h` or
the legacy `.cpp` source rather than taking the finding on faith) and fixed
below. The reviewer's own closing assessment — "the central finding is
valid... separating entry authority from post-fill management is the
correct architectural direction" — is unchanged; every fix below is local
to already-flawed detail, not a reopening of that central finding.

**Critical:**
1. **The post-fill realized-risk formula (section 13) was dimensionally
   wrong.** `|AverageFillPrice - StopPrice| x Quantity` is price-units times
   a contract count, not currency — comparing it directly against
   `ProposedActionAggregatePlannedStopRisk` (currency) could silently
   approve excess risk on any instrument where currency-per-tick isn't
   exactly 1. Fixed: convert through `sc.TickSize` and
   `sc.CurrencyValuePerTick` (confirmed present in the installed
   `sierrachart.h` before relying on it) before comparing. **Superseded by
   Revision 4's own scope correction (part 2): section 13 no longer
   compares a currency figure at all — the whole risk-budget comparison
   this fix repaired was later removed, not just re-fixed. Preserved here
   only as an accurate record of a real dimensional bug against the
   machinery that existed at the time.**
2. **Partial-fill ownership was unresolved.** The contract published the
   handoff using whatever quantity was currently filled without ever
   stating whether the unfilled remainder was cancelled first — a
   filled-3-of-4 trade could still have its exposure changing after the
   candle manager had already taken over. Fixed (section 12): the Entry
   Authority retains ownership until the parent order reaches a genuine
   terminal state (fully filled, or partial-fill-plus-confirmed-cancellation
   of the remainder) — section 11's table is never published from a
   still-working parent order.

**High:**
3. **The two-instance topology (section 2) had no executable design** — no
   input for the master-handoff target, no stated cross-instance write
   mechanism, no recalc-ownership split. Fixed (section 2, expanded): an
   explicit `Master Handoff Target Chart/Study` input; confirmed
   `sc.SetPersistentIntForChartStudy`/`SetPersistentFloatForChartStudy`/
   `SetPersistentDoubleForChartStudy`/`SetPersistentInt64ForChartStudy` all
   exist in the installed header, so cross-instance writes are a verified
   capability, not an assumption; the target is a minimal placeholder
   instance with no recalc logic of its own.
4. **The architecture doc's amendment was incomplete.** The new "Order
   Management" section was corrected in the prior pass, but the earlier
   "Cycle outcomes" section still said, in three places, that
   `PROPOSED_ACTION` is forwarded to and reported on by "the candle baton
   manager" specifically — directly contradicting the two-component model.
   Fixed: all three spots (and the frozen operating-principle quote)
   generalized to "Order Management," with a note that which physical
   component that means depends on trade-lifecycle phase (Entry Authority
   for an entry, the existing candle manager afterward).
5. **`EXECUTABLE_INSTRUCTION` was redefined incorrectly.** The architecture
   defines it as a validated, concrete instruction safe to submit — Revision
   1 instead reported it only after submission, confirmed fill,
   reconciliation, and handoff publication, which is an execution *result*,
   not an instruction. Fixed (section 14): `EXECUTABLE_INSTRUCTION` is now
   reported at submission, matching the architecture exactly; a new,
   explicitly-not-an-architecture-concept `ExecutionOutcome` field (plus its
   own `ExecutionOutcomeSeq`) tracks what happens to the submitted order
   afterward.
6. **`NO_INSTRUCTION` and `DECLINED` overlapped and contradicted each
   other** — the same failure cases (expired, arbitration-lost, failed
   revalidation/risk-check) were each described, in different places, as
   belonging to both. `DECLINED` also had no basis as a third disposition
   value — the architecture defines exactly two, plus an *optional* declined
   variant, not a mandatory third category. Fixed: `OrderMgmtDispositionType`
   now has only the architecture's own two values;
   `ReasonCode` (common envelope PV 7) carries the specific reason in every
   case. A disposition record is also no longer published when nothing was
   pending at all (silence, matching every other layer's own
   `NOT_APPLICABLE`-shaped silence).
7. **Restart-safe idempotency state was insufficient.** The in-flight
   record held only slot/sequence/bar-index — not enough to reliably
   correlate a specific broker order back to a proposal after a restart,
   especially with unrelated working orders present. Fixed (sections 10 and
   16): the marker now includes the platform-assigned entry and stop order
   IDs (the actual restart-correlation key — everything else is re-derivable
   from the platform via `sc.GetOrderByOrderID` once the ID is known),
   requested side/quantity, and submission time.
8. **The time-in-force claim was factually wrong.** Revision 1 claimed
   "immediate/day-session" matched the legacy baton; the actual
   `K_Util_ModeBaton_V1.cpp` uses `SCT_TIF_GOOD_TILL_CANCELED` at its own
   order-construction sites. Fixed (section 7): corrected to
   `SCT_TIF_GOOD_TILL_CANCELED`, verified directly against the source rather
   than re-asserted.
9. **Arbitration scope was overstated as system-wide.** The scalar
   master-handoff PVs only prove one trade per *manager instance*, not
   across every symbol/account/instance in a deployment. Fixed (section 6):
   arbitration is explicitly scoped to the slots feeding one Entry
   Authority instance / one master-handoff target / one symbol-account
   pair; cross-instance competition is called out as unproven and tied to
   the still-open account/instrument-identity question.

**Medium:**
10. **Arbitration had a snapshot-before-side-effects gap.** Evaluating
    slots sequentially and submitting slot 1 immediately would make slot
    3's later evaluation fail an ordinary lifecycle check instead of being
    correctly attributed to arbitration. Fixed (section 6): explicit
    collect/choose/act phases — every slot is evaluated against one
    snapshot taken before any submission occurs this cycle.
11. **The emergency-stop guarantee overclaimed what's physically
    possible** — "never left without a stop, even transiently" cannot be
    true; some interval between fill and stop-attach failure always exists.
    Fixed (section 12 and worked example 8): reworded to the actually
    enforceable rule — immediate detection, no handoff until protected or
    flattened, best-effort protection attempted immediately, emergency
    flatten if that can't be confirmed within a short bound.
12. **The handover recommended starting implementation despite
    acknowledged blockers.** `claude-handover.md` said all seven open
    questions block implementation, then suggested resolving only naming
    before implementing. Fixed in that file directly (see its own updated
    text) — it now states plainly that every open question, plus this
    revision's own fixes, must be resolved first.

------------------------------------------------------------------------

# Changes from Revision 2

A second independent external review — after verifying the Revision 1 -> 2
fixes directly (the reviewer confirmed the ACSIL APIs and the architecture
wording itself, not just re-reading the prose) — found three further
material issues, none reopening anything Revision 2 fixed:

1. **The execution record could lose its proposal correlation (a real bug,
   same failure shape as the composite-identity bugs found repeatedly
   earlier in this project, recurring one document later).** Revision 2's
   single disposition record shared its identity fields
   (`SourceComposerChartNumber`/`StudyID`/`ProducerTypeID`/
   `SourceProposedActionSeq`) between the general per-cycle decision report
   and the in-flight execution's own tracking. Worked failure: order A
   (slot 1) is submitted and in flight; before it fills, a different slot's
   proposal is correctly evaluated and produces `NO_INSTRUCTION` — writing
   that decision into the *same* identity fields silently reassigns them
   away from order A. When order A later fills, `ExecutionOutcome` updates
   on a record now describing an unrelated proposal.
   `ExecutionOutcomeSeq` alone doesn't catch this, since it only detects
   *that* the outcome changed, not whether the identity underneath it
   changed too. Fixed (section 14): split into two separate records — a
   general, freely-overwriteable **decision record**, and a **self-contained
   execution record** whose identity fields are fixed once, at submission,
   on entirely separate PV numbers, and never touched by any other slot's
   decision until that specific execution reaches a terminal outcome.
2. **One disposition slot cannot represent every simultaneous arbitration
   result in the same cycle.** Section 6 said to publish `NO_INSTRUCTION`
   for every losing slot and then `EXECUTABLE_INSTRUCTION` for the winner —
   but with one sticky record, each write overwrites the last before any
   consumer can observe the intermediate ones; only the final write is ever
   seen. Fixed (section 6 and section 14): **losers are diagnostic-log-only,
   never separately published** — the decision record reports only the
   winning slot's outcome each cycle (or the sole slot's, when only one was
   pending), matching this framework's standing "log the skip rate, don't
   build infrastructure for it" principle rather than inventing bounded
   per-slot records for a case fully diagnosable from the log.
3. **PV 43 was not guaranteed to be the final handoff commit write.**
   Section 11's table listed PV 43 (the candle manager's own edge/commit
   signal) before PV 52-54 (direction/quantity/stop distance) — a direct
   violation of this project's own repeatedly-established "payload first,
   sequence-committed-last" convention, which would let a consumer observe
   a new leader ID paired with still-stale payload. Fixed: the table is now
   presented in explicit, required write order, with PV 43 mandated as the
   strictly last write, after every order-ID and payload field.

------------------------------------------------------------------------

# Changes from Revision 3 (user-directed scope correction, part 1, 2026-07-19)

**Not a review round — a project-wide scope boundary set directly by the
user, applied here and across every upstream TradePol contract in the same
pass.** Chakra is a trade-decision framework: it decides whether to
propose, close, adjust a stop, or change a target. **It does not own
portfolio or account-level risk management.** The user configures the
chart, instrument, and trade account manually — potentially permanently —
and Chakra, including the Entry Authority this document designs, operates
within that manually configured context. This removes the previously
recommended next task (a
`Chakra_ExecutionContext_AccountInstrument_Interface_Contract.md` general-
purpose account/instrument context service) as unnecessary, and corrects
the following in this document specifically:

1. **The Execution Context contract is no longer a dependency of this
   document, and is not going to be written.** Every place below that
   previously deferred account/instrument handling to "the still-open
   account/instrument-identity question" now states the resolution
   directly: the Entry Authority is manually configured, per instance, for
   the specific symbol and trade account it feeds (ordinary chart/study
   wiring, the same mechanism this codebase already uses everywhere else) —
   not discovered, not published as a Chakra-wide authoritative identity,
   and not mediated through any new shared service.
2. **Section 6's scope note previously tied cross-instance/cross-symbol
   coordination to the (now-cancelled) account/instrument-identity
   question, implying a future contract would eventually resolve it.**
   Corrected: coordination across multiple Entry Authority/candle-manager
   instance pairs on different symbols or accounts is **out of scope by
   design**, not a deferred gap — each instance is independently and
   manually configured, and this document does not attempt to arbitrate or
   share risk capacity between separately configured instances.
3. **Section 8's "instrument and account identity" revalidation bullet was
   marked provisional pending that same question.** Corrected: it is not
   provisional — it is a basic consistency check (the platform's current
   position/order state for the manually-configured symbol/account still
   matches what was assumed upstream), exactly the kind of "immediately
   before acting" validation this section already performs for every other
   check in its list.
4. **Open question 2 ("what is the authoritative account/instrument
   identity source?") is closed**, not carried forward — manual
   configuration is the answer, and no further design work resolves it
   further. The former open questions 3-7 are renumbered 2-6 below; open
   question 6 (arbitration ever becoming a priority policy) is reworded to
   drop "portfolio-arbitration" framing, since a true cross-account
   portfolio arbitration layer is excluded by this same scope boundary, not
   a possible future extension of this document.

**Nothing about the central finding, the two-instance PV topology, the
correlation/idempotency/fill-handling/emergency-remediation rules
(sections 8-13), the PV write-order convention (section 11), or the
`K_Util_Candle_Baton_Manager_V1` zero-code-change conclusion changes.**
This pass only removes infrastructure that was never actually required by
anything in this document's own design — the Entry Authority was already
specified as validating against its own manually-wired symbol/account
(section 8) in every revision through Revision 3; only the open-question
framing around *where that identity ultimately comes from* is corrected
here.

------------------------------------------------------------------------

# Changes from Revision 4, part 2 (user-directed scope correction, 2026-07-19)

**A direct user correction — the first scope-correction pass (Revision 4,
above) narrowed account/instrument-identity framing but left the entire
dollar-risk reconciliation machinery in section 13 in place.** The user
restated the boundary literally: Chakra does not perform monetary,
account, or portfolio risk management. `StopManager`'s own Revision 5 and
`PositionSizer`'s own Revision 6 (companion passes) removed every field
and sizing method that produced a currency-denominated risk figure
upstream of this document — this document's own section 13 depended
entirely on those now-removed fields and is corrected to match.

1. **Section 13, retitled "Post-fill slippage handling," rewritten to
   compare a whole-tick price distance against a documented tick
   tolerance, never a currency-converted risk figure against a dollar
   budget.** **This revision compared the wrong distance — the *entire*
   fill-to-stop distance against the tolerance, not the *increase* over
   the planned distance — a real bug caught on review and fixed in
   Revision 6, below; every ordinary trade would have failed a tight
   tolerance even with a perfect fill.** The `sc.CurrencyValuePerTick`
   conversion and the comparison
   against `ProposedActionAggregatePlannedStopRisk` (itself retired by
   `EntryComposer`'s own Revision 4) are both removed. **The three
   remedies (retain, tighten, emergency-exit) are unchanged** — this is
   still a genuine execution-safety responsibility (detecting and
   remediating a bad fill), squarely within what this document retains
   under the project's scope boundary (order submission, fills, partial
   fills, rejection, duplicate prevention, stop attachment, emergency
   flattening); only the dollar-risk framing around *when* to apply a
   remedy is removed, since it was never actually required by
   StopManager's own hard gate (a price-reconciliation requirement, not a
   risk-budget one — see EntryComposer's own matching Revision 4
   changelog).
2. **The execution record's `RealizedStopRisk` field (section 14, Float
   PV 1) renamed to `RealizedDistanceTicks`** — same PV slot, now a
   whole-tick price distance rather than a currency amount. **This name,
   and the comparison it fed, both had a real bug — see "Changes from
   Revision 5" below; the field is renamed again there, to
   `AdverseDistanceIncreaseTicks`.**
3. **`K_CHAKRA_ORDERMGMT_REASON_POST_FILL_RISK_RETAINED` renamed to
   `..._POST_FILL_SLIPPAGE_RETAINED`** (value 11, unchanged) — reflects
   what is actually being retained (excess tick distance, not excess
   dollar risk).
4. **Worked examples 1 and 7 updated** to describe a tick-distance check
   instead of a currency-converted risk comparison.

**Nothing about the central finding, the two-instance PV topology, the
correlation/idempotency/fill-handling rules (sections 8-12), stop
attachment/emergency-flattening requirements, the PV write-order
convention (section 11), or the `K_Util_Candle_Baton_Manager_V1`
zero-code-change conclusion changes.** Order Management's execution-safety
responsibilities are fully retained — only the dollar-risk accounting
layered on top of one of them is removed.

------------------------------------------------------------------------

# Changes from Revision 7 (independent review-correction pass, 2026-07-19)

**A real gap found on review, same shape as EntryGate's and StopManager's
own matching fixes — section 8's revalidation checklist and the section 4
manager-readiness summary both claimed to check that the Entry Authority's
configured symbol/account "still matches what upstream assumed."** No
upstream record (`EntryCandidate`, `SizedCandidate`, `StopGeometry`, or
`ProposedAction`) publishes an instrument or account field — there is
nothing "upstream assumed" to compare against, so this was never an
actually-performable check as worded. Fixed (sections 4 and 8): reframed
as what the Entry Authority can actually do — confirm its own manually
configured symbol/account are valid, confirm the chart/study wiring
feeding this slot is the intended configured wiring, and query platform
position/working-order state against that configured pair. This is
configuration and platform-state validation, not an identity comparison.
**Nothing about the still-flat/no-incompatible-entry/no-opposing-exposure
checks, or any other part of section 8, changes.**

------------------------------------------------------------------------

# Changes from Revision 6 (independent review-correction pass, 2026-07-19)

**A high-severity gap found on review of Revision 6's own fix — the
corrected formula was still unsafe for a specific, real fill pattern.**

1. **The tick-distance formula (section 13, step 1) only measures
   magnitude, and magnitude alone cannot distinguish "the fill was a bit
   worse than planned" from "the fill crossed through the stop entirely."**
   Worked failure: a `LONG` position's approved stop at 4488.00, but the
   actual fill confirms at 4485.00 — *below* the stop. `|4485.00 -
   4488.00|` is a small number, and once compared against the planned
   distance it could even read as favorable — while the real situation is
   that the stop is on the wrong side of the position entirely, and may
   already be invalid or immediately triggerable. Revision 6's formula had
   no way to detect this, because it never checked *which side* of the
   fill the stop was on, only how far away.
2. **Fixed: a directional validity gate (section 13, new step 0), checked
   before any distance/tolerance math, not folded into the existing
   retain/tighten/emergency-exit remedy set.** `LONG` requires
   `ProposedActionStopPrice < AverageFillPrice`; `SHORT` requires
   `ProposedActionStopPrice > AverageFillPrice`. Failing this gate is not
   an ordinary slippage-tolerance case — it is recognized as the same
   underlying problem section 12's stop-attach-failure emergency
   remediation already exists to solve (a position without a valid
   protective stop), reached a second way, and now explicitly documented
   as such in section 12 itself: a best-effort attempt to replace the
   invalid stop with a freshly computed, correctly-sided one, or an
   emergency flatten if that cannot be confirmed within the bounded retry
   window. New `ReasonCode = POST_FILL_STOP_INVALID_SIDE` (value 14)
   distinguishes this trigger from an ordinary stop-attach failure in the
   log, while both converge on the same `ExecutionOutcome =
   EMERGENCY_EXIT_POST_FILL` if flattening is ultimately required.
3. **New worked example 9** demonstrates the failure this gate catches.

**Nothing about the Revision 6 fix's own core correction (comparing the
*increase* over planned distance, not the raw distance) is reopened —
that fix remains correct for every fill that stays on the correct side of
the stop, which is the ordinary case this gate does not intercept.**

------------------------------------------------------------------------

# Changes from Revision 5 (independent review-correction pass, 2026-07-19)

**A critical math error found on review of the Revision 5 rewrite,
confirmed real and fixed — not a scope change, a bug fix.**

1. **Section 13's tolerance check compared the wrong quantity.**
   `ActualDistanceTicks = |AverageFillPrice - ProposedActionStopPrice| /
   sc.TickSize` is the fill's **entire** distance to the stop — comparing
   that directly against a small documented tolerance (e.g. 2-3 ticks)
   would fail essentially every ordinary trade, since a normal stop is
   placed many ticks away by design (StopManager's own minimum-distance
   validation, its section 8, already requires this). A 20-tick stop with
   a *perfect, zero-slippage* fill would still "fail" a 2-tick tolerance
   under the buggy formula. **What the check actually needs to measure is
   slippage — the increase over the *originally planned* distance, not
   the distance itself.** Fixed (section 13):
   ```text
   ActualDistanceTicks          = |AverageFillPrice - ProposedActionStopPrice| / sc.TickSize
   AdverseDistanceIncreaseTicks = ActualDistanceTicks - ProposedActionStopDistanceTicks
   ```
   compared as `max(AdverseDistanceIncreaseTicks, 0)` against the
   documented tolerance — a fill exactly at the planned distance now
   correctly reads as zero adverse increase, not a near-certain violation.
2. **Favorable slippage must never be treated as a violation.** A
   negative `AdverseDistanceIncreaseTicks` (the fill landed on the
   favorable side of the planned distance) is explicitly excluded from
   the tolerance comparison by the `max(..., 0)` clamp — this is stated
   directly in section 13 now, not left to be inferred from the formula
   alone.
3. **The execution record's field (section 14, Float PV 1) renamed a
   second time**: `RealizedDistanceTicks` (Revision 5's name, itself a
   raw fill-to-stop distance) -> `AdverseDistanceIncreaseTicks` (the
   actual compared quantity). The field stores the signed value,
   unclamped — a negative reading is meaningful diagnostic information
   (how much better the fill was than planned), not something to discard;
   only the tolerance *comparison* in section 13 clamps it.
4. **Worked examples 1 and 7 updated** to describe the corrected
   comparison.
5. **Stale open-question cross-reference numbers fixed (low severity, same
   review pass)**: the emergency-stop open question is cited as "open
   question 3" at two spots that still said "4" (sections 12 and the
   worked examples — Revision 3's renumbering missed these two inline
   references); a reference to "EntryComposer's own open question 5" for
   the arbitration question corrected to "4"; a reference to "EntryComposer's
   own open question 3" for the wait-for-refresh question corrected to
   "2." This document's own open-questions list numbering (1-6) is
   unaffected — only inline cross-references elsewhere in the prose were
   stale.

**Nothing else about section 13's structure (three remedies: retain,
tighten, emergency-exit; the requirement that the remedy applies before
section 11's table is published) or any other section of this document
changes.**

------------------------------------------------------------------------

# Core responsibility

**Frozen:**

> The Order Management Entry Authority consumes a fully-validated
> `PROPOSED_ACTION` from exactly the Entry Composer instance(s) it is
> configured to watch, independently revalidates it, and either reports
> `NO_INSTRUCTION` (nothing produced — arbitration lost, expired, failed
> revalidation, drift exceeded) or translates it into a concrete, submittable
> entry order and reports `EXECUTABLE_INSTRUCTION` at the moment that order
> is submitted (section 14 — this matches the base architecture's own
> definition of the term; it is not deferred until fill). What happens to
> that submitted order afterward (filled and handed off, rejected,
> cancelled, timed out) is tracked separately as this contract's own
> `ExecutionOutcome` (section 14), decoupled from the
> `EXECUTABLE_INSTRUCTION`/`NO_INSTRUCTION` vocabulary the architecture
> already defines. It does not discover setups, judge entry acceptance,
> compute quantity or stop geometry, or manage a trade once it is confirmed
> live — that remains `K_Util_Candle_Baton_Manager_V1`'s job, entirely
> unchanged.

**The Entry Authority must not:**
- Re-run or second-guess EntryGate/PositionSizer/StopManager/Composer's own
  judgment about setup validity, sizing, or geometry — only revalidate that
  their *conclusions* are still safe to act on right now (section 8).
- Compute a new quantity or stop distance. It may only **tighten** an
  already-approved stop under the narrow, documented post-fill remedy
  (section 13), exactly the same "risk-non-increasing adjustment only"
  discipline the Composer itself uses for pre-fill drift (EntryComposer
  section 7).
- Manage a trade once it is confirmed live — that is
  `K_Util_Candle_Baton_Manager_V1`'s exclusive responsibility from that
  point on.
- Write into `K_Util_Candle_Baton_Manager_V1`'s own local PV namespace
  (120+), or into any existing master-baton PV index not listed in section
  11's table, per that manager's own already-established "Rule 1: do not
  repurpose or reinterpret any existing master baton PV index."

------------------------------------------------------------------------

# 1. Naming and identity — one point deliberately left for the user

Every other Chakra layer follows the `K_Chakra_<LayerTag>_<Name>_V<n>`
naming convention. **Order Management is explicitly exempted from that
convention** by a prior, explicit user directive (recorded because it
mattered enough to state outright): it stays as the existing
`K_Util_Candle_Baton_Manager_V1`, untouched. That directive was made before
this document's central finding existed — before it was known that a
*second*, new physical component (the Entry Authority) would be needed at
all. Whether the Entry Authority should:
- get a `K_Util_*` name (treated as squarely Order Management, following
  the existing baton-family naming already used for
  `K_Util_ModeBaton_V1`/`K_Util_Baton_V3`/etc.), or
- get a `K_Chakra_*` name (treated as Chakra-facing enough — it is, after
  all, the one component in this whole pipeline that reads `ProposedActionSeq`
  and reports a Chakra-shaped disposition record) despite the standing
  exemption,

is **not decided here** — it is a direct consequence of this document's own
central finding, not something this document should silently resolve on
the user's behalf. See open question 1.

**Whatever it is named, its identity for this contract's purposes:**
- `ComponentKind = K_CHAKRA_KIND_ORDERMGMT` (envelope Revision 9) — for its
  own disposition record only (section 14), never for the legacy
  master-handoff PV block, which has no Chakra envelope at all (section 2).
- `ProducerTypeID`, Order-Management-local:
  ```cpp
  enum K_CHAKRA_ORDERMGMT_TYPE
  {
      K_CHAKRA_ORDERMGMT_TYPE_NONE                    = 0,
      K_CHAKRA_ORDERMGMT_TYPE_ENTRY_EXECUTION_AUTHORITY = 1
      // one member per concrete Entry Authority implementation, if more than one is ever built
  };
  ```
- `PayloadSchemaVersion` (int32 PV 20), scoped per `ProducerTypeID`, unchanged
  convention.

------------------------------------------------------------------------

# 2. Why the disposition record and the legacy handoff block are two different persistent-storage instances

**A real, load-bearing constraint found while designing this contract, not
a style preference:** the legacy master-handoff contract
(`K_Candle_Baton_Manager_V1_Design.md`, "Existing Persistent Variables
Already In Use") reserves int32 PVs **18 through 76** on whatever instance
is wired as `Master Trade Chart/Study` — including, critically, PV 20-25
and PV 34-38 (order IDs) and PV 43/52/53/54 (leader ID, direction,
quantity, stop level), all **hardcoded, non-configurable** numbers inside
`K_Util_Candle_Baton_Manager_V1.cpp` itself.

The Chakra envelope's own PV-ownership convention reserves int32 PV 20-49
for "layer-specific public payload" and 50-89 for private state — **this
overlaps the legacy reservation almost entirely.** If a single physical
study instance tried to be both the master-handoff publisher *and* a
standards-compliant Chakra envelope producer, PV 20 cannot simultaneously
mean `PayloadSchemaVersion` (Chakra convention) and `StopAllOrderID`
(legacy, already-shipping convention) — a direct, unresolvable collision.

**Resolution: the Entry Authority uses two separate persistent-storage
instances for two separate audiences, not one instance serving both:**
1. **The master-handoff instance** — whatever chart+study is wired into
   `K_Util_Candle_Baton_Manager_V1`'s `Master Trade Chart/Study` input.
   Publishes **only** the legacy PV block (section 11's table), with **no**
   Chakra envelope at all — it doesn't need one, because
   `K_Util_Candle_Baton_Manager_V1` has no Chakra awareness and never
   will.
2. **The disposition instance** — a standard Chakra envelope producer
   (`ComponentKind = ORDERMGMT`), publishing the audit/delivery records in
   section 14, read by the master decision cycle for `DecisionFrame`
   purposes and by any diagnostic tooling. Uses the standard PV ranges
   exactly like every other Chakra layer, with no exception needed.

**Concrete topology (this was left as "not mandated" in an earlier draft of
this section — that was a real gap, not a legitimate degree of freedom, and
is closed here):**

- **Cross-study persistent writes are confirmed to exist in the installed
  ACSIL header, not assumed**: `sc.SetPersistentIntForChartStudy(...)`,
  `sc.SetPersistentFloatForChartStudy(...)`,
  `sc.SetPersistentDoubleForChartStudy(...)`, and
  `sc.SetPersistentInt64ForChartStudy(...)` all exist alongside the
  already-used `GetPersistent*FromChartStudy` read accessors. This means a
  single study instance *can* write into a *different* instance's
  persistent storage directly — the two-instance design does not require
  guessing at an unverified capability.
- **The Entry Authority's own logic instance is the disposition instance**
  (section 14) — it is the primary decision-owner (sections 3-13 all run
  here), and it publishes its own standard Chakra envelope on **its own**
  persistent storage, standard PV ranges, no exception.
- **The master-handoff instance is a distinct, separate study instance**,
  configured on the Entry Authority via a new, explicit chart+study combo
  input — `Master Handoff Target Chart/Study` — analogous to every other
  cross-study reference already used throughout this codebase (e.g. the
  candle manager's own `Trail Reference Study`/`BE Reference Study`
  inputs). **This input's value must be configured by the user to point at
  the exact same chart+study the candle manager's own `Master Trade
  Chart/Study` input points at** — this is ordinary cross-study wiring
  discipline, not auto-discovery; nothing in ACSIL lets one study read
  another study's *input* values (only its *persistent storage*), so the
  Entry Authority cannot verify this wiring matches automatically. A
  concrete implementation should log its configured target chart+study on
  every disposition-record write specifically so a wiring mismatch is
  visible in the log even though it cannot be automatically validated.
- **The master-handoff target instance itself is a minimal placeholder
  study** — its only job is to exist (have a chart+study identity) so it
  has persistent storage to be written into; it needs no computation of its
  own, no `SetDefaults` beyond the bare minimum, and in particular **no
  recalculation/lifecycle logic of its own** for this contract's purposes.
  All recalc-safety (never writing during a recalc pass, section 15) is the
  writer's (the Entry Authority's) responsibility, exercised at the moment
  of the cross-study write call — the passive target has nothing to
  baseline.
- **"Synchronization" of the cross-instance write is nothing more than an
  ordinary, synchronous function call** — ACSIL's single-threaded,
  serialized per-study calculation model (the same model this whole
  framework's "commit-order convention, not a proven atomicity guarantee"
  language already relies on, per the envelope contract) means there is no
  concurrency concern to design around here: the Entry Authority calls
  `sc.SetPersistentIntForChartStudy(...)` (etc.) directly, in order, exactly
  as section 11's table specifies, and the write is visible to the
  candle manager's own next evaluation exactly like any other persistent
  variable already is.
- **A single-instance design that instead moves the Chakra payload to PV
  90+ (the envelope's own "uncommitted / future use" range) to dodge the
  collision was considered and rejected** — it would create a permanent,
  easy-to-forget special case in the PV table for exactly one producer,
  versus paying for two instances (one of them a near-empty placeholder)
  once, up front, and keeping both contracts exactly standard forever
  after.

------------------------------------------------------------------------

# 3. Inputs — consuming `ProposedActionSeq`, possibly from more than one Composer source

**Unlike the Entry Composer (wired to exactly one EntryGate/PositionSizer/
StopManager triad), the Entry Authority is realistically wired to more than
one Composer instance** — multiple independent strategies, each with its
own Composer, all ultimately competing for the same one-trade-at-a-time
capacity this codebase's actual position/order model provides (section 6).
This document supports up to 5 configured Composer source slots, matching
the existing "up to 5" convention already used throughout this codebase
(managed contracts/legs) rather than inventing a different scale.

Per configured slot (chart + study, exactly the same wiring convention as
every other cross-study reference in this framework):

- **Identity validation**, independently per slot, before any payload field
  is read: `SchemaVersion -> ComponentKind = TRADEPOL -> PolicyCategory =
  ENTRY_COMPOSER -> ProducerTypeID -> PayloadSchemaVersion -> payload` —
  unchanged chain from every upstream TradePol contract.
- **Gate**: `ProposedActionStatus != NONE && ProposedActionSeq !=
  LastConsumedProposedActionSeq[slot]` — `!=`, never `>`, same reasoning
  restated in every sibling contract: a producer storage reset to a lower
  sequence must remain observable.
- **Composite correlation identity, satisfied directly by wiring — no new
  upstream field needed.** The user-facing requirement here ("chart + study
  + `ProducerTypeID` + sequence") is exactly what every consumer in this
  framework already does for its *own* immediate producer: this Entry
  Authority is configured, per slot, with a specific chart+study; it reads
  that producer's own common-envelope `ComponentKind`/`ProducerTypeID`
  directly (standard identity validation, not a cross-reach into someone
  else's payload); and `ProposedActionSeq` (payload int64 PV 16) is the
  delivery key. This is **not** the same problem the Composer itself solved
  for `SizedCandidate`/`StopGeometry` (section 3 of that contract) — that
  problem existed because the Composer needed to cross-validate a *third
  party's* identity (the shared upstream EntryGate) embedded inside two
  *different* producers' records. Here, the producer being identified
  *is* the one the Entry Authority is directly, immediately reading from.
  The (chart, study, `ProducerTypeID`, `ProposedActionSeq`) tuple is
  therefore already a safe composite identity with zero upstream changes,
  the same way it already is for every other single-hop consumer in this
  framework (e.g. Composer reading its own wired EntryGate's `CandidateSeq`
  directly).
- **Lifecycle baselining**, all four boundaries, independently per slot
  (first attachment, rewiring, the Entry Authority's own
  `IsFullRecalculation`, its own disable -> re-enable) — the unified
  `slotBaselined` mechanism, unchanged, applied to up to 5 independently
  rewireable slots instead of one.
- **Storage-reset handling**: capture the prior sequence before mutating,
  branch forward (gap-logged) vs. backward (probable storage reset,
  re-baseline without consuming) — per slot, independently.
- **Expiry**: `ProposedActionValidUntilDateTime`, checked against this
  component's own evaluation time, never `PertainsToDateTime` (doesn't
  apply anywhere in this chain).
- **Latest-wins**: each slot's `ProposedActionSeq` is a single-slot sticky
  record under the Composer's own latest-approved-wins policy — a missed
  intermediate proposal is detected via the standard gap-logging, not
  retried.

------------------------------------------------------------------------

# 4. Manager-ready criteria

A proposal is **manager-ready** — eligible to be considered for actual
submission — only once **all** of the following hold. Falling short of any
one of these is not a partial or degraded submission; it is either
`NO_INSTRUCTION` or a declined disposition (section 14), never a partial
order:

1. Identity-validated per section 3, from a configured, currently-baselined
   slot.
2. Gate passed: genuinely new `ProposedActionSeq` for that slot (section 3).
3. Not expired: current evaluation time `<= ProposedActionValidUntilDateTime`.
4. `ProposedActionResolvedSide` is `LONG` or `SHORT` (never unset),
   `ProposedActionQuantity > 0`, `ProposedActionStopDistanceTicks` positive
   and sane.
5. **Won arbitration this cycle**, if more than one configured slot has a
   simultaneously eligible proposal (section 6).
6. **Passed independent lifecycle/position/working-order revalidation, and
   the Entry Authority's own configured-symbol/account and platform-state
   check** (section 8; narrowed on review — this is configuration and
   platform-state validation, not a comparison against an upstream
   instrument/account identity that is never actually published) — never
   inherited from the Composer's own section 6 validation, which may
   already be stale by the time this component evaluates.
7. **Passed the final pre-submission price-risk check** (section 9).

Only a proposal that clears all seven is actually submitted. Nothing here
computes a *new* quantity, side, or stop distance — every one of these
checks either passes the already-approved values through unchanged or
declines the proposal outright.

------------------------------------------------------------------------

# 5. (reserved — folded into sections 3 and 6, no separate mechanism needed)

------------------------------------------------------------------------

# 6. Multi-source arbitration and the single-live-trade constraint

**A second real, load-bearing finding grounded in the actual manager, not
an assumption:** the existing master-handoff contract has no way to
represent more than one concurrent trade **on one master-handoff instance**
at all. There is exactly one leader ID (PV 43), one direction (PV 52), one
quantity (PV 53) — this is a **single-trade-at-a-time model per
manager/master-handoff instance**, with up to 5 stop/target *legs* existing
only to scale *one* trade's quantity out across up to 5 partial exits, not
to represent 5 independent simultaneous positions. This directly shapes
arbitration for the slots that feed *this* Entry Authority instance and
*this* master-handoff target: it is not a question of reconciling
non-conflicting sizes or sides from different sources (EntryComposer's own
open question 4 named this as one *possible* shape, unresolved) — under
this codebase's actual data model, **any two simultaneously-eligible
proposals feeding the same manager/master-handoff instance are mutually
exclusive by construction**, regardless of side or size.

**Scope, stated precisely (an earlier draft of this section overstated
this as "system-wide," which cannot yet be proven true — corrected on
review):** the single-trade constraint is proven only for **one
master-handoff instance** — i.e., one specific `K_Util_Candle_Baton_Manager_V1`
deployment and whatever single symbol/account it manages, which the user
configures directly (manual chart/instrument/account setup, per the
project's own scope boundary — see "Changes from Revision 3"). **This
document's arbitration rule is scoped to exactly that: the up to 5
configured Composer slots feeding *one* Entry Authority instance, which
itself feeds *one* master-handoff instance for *one* symbol/account.**
**Coordination across multiple Entry Authority/candle-manager instance
pairs on different symbols or accounts is out of scope by design, not a
deferred gap** — Chakra does not own portfolio or account-level risk
management; if the user runs more than one instance, each is independently
and manually configured for its own symbol/account, and this document does
not attempt to arbitrate, discover, or share risk capacity between them.

**Resolution (this document's own contribution — EntryComposer's open
question 5 named "left entirely to Order Management's own existing safety
nets" as one candidate location and flagged it as unconfirmed; this section
is that confirmation, designed rather than assumed):**

- **Evaluation is snapshot-before-side-effects, not evaluate-and-submit
  sequentially — a real race condition in an earlier draft of this
  section, caught on review.** Submitting slot 1's order immediately upon
  finding it manager-ready, then evaluating slot 3 *after* that submission
  has already changed live order/position state, would make slot 3 fail
  section 8's ordinary "no incompatible working entry" check instead of
  being correctly attributed to arbitration — a materially different,
  misleading diagnostic. Fixed, in three explicit phases within one cycle:
  1. **Collect**: evaluate every configured slot's manager-readiness
     (section 4, including section 8's revalidation) against **one single,
     consistent snapshot** of lifecycle/position/order state captured once
     at the start of this phase — no slot's evaluation may observe a
     side effect of another slot's evaluation within this same phase,
     because no submission has happened yet.
  2. **Choose**: among every slot found manager-ready against that
     snapshot, the first in fixed configuration order (slot 1 before slot
     2, etc.) is the winner.
  3. **Act**: for every *other* slot that was manager-ready against the
     snapshot, log `ReasonCode = ARBITRATION_LOST` (section 17) — using the
     snapshot's own facts, never a post-submission re-check that would
     misattribute the loss to an ordinary lifecycle failure — and only then
     proceed to actual submission (section 9's final drift re-check, then
     section 10) for the winner. **Losers are diagnostic-log-only, never
     separately published to section 14's decision record — a real gap in
     an earlier draft, caught on review.** Section 14's decision record is
     one single sticky slot, the same "latest state, not a bus" shape every
     other single-slot record in this framework already has; if a losing
     slot's `NO_INSTRUCTION` were published there and then immediately
     overwritten by the winner's `EXECUTABLE_INSTRUCTION` in the same
     evaluation, no consumer could ever have observed the loser's record at
     all — publishing something no reader can ever see is not meaningfully
     different from not publishing it, so this document does not pretend
     otherwise: **the decision record reports only the winning slot's
     outcome each cycle** (or the sole slot's, when only one was pending);
     every other slot's arbitration result is fully diagnosable from the
     log, matching this framework's standing "log the skip rate, don't
     build infrastructure for it" principle (the base architecture's own
     MCtx-conflict-handling stance, applied here to competing entry
     sources).
- This reuses the exact same lifecycle check every slot already needs for
  itself (section 8) — no new arbitration engine, consistent with the base
  architecture's own "component-scoped fallback, not a shared registry"
  principle, applied here to competing entry sources instead of competing
  context combinations.
- **This is a deliberate, documented, simple default** — not a claim that
  it is optimal. A future deployment wanting true multi-strategy priority
  (e.g. "strategy A always outranks strategy B") can express that as a
  documented, per-slot priority ordering using exactly this same mechanism
  (the fixed evaluation order *is* the priority order) without needing a
  new design.

------------------------------------------------------------------------

# 7. Entry order style, price intent, and time-in-force ownership

**Resolves EntryComposer's own open question 3, exactly as that document's
own text anticipated ("Order Management's own contract adopts an explicit,
documented policy for converting `NONE` into a defined order form").**

- **`ProposedActionEntryStyle = NONE`** (the only value any upstream
  producer currently emits, per the Composer contract's own explicit
  implementation blocker) is resolved by this document's own explicit,
  documented default policy: **an unspecified entry style always submits as
  a market order**, at the Entry Authority's own current price reference
  (section 9). This is a genuine policy choice made here, not a
  rediscovery of missing upstream information — a future upstream
  `EntryStyle` source, if one is ever built, would supersede this default
  for whichever proposals actually carry a real value.
- **Time-in-force**: not carried by any upstream record. **This document's
  own default, verified directly against the legacy source rather than
  assumed (an earlier draft misstated this — corrected on review):**
  `K_Util_ModeBaton_V1.cpp` submits its entry orders with
  `NewOrder.TimeInForce = SCT_TIF_GOOD_TILL_CANCELED` (confirmed at that
  file's own order-construction sites, not day-session as an earlier draft
  of this document incorrectly claimed). This document adopts the same
  `SCT_TIF_GOOD_TILL_CANCELED` value for the Entry Authority's own entry
  submission, for the same reason cited for the market-order default above:
  matching existing, already-working platform behavior rather than
  introducing a new, undocumented convention. A future upstream source
  could supply an explicit time-in-force; until then, this default applies
  uniformly.
- **Price intent** for a market-style submission is the Entry Authority's
  own freshly-read current price at submission time (section 9) — not
  `ProposedActionStopReferencePrice` (that field describes the *stop*
  geometry's reference price, an entirely different concept) and not
  cached from an earlier cycle.

------------------------------------------------------------------------

# 8. Independent final revalidation — immediately before submission

**Never inherited from the Composer's own section 6 validation, which may
already describe stale conditions by the time this component runs** — the
same "independent re-check at every consumption stage" rule this framework
has required since EntryGate's own Revision 3, applied here one stage
further downstream, immediately before an irreversible action (submitting
a real order):

- **Still flat, for this instance's configured symbol/account** —
  `sc.GetTradePositionForSymbolAndAccount` (or equivalent) shows no open
  position for the specific symbol/account this Entry Authority instance
  feeds (not a system-wide claim across every symbol/account in the
  platform — see section 6's own scope correction).
- **No incompatible pending or working entry** — from *any* configured
  slot (section 6), not just this one.
- **No opposing exposure.**
- **The Entry Authority's own manually configured symbol and trade
  account are valid, and position/working-order queries are made for
  that configured pair** — a configuration and platform-state check, not
  a comparison against upstream identity (corrected on review: no
  upstream record — `EntryCandidate`, `SizedCandidate`, `StopGeometry`,
  or `ProposedAction` — publishes an instrument or account field, so
  there is nothing "upstream assumed" to match against; the earlier
  wording implied a comparison this component cannot actually perform).
  What this check verifies: the chart/study wiring feeding this slot is
  the intended configured wiring, the configured symbol/account
  themselves are valid, and platform position/order state is queried
  against that same configured pair. No separate identity source is
  required (see "Changes from Revision 3").
- **Quantity strictly positive and within *current* hard limits** —
  reapplies PositionSizer's own "a quantity valid when computed may no
  longer respect limits that have since changed" rule one stage further.
- **Stop remains on the risk-correct side** and stop distance/tick metadata
  remain valid — re-verified independently, never trusted from
  StopManager's or the Composer's own earlier pass.
- **The specific `ProposedActionSeq` being acted on is still the
  currently-published one for its slot** (re-read fresh immediately before
  submission, not cached from when the gate first fired earlier in this
  same evaluation) — a rewiring or a newer proposal could theoretically
  land in the gap between "observed" and "about to submit."

**If any check fails: decline the proposal, consume the sequence (section
3's gate already did this), no order submitted, log the specific failing
check** (section 14's reason codes).

------------------------------------------------------------------------

# 9. Pre-submission price-risk validation — the final gate before an irreversible action

**Distinct from, and in addition to, the Composer's own pre-fill drift gate
(EntryComposer section 7)** — that check ran at proposal-assembly time,
which may be one or more master cycles before this component actually
attempts submission. This document requires one more, final check,
immediately before the order is actually sent:

- Re-read the current price reference (same source and tolerance
  convention as section 7 requires the Composer to document for its own
  gate — this document inherits that same obligation: a concrete Entry
  Authority implementation must document its specific ACSIL price source).
- Compare against `ProposedActionStopReferencePrice` (StopManager's
  original indicative reference, self-contained in the proposal) using a
  tolerance **at least as strict as** the Composer's own
  `PriceDriftToleranceIdentifier` for that proposal — submission-time drift
  compounds the Composer's own already-checked drift, it does not replace
  it.
- **Outside tolerance: decline.** `OrderMgmtDispositionType =
  NO_INSTRUCTION`, `ReasonCode = PRICE_DRIFT_EXCEEDED_AT_SUBMISSION`. No
  order is submitted, so this is `NO_INSTRUCTION` (nothing produced), not
  `EXECUTABLE_INSTRUCTION` — see section 14 for why those two are the only
  disposition types and `ReasonCode` (common envelope PV 7) carries the
  "why," not a third disposition value. This is the safe default and the
  only option this document specifies — the Composer's own contract already
  named "wait for refresh" as unconfirmed-to-work (its own open question
  2); this component does not attempt it either.
- **This check runs even when submission would otherwise proceed
  immediately after the section 3 gate fires in the same evaluation** — it
  is not skippable because "the proposal was just observed a moment ago."

------------------------------------------------------------------------

# 10. Submission mechanics and idempotency

- Once a proposal is manager-ready (section 4) and wins arbitration
  (section 6): submit one entry order — side from `ProposedActionResolvedSide`,
  quantity from `ProposedActionQuantity`, style per section 7 — and attach
  one initial stop order at `ProposedActionStopPrice`. **No target order is
  submitted at entry** — no upstream contract supplies a target at this
  stage (EntryComposer's own open question 6: targets are a real, future,
  not-yet-designed extension, not an oversight). This is safe and requires
  no change to `K_Util_Candle_Baton_Manager_V1`: its own target-order
  handling already tolerates a zero/unset target order ID per leg.
- **Idempotency, with a restart-safe record — an earlier draft of this
  marker was too thin to actually correlate a broker order back to a
  proposal after a restart; expanded here, caught on review.** Before
  submitting, record a private, per-slot "submission in flight" marker
  containing, at minimum: the specific `ProposedActionSeq` being acted on;
  the submitting slot; the evaluation bar/time submission was attempted;
  the requested side/quantity; and, as soon as each becomes known, the
  platform-assigned entry order ID and stop order ID. **The order ID(s) are
  the authoritative restart-correlation key** — everything else about a
  submitted order (its current status, actual fill quantity, symbol,
  account) is always re-derivable from the platform itself via
  `sc.GetOrderByOrderID(...)`/position queries once the order ID is known,
  so this record does not need to separately persist a redundant,
  potentially-stale copy of symbol/account/quantity beyond what's needed to
  recognize *which* order this marker refers to before an ID is assigned.
  This marker is checked on every subsequent evaluation until it resolves
  (fill confirmed to a terminal state per section 12, or
  rejected/cancelled/timed out) — **while a submission is in flight for any
  slot, no other slot may also submit** (enforced by section 8's "no
  incompatible pending or working entry" check reading this same marker,
  not a separate mechanism).
- **A given `ProposedActionSeq` is never submitted twice.** Section 3's
  event gate already ensures it is only observed once as "new"; the
  in-flight marker additionally ensures a slow-to-resolve submission cannot
  be re-triggered by a later evaluation of the *same* still-current
  proposal.
- **If the platform's order-submission call itself fails synchronously**
  (the broker/platform rejects at submission time): this is still
  `OrderMgmtDispositionType = EXECUTABLE_INSTRUCTION` — Order Management
  did produce and attempt a concrete, submittable instruction, matching the
  architecture's own definition of that term (section 14) — with
  `ExecutionOutcome = REJECTED_AT_SUBMISSION`, `ReasonCode =
  SUBMISSION_REJECTED`. The in-flight marker is cleared, no master-handoff
  PV is touched (section 11 — nothing is written until fill confirmation).
  **The consumed `ProposedActionSeq` is not retried** — this is the
  framework's standing consume-and-drop default (envelope contract,
  "Failure policy"); a fresh attempt requires a fresh `ProposedActionSeq`
  from a new Composer decision, not a retry of the old one against
  conditions that already changed once.

------------------------------------------------------------------------

# 11. Deferred master-handoff publication — the literal crossing point into `K_Util_Candle_Baton_Manager_V1`

**The master-handoff PV block is written only once, and only after the
entry fill is actually confirmed** — never at submission time, and never
speculatively. This is a deliberate design choice, not an oversight: writing
it at submission time would risk exposing a not-yet-real trade to the
candle manager's sync logic if the order is later rejected, partially
filled in an unexpected way, or times out unfilled. Deferring publication
until fill confirmation means the candle manager's already-working sync
detection (`shouldSyncNewTrade`) never needs to special-case a phantom or
partial trade — it only ever sees a fully-confirmed one, exactly as it
already does for the legacy baton today.

**Written to whatever chart+study is configured as `Master Trade Chart/Study`
on the `K_Util_Candle_Baton_Manager_V1` instance being fed** (this document
does not change that input or its meaning). **`PV 43` must be written
strictly last, after every other field in this table — a real ordering
violation in an earlier draft, caught on review.** `PV 43` is the exact
edge/commit signal the candle manager's own `shouldSyncNewTrade` polls
(`currentTradeLeaderNumberAsIdFromMaster`); the manager can observe the
incremented leader ID the instant it's written and immediately act on
whatever payload (order IDs, direction, quantity, stop distance) is
currently sitting in the other fields. Writing `PV 43` before those fields
are settled — as an earlier draft's table ordering implied — would let a
consumer observe a new leader ID paired with stale or partially-written
payload, exactly the "payload first, sequence-committed-last" violation
this project's own commit-order convention (envelope contract, and every
layer built on it) already exists to prevent. The table below is presented
in **required write order**, not just for reference:

| Write order | PV | Field | Value written by the Entry Authority |
|---|---|---|---|
| 1 | 20 | `StopAllOrderID` | 0, unless a single aggregate stop order model is used instead of per-leg stops (implementation choice, not fixed by this contract) |
| 2 | 21-25 | `Target1-5OrderID` | 0 — no target submitted at entry (section 10) |
| 3 | 34-38 | `Stop1-5OrderID` | The confirmed stop order's ID in slot 1; 0 for slots 2-5 (V1 submits one stop; multi-leg scale-out stops are a future extension, not designed here) |
| 4 | 52 | `mainChartTradeDirection` | `1` for long, `-1` for short — copied from `ProposedActionResolvedSide` |
| 5 | 53 | `mainChartTradeQuantity` | The **actually confirmed filled** quantity, read from the live position after fill, never the originally-requested `ProposedActionQuantity` if they differ (partial fill, section 12) |
| 6 | 54 | `mainChartTradeStopLevel` (double) | The confirmed stop **distance** in price units — `|AverageFillPrice - StopPrice|` — matching exactly how the candle manager already consumes this field (`syncedMasterStopOffset`, added/subtracted from `AveragePrice`), **not** an absolute price |
| 7 (last) | 43 | `currentTradeLeaderNumberAsId` | **Incremented** (never just set) from this instance's own prior value, exactly matching the legacy baton's own `currentTradeLeaderNumberAsId++` semantic — this is what the candle manager's edge-detection actually watches, and it must be the final write of this whole table |

**All other legacy-reserved PVs (18-19, 26-33, 39-42, 44-51, 55-76) are left
untouched** — the candle manager does not read them, and the Entry
Authority has no reason to write them; per the legacy contract's own "Rule
1," they remain reserved regardless of whether anything currently uses
them.

**Once these are written, `K_Util_Candle_Baton_Manager_V1` takes over with
zero further involvement from Chakra** — its own already-existing,
already-tested sync/BE/trail/target/close logic runs exactly as it does
today for a legacy-baton-originated trade. This document's own
responsibility ends the moment fill is confirmed and this table is
published.

------------------------------------------------------------------------

# 12. Partial fills, rejected orders, cancellations, timeouts, and emergency remediation

- **Partial-fill ownership — resolved explicitly, not left implicit (a real
  gap in an earlier draft of this section):** the Entry Authority retains
  full responsibility for the submitted entry order until it reaches a
  **terminal state** — either fully filled with no remaining working
  quantity, or partially filled *and* the remaining working quantity is
  actively cancelled and that cancellation is confirmed. Concretely, on
  observing a partial fill:
  1. Immediately request cancellation of the remaining unfilled quantity —
     never leave a partial fill's remainder passively working while
     treating the trade as "handed off."
  2. Wait for the platform to confirm the remainder is actually cancelled
     (or query order status until it reports a terminal state) — the
     remaining quantity filling *anyway*, in the gap before cancellation is
     confirmed, is treated as a further partial fill and re-evaluated from
     step 1, not silently absorbed.
  3. **Only once the parent order is terminal** (nothing left working) does
     the Entry Authority treat the fill as confirmed and proceed to section
     13's reconciliation and section 11's publication, using whatever
     quantity actually, finally filled.
  This means exposure can never keep changing after "handoff" — the
  quantity published at PV 53 (section 11) is always the final, terminal
  fill quantity, never a snapshot taken while the remainder could still
  fill unpredictably.
- **Order rejected at submission**: covered in section 10 — reported per
  section 14 (`EXECUTABLE_INSTRUCTION` / `ExecutionOutcome =
  REJECTED_AT_SUBMISSION`), no PV block written, sequence consumed, no
  retry of the same `ProposedActionSeq`.
- **Cancellations / timeouts**: a configurable, code-versioned timeout
  (per this framework's standing "no runtime-editable thresholds" rule)
  bounds how long an entry order may sit completely unfilled. On timeout:
  cancel the order, clear the in-flight marker, do not write any
  master-handoff PV (nothing was ever written for an unconfirmed attempt —
  section 11), `ExecutionOutcome = CANCELLED_TIMEOUT`, `ReasonCode =
  SUBMISSION_TIMEOUT`, sequence remains consumed, no retry of the same
  `ProposedActionSeq`. If the order actually fills (fully or partially) in
  the gap between the timeout firing and the cancel request landing, that
  is a fill, not a timeout — handle it per the partial/full-fill rule
  above, not as a cancellation.
- **Emergency remediation — a hard rule on required behavior, stated as
  what is actually achievable, not overclaimed:** if the intended stop
  order itself fails to attach after a confirmed fill (a distinct platform
  failure from the entry fill succeeding), the position necessarily exists
  briefly with no working stop — that interval cannot be made to not exist,
  and this document does not claim otherwise. **A second, distinct trigger
  for this same mechanism (added in Revision 7): the stop order attaches
  successfully, but the actual fill price crosses through or past it, so
  the attached stop is no longer on the protective side of the position —
  section 13's directional validity gate is what detects this case.** Both
  triggers converge on the same requirement, because both describe the
  same underlying fact — the position does not currently have a valid
  protective stop. **What is required and non-negotiable**: immediate
  detection; the master-handoff table (section 11) must not be published
  until either a valid stop is in place or an emergency remediation
  completes; a best-effort attempt (attach the stop, or replace an invalid
  one with a freshly computed, correctly-sided one) follows immediately;
  and — if that best-effort attempt cannot be confirmed within a short,
  bounded retry window — an immediate emergency flatten of the position,
  rather than continuing to retry indefinitely with the position sitting
  unprotected or mis-protected. The exact emergency-stop parameters and
  retry-window bound are an implementation detail left open (open question
  3); the requirement that the Entry Authority never simply moves on while
  a fill sits with no *valid* stop and no active remediation in progress is
  not optional.

------------------------------------------------------------------------

# 13. Post-fill slippage handling — closing StopManager's hard gate

**Rewritten in Revision 4 (user-directed scope correction, part 2 — see
"Changes from Revision 4, part 2" below) — this is now a price/execution-
quality check, not a dollar-risk-budget reconciliation.** The prior
revision compared a currency-converted realized-risk figure against
`ProposedActionAggregatePlannedStopRisk` (a dollar risk budget composed
from PositionSizer/StopManager output). **Both the currency conversion and
the dollar budget it compared against are gone**: `StopManager`'s own
Revision 5 removed the currency-per-tick fields that conversion depended
on, `PositionSizer`'s own Revision 6 removed every sizing method that
produced a risk budget in the first place, and `EntryComposer`'s own
Revision 4 retired `ProposedActionAggregatePlannedStopRisk` entirely — see
each document's own "Changes" section. **This section is what StopManager's
own hard gate (its section 9) and the Composer's own section 7 have both
been waiting for, all the way down this pipeline** — but what that gate
actually requires is a *price* reconciliation, not a *risk* one; Revision
3's dollar framing was always more than the gate itself demanded (see
EntryComposer's own Revision 4 changelog, which makes the same point from
the proposal-field side).

The inputs are already on the proposal (EntryComposer section 8):
`ProposedActionStopReferencePrice`, `ProposedActionStopPrice`,
`ProposedActionStopDistanceTicks`, `ProposedActionDriftReferencePrice`,
`ProposedActionDriftAmount`, `PriceDriftToleranceIdentifier`.

Immediately once fill is confirmed (per section 12's terminal-state rule),
before publishing section 11's table:

0. **Directional validity gate — checked first, before any distance or
   tolerance math, and never bypassed by a small or even favorable-looking
   raw distance (a real gap found on review, added in Revision 7): confirm
   the already-attached stop is still on the protective side of the actual
   fill.**
   ```text
   LONG:  ProposedActionStopPrice < AverageFillPrice
   SHORT: ProposedActionStopPrice > AverageFillPrice
   ```
   A fast or adverse fill can cross through or past the approved stop
   price entirely — e.g. a `LONG` fill landing *below* `ProposedActionStopPrice`.
   In that case the absolute distance formula in step 1 below would report
   a *small* (or even, by the signed math, negative/"favorable") distance,
   because it only measures magnitude — it has no way to express "the stop
   is now on the wrong side of the position and may already be invalid or
   immediately triggerable." **If this gate fails, do not proceed to steps
   1-4 below at all** — this is not an ordinary slippage-tolerance case; it
   is section 12's own emergency-remediation trigger (a position without a
   valid protective stop), reached a second way. Log
   `ReasonCode = POST_FILL_STOP_INVALID_SIDE`, then follow section 12's
   emergency-remediation mechanism exactly: a best-effort attempt to
   replace the invalid stop with a freshly computed, correctly-sided one
   (StopManager's own risk-safe rounding rule, that contract's section 7,
   computed fresh against the actual fill rather than the pre-fill
   reference), and — if that cannot be confirmed within section 12's own
   bounded retry window — an immediate emergency flatten. Section 11's
   table is not published until this resolves, exactly as section 12
   already requires for its own trigger case.
1. **Compute the actual fill's distance from the (fixed) approved stop
   price, and compare it against the *originally planned* distance — never
   test the raw distance against a tolerance directly (a real bug in an
   earlier draft of this section, caught on review: comparing the whole
   stop distance to a small tick tolerance would fail every ordinary trade
   — a ordinary 20-tick stop would never pass a 2-tick tolerance even with
   a perfect, zero-slippage fill; what must be measured is the *increase*
   caused by slippage, not the distance itself).** Plain price-grid
   arithmetic throughout, no currency conversion of any kind:
   ```text
   ActualDistanceTicks          = |AverageFillPrice - ProposedActionStopPrice| / sc.TickSize
   AdverseDistanceIncreaseTicks = ActualDistanceTicks - ProposedActionStopDistanceTicks
   ```
   `ProposedActionStopDistanceTicks` is StopManager's own already-approved,
   already-rounded planned distance (self-contained on the proposal,
   EntryComposer section 8) — the same reference point the comparison is
   measuring deviation *from*, not a fresh recomputation from
   `ProposedActionStopReferencePrice`.
2. **Only a *positive* `AdverseDistanceIncreaseTicks` represents excess
   risk — a negative value (the fill landed closer to the stop-favorable
   side than planned, e.g. a better entry price) is favorable slippage and
   must never be treated as a violation, clamped, or otherwise flagged.**
   Compare `max(AdverseDistanceIncreaseTicks, 0)` against a documented,
   code-versioned tick tolerance (the same versioning discipline as every
   other tolerance in this framework — e.g. the Composer's own
   `PriceDriftToleranceIdentifier`, section 9's submission-time tolerance).
3. **Within tolerance** (including every favorable/zero case): keep the
   stop exactly as originally approved. Ordinary case.
4. **Outside tolerance — in order of preference, each an explicit,
   documented choice, never a silent default of "do nothing":**
   - **Retain**: accept the extra distance as-is, log it plainly
     (`ReasonCode = POST_FILL_SLIPPAGE_RETAINED`) — legitimate when the
     excess is small and adjusting the stop isn't practical post-fill.
   - **Tighten the stop** distance to bring it back toward the originally
     approved distance — **never widen it**, the same risk-non-increasing
     constraint the Composer itself used for pre-fill drift adjustment
     (EntryComposer section 7). `ReasonCode = POST_FILL_STOP_TIGHTENED`.
   - **Emergency-exit** immediately if neither retaining nor tightening is
     acceptable (e.g. the slippage was severe enough that no reasonable
     stop keeps the position anywhere near its intended risk profile) —
     `ReasonCode = POST_FILL_EMERGENCY_EXIT`.
5. Whichever remedy applies, it is applied **before** section 11's table is
   published, so `K_Util_Candle_Baton_Manager_V1` only ever syncs onto a
   trade whose stop already reflects the post-fill-reconciled reality, not
   the pre-fill indicative one.

**This is the only post-fill remedy this framework defines anywhere** — no
upstream document attempts it, by design; every one of them named this
document as where it would finally be decided. **It remains a genuine
execution-safety responsibility** (detecting and remediating bad fills),
squarely within what Order Management retains under this project's scope
boundary — only the *dollar-risk-budget* framing around it, which was
never actually load-bearing for the gate itself, has been removed.

------------------------------------------------------------------------

# 14. Disposition and execution records — acknowledgement back to Chakra

**Own standard Chakra envelope** (`ComponentKind = ORDERMGMT`, section 1),
on the **disposition instance** (section 2) — never the master-handoff
instance.

**Two separate records, not one shared record with two sequences — a real
correlation-loss bug in an earlier draft of this section, caught on
review.** That draft put `SourceComposerChartNumber`/`StudyID`/
`ProducerTypeID`/`SourceProposedActionSeq` and `ExecutionOutcome` in the
**same** single sticky slot as the general per-cycle decision report. Worked
failure: order A (from slot 1) is submitted and in flight
(`EXECUTABLE_INSTRUCTION` / `SUBMITTED_PENDING_FILL`, identity fields
pointing at slot 1's proposal). Before it fills, a newer proposal from a
*different* slot is evaluated and correctly produces `NO_INSTRUCTION`
(e.g. declined because order A is still in flight) — but writing that
decision into the *same* fields overwrites the identity that order A's
in-flight tracking was relying on. When order A later fills, the code
updates `ExecutionOutcome` on a record whose `SourceComposerChartNumber`/
`SourceProposedActionSeq` now describe the unrelated later decision, not
order A. `ExecutionOutcomeSeq` alone does not prevent this — it detects
*that* something changed, not *whether it changed the right record's
identity along with it*. Fixed by giving the execution lifecycle its own,
separate, self-contained record whose identity fields are copied in once,
at submission, and never touched again by anything else until that
specific order reaches a terminal state:

## Decision record (general, single-sticky-slot audit — overwriteable every cycle)

Reports which of the architecture's two disposition values Order
Management decided **this cycle**, for **whichever slot won arbitration**
(section 6) — losing slots are diagnostic-log-only, never given their own
publication (section 6's own fix, above), so this record never needs to
represent more than one simultaneous outcome.

| Field | Namespace/PV | Purpose |
|---|---|---|
| `PayloadSchemaVersion` | Int32 PV 20 | Envelope convention |
| `OrderMgmtDispositionType` | Int32 PV 21 | `NONE=0`/`NO_INSTRUCTION=1`/`EXECUTABLE_INSTRUCTION=2` — matches the architecture's own two values exactly, no third value |
| `SourceComposerChartNumber` | Int32 PV 22 | Which slot's decision this is |
| `SourceComposerStudyID` | Int32 PV 23 | Same |
| `SourceComposerProducerTypeID` | Int32 PV 24 | Read from that producer's own envelope at consumption time |
| `ResolvedSide` | Int32 PV 25 | Self-contained copy |
| `SubmittedQuantity` | Int32 PV 26 | Requested (pre-fill) quantity |
| `OrderMgmtDecisionSeq` | Int64 PV 10 | Increments only when a genuine `PROPOSED_ACTION` was evaluated and a real decision resulted — never for "nothing pending" (below), never re-incremented by anything happening to a separately-tracked in-flight execution |
| `SourceProposedActionSeq` | Int64 PV 11 | Self-contained — which proposal this decision is about |
| `SourceComposerDecisionSeq` | Int64 PV 12 | Non-critical diagnostic linkage only, per the established audit/delivery split |
| `DecisionAvailabilityDateTime` | Datetime PV 10 | When this decision was made |

**No disposition is published at all when there was nothing pending to
evaluate** (an earlier draft's `NO_INSTRUCTION` definition wrongly included
"no pending proposal this cycle," which both self-contradicted this
section's own "meaningful transitions only" logging rule and conflicted
with the architecture's own "for each `PROPOSED_ACTION` forwarded" framing
— fixed by removing that case entirely; "nothing was pending" is silence,
matching every other layer's own `NOT_APPLICABLE`-shaped silence, not a
reportable disposition).

## Execution record (self-contained, pinned per in-flight order until terminal)

**Created and its identity fields fixed the moment `EXECUTABLE_INSTRUCTION`
is decided** (submission attempted) — never reassigned to a different
proposal's identity until this specific execution reaches a terminal
`ExecutionOutcome`. Because idempotency (section 10) already guarantees at
most one submission in flight at a time across all slots, this record never
needs to represent more than one occupant either — the same "at most one
thing in flight" fact that makes section 6's arbitration rule sufficient
also makes a single execution record sufficient here.

| Field | Namespace/PV | Purpose |
|---|---|---|
| `ExecutionOutcome` | Int32 PV 27 | See enum below; `NONE=0` when no execution is currently pinned |
| `ExecutionSourceComposerChartNumber` | Int32 PV 28 | **Fixed at submission, immutable until terminal** — which slot's proposal this specific order came from |
| `ExecutionSourceComposerStudyID` | Int32 PV 29 | Same |
| `ExecutionSourceComposerProducerTypeID` | Int32 PV 30 | Same |
| `ExecutionResolvedSide` | Int32 PV 31 | Fixed at submission |
| `ExecutionRequestedQuantity` | Int32 PV 32 | Fixed at submission |
| `ConfirmedFillQuantity` | Int32 PV 33 | 0 until a terminal fill (section 12) |
| `MasterHandoffTradeLeaderId` | Int32 PV 34 | The PV 43 value this execution resulted in publishing (section 11) — 0 if none published |
| `EntryOrderID` | Int32 PV 35 | Platform order ID, once known — the restart-correlation key (section 10) |
| `StopOrderID` | Int32 PV 36 | Platform order ID of the attached (or emergency) stop, once known |
| `ExecutionOutcomeSeq` | Int64 PV 13 | Increments every time `ExecutionOutcome` itself changes (submitted -> filled, or submitted -> rejected, etc.) |
| `ExecutionSourceProposedActionSeq` | Int64 PV 14 | **Fixed at submission** — which proposal this execution is about; never reassigned mid-flight |
| `SubmissionAvailabilityDateTime` | Datetime PV 11 | When *this* execution's submission was attempted |
| `FillConfirmedDateTime` | Datetime PV 12 | 0/unset until a terminal fill (section 12) |
| `AdverseDistanceIncreaseTicks` | Float PV 1 | Section 13 — renamed from `RealizedStopRisk` in Revision 4, and corrected in Revision 5 from a raw fill-to-stop distance to the *increase* over the originally planned distance (`ActualDistanceTicks - ProposedActionStopDistanceTicks`); a currency figure through Revision 3, a whole-tick price-distance measure since. Never negative-clamped in storage — a negative value (favorable slippage) is a legitimate, meaningful reading, only excluded from the tolerance *comparison* (section 13), not from what this field records. |
| `AverageFillPrice` | Float PV 2 | 0 until confirmed |

**This record is only ever overwritten by a *new* submission's identity
once the current occupant has reached a terminal `ExecutionOutcome`**
(`FILL_CONFIRMED_HANDOFF_PUBLISHED`/`REJECTED_AT_SUBMISSION`/
`CANCELLED_TIMEOUT`/`EMERGENCY_EXIT_POST_FILL`) — which section 10's
idempotency rule already guarantees, since no other slot may submit while
one is in flight. A decision record publication (above) for an unrelated
slot's `NO_INSTRUCTION` never touches this record's fields at all — the
two are on entirely separate PV numbers precisely so one can never
overwrite the other.

## Shared `ReasonCode`

**One common-envelope `ReasonCode` field (PV 7, int32, common range 1-7)
is shared by both records — no separate payload-range `ReasonCode` field
exists** (a first draft of this section added one, duplicating PV 7's own
purpose; removed on review, matching every other TradePol contract's own
practice of populating the one shared field rather than inventing a
second). **Explicit rule for which record it currently describes** (needed
now that there are two records that can each write it): `ReasonCode`
always reflects whichever record was written most recently in the current
evaluation. On a cycle where a new submission both decides
`EXECUTABLE_INSTRUCTION` (decision record) **and** immediately pins
`SUBMITTED_PENDING_FILL` (execution record) — the ordinary case — it
reflects the execution record's value, since that write is the later,
more specific one in commit order.

```cpp
enum K_CHAKRA_ORDERMGMT_DISPOSITION
{
    K_CHAKRA_ORDERMGMT_DISPOSITION_NONE                   = 0,
    K_CHAKRA_ORDERMGMT_DISPOSITION_NO_INSTRUCTION         = 1,
    K_CHAKRA_ORDERMGMT_DISPOSITION_EXECUTABLE_INSTRUCTION = 2
    // exactly the architecture's own two values -- no third "declined" value;
    // ReasonCode (common envelope PV 7) always carries the specific reason
    // (arbitration lost, expired, drift exceeded, etc.)
};

enum K_CHAKRA_ORDERMGMT_EXECUTION_OUTCOME
{
    K_CHAKRA_ORDERMGMT_EXECUTION_OUTCOME_NONE                          = 0,
    K_CHAKRA_ORDERMGMT_EXECUTION_OUTCOME_SUBMITTED_PENDING_FILL        = 1,
    K_CHAKRA_ORDERMGMT_EXECUTION_OUTCOME_FILL_CONFIRMED_HANDOFF_PUBLISHED = 2,
    K_CHAKRA_ORDERMGMT_EXECUTION_OUTCOME_REJECTED_AT_SUBMISSION        = 3,
    K_CHAKRA_ORDERMGMT_EXECUTION_OUTCOME_CANCELLED_TIMEOUT             = 4,
    K_CHAKRA_ORDERMGMT_EXECUTION_OUTCOME_EMERGENCY_EXIT_POST_FILL      = 5
};

enum K_CHAKRA_ORDERMGMT_REASON_CODE
{
    K_CHAKRA_ORDERMGMT_REASON_NONE                            = 0,
    K_CHAKRA_ORDERMGMT_REASON_PRODUCER_STORAGE_RESET          = 1,
    K_CHAKRA_ORDERMGMT_REASON_LIFECYCLE_INELIGIBLE            = 2,
    K_CHAKRA_ORDERMGMT_REASON_ARBITRATION_LOST                = 3,
    K_CHAKRA_ORDERMGMT_REASON_EXPIRED                         = 4,
    K_CHAKRA_ORDERMGMT_REASON_PRICE_DRIFT_EXCEEDED_AT_SUBMISSION = 5,
    K_CHAKRA_ORDERMGMT_REASON_INVALID_QUANTITY_OR_GEOMETRY    = 6,
    K_CHAKRA_ORDERMGMT_REASON_SUBMISSION_REJECTED             = 7,
    K_CHAKRA_ORDERMGMT_REASON_SUBMISSION_TIMEOUT              = 8,
    K_CHAKRA_ORDERMGMT_REASON_EXECUTABLE_INSTRUCTION_SUBMITTED = 9,
    K_CHAKRA_ORDERMGMT_REASON_FILL_CONFIRMED_HANDOFF_PUBLISHED = 10,
    K_CHAKRA_ORDERMGMT_REASON_POST_FILL_SLIPPAGE_RETAINED     = 11,   // renamed in Revision 4, was POST_FILL_RISK_RETAINED
    K_CHAKRA_ORDERMGMT_REASON_POST_FILL_STOP_TIGHTENED        = 12,
    K_CHAKRA_ORDERMGMT_REASON_POST_FILL_EMERGENCY_EXIT        = 13,
    K_CHAKRA_ORDERMGMT_REASON_POST_FILL_STOP_INVALID_SIDE     = 14   // new in Revision 7, section 13's directional validity gate
    // extend per concrete implementation as needed, numbered above these common ones
};
```

**Exact meanings, restated precisely for this boundary:**
- **`PROPOSED_ACTION`** — not this document's own output; it is what this
  document *consumes* (produced exclusively by the Entry Composer,
  architecture doc). Included here only to state the boundary precisely:
  this document never emits one.
- **`NO_INSTRUCTION`** — a `PROPOSED_ACTION` **was** evaluated this cycle,
  belonged to the slot that won arbitration (or was the only slot pending),
  and Order Management still produced nothing executable from it: expired,
  failed final revalidation, or price-risk check exceeded. **Never
  "arbitration lost"** — an earlier draft of this list included it, which
  contradicted section 6's own log-only rule for losers (a losing slot's
  `ARBITRATION_LOST` is never published to this record at all, so it can
  never be one of the reasons a *published* `NO_INSTRUCTION` cites; fixed
  by removing it). Never used for "there was nothing to evaluate at all"
  (silence, above).
- **`EXECUTABLE_INSTRUCTION`** — Order Management validated the proposal
  and translated it into a concrete, submittable order, **and attempted
  submission** — reported at that moment, matching the architecture's own
  definition precisely. This is **not** deferred until fill (an earlier
  draft of this document incorrectly redefined the term to mean
  "submitted, filled, reconciled, and handed off," which contradicts the
  architecture's own "safe to actually submit" wording — fixed). What
  happens afterward is tracked by `ExecutionOutcome`, this contract's own
  extension, not a redefinition of the architecture term:
  - `SUBMITTED_PENDING_FILL` — order accepted by the platform, not yet
    resolved.
  - `FILL_CONFIRMED_HANDOFF_PUBLISHED` — terminal fill confirmed (section
    12), section 13's reconciliation applied, section 11's table
    published. The success path.
  - `REJECTED_AT_SUBMISSION` — the platform synchronously rejected the
    submission attempt itself (section 10). Still `EXECUTABLE_INSTRUCTION`
    at the `OrderMgmtDispositionType` level — a concrete instruction *was*
    produced and attempted — with this `ExecutionOutcome` recording that it
    did not actually go live.
  - `CANCELLED_TIMEOUT` — submitted, never filled, cancelled after the
    configured timeout (section 12).
  - `EMERGENCY_EXIT_POST_FILL` — the emergency flatten path was taken, from
    any of section 12's now-two trigger conditions (stop-attach failure, or
    section 13's directional validity gate failing) or section 13's own
    excess-slippage emergency-exit remedy — all three converge on the same
    outcome value; `ReasonCode` (`POST_FILL_EMERGENCY_EXIT` or
    `POST_FILL_STOP_INVALID_SIDE`) distinguishes which one actually fired.

------------------------------------------------------------------------

# 15. Recalculation, disable/re-enable, attachment, and restart behavior

- **`IsFullRecalculation`**: baseline every slot's `slotBaselined`/
  `LastConsumedProposedActionSeq` to whatever each configured Composer
  source's *current* `ProposedActionSeq` is, without consuming it — the
  same rule every consumer in this framework already follows. **Never
  submit an order, and never write the master-handoff table, while this
  component's own `IsFullRecalculation` is true** — this is the same
  blanket safety rule the envelope contract already states for every
  consumer with real-world side effects, restated here because this is the
  one component in the entire framework where it actually gates a live
  order.
- **Disable -> re-enable**: same baselining, applied at this boundary too.
  Any in-flight submission marker (section 10) active at disable time is
  **not** silently abandoned — it must still resolve (fill-confirmed,
  rejected, or timed out) even across a disable/re-enable cycle, since an
  order already live at the broker does not stop existing because this
  study was toggled off. Only *new* proposal consumption is gated by
  enable state, not resolution of an already-in-flight submission.
- **First attachment / rewiring** to a Composer source slot: same unified
  `slotBaselined` mechanism as every other consumer, per slot.
- **Restart** (platform restart, DLL reload): the in-flight submission
  marker is itself persistent-variable-backed (private range, section 16),
  so a restart does not lose track of an order that was submitted but not
  yet confirmed — on resume, the Entry Authority must re-query actual
  order/position state before doing anything else, rather than trusting
  its own pre-restart marker blindly (the marker says "I believed a
  submission was in flight"; the platform's own order/position state is
  authoritative about what actually happened while this study wasn't
  running).

------------------------------------------------------------------------

# 16. Persistent-variable ownership and collision avoidance

**Disposition instance** (section 2), standard Chakra ranges, no exception
needed (section 14's two records, above, already assign int32 PVs 20-36
public and int64 PVs 10-14 public, split across the decision record and the
self-contained execution record; 1-2 float public; remaining public/private
ranges follow the envelope's standard convention untouched). **Expanded
from an earlier draft** to actually support restart-safe correlation
(section 10) — the in-flight marker needs more than a bare sequence number
and bar index to survive a restart usefully:

```
int32 private 50-89:
  slotBaselined[1..5]                 -- one per configured Composer slot
  storedComposerChartNumber[1..5]     -- rewiring-detection bookkeeping
  storedComposerStudyID[1..5]
  inFlightSlot                        -- 0 = none; else which slot has a submission in flight
  inFlightSubmissionBarIndex
  inFlightRequestedSide               -- section 10's marker: LONG/SHORT, set before submission
  inFlightRequestedQuantity           -- section 10's marker: requested quantity, set before submission
  inFlightEntryOrderID                -- 0 until the platform assigns an order ID; the restart-correlation key
  inFlightStopOrderID                 -- 0 until the stop attaches (or an emergency stop is applied)
  inFlightExecutionOutcome            -- private working copy of section 14's ExecutionOutcome while still in flight

int64 private 30-59:
  LastConsumedProposedActionSeq[1..5] -- one per configured Composer slot
  inFlightProposedActionSeq           -- the specific ProposedActionSeq currently in flight, 0 = none

datetime private 30+:
  inFlightSubmissionDateTime          -- when submission was attempted, set before the platform call
```

**On restart, the Entry Authority must re-query actual order/position
state via `sc.GetOrderByOrderID(inFlightEntryOrderID, ...)` (once that ID
is non-zero) and the live position, rather than trusting these private
fields' own last-known values** (section 15) — the private fields exist so
the Entry Authority knows *which* order to re-query, not as a substitute
for re-querying the platform's own authoritative state.

**Master-handoff instance** (section 2) — no Chakra envelope, no PV range
exception, only the exact table in section 11, and only those numbers.
Never repurpose 18-19, 26-33, 39-42, 44-51, or 55-76 on that instance for
anything this document needs — if a future need arises, it requires a new,
explicitly-reserved number outside the legacy range entirely, per that
contract's own already-established "Rule 1."

------------------------------------------------------------------------

# 17. Logging, auditability, bounded retention, and failure diagnostics

- **Sparse logging at meaningful transitions only** — every
  `OrderMgmtDecisionSeq` advance and every `ExecutionOutcomeSeq` advance
  (section 14, now two independent sequences on two independent records),
  matching this framework's standing "log transitions, not every cycle"
  discipline. A proposal that remains not-yet-manager-ready across several
  cycles (e.g. still waiting on the in-flight marker to clear before a new
  one can be considered) is not re-logged every cycle. Every losing slot's
  `ARBITRATION_LOST` determination (section 6) is logged here even though
  it is never separately published to the decision record.
- **Every decline carries its specific `ReasonCode`** — never a bare
  rejection with no diagnostic detail, matching every prior layer's own
  reason-code discipline.
- **The in-flight submission marker's full lifecycle is logged**:
  submission attempted, fill confirmed (with the section 13 reconciliation
  outcome), or resolved as rejected/cancelled/timed out — this is the one
  place in the entire framework where a "nothing happened yet, still
  waiting" state corresponds to a real order sitting at a real broker, so
  it warrants more visibility than an ordinary Trade-Policy wait state.
- **Bounded retention**: current disposition record only, by default,
  matching the architecture's own `DecisionFrame` retention discipline —
  no unbounded history of past dispositions is required by this contract.
- **Failure diagnostics for emergency remediation** (section 12's
  unprotected-fill case, section 13's emergency-exit case) must log with
  enough detail (confirmed fill price/quantity, attempted stop parameters,
  actual platform error if any) to reconstruct what happened after the
  fact — these are the two cases in this entire document where "log it and
  move on" is not sufficient without also flagging loudly, since they
  represent a live position that briefly existed in an unintended risk
  state.

------------------------------------------------------------------------

# 18. Worked examples

1. **Ordinary success.** Slot 1's Composer publishes `ProposedActionSeq =
   9`, passes identity/gate/expiry, section 6's snapshot-phase finds no
   other slot manager-ready this cycle, section 8's revalidation passes
   (flat, no incompatible orders), section 9's drift re-check passes. The
   **decision record** publishes `OrderMgmtDispositionType =
   EXECUTABLE_INSTRUCTION` (`OrderMgmtDecisionSeq` advances) for slot 1's
   proposal; the **execution record** is simultaneously pinned with slot
   1's identity (chart/study/`ProducerTypeID`/`ProposedActionSeq`, fixed
   from this point on) and `ExecutionOutcome = SUBMITTED_PENDING_FILL`,
   before the fill is even known. Entry submitted (market order,
   `SCT_TIF_GOOD_TILL_CANCELED`, section 7). Stop attaches. Fill reaches a
   terminal state (section 12) at the requested quantity; section 13 finds
   `AdverseDistanceIncreaseTicks` at or below zero (the fill landed at or
   better than the planned distance, plain price-grid arithmetic, no
   currency conversion — Revision 6). Section 11's table is
   published: PV 20/21-25/34-38/52/53/54 written first, PV 43 incremented
   **last**. The execution record's `ExecutionOutcome` transitions to
   `FILL_CONFIRMED_HANDOFF_PUBLISHED` (`ExecutionOutcomeSeq` advances; the
   decision record's `OrderMgmtDecisionSeq` does not — it already recorded
   its own decision at submission and is untouched by this record's own
   later transition). On the next `K_Util_Candle_Baton_Manager_V1`
   evaluation, its own already-working sync logic detects the new leader ID
   and takes over — no code in that file changed to make this happen.
2. **Arbitration loss — snapshot-based, log-only, never overwrites the
   winner's decision record.** Slot 1 and slot 3 both look manager-ready.
   Section 6's collect phase evaluates both against one snapshot taken
   before any submission this cycle; slot 1 wins (configuration order).
   Slot 3's loss is evaluated from *that same snapshot*, not re-evaluated
   after slot 1 has already submitted (the race an earlier draft of this
   design had) — but it is **logged only** (`ReasonCode = ARBITRATION_LOST`,
   section 17), never published to the decision record, since that record
   already publishes slot 1's `EXECUTABLE_INSTRUCTION` this same cycle and
   a single sticky slot cannot represent two simultaneous outcomes (section
   6's own fix, and this section's own two-record design — an earlier draft
   of this example wrongly showed slot 3 being *published* as
   `NO_INSTRUCTION`/`ARBITRATION_LOST`, which the single decision record
   could never actually let a consumer observe before slot 1's own
   `EXECUTABLE_INSTRUCTION` overwrote it in the same evaluation). Slot 3's
   `ProposedActionSeq` is consumed (not retried) — a genuinely new
   opportunity for that strategy requires its own Composer to produce a new
   `ProposedActionSeq` later.
3. **Price drift at submission.** Slot 2's proposal passed the Composer's
   own pre-fill drift gate one cycle ago; by the time this component
   evaluates it, price has moved further and now exceeds this document's
   own submission-time tolerance. `OrderMgmtDispositionType =
   NO_INSTRUCTION`, `ReasonCode = PRICE_DRIFT_EXCEEDED_AT_SUBMISSION`. No
   order submitted, no master-handoff PV touched.
4. **Submission rejected by the platform.** Order sent, broker/platform
   returns a synchronous rejection (e.g. insufficient margin). A concrete
   instruction *was* produced and attempted, so
   `OrderMgmtDispositionType = EXECUTABLE_INSTRUCTION`, `ExecutionOutcome =
   REJECTED_AT_SUBMISSION`, `ReasonCode = SUBMISSION_REJECTED`. In-flight
   marker cleared, no master-handoff PV block written. The
   `ProposedActionSeq` is not retried.
5. **Timeout with no fill.** Order submitted (`EXECUTABLE_INSTRUCTION` /
   `SUBMITTED_PENDING_FILL` recorded at submission), sits unfilled past the
   configured timeout. Cancelled. No PV block was ever written (deferred
   publication, section 11), so the candle manager never saw anything.
   `ExecutionOutcome` transitions to `CANCELLED_TIMEOUT`, `ReasonCode =
   SUBMISSION_TIMEOUT`.
6. **Partial fill — remainder actively cancelled before handoff, not
   silently handed off mid-exposure.** Requested quantity 4; 3 fill,
   1 remains working. Per section 12's ownership rule, the Entry Authority
   requests cancellation of the remaining 1 immediately and waits for
   confirmation (not treating 3-of-4 as terminal on its own). Once the
   cancellation confirms (or the last unit also fills, whichever the
   platform reports), the order is terminal at whatever quantity actually,
   finally resulted — say 3. Section 11's PV 53 is written as 3, never
   speculatively written the moment the first partial fill was observed.
   Section 13's slippage check uses the confirmed, terminal 3.
7. **Post-fill slippage beyond tolerance, stop tightened.** Fill price lands
   materially past the planned distance from `ProposedActionStopPrice` —
   e.g. planned distance 12 ticks, actual fill-to-stop distance 19 ticks.
   Section 13 computes `AdverseDistanceIncreaseTicks = 19 - 12 = 7` (plain
   price-grid arithmetic, no currency conversion — Revision 6), beyond the
   documented tick tolerance; the tighten remedy is applied (stop distance
   reduced, never increased) before section 11 publishes — the candle
   manager syncs onto the
   *already-reconciled* stop, never the original indicative one.
8. **Stop attach fails after a confirmed fill — emergency remediation,
   stated as what is actually achievable.** Entry fills; the intended stop
   order itself fails to attach (a distinct platform error). The position
   necessarily exists briefly with no working stop — this example does not
   claim otherwise. Section 11's table is **not** published while this is
   unresolved. A best-effort emergency stop is attempted immediately; if it
   cannot be confirmed within the bounded retry window (open question 3),
   an emergency flatten follows instead of retrying indefinitely.
   `ExecutionOutcome = EMERGENCY_EXIT_POST_FILL` if the flatten path is
   taken. Logged loudly per section 17 either way.
9. **Fill crosses through the approved stop — directional validity gate,
   not an ordinary slippage case (new in Revision 7).** `LONG` proposal,
   `ProposedActionStopPrice = 4488.00`. A fast adverse fill confirms at
   `AverageFillPrice = 4485.00` — *below* the approved stop. Step 1's raw
   distance formula alone would report `|4485.00 - 4488.00| / TickSize`,
   a small magnitude that could even read as "favorable" once compared
   against the planned distance — masking the real problem. Section 13's
   step 0 directional validity gate catches this first: `LONG` requires
   `ProposedActionStopPrice < AverageFillPrice`, which is false here
   (`4488.00 < 4485.00` is false). Steps 1-4 are never reached.
   `ReasonCode = POST_FILL_STOP_INVALID_SIDE`; section 12's emergency-
   remediation mechanism runs — a best-effort attempt to replace the stop
   with a freshly computed, correctly-sided one against the actual
   4485.00 fill, or, if that cannot be confirmed within the bounded retry
   window, an immediate emergency flatten
   (`ExecutionOutcome = EMERGENCY_EXIT_POST_FILL`). Section 11's table is
   not published until whichever path resolves.
10. **Recalculation never touches a live order.** This component enters
   `IsFullRecalculation` while an entry is in flight at the broker. Per
   section 15, no new submission occurs and no master-handoff write
   happens during recalculation — but the in-flight marker's *resolution*
   (fill or rejection, once it actually occurs) is not abandoned; it is
   picked up and processed once normal evaluation resumes.

------------------------------------------------------------------------

# Unresolved design questions — deliberately left open

Only questions that affect real implementation. Every upstream contract's
own already-settled decisions are not reopened here.

1. **Should the Entry Authority get a `K_Util_*` name (Order Management
   family) or a `K_Chakra_*` name** (section 1), given the user's own prior
   directive exempted Order Management from the Chakra naming convention
   before this document's central finding (a second physical component)
   existed? **Until resolved, this document refers to it only descriptively
   ("the Entry Authority")** — a concrete implementation cannot be started
   without the user deciding this, since it is a naming-convention question
   this document cannot answer on the user's behalf.
2. **What is the Entry Authority's own current-price reference source and
   submission-time drift tolerance** (section 9)? Directly inherits
   EntryComposer's own open question 1 one stage further downstream — no
   prior Chakra contract needed a live, ungated price read before this
   pipeline reached the Composer, and now a *second* independent read is
   needed here. **Until resolved, section 9's gate cannot be concretely
   implemented** — a concrete Entry Authority must pick and document a
   specific ACSIL price source and tolerance constant, distinct from (but
   at least as strict as) the Composer's own, before shipping.
3. **What are the concrete emergency-stop parameters and bounded retry
   window** (section 12) when the intended stop order itself fails to
   attach after a confirmed fill? The *requirement* — immediate detection,
   no handoff-until-protected-or-flattened, best-effort protection attempted
   immediately, emergency flatten if that cannot be confirmed within a
   short bound — is fixed and non-negotiable; the specific fallback
   distance/price and the exact retry-window bound are not designed here.
   **Until resolved, a concrete implementation must supply its own
   documented emergency-stop policy** before this component can be
   considered safe to run live.
4. **Does a future multi-leg entry (splitting one Composer's approved
   quantity across more than one stop leg at entry, rather than the single
   stop this document designs) ever get built**, and if so, does section
   11's PV-34-only-populated table need to populate PV 35-38 as well? Not
   needed by anything upstream today (PositionSizer/StopManager both
   currently produce one quantity/one stop geometry per candidate) — flagged
   in case a future PositionSizer extension changes that.
5. **Does the arbitration rule in section 6** (fixed configuration order,
   first-manager-ready-wins), scoped to the slots feeding *one*
   manually-configured Entry Authority instance, ever need to become a
   more elaborate priority policy rather than plain configuration order? A
   true cross-account/cross-symbol **portfolio**-arbitration layer is
   excluded outright by this project's scope boundary (Chakra does not own
   portfolio or account-level risk management — "Changes from Revision 3"),
   not a possible future extension of this document; this question is only
   about whether same-instance slot priority ever needs to be more than
   fixed configuration order. Directly inherits EntryComposer's own open
   question 5, now resolved only as far as "some deterministic rule must
   exist and this is the simplest one" — a documented, deliberate default,
   not a claim of optimality.
6. **How does a future `TrailingManager`/`ProfitManager` (open-position
   Trade Policy phases, entirely out of scope for both this document and
   every TradePol contract so far) eventually feed proposed changes to an
   *already-open* trade back through Order Management?** This document only
   designs the entry handoff; post-entry Chakra-originated proposals (a
   trailing-stop adjustment, a target change) are a distinct, not-yet-designed
   extension — `K_Util_Candle_Baton_Manager_V1`'s own existing internal
   trail/target strategies remain the only mechanism for an already-open
   trade today, per this document's own scope.

Do not hide an implementation blocker inside prose above — each item states
plainly what remains unavailable until it is resolved.

------------------------------------------------------------------------

# Hostile review performed before Revision 1's publication

This document's design went through an adversarial pass against every
upstream contract and the actual manager source before being written up in
final form above (not after, as a separate visible revision) — the
findings below are recorded so a future reader understands *why* the
design looks the way it does, rather than treating these as arbitrary
choices:

1. **The PV-namespace collision (section 2) is real and would have broken
   the contract silently if missed.** A first-pass design that let one
   physical study instance be both the master-handoff publisher and a
   standards-compliant Chakra envelope producer would have PV 20 meaning
   two different things simultaneously — caught by cross-referencing the
   envelope's own PV-ownership table (20-49 public payload) directly
   against the legacy design doc's own reserved-PV list (18-76), which
   overlap almost completely. Resolved by requiring two persistent-storage
   instances rather than inventing a per-producer numeric exception.
2. **The single-live-trade constraint (section 6) is real, found by reading
   the manager's own PV semantics, not assumed.** `mainChartTradeDirection`/
   `mainChartTradeQuantity` (PV 52/53) and the leader ID (PV 43) are each a
   single scalar, not a per-leg array — the "up to 5" pattern elsewhere in
   this file is for scale-out legs of *one* trade, never for 5 independent
   concurrent trades. A design that treated multiple simultaneous
   Composer-sourced proposals as reconcilable-by-side-or-size (one candidate
   reading of EntryComposer's own open question 4) would have been
   incompatible with the actual data model. Resolved with a strict
   mutual-exclusion arbitration rule instead (section 6).
3. **Deferred master-handoff publication until fill confirmation (section
   11) was checked against the candle manager's own sync logic
   specifically** (`shouldSyncNewTrade`, `flatPositionReadCount` handling)
   to confirm publishing only a fully-confirmed trade introduces no edge
   case that code doesn't already handle — it doesn't; the manager already
   tolerates re-reading a flat/zero-order state indefinitely before a
   sync, so nothing needs to change there either.
4. **Checked that `sc.GetPersistentInt64`/`GetPersistentInt64FromChartStudy`
   actually exist in the installed `sierrachart.h`** before relying on the
   envelope's int64 PV convention for this document's own disposition
   record — confirmed present (matching this project's own established
   discipline of verifying an ACSIL API claim against the installed header
   before depending on it, as the envelope contract's own Revision 5
   changelog already did for `sc.GetCurrentDateTime`/`sc.BaseDateTimeIn`).
5. **Considered and rejected treating `K_Util_Candle_Baton_Manager_V1`
   itself as extensible with new entry-submission logic**, instead of
   introducing a separate Entry Authority. Rejected because it would
   reverse that study's own explicit, already-documented, already-shipped
   non-goal ("no order entry logic... slave/follower manager only") — a
   deliberate prior design decision, not an accidental gap, and not this
   document's decision to reverse unilaterally.

No contradiction was found between this document's design and any
already-approved upstream contract's own settled decisions (EntryComposer
Revision 2, StopManager Revision 3, PositionSizer Revision 4, EntryGate
Revision 4, envelope Revision 8) — only the two additive upstream
corrections already listed above, and the findings above, which are
internal to this document's own design rather than corrections to anything
upstream.
