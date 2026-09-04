# Chakra — Trade Policy: Entry Composer Interface Contract (Draft, Revision 7)

Status: **design only, no ACSIL implementation yet.**

Builds directly on:
- `Algo_Overlay_Framework_Architecture.md`, including the master decision
  cycle, `POLICY_DECISION`, `PROPOSED_ACTION`, and bounded `DecisionFrame`
  rules.
- `Chakra_Common_Interface_Envelope_Contract.md` (approved, Revision 8).
- `Chakra_TriggerStr_Interface_Contract.md` (Revision 3).
- `Chakra_TradePol_EntryGate_Interface_Contract.md` (Revision 4 — this pass
  required one small additive correction to that document; see "Upstream
  correction" below).
- `Chakra_TradePol_StopManager_Interface_Contract.md` (Initial Risk
  Geometry, Revision 9 — amended alongside this document to clarify the
  tick-to-price-unit conversion this document's own publisher relies on;
  see that document's own "Changes from Revision 8").
- `Chakra_ModeBatonSignalPublisher_Interface_Contract.md` (Revision 1) —
  this document's own terminal step as of Revision 6; see "Changes from
  Revision 5," below.

**`Chakra_TradePol_PositionSizer_Interface_Contract.md` is not a
dependency of this document as of Revision 7.** It is not merely
superseded-with-a-banner elsewhere — as of this revision, this document's
own normative sections no longer reference `SizedCandidate`, PositionSizer
identity, or quantity correlation anywhere. See "Changes from Revision 6,"
immediately below, for the full removal.

**This document designs only the pre-entry Entry Composer** — the
component every prior Trade Policy contract has referred to only as "the
Entry Composer," a reserved placeholder name (EntryGate contract, section
4). Nothing else changes scope: `TrailingManager`, `ProfitManager`, and an
eventual open-position phase for `StopManager` all remain out of scope
here. There is no Order Management component in the v1 live path at all
(see "Changes from Revision 5" and "Changes from Revision 6," below).

------------------------------------------------------------------------

# Changes from Revision 6 (normative cleanup — SizedCandidate/PositionSizer removed from the live path, 2026-07-25)

**A direct correction, not a review round: Revision 6 added a banner and a
changelog entry pointing `ProposedActionSeq` at the new Mode Baton Signal
Publisher, but left the entire normative body of the document — the
correlation invariant, the pending-assembly state machine, the delivery
record, the worked examples — still describing a **three**-input assembly
(`EntryCandidate` + `SizedCandidate` + `StopGeometry`). That is not what
the live v1 path is. Mode Baton owns quantity for mode 25 (source-audit
finding, "Changes from Revision 5" above); there is no reason for this
Composer to wait on, correlate against, or carry forward a `SizedCandidate`
it has nowhere to send. This pass rewrites every normative section so the
document actually matches the two-input architecture it claims to
implement, rather than layering a correction on top of a design that still
structurally requires the removed component.**

**Live v1 input, restated plainly:**

```text
EntryCandidate (EntryGate) + StopGeometry (StopManager)
    -> Entry Composer
    -> Mode Baton Signal Publisher (Chakra_ModeBatonSignalPublisher_Interface_Contract.md)
```

**Every normative section below (Core responsibility through the open
questions list) is rewritten, not merely amended, to reflect this.**
Sections "Changes from Revision 1" through "Changes from Revision 5"
below this one are **preserved verbatim as historical record** — they
describe what was true and correctly fixed at the time, including the
three-input composite-correlation design this revision now removes. Where
a historical section's content has since been superseded outright rather
than merely built upon, that is stated in this section, not by editing the
historical text itself:

- The composite hard-correlation invariant (`EntryGateIdentity(SizedCandidate)
  == EntryGateIdentity(StopGeometry) == ...`, "Changes from Revision 1"
  item 2) is **narrowed from a three-way to a two-way comparison**
  (`EntryCandidate` vs. `StopGeometry` only) — the underlying lesson
  (bare `CandidateSeq` is not a safe correlation key; composite identity
  is required) still stands and still governs the remaining two-way
  check, unchanged in kind.
- Every PositionSizer-specific field in the decision record and every
  `SizedCandidate`-specific field in the delivery record (both introduced
  across Revisions 1-4) is **retired, not deleted from history** — left
  unused in the enum/PV tables below with an explicit retirement note,
  the same convention already used for the dollar-risk fields Revision 4
  retired.
- Worked examples describing size arrival, size mismatch, or size-vs-
  geometry ordering are removed from the live examples list (section 16)
  — they describe a scenario that can no longer occur, not a scenario
  this document still needs to handle correctly.
- The price-drift source/tolerance and wait-for-refresh-viability
  questions (now questions 1-2 in the renumbered list) are otherwise
  unaffected by this pass — they were never about PositionSizer. The
  profit-target question (now question 4) and the arbitration question
  (now question 3) are separately updated to reflect the Mode Baton
  Signal Publisher's own new role — see the open-questions list itself.

------------------------------------------------------------------------

# Changes from Revision 5 (Mode Baton integration boundary, 2026-07-25)

**Direct consequence of a source audit of `K_Strat_Bhaskar_V1.cpp`,
`K_Strat_Bhaskar_V2.cpp`, and `K_Util_ModeBaton_V1.cpp`, not a review
round.** The audit found that Order Management's greenfield Entry
Authority design (the consumer this document's `ProposedActionSeq` was
built to feed) is unnecessary — `K_Util_ModeBaton_V1.cpp` already performs
order submission, fill handling, and attached-stop management, live and
proven. See `Chakra_OrderManagement_Handoff_Interface_Contract.md`'s own
new superseded banner.

1. **Section 12 ("Downstream consumption") is corrected: `ProposedActionSeq`
   no longer terminates at an Entry Authority.** It terminates at
   `Chakra_ModeBatonSignalPublisher_Interface_Contract.md`, which converts
   this document's own proposal content (reference price, stop geometry,
   resolved side) plus two genuinely new computations (target prices,
   invalidation prices — neither existed anywhere upstream before Revision
   1 of that document) directly into Mode Baton's `PV51`-`PV74` record.
   Every event-pattern consumption rule section 12 already specified
   (baselining, sequence-gap detection, expiry, consume-and-drop,
   independent lifecycle revalidation) is unchanged — only *who* consumes
   changes, not *how*.
2. **`ProposedActionEntryStyle` (section 8, `PV39`) is now moot for the
   version-one live path.** Mode Baton's mode-25 entry is always a market
   order regardless of what this field says (source-audit finding); the
   field is retained in this document's own delivery record for schema
   stability but the publisher does not read it.
3. **Quantity (`ProposedActionQuantity`, section 8) is now moot for the
   version-one live path.** Source audit confirmed Mode Baton's mode-25
   sizing path never reads the published quantity field (`PV53`) —
   position size is controlled entirely by the consuming Mode Baton
   instance's own local configuration. This document's own quantity field
   is unaffected (still populated from whatever upstream source this
   Composer is wired to, if any), but the publisher writes `PV53 = 0`
   regardless — see `Chakra_ModeBatonSignalPublisher_Interface_Contract.md`
   section 8.
4. **No change to sections 1-11, 13-16, or the audit/delivery-record
   separation.** The correlation-identity, expiry, drift-gate, and
   recalculation rules this document already established are unaffected —
   they govern the input side of the new publisher exactly as they
   governed the old Entry Authority's input side.

------------------------------------------------------------------------

# Upstream correction: `PolicyCategory` catalog was missing this role

**Identified, not silently worked around.** `K_CHAKRA_TRADEPOL_CATEGORY`
(defined in the EntryGate contract, section 1) is the canonical,
TradePol-layer-wide catalog every sub-component contract reuses rather
than redefining. It reserved five values
(`ENTRY_GATE`/`POSITION_SIZER`/`STOP_MANAGER`/`TRAILING_MANAGER`/
`PROFIT_MANAGER`) but never reserved one for the Entry Composer itself,
because the Composer didn't have its own contract pass yet when that enum
was written.

**Smallest necessary upstream correction, made directly in the EntryGate
contract** (now Revision 4): added `K_CHAKRA_TRADEPOL_CATEGORY_ENTRY_COMPOSER
= 6`, purely additive — a new value appended to the end, no existing value
renumbered or reinterpreted, nothing else in EntryGate's own contract
touched. **Reusing an existing category (e.g. `ENTRY_GATE`, on the
reasoning that the Composer consumes EntryGate's output) was considered
and rejected** — the Composer is a structurally distinct role with its own
identity, own decision enum, and own delivery record, exactly the same
reasoning that gave `PositionSizer` and `StopManager` their own categories
rather than folding them into `ENTRY_GATE`. See the EntryGate contract's
own "Changes from Revision 3" section for the full note.

------------------------------------------------------------------------

# Changes from Revision 1

Four material issues from a review pass — two blocking correctness bugs,
one design gap requiring smallest-necessary upstream corrections to
PositionSizer and StopManager, and one internal contradiction — plus one
implementation blocker made explicit rather than left implicit. The
bounded pending slot, triad-matching concept, expiry chain, drift gate,
audit/delivery split, latest-wins behavior, and Order Management boundary
were all confirmed correct and are not reopened.

1. **The candidate gate used `>` instead of `!=`, silently breaking
   storage-reset detection (blocking).** Section 2 correctly used
   `CandidateSeq != LastConsumedEntryCandidateSeq`, but section 4's own
   pseudocode switched to `CandidateSeq > LastConsumedEntryCandidateSeq` —
   under `>`, a producer storage reset to a *lower* sequence value can
   never be observed as "something changed," the exact bug class already
   fixed once in the envelope contract's own gap-detection pattern. Fixed:
   section 4 now uses `!=` throughout, with explicit forward/backward/equal
   branching (forward = ordinary new candidate, with gap logging;
   backward = probable storage reset, abandon pending assembly and
   re-baseline without consuming; equal = nothing new) — see section 4.
2. **`CandidateSeq` alone is not a globally safe correlation identity
   (blocking).** The hard correlation invariant (section 3) compared only
   bare sequence numbers. After a rewiring event or producer
   reconstruction, two *different* EntryGate instances can independently
   arrive at the same numeric `CandidateSeq` — the Composer could
   numerically "match" a `SizedCandidate` and a `StopGeometry` that
   actually came from different EntryGate producers. Fixed: correlation is
   now a **composite identity** — `SourceEntryGateChartNumber`/
   `...StudyID`/`...ProducerTypeID`/`...CandidateSeq` together (section 3).
   **This required a smallest-necessary upstream correction to two
   already-approved documents**, since the composite identity has to live
   in the *delivery* records, not the overwriteable audit records:
   `Chakra_TradePol_PositionSizer_Interface_Contract.md` bumped to
   Revision 4 (added the composite EntryGate identity to the
   `SizedCandidate` delivery record) and
   `Chakra_TradePol_StopManager_Interface_Contract.md` bumped to Revision 3
   (same addition to the `StopGeometry` delivery record) — see each
   document's own "Changes from Revision N" section. Neither correction
   touches anything else in those contracts. **Related correction**:
   rewiring of the Composer's own EntryGate source mid-assembly now
   abandons the pending assembly outright and re-baselines, rather than
   continuing to wait on the same numeric `CandidateSeq` from what is now a
   different producer (section 4) — rewiring of PositionSizer or
   StopManager alone does not have this effect, since the EntryGate
   candidate being assembled hasn't itself changed.
3. **Trigger provenance was claimed to be cross-validated across the triad,
   but the delivery records it would need don't carry that data
   (blocking).** Section 3 (Revision 1) said the Composer validates
   Trigger event type/sequence agreement across `EntryCandidate`,
   `SizedCandidate`, and `StopGeometry` — but PositionSizer's and
   StopManager's own self-contained delivery records never carried Trigger
   provenance at all; it exists only in their **decision** records
   (overwriteable audit trails that may already describe a later,
   unrelated decision by the time the Composer reads them). Validating a
   sticky delivery record by reaching into a different, mutable audit
   record is exactly what the audit/delivery split exists to prevent.
   **Resolved by choosing one of the two options the review offered**:
   Trigger provenance is now sourced solely and authoritatively from the
   matching `EntryCandidate`'s own self-contained copy — never cross-checked
   against PositionSizer's/StopManager's own decision-record copies — and
   the composite EntryGate identity (item 2) is the sole correlation
   mechanism (section 3). `ProposedActionSourceTriggerEventSeq` was also
   added to the proposed-action delivery record itself (section 8), so
   Order Management doesn't need to reach anywhere for this either. **Note:
   EntryGate's own candidate record only self-contains
   `CandidateSourceTriggerEventSeq`, not an `EventType` field** — a
   pre-existing gap this pass surfaces but does not fix (also latent in
   PositionSizer's/StopManager's own decision records, out of this pass's
   scope). Rather than carry an unverifiable value forward, this contract's
   own records carry `SourceTriggerEventSeq` only, no `EventType` field —
   see section 3.
4. **Recalculation behavior was internally contradictory.** Section 13
   said private trackers "re-baseline" at recalc-entry (implying the live
   gate would see no difference and report nothing new) and also that an
   in-progress candidate is "re-observed as a fresh candidate" after
   recalculation (implying the opposite) — both cannot be true, and
   Revision 1 never said which one governs. Fixed: baselining wins,
   explicitly. Recalc-entry abandons any pending assembly, baselines every
   tracked sequence to the *current* value without consuming it, and waits
   for a genuinely newer value via the normal live gates afterward — a
   candidate already sitting in a producer's slot at the moment of
   baselining is never treated as newly actionable (section 13).

**Implementation blocker made explicit, not smoothed over**:
`ProposedActionEntryStyle = NONE` is complete only with respect to this
contract's own scope (candidate/size/geometry, correlated and validated) —
it is not necessarily manager-ready. Before implementation, either the
Composer must receive an authoritative entry intent/order style from some
not-yet-designed upstream source, or Order Management must adopt an
explicit policy converting `NONE` into a defined order form. Section 8 and
section 11 both now state this directly rather than letting "complete"
read as "unconditionally executable" (open question 4 at the time this
paragraph was written, before this document's own account/instrument open
question was later removed and everything after it renumbered down by
one — this is now open question 3, and resolved: see that entry).

------------------------------------------------------------------------

# Changes from Revision 4 (independent review-correction pass, 2026-07-19)

Four findings from a review of the Revision 4 scope-correction pass:

1. **A real gap in section 6's own "corrected source" fix, mirroring
   EntryGate's own "Changes from Revision 6" fix.** Section 6 attributed
   "no incompatible pending or working entry" to the corrected platform
   source, but the platform cannot see an unsubmitted, "pending"
   Chakra-internal proposal — only actual working orders. Fixed: the
   check is narrowed to "no incompatible working entry"; an in-flight
   submission conflict is exclusively the Entry Authority's own
   idempotency check.
2. **Section 3's own hard-correlation-invariant mismatch list still named
   "instrument/account" mismatches as a producer-contract violation to
   detect**, even though this same document's Revision 4 pass had already
   established (a few paragraphs earlier in the same section) that no
   upstream record publishes a comparable instrument/account value.
   Fixed: removed from the mismatch list, with a pointer back to the
   earlier correction so the two don't drift apart again.
3. **Three open questions (3, 5, 7) were stale, not actually open** — each
   is resolved by `Chakra_OrderManagement_Handoff_Interface_Contract.md`
   (entry style default, post-fill remediation ownership, and the
   Composer-to-candle-manager physical relationship via the Entry
   Authority, respectively), but this document's own list still described
   them as unresolved. Fixed: marked resolved in place, kept in the list
   (not deleted/renumbered) so a reader sees the resolution and its source
   directly.
4. **Multiple inline cross-references to this document's own open
   questions, scattered through the prose outside the open-questions list
   itself, were stale by one or two — a lingering effect of the account/
   instrument question's removal (its own Revision 3 pass) shifting every
   later number down by one, which the prose citations were never
   individually re-checked against.** Found and fixed at every site: the
   `ProposedActionEntryStyle` implementation-blocker note (was "open
   question 4," now correctly "3," with a note on why the number moved);
   the current-price-source paragraphs (section 7, two sites, were "open
   question 2," now correctly "1"); the wait-for-refresh paragraph (was
   "open question 3," now correctly "2"); the post-fill-remediation-inputs
   paragraph (was "open question 6," now correctly "5"); the arbitration-
   fallback paragraph (section 15, was "open question 5," now correctly
   "4"). None of these were flagged by number in the earlier scope-
   correction passes — each is a pre-existing prose citation that drifted
   out of sync with the list itself, not a new mistake introduced by this
   pass.

------------------------------------------------------------------------

# Changes from Revision 3 (user-directed scope correction, part 2, 2026-07-19)

**A direct user correction — the first scope-correction pass (Revision 2,
below) closed the account/instrument-identity *question* but left stale
statements elsewhere in this document still describing identity checks as
"provisional pending" it, and left two dollar-risk fields in place that
depended on machinery `StopManager`/`PositionSizer` have since removed.**

1. **Section 3's "instrument identity"/"account identity" consistency
   checks removed, not just reworded.** These claimed to compare a value
   across `EntryCandidate`/`SizedCandidate`/`StopGeometry` — but none of
   those three records has ever published a concrete, comparable
   instrument or account field; there was never anything to compare.
   Correlation across the triad comes from manual wiring (one Composer
   instance, one EntryGate/PositionSizer/StopManager triad, section 1),
   not from a field comparison. Section 6's matching checklist bullet and
   section 8's "provisional" framing are corrected the same way.
2. **`ProposedActionStopRiskPerContract`/`ProposedActionAggregatePlannedStopRisk`
   retired (section 8; PV 3/4 left unused, not reassigned).** Both
   depended on `StopGeometryDistanceCurrencyPerContract`, which
   `StopManager`'s own Revision 5 removed — there is no longer a
   dollar-risk figure anywhere upstream of this document to copy or
   compose. The core-responsibility section's composition example is
   updated to match; StopManager's hard gate (its section 9) needs the
   *price* fields already published, not a risk figure, so removing these
   two does not weaken that gate.
3. **`K_CHAKRA_ENTRYCOMPOSER_REASON_INSTRUMENT_ACCOUNT_MISMATCH` (value 12)
   retired** — a reason code for a comparison this document no longer
   claims to perform. Value left unused, not reassigned.
4. **Where instrument/account consistency actually matters**: the Entry
   Authority, immediately before submission, against its own manually
   configured execution environment
   (`Chakra_OrderManagement_Handoff_Interface_Contract.md` section 8) —
   stated directly in sections 3, 6, and 8 below rather than left implicit.

**Nothing about the hard correlation invariant (composite EntryGate
identity), the bounded pending-assembly slot, the expiry chain, the price-
drift gate, the audit/delivery split, or the `PROPOSED_ACTION`/Order-
Management boundary changes.**

------------------------------------------------------------------------

# Changes from Revision 2 (user-directed scope correction, 2026-07-19)

**Not a review round — a project-wide scope boundary set directly by the
user, applied identically across every upstream TradePol contract in the
same pass (see `Chakra_OrderManagement_Handoff_Interface_Contract.md`'s own
"Changes from Revision 3" for the full statement).** Chakra is a
trade-decision framework; it does not own portfolio or account-level risk
management. The user configures the chart, instrument, and trade account
manually, and every Chakra component — including this one — operates
within that manually configured context. This cancels the previously
planned `Chakra_ExecutionContext_AccountInstrument_Interface_Contract.md`
and corrects the following here:

1. **Open question 1 ("what is the authoritative account/instrument
   identity source?") is closed, not carried forward.** This contract's
   correlation checks (section 3) and proposal content (section 8) were
   never actually blocked on a missing identity *source* — the Composer is
   wired, by configuration, to one specific EntryGate/PositionSizer/
   StopManager triad (section 1's "one instance, one coherent concept"
   rule), and that triad is itself wired to the manually configured
   chart/instrument/account. Nothing in this document needs a separate
   account/instrument service; where instrument/account fields appear in
   the proposal record, they are self-contained copies from the
   already-validated upstream chain, not a new lookup.
2. **Section 15's arbitration discussion named "a future
   portfolio/entry-arbitration component sitting above all Composer
   instances" as one of three candidate homes for cross-instance
   arbitration.** That option is now excluded outright, not merely
   undesigned — a portfolio-level arbitration layer is exactly the kind of
   account/portfolio-level infrastructure this project's scope boundary
   excludes. The other two candidates (a local, per-instance precedence
   rule; or Order Management's own existing safety nets) remain available,
   and the second of those is now **concretely resolved**, not just
   "plausible": `Chakra_OrderManagement_Handoff_Interface_Contract.md`
   section 6 defines exactly this — fixed configuration-order,
   first-manager-ready-wins arbitration across the Composer sources feeding
   one Entry Authority instance. Open question 5, below, is updated to
   point at that resolution rather than leaving all three options open.

**Nothing about the core boundary, the hard correlation invariant, the
bounded pending-assembly slot, the expiry chain, the price-drift gate, the
audit/delivery split, or the `PROPOSED_ACTION`/Order-Management boundary
changes.**

------------------------------------------------------------------------

# Core responsibility

**Frozen, rewritten in Revision 7 to drop `SizedCandidate` (see "Changes
from Revision 6" — quantity is not this pipeline's concern for v1; Mode
Baton owns it):**

> The Entry Composer assembles one complete, internally correlated entry
> proposal from an accepted `EntryCandidate` and approved initial
> `StopGeometry`, then forwards that `PROPOSED_ACTION` to the Mode Baton
> Signal Publisher. It does not discover setups, re-judge acceptance,
> compute quantity, compute stop geometry, or touch any order.

**The Entry Composer must not:**

- Discover triggers (TriggerStr's job entirely).
- Reconsider EntryGate's accept/reject judgment.
- Compute or carry a position-sizing decision — for the v1 live path,
  quantity is not part of this pipeline at all (Mode Baton's own local
  configuration governs size for mode 25; see
  `Chakra_TradePol_PositionSizer_Interface_Contract.md`'s own superseded
  banner). This is not merely "not this Composer's job" in the sense
  PositionSizer's job was previously described — for v1 there is no
  component in this chain that computes or publishes a quantity Mode
  Baton will honor.
- Calculate stop geometry (StopManager's job).
- Calculate profit targets, unless a future target contract explicitly
  supplies them as a fourth correlated input — not invented here. (The
  Mode Baton Signal Publisher computes target *prices* for `PV56`/`PV57`
  from the stop geometry's reference price and a configured tick offset —
  that is the publisher's own new responsibility, not this Composer's,
  and not a profit-target *policy* in the `ProfitManager` sense.)
- Submit, modify, or cancel any order.
- Claim an order was accepted or executed.
- Produce an `EXECUTABLE_INSTRUCTION` — that disposition does not exist
  in the v1 live path at all (no Entry Authority, no Order Management
  component); the Mode Baton Signal Publisher's own output is a PV write,
  not a disposition.
- Manage an open position.

**What the Composer *is* allowed to do with the numbers it assembles**:
combine already-computed values via simple, non-judgmental arithmetic that
introduces no new information — composition, never calculation of stop
geometry itself. No composition example currently exists in this
contract; the principle is stated for any future field that might need
it.

Naming: `K_Chakra_TradePol_EntryComposer_<DescriptiveName>_V<n>`.

**The Entry Composer is event/decision-style**, like every other Trade
Policy sub-component with a staged pipeline in front of it, and needs the
same two-record separation (audit decision trail vs. correctness-critical
delivery record) — see section 9. **As of Revision 7, it requires two
independently-published, cross-correlated inputs** (`EntryCandidate`,
`StopGeometry`) — reduced from three in Revision 6 and earlier, since
`SizedCandidate` is no longer part of the v1 live path.

------------------------------------------------------------------------

# 1. Naming and identity

- **`ComponentKind`** is always `K_CHAKRA_KIND_TRADEPOL`.
- **`PolicyCategory`** is `K_CHAKRA_TRADEPOL_CATEGORY_ENTRY_COMPOSER`
  (int32 PV 21, newly reserved above).
- **`ProducerTypeID`** is scoped to the Entry Composer specifically, same
  per-category scoping every other TradePol sub-component uses:
  ```cpp
  enum K_CHAKRA_TRADEPOL_ENTRYCOMPOSER_TYPE
  {
      K_CHAKRA_TRADEPOL_ENTRYCOMPOSER_TYPE_NONE       = 0,
      K_CHAKRA_TRADEPOL_ENTRYCOMPOSER_TYPE_ORBREAKOUT = 1,
      // one member per concrete named Entry Composer policy, assigned as each is built
  };
  ```
- **`PayloadSchemaVersion`** (int32 PV 20) scoped per `ProducerTypeID`,
  unchanged convention.

**One Entry Composer instance is wired to exactly one EntryGate source,
and to exactly the one `StopManager` instance that itself consumes that
same EntryGate's candidate (narrowed from an EntryGate/PositionSizer/
StopManager triad to an EntryGate/StopManager pair in Revision 7 — see
"Changes from Revision 6").** This is not a new rule invented here — it is
the same "one instance, one coherent concept" discipline every layer in
this framework already follows (one Sensor = one measurement family, one
MCtx = one context concept, one TriggerStr = one setup, one EntryGate =
one entry policy for one trigger concept). It is also load-bearing here
specifically: the hard correlation invariant (section 3) only has a
well-defined answer when the Composer already knows, by configuration,
which two producers' outputs are supposed to belong together — it does
not discover this correspondence dynamically, and does not arbitrate
among multiple unrelated EntryGate/StopManager pairs (section 15).

**Identity validation order** (TradePol layer, unchanged):

```text
SchemaVersion -> ComponentKind -> PolicyCategory -> ProducerTypeID
  -> PayloadSchemaVersion -> payload
```

Applied independently to each of the three configured source producers
before any of their payload fields are read (section 2) — not just to the
Composer's own identity.

------------------------------------------------------------------------

# 2. Inputs — two independently-published, event-style delivery records

**Narrowed from three to two in Revision 7** (see "Changes from Revision
6") — `SizedCandidate`/PositionSizer removed outright, not merely
de-prioritized. The Composer consumes two sticky delivery records, each
published by a different producer on its own schedule, each already fully
specified by its own contract. Nothing here changes either of them — this
section only states how the Composer reads them.

| Input | Producer | Gate | Correlation field(s) |
|---|---|---|---|
| `EntryCandidate` | EntryGate | `CandidateStatus != NONE && CandidateSeq != LastConsumedEntryCandidateSeq` | `CandidateSeq`, read from this Composer's own wired EntryGate chart/study — see section 3 |
| `StopGeometry` | StopManager | `StopGeometryStatus == ACTIVE && StopGeometrySeq != LastObservedStopGeometrySeq` | Composite: `StopGeometrySourceEntryGateChartNumber`/`...StudyID`/`...ProducerTypeID`/`...CandidateSeq` together (see section 3) |

Each input carries its own full discipline, unmodified from its own
contract:

- **Identity validation**, independently, before any payload field is
  read: `SchemaVersion -> ComponentKind = TRADEPOL -> PolicyCategory =
  {ENTRY_GATE|STOP_MANAGER} -> expected ProducerTypeID ->
  PayloadSchemaVersion -> payload`.
- **Lifecycle baselining, all four boundaries, independently per source**
  (the Composer's own initial attachment to that source, rewiring, the
  Composer's own `IsFullRecalculation`, the Composer's own disable ->
  re-enable) — the unified `slotBaselined` mechanism established since the
  envelope contract's Revision 6/7, applied here to two independently
  rewireable dependencies.
- **Storage-reset handling**: gap detection captures the prior sequence
  before mutating the tracking field, branching forward-gap-logged versus
  backward-movement-logged-as-probable-storage-reset — per source,
  independently, since each producer's storage can reset on its own
  schedule.
- **Expiry**: each record's own `...ValidUntilDateTime`, checked against
  the master cycle's own evaluation time — never `PertainsToDateTime`
  anywhere in this chain (section 5).
- **Source provenance**: each record's own chart/study/`ProducerTypeID`
  triad, retained and carried into the Composer's own decision record
  (section 9), not discarded.
- **Latest-wins behavior**: both are single-slot sticky records under
  their own producer's latest-(approved)-wins overwrite policy — the
  Composer must assume either can be silently superseded by its own
  producer between one Composer evaluation and the next, and detect this
  via the standard sequence-gap logging on its own tracking field, same as
  every other consumer in this framework.

**`PublicationSeq`, each producer's own general decision-audit sequence
(`DecisionSeq`/`GeometryDecisionSeq`), and arrival time are never used as
delivery identity for either input.** Only `CandidateSeq`/`StopGeometrySeq`
(the correctness-critical delivery keys each producer's own contract
already established) identify "is there a new record."

**Note on the `StopGeometrySeq` gate above**: unlike a classic one-shot
event consume, the Composer's own tracking field for it
(`LastObservedStopGeometrySeq`) exists only for gap-detection/storage-reset
bookkeeping on the producer's own sequence — it is **not** the mechanism
that decides whether the record is usable for the currently pending
candidate. That decision is the correlation check (section 3), re-evaluated
fresh every cycle against whatever the record currently, actually
contains — because it is a single-slot sticky record that can still be
waiting to update to match a still-pending candidate.

------------------------------------------------------------------------

# 3. Hard correlation invariant

**Narrowed from a three-way to a two-way comparison in Revision 7** (see
"Changes from Revision 6") — `SizedCandidate` removed from every side of
this check. **This is still the load-bearing rule of the entire contract,
and it must still be a composite-identity comparison, never a bare
sequence-number comparison** (the underlying lesson from the original
three-way design, "Changes from Revision 1," is unchanged in kind — only
the number of things being compared shrank):

```text
EntryGateIdentity(StopGeometry)
    == EntryGateIdentity(this Composer's own wired EntryGate source)

  where EntryGateIdentity(X) means the composite:
    (SourceEntryGateChartNumber, SourceEntryGateStudyID,
     SourceEntryGateProducerTypeID, SourceEntryGateCandidateSeq)
```

**`CandidateSeq` alone is not a globally safe correlation identity.**
After a rewiring event or a producer reconstruction, two *different*
EntryGate instances can independently arrive at the same numeric
`CandidateSeq` value — comparing only the bare integer risks the Composer
silently assembling a proposal from a `StopGeometry` that numerically
agrees but actually came from an unrelated EntryGate producer. The
composite identity above is what actually identifies "the same candidate."
This is why `StopGeometrySourceEntryGateChartNumber`/`...StudyID`/
`...ProducerTypeID` (StopManager contract, Revision 3) exists as a
self-contained field in that delivery record.

**This Composer's own wired EntryGate chart/study identity is fixed by
configuration (section 1), and its `ProducerTypeID` is confirmed at every
identity-validation pass (section 2)** — it is not discovered from the
`StopGeometry` record itself; that record is checked *against* it.

A proposal may only ever be assembled from a pair that satisfies this
exactly. **Never combine the latest active `EntryCandidate` and the latest
active `StopGeometry` merely because both happen to be non-`NONE`/`ACTIVE`
at the same evaluation — each is an independent single-slot sticky record,
updated by a different producer on a different schedule, and "both
currently hold something" proves nothing about whether they hold the
*same* something.** A mismatched pair is not a degraded or partial
proposal — it is not a proposal at all.

**Also validated for consistency, once the primary composite correlation
above holds** (mismatch on any of these is treated identically to a
composite-identity mismatch — see section 14):

- **Resolved side** — `CandidateResolvedSide == StopGeometryResolvedSide`.
  Both are supposed to be the same preserved value, never independently
  re-derived by any producer (EntryGate section 7, StopManager section
  4) — any divergence is a producer-side contract violation, not something
  to silently resolve by picking one.
- **Instrument/account identity is not a Composer-level check.** Neither
  `EntryCandidate` nor `StopGeometry` publishes a concrete, comparable
  instrument or account value — each upstream component simply operates
  within whatever chart/study it is manually wired to (per this project's
  own scope boundary: the user configures the chart, instrument, and
  trade account manually, and every component operates inside that
  configuration). Consistency across the pair is therefore guaranteed
  **by construction** — one Composer instance is wired to exactly one
  EntryGate/StopManager pair (section 1), both sharing the same manually
  configured chart context — not by comparing a field that does not
  exist. Claiming to validate "instrument identity" or "account identity"
  here would be documenting a check against data nothing publishes.
- **Original Trigger event identity/sequence — sourced solely and
  authoritatively from the matching `EntryCandidate`, never cross-checked
  against StopManager's own copy.** The `StopGeometry` delivery record
  does not carry Trigger provenance at all — `SourceTriggerEventSeq`/
  `SourceTriggerEventType` exist only in StopManager's own **decision**
  record (its audit trail, section 10 in that contract), which is
  single-slot, overwriteable, and may already describe a later, unrelated
  decision by the time the Composer reads it. Reaching into a mutable
  audit record to validate a sticky delivery record is exactly the
  mistake this framework's audit-vs-delivery split exists to prevent
  (section 9 below). The composite EntryGate identity (above) is the
  authoritative correlation mechanism; Trigger provenance is carried
  through purely for traceability, taken only from the `EntryCandidate`
  record's own self-contained field (`CandidateSourceTriggerEventSeq`,
  EntryGate contract section 10), and is never treated as a second,
  independent correlation check. **Note on `EventType` specifically**:
  EntryGate's own candidate record self-contains
  `CandidateSourceTriggerEventSeq` only, not an `EventType` field — a
  pre-existing gap in EntryGate's own contract this document does not fix.
  This Composer's own decision and delivery records **carry only
  `SourceTriggerEventSeq`/`ProposedActionSourceTriggerEventSeq`** (sections
  9/8) — no `EventType` field.
- **Relevant producer identities** — the `StopGeometry` record's own
  chart/study/`ProducerTypeID` matches what this Composer instance is
  actually configured to consume (section 1's one-pair-per-instance
  assumption) — this is the same composite check as the primary invariant
  above, not a separate one; a mismatch here means either a genuine
  candidate/geometry mismatch, or a rewiring the Composer hasn't
  baselined against yet (section 2) — either way, not a proposal.
- **Master-cycle provenance, where required** — if a concrete Composer
  policy chooses to require same-cycle provenance consistency (stricter
  than this contract's own default, which tolerates multi-cycle assembly —
  section 4), that choice must be explicit and documented per instance,
  not silently assumed.

**Disposition on mismatch — waiting, not dropping, unless the mismatch
itself proves the pending candidate can never complete:**

- If `StopGeometry` currently correlates to a candidate *other than* the
  one currently pending (an older, already-superseded `CandidateSeq`, or
  a not-yet-relevant future one) — that input is simply treated as **not
  yet matching** the pending candidate (section 4); nothing is dropped, no
  error is logged beyond ordinary diagnostic detail, the Composer keeps
  waiting.
- If the correlated `CandidateSeq` on `StopGeometry` belongs to a
  candidate that the Composer has already reached a terminal outcome for
  (expired, lifecycle-stale, superseded, or already proposed) — that
  input can never become usable for a new proposal; log it as a
  correlation mismatch (section 14) at most once per occurrence (not
  every cycle), and continue waiting on the actual pending candidate as
  normal.
- Side/Trigger-provenance mismatches **within an otherwise-`CandidateSeq`-
  matching pair** indicate a producer-side contract violation, not an
  ordinary waiting state — consume-and-drop the pending candidate
  outright, `SKIP_UNDECIDABLE`-equivalent (`ComposerDecisionType =
  CORRELATION_MISMATCH`, section 14), log with full detail, and never
  attempt to guess which of the disagreeing records is "right." **Not
  instrument/account — no upstream record publishes a comparable
  instrument or account value, so there is nothing to detect a mismatch
  in; consistency there comes from manual wiring, not a runtime check.**

This avoids accidental cross-candidate assembly by construction: nothing
is ever assembled from a pair that doesn't satisfy the invariant, and a
transient mismatch (the ordinary case, since the two producers publish
independently) is just an unresolved wait, never a reason to substitute a
different, non-matching record.

------------------------------------------------------------------------

# 4. Pending assembly state

**Private, per-Composer-instance mechanism — bounded, not a queue.**
Because one Composer instance is wired to exactly one EntryGate source
(section 1), and `CandidateSeq` is itself a single-slot sticky value, at
most **one** candidate is ever meaningfully "in flight" for a given
Composer instance at a time. This keeps the mechanism a small, fixed set
of private trackers, not an unbounded structure — **guaranteed delivery
and unbounded candidate queueing are explicitly not chosen here**,
consistent with every other layer in this framework.

**Private trackers** (namespace ranges per the envelope's PV-ownership
convention — int32 private 50-89, int64 private 30-59 — exact PV numbers
left to the concrete implementation, not pinned here):

- `LastConsumedEntryCandidateSeq` (int64) — the highest `CandidateSeq`,
  from this Composer's own wired EntryGate producer, for which this
  Composer has reached **any** terminal outcome (approved, expired,
  lifecycle-stale, superseded, or correlation-mismatch-abandoned). **The
  gate is `!=`, never `>`**: `CandidateStatus != NONE && CandidateSeq !=
  LastConsumedEntryCandidateSeq`. Using `>` would make a producer storage
  reset to a *lower* sequence permanently unobservable.
- `PendingCandidateSeq` (int64, `0` = nothing pending) — the specific
  `CandidateSeq` currently being assembled, if any, always understood
  together with this Composer's own wired EntryGate chart/study/
  `ProducerTypeID` identity (fixed by configuration, section 1) — the pair
  together, not `PendingCandidateSeq` alone, is "which candidate."
- `LastObservedStopGeometrySeq` (int64) — gap-detection/storage-reset
  tracking on the producer's own sequence (section 2), independent of
  matching status. **`LastObservedSizedCandidateSeq` is retired as of
  Revision 7** — there is no second producer to track (see "Changes from
  Revision 6").
- Two `slotBaselined` flags (int32), one per source producer (EntryGate/
  StopManager) — the unified attachment/rewiring/recalc/disable-reenable
  mechanism, applied per dependency. **Reduced from three flags to two in
  Revision 7** (the PositionSizer slot is retired, not merely unused).

**The Composer must be able to observe the two inputs in either order** —
candidate then geometry, or geometry then candidate (a producer publishing
ahead of the candidate it correlates to is possible whenever StopManager's
own retry/pending logic resolves a *previous* candidate's geometry on the
same cycle a *new* `EntryCandidate` first appears; not disallowed by its
contract). Concretely:

```text
Each cycle:

1. Gate: CandidateStatus != NONE && CandidateSeq != LastConsumedEntryCandidateSeq?
   If false, go to step 2 with whatever candidate (if any) is already pending.
   If true, capture previousSeq = LastConsumedEntryCandidateSeq BEFORE any
   mutation, then branch:

     FORWARD (CandidateSeq > previousSeq) -- the ordinary case:
       -> if PendingCandidateSeq != 0 and CandidateSeq != PendingCandidateSeq:
            the previously-pending candidate is SUPERSEDED (section 11) --
            log it, advance LastConsumedEntryCandidateSeq to the OLD
            PendingCandidateSeq (the newly-observed CandidateSeq becomes
            LastConsumedEntryCandidateSeq only once IT reaches its own
            terminal outcome), clear matched-geometry state.
       -> set PendingCandidateSeq = the new CandidateSeq.
       -> if previousSeq != 0 and CandidateSeq - previousSeq > 1, log a
          forward sequence gap -- an intermediate EntryGate candidate this
          Composer never got to evaluate at all (EntryGate's own
          latest-wins overwrite, not a drop by this Composer).

     BACKWARD (CandidateSeq < previousSeq) -- a probable producer storage
     reset on the wired EntryGate source:
       -> abandon any in-flight PendingCandidateSeq assembly outright --
          clear PendingCandidateSeq and all matched-geometry state to
          neutral, exactly as for an EntryGate rewiring event (below).
       -> log distinctly as a probable storage reset, not an ordinary gap.
       -> re-baseline LastConsumedEntryCandidateSeq = CandidateSeq WITHOUT
          treating the record now sitting in the slot as newly actionable
          this same evaluation -- baselining and consuming are never the
          same call, the rule already governing every attachment/rewiring
          boundary in this framework. A genuinely new candidate is only
          picked up on a later cycle, once CandidateSeq once again differs
          from the just-rebaselined LastConsumedEntryCandidateSeq.

     EQUAL: unreachable under the gate above (the gate already requires
     CandidateSeq != LastConsumedEntryCandidateSeq); listed only for
     completeness with this framework's other gap-detection descriptions.

2. Does the CURRENT StopGeometry record's own composite identity
   (StopGeometrySourceEntryGateChartNumber/...StudyID/...ProducerTypeID/
   ...CandidateSeq) match (this Composer's wired EntryGate identity,
   PendingCandidateSeq) -- the hard correlation invariant, section 3? If
   yes, it is "matched" for this cycle -- re-checked fresh every cycle, not
   cached as a one-time fact, since the sticky slot can itself move on to a
   different candidate later.
3. If matched: proceed to final validation (section 6).
4. If not matched: remain pending -- see "Waiting ends when" below.
```

**Waiting ends when:**

- **The matching component arrives** — proceed to section 6.
- **The earliest relevant expiry is reached** — section 5; consume-and-drop
  as `SKIP_EXPIRED`.
- **The candidate is superseded** under the declared latest-wins rule
  (step 1 above; also section 11) — the old `PendingCandidateSeq` is
  abandoned, not silently merged with the new one.
- **Lifecycle eligibility becomes false** (section 6, checked every
  pending cycle, not only at final validation) — consume-and-drop as
  `SKIP_LIFECYCLE`.
- **A required producer resets or rewires — treatment depends on which
  producer:**
  - **EntryGate itself** (this Composer's own primary candidate source)
    resets or rewires mid-assembly: **abandon the pending assembly
    outright**, exactly as for the backward-sequence storage-reset case in
    step 1 above. Re-baseline that `slotBaselined` flag and
    `LastConsumedEntryCandidateSeq` to the new/reset producer's current
    value without consuming; clear `PendingCandidateSeq` and all
    matched-geometry state to neutral. **Do not continue waiting on the
    same numeric `PendingCandidateSeq` against what is now a different
    EntryGate producer** — the composite identity that defined "which
    candidate" no longer holds the instant the wired producer's own
    chart/study identity changes underneath it.
  - **StopManager** resets or rewires mid-assembly: only that producer's
    own tracking re-baselines (`slotBaselined`,
    `LastObservedStopGeometrySeq`); the pending `EntryCandidate` itself is
    untouched (it isn't the producer that changed), and the pending
    candidate simply returns to "waiting on geometry," now from the
    newly-rewired producer, rather than being torn down. A rewiring on the
    geometry side alone does not invalidate which EntryGate candidate is
    being assembled.
- **The Composer is disabled or enters recalculation** — standard
  disable-entry/recalc-entry clearing (section 13): `PendingCandidateSeq`
  and all matched-state trackers reset to their neutral values; nothing
  live is emitted during recalculation regardless of what was pending
  before it started.

**Do not design this as an unbounded queue of pending candidates.** One
`PendingCandidateSeq` slot, replaced under supersession, is the entire
mechanism.

------------------------------------------------------------------------

# 5. Expiry

**The composed proposal's expiry can only tighten upstream expiry, never
extend it — the same transitively-bounded chain established by
StopManager (its section 11), one stage further. Reduced from three
upstream bounds to two in Revision 7** (see "Changes from Revision 6"):

```text
ProposedActionValidUntilDateTime
    <= min(
        EntryCandidateValidUntilDateTime,
        StopGeometryValidUntilDateTime
    )
```

Computed once, at the moment both inputs are matched and validation
(section 6) begins — using the master cycle's own evaluation time and each
record's own **availability**-anchored `...ValidUntilDateTime`, never
`PertainsToDateTime` (which does not apply anywhere in this chain — every
upstream producer in this pipeline already resolves availability-based
expiry, per TriggerStr section 15's original rule, inherited by every
layer since).

**While a candidate remains pending (section 4) and the geometry input has
not arrived yet**, expiry is still checked every cycle, using whichever of
the two `...ValidUntilDateTime` values are currently available (the
`EntryCandidate`'s own is always available the moment it's pending;
`StopGeometry`'s only once it has actually arrived and matched at least
once, even if a later evaluation finds the sticky slot has moved on to
something else). **If the current evaluation time exceeds any available
one of these, the pending assembly cannot complete — consume and drop,
`SKIP_EXPIRED`**, regardless of whether geometry has arrived yet.

**Expired components are consumed/dropped with the detecting Composer's
own reason code** (section 14) — never a downstream diagnostic borrowed
from EntryGate/StopManager, per the self-referential `ReasonCode`
principle already established throughout this framework.

------------------------------------------------------------------------

# 6. Final pre-proposal validation

**Immediately before emitting a proposal, independently revalidate —
never rely solely on EntryGate's or StopManager's own earlier validation,
exactly the "independent re-check at every consumption stage" rule this
framework has required since EntryGate's own Revision 3. Quantity checks
removed in Revision 7** (see "Changes from Revision 6" — there is no
quantity in the v1 live path to validate):

- Still flat, where required (platform order/position state, read
  directly for the manually configured symbol/account — never
  `K_Util_Candle_Baton_Manager_V1`, which has no pre-fill visibility; same
  corrected source as EntryGate's own section 6).
- No incompatible working entry (from this or any other candidate path) —
  same corrected source. **Not "pending" — the platform has no concept of
  an unsubmitted Chakra-internal proposal, so this Composer cannot check
  it. There is no idempotency check downstream of this document in the v1
  live path (no Entry Authority exists) — the Mode Baton Signal Publisher
  publishes a signal, and Mode Baton's own existing lifecycle/duplicate-
  entry handling (source-audited, not designed here) governs from that
  point.**
- No opposing exposure.
- Lifecycle/trade-synchronization state usable.
- Stop remains on the risk-correct side (StopManager section 8's
  construction-sanity check, re-verified independently rather than
  trusted from StopManager's own earlier pass).
- Stop distance and tick metadata remain valid.
- Both source records remain active and unexpired (section 5, re-checked
  at the moment of emission, not just during the pending-wait loop — time
  keeps passing right up to commit).
- Candidate/geometry correlation (section 3) — re-checked one final time
  immediately before commit, not just when matching was first detected,
  since a rewiring or supersession could theoretically land in the gap
  between "matched" and "about to emit."
- Resolved side agrees across both inputs (section 3).
- No duplicate proposal has already been emitted for this candidate — see
  section 9's overwrite/duplicate-suppression rule.

**Instrument/account identity is deliberately not one of these checks.**
The Composer has no comparable field to check it against (section 3,
above). There is no downstream component in the v1 live path that
performs an account/instrument submission-time check either — Mode Baton
itself is manually wired to one configured symbol/account, per this
project's own standing scope boundary; this document does not claim any
component re-validates that wiring at runtime.

**If any check fails: consume-and-drop the pending candidate, produce no
`ProposedAction`, log the specific failing check's own reason (section
14).** A failed final validation is not "revert to waiting" — the
candidate has reached a terminal outcome (section 4's "any terminal
outcome" definition), even though the input had, until this check,
appeared matched and current.

------------------------------------------------------------------------

# 7. Reference-price drift hard gate

**As of Revision 9 of the StopManager contract, StopManager's own former
hard gate (which required a downstream post-fill policy to exist before
its geometry could become part of an executable entry) is retired** — see
that document's own "Changes from Revision 8." This section is no longer
"the pre-fill half of a policy StopManager's contract requires" — it is
this Composer's own, independent, self-standing pre-publish safeguard: a
final sanity check that the price hasn't moved materially between when
StopManager computed its geometry and when this Composer is about to
publish. It exists on its own merits, not to satisfy an obligation from
another document.

**Before a matched `StopGeometry` becomes part of a `PROPOSED_ACTION`**,
the Composer compares StopManager's own indicative `StopGeometryReferencePrice`
against the Composer's own **current, authoritative, freshly-read
pre-submission price reference** — evaluated at proposal-emission time,
never cached from an earlier cycle. **This document does not invent or
assume a specific ACSIL price source** (e.g. last trade, bid/ask midpoint
on the master decision chart) — a concrete Composer implementation must
document exactly which one it uses and why; this is flagged as an
unresolved dependency (open question 1, corrected on review — this
paragraph previously cited the wrong number), not silently decided here,
since no prior Chakra contract has needed a live, ungated price read of
this kind before.

**Dispositions, in order of preference, each requiring an explicit,
documented choice per concrete Composer policy — never silently defaulted
without being stated:**

- **Within tolerance** — accept the existing, already-matched geometry and
  size as-is. This is the expected, ordinary case.
- **Outside tolerance, default disposition: reject.** Consume-and-drop the
  pending candidate, `ComposerDecisionType = PRICE_DRIFT_EXCEEDED`
  (section 14), no proposal emitted. Safest default — requires no further
  design, and matches this framework's general "don't guess, don't
  silently proceed" posture whenever a hard gate fails.
- **Outside tolerance, optional alternative: wait for a newly-correlated
  refresh.** A concrete Composer policy may instead choose to treat the
  candidate as still pending (section 4) rather than dropping it outright
  — waiting for `StopManager` to publish a **new** `StopGeometrySeq`,
  still correlated to the same `CandidateSeq`, before the candidate's own
  expiry. This requires `StopManager` to actually be configured to
  re-evaluate and republish for an already-approved candidate (its
  contract does not currently specify this as automatic behavior) —
  **flagged as an unresolved cross-component dependency if chosen** (open
  question 2), not assumed to work by default.
- **Adjust — narrowly permitted only under a documented, provably
  risk-non-increasing policy, never as a default.** The Composer must
  never recompute stop geometry or quantity itself (frozen prohibitions,
  above) — "adjust" here can only mean substituting the Composer's own
  freshly-read current price into the proposal's own drift-tracking fields
  (section 8) for downstream reference, never altering
  `StopPrice`/`StopDistanceTicks`/`Quantity` themselves. A concrete policy
  choosing this option must document, in its own contract or
  configuration, the specific proof that doing so cannot increase realized
  risk versus the already-approved geometry and size — absent that proof,
  this option is not available, and **reject remains the default.**

**Do not silently accept stale geometry** — the comparison above is
mandatory on every proposal, not a spot-check.

**Post-fill reconciliation is not attempted anywhere in the v1 live
path — not by this Composer, and not by any downstream Chakra
component.** The superseded Order Management Handoff design would have
performed a post-fill slippage check; that design is not being built (see
`Chakra_OrderManagement_Handoff_Interface_Contract.md`'s own superseded
banner). What actually happens after a signal is published: Mode Baton
executes and manages the trade using its own existing, tested behavior,
which — per the source audit — does not include any reference-vs-fill
reconciliation of its own either. **This document deliberately inherits
that gap rather than papering over it**: no Chakra component reconciles a
fill against the pre-submission reference price. The fields this section
publishes (`ProposedActionStopReferencePrice`, `ProposedActionDriftReferencePrice`,
`ProposedActionDriftAmount`, `PriceDriftToleranceIdentifier` — section 8)
exist for **traceability and this Composer's own pre-publish gate only**;
nothing currently reads them after publication.

**Tolerance is a code-versioned constant per concrete Composer/policy** —
`PriceDriftToleranceIdentifier` (section 8) names which documented,
code-versioned tolerance value applies, never a runtime-editable input,
per this framework's standing versioning rule — **unless the architecture
already defines an authoritative runtime tolerance source**, which it does
not currently (open question 1).

------------------------------------------------------------------------

# 8. Proposal content

**The `PROPOSED_ACTION` delivery record is genuinely self-contained**,
including for the correctness-critical composite correlation identity and
Trigger provenance: the Mode Baton Signal Publisher reads it without
needing to separately reach into EntryGate/StopManager, **and without
needing to reach into this Composer's own decision record either for
anything correctness-critical** — `ComposerDecisionSeq` is a shared,
overwriteable audit sequence, and by the time the publisher reads a
proposal, that decision record may already describe a later, unrelated
decision. This is exactly the lesson section 3 restates for
`StopGeometry`, applied here to the Composer's own output: never require a
consumer to resolve correctness by reaching into a mutable
audit record. Only genuinely non-critical, deeper diagnostic detail (e.g.
per-producer `PayloadSchemaVersion` values) is left to the decision record
via `ProposedActionSourceComposerDecisionSeq`.

| Field | Namespace/PV | Purpose |
|---|---|---|
| `ProposedActionStatus` | Int32 PV 34 | `NONE = 0` / `ACTIVE = 1` |
| `ProposedActionType` | Int32 PV 35 | Action type — currently only a bare entry (`ENTRY = 1`); reserved for future distinct types once other proposal shapes exist |
| `ProposedActionResolvedSide` | Int32 PV 36 | Self-contained copy — `LONG`/`SHORT` only |
| ~~`ProposedActionQuantity`~~ | ~~Int32 PV 37~~ | **Retired in Revision 7** — sourced from `SizedCandidateFinalQuantity`, which no longer exists as an input to this document (`SizedCandidate` removed; see "Changes from Revision 6"). Mode Baton owns quantity for mode 25 and ignores any published value regardless (source-audit finding). PV 37 is left unused, not reassigned, same convention as the Revision 4 dollar-risk retirements. |
| `ProposedActionStopDistanceTicks` | Int32 PV 38 | Self-contained copy of `StopGeometryDistanceTicks` |
| `ProposedActionEntryStyle` | Int32 PV 39 | Entry intent/order style, **if known** — see note below; `NONE = 0` is the honest default until an authoritative source exists |
| `PriceDriftToleranceIdentifier` | Int32 PV 40 | Section 7 — which code-versioned tolerance was applied |
| `ProposedActionSourceEntryGateChartNumber` | Int32 PV 41 | Self-contained copy — part of the composite correlation identity (Revision 2, section 3) |
| `ProposedActionSourceEntryGateStudyID` | Int32 PV 42 | Self-contained copy — composite correlation identity |
| `ProposedActionSourceEntryGateProducerTypeID` | Int32 PV 43 | Self-contained copy — composite correlation identity |
| `ProposedActionSeq` | Int64 PV 16 | The delivery sequence — section 9 |
| `ProposedActionSourceComposerDecisionSeq` | Int64 PV 17 | Links back to `ComposerDecisionSeq` (section 9) — deeper, non-critical diagnostic detail only, never used to resolve correlation |
| `ProposedActionSourceEntryGateCandidateSeq` | Int64 PV 18 | Self-contained copy — paired with PV 41-43 above, together the correlation key (section 3) |
| ~~`ProposedActionSourceSizedCandidateSeq`~~ | ~~Int64 PV 19~~ | **Retired in Revision 7** — no `SizedCandidate` input exists to source it from. PV 19 left unused, not reassigned. |
| `ProposedActionSourceStopGeometrySeq` | Int64 PV 20 | Self-contained copy |
| `ProposedActionMasterCycleSeq` | Int64 PV 21 | Self-contained copy — which decision cycle produced this proposal |
| `ProposedActionSourceTriggerEventSeq` | Int64 PV 22 | Self-contained copy, sourced from the matching `EntryCandidate` only (section 3) — Trigger provenance, new in Revision 2 |
| `ProposedActionAvailabilityDateTime` | Datetime PV 14 | Proposal creation/availability time |
| `ProposedActionValidUntilDateTime` | Datetime PV 15 | Section 5 |
| `ProposedActionStopReferencePrice` | Float PV 1 | Self-contained copy of `StopGeometryReferencePrice` — StopManager's original indicative reference |
| `ProposedActionStopPrice` | Float PV 2 | Self-contained copy of `StopGeometryStopPrice` |
| ~~`ProposedActionStopRiskPerContract`~~ | ~~Float PV 3~~ | **Retired in Revision 4** — sourced from `StopGeometryDistanceCurrencyPerContract`, which no longer exists (StopManager no longer computes a currency risk figure; see "Changes from Revision 3"). PV 3 is left unused, not reassigned. |
| ~~`ProposedActionAggregatePlannedStopRisk`~~ | ~~Float PV 4~~ | **Retired in Revision 4** — depended on the field above. PV 4 is left unused, not reassigned. |
| `ProposedActionDriftReferencePrice` | Float PV 5 | The Composer's own freshly-read current price at proposal-emission time (section 7) |
| `ProposedActionDriftAmount` | Float PV 6 | `\|ProposedActionDriftReferencePrice - ProposedActionStopReferencePrice\|`, the value compared against tolerance |

**`ProposedActionEntryStyle` — retained for schema stability, moot for the
v1 live path (see "Changes from Revision 5").** `NONE = 0` is always what
this Composer publishes; the Mode Baton Signal Publisher does not read
this field, since Mode Baton's mode-25 entry is always a market order
regardless (source-audit finding, `K_Util_ModeBaton_V1.cpp`). This is no
longer an implementation blocker — there is no downstream component
waiting on an entry-style policy decision that doesn't exist. If a future
integration path other than Mode Baton mode 25 is ever built, this field
would need an actual source; not needed for v1.

**Instrument/account identity is not a field this proposal carries.** No
upstream record publishes a comparable value, correlation across the pair
is guaranteed by manual wiring instead (section 3), and there is no
downstream component in the v1 live path that performs an account/
instrument check of its own — Mode Baton is manually wired to one
configured symbol/account, unmodified.

**What the Mode Baton Signal Publisher needs from this record**
(`Chakra_ModeBatonSignalPublisher_Interface_Contract.md` sections 3-4):
`ProposedActionStopReferencePrice` and `ProposedActionStopPrice`, to
derive `PV54`/`PV71` and, together with a configured tick offset, `PV56`/
`PV57`. `ProposedActionDriftReferencePrice`/`ProposedActionDriftAmount`/
`PriceDriftToleranceIdentifier` are **not** read by the publisher — they
exist purely as this Composer's own traceability record of its section 7
pre-publish gate, per that section's own corrected note (no downstream
post-fill reconciliation exists to consume them).

------------------------------------------------------------------------

# 9. Audit record versus delivery record

**The same two-record separation established by EntryGate and StopManager,
applied here again — never `PublicationSeq`, and never `ComposerDecisionSeq`,
as the proposal's own delivery key.**

## Composer decision record (audit)

**`ComposerDecisionSeq`** increments for every **meaningful** composition
decision — not every master-cycle evaluation while a candidate merely
continues waiting (see the logging discipline below).

| Field | Namespace/PV | Purpose |
|---|---|---|
| `PayloadSchemaVersion` | Int32 PV 20 | Envelope convention |
| `PolicyCategory` | Int32 PV 21 | Section 1 |
| `ComposerDecisionType` | Int32 PV 22 | Section 6/14 |
| `ResolvedSide` | Int32 PV 23 | Preserved, self-contained copy |
| `SourceEntryGateChartNumber` | Int32 PV 24 | Provenance |
| `SourceEntryGateStudyID` | Int32 PV 25 | Provenance |
| `SourceEntryGateProducerTypeID` | Int32 PV 26 | Provenance |
| ~~`SourcePositionSizerChartNumber`~~ | ~~Int32 PV 27~~ | **Retired in Revision 7** — no `PositionSizer` input exists (see "Changes from Revision 6"). Left unused, not reassigned. |
| ~~`SourcePositionSizerStudyID`~~ | ~~Int32 PV 28~~ | **Retired in Revision 7**, same reason. |
| ~~`SourcePositionSizerProducerTypeID`~~ | ~~Int32 PV 29~~ | **Retired in Revision 7**, same reason. |
| `SourceStopManagerChartNumber` | Int32 PV 30 | Provenance |
| `SourceStopManagerStudyID` | Int32 PV 31 | Provenance |
| `SourceStopManagerProducerTypeID` | Int32 PV 32 | Provenance |
| `ComposerDecisionSeq` | Int64 PV 10 | This section |
| `SourceEntryGateCandidateSeq` | Int64 PV 11 | The correlation key (section 3) |
| ~~`SourceSizedCandidateSeq`~~ | ~~Int64 PV 12~~ | **Retired in Revision 7** — no `SizedCandidate` input exists. Left unused, not reassigned. |
| `SourceStopGeometrySeq` | Int64 PV 13 | Populated once matched |
| `SourceTriggerEventSeq` | Int64 PV 14 | Carried forward |
| `MasterCycleSeq` | Int64 PV 15 | Which decision cycle |
| `SourceEntryCandidateValidUntilDateTime` | Datetime PV 10 | Propagated |
| ~~`SourceSizedCandidateValidUntilDateTime`~~ | ~~Datetime PV 11~~ | **Retired in Revision 7** — no `SizedCandidate` input exists. Left unused, not reassigned. |
| `SourceStopGeometryValidUntilDateTime` | Datetime PV 12 | Propagated, once known |
| `SourceTriggerAvailabilityDateTime` | Datetime PV 13 | Provenance |

Sticky until superseded by a newer decision or cleared at the Composer's
own disable-entry/recalc-entry — the same rule every event-style layer in
this framework already follows.

## Proposed-action delivery record (section 8)

**`ProposedActionSeq` increments only when a complete proposal is
approved and emitted** — never on any waiting/rejected/dropped decision.
This is the correctness-critical delivery interface to the Mode Baton
Signal Publisher, the same reasoning that produced every prior layer's own
delivery sequence: a shared sequence would let a later rejection/skip for
a *different* candidate silently erase an already-approved, not-yet-
consumed proposal.

**Decisions the Composer's own enum must distinguish** (section 14 gives
the full enum and mapping):

- Proposal approved.
- Waiting for matching geometry.
- Correlation mismatch.
- Expired.
- Lifecycle stale.
- Price drift exceeded.
- Duplicate suppressed.
- Undecidable.

**Logging discipline — be careful with "waiting":** entering a wait state
for the missing geometry input is logged **once**, on the cycle the wait
state is first entered (a meaningful state transition) — not re-logged
every subsequent master cycle the candidate merely continues waiting.
`ComposerDecisionSeq` likewise only advances on a genuine transition
(entering a new decision/wait state, or reaching a terminal outcome), never
on "still waiting, nothing changed since last cycle."

------------------------------------------------------------------------

# 10. Commit order

**For an approved proposal:**

```text
envelope
+ complete composer decision payload
+ complete proposed-action payload
    -> ComposerDecisionSeq
    -> ProposedActionSeq
    -> PublicationSeq
```

**For reject/skip/state-transition decisions:**

```text
envelope
+ decision payload
    -> ComposerDecisionSeq
    -> PublicationSeq
```

**Rejects and waiting-state transitions must never overwrite a previous
unconsumed proposal.** `ComposerDecisionSeq` moving forward on its own
never touches `ProposedActionSeq` or the sticky proposed-action record —
identical to every prior layer's own overwrite protection.

**A later successful proposal may overwrite an earlier, unconsumed one
only under the explicitly declared latest-approved-wins rule**, matching
this framework's established default (TriggerStr's `EventSeq`, EntryGate's
`CandidateSeq`, StopManager's `StopGeometrySeq`) — the downstream consumer
(the Mode Baton Signal Publisher) detects a missed intermediate proposal
via the same sequence-gap logging already required everywhere else in
this framework. Since one Composer instance tracks at most one pending
candidate at a time (section 4), this case is rarer here than upstream (it
requires an *already-approved-but-unconsumed* proposal, followed by an
*entirely new* candidate completing assembly before the publisher consumes
the first) but is not impossible, and is not silently disallowed.

**Neutralize every inapplicable field on each relevant publication** —
never let a stale value from a prior proposal survive into an unrelated
publication under the atomic-publication model. Concretely: a
reject/skip/waiting-state publication (decision-only, no proposed-action
payload) must not be interpreted as clearing the sticky proposed-action
record — it simply doesn't touch it, per the overwrite rule above — but
any *new* approved proposal must write every one of section 8's fields
explicitly, never leaving any of them holding a prior proposal's stale
value.

------------------------------------------------------------------------

# 11. Outcomes and boundary

**Mapped consistently to the architecture's five generic Trade Policy
outcomes** (`Algo_Overlay_Framework_Architecture.md`, "Decision Cycle and
Master Clock"):

- **`NOT_APPLICABLE`** — nothing consumed this cycle at all (no pending
  candidate, no newly-observed candidate, lifecycle irrelevant).
- **`NO_ACTION`** — relevant and evaluated, but genuinely nothing to do
  (no new `EntryCandidate`, and no candidate currently pending).
- **`POLICY_DECISION`** — a candidate is being tracked and the Composer
  produced a real decision about it this cycle: entering a wait state
  (logged once, section 9), a correlation mismatch, an expiry, a
  lifecycle-stale drop, a price-drift rejection, a superseded pending
  candidate, or a duplicate suppression. **Waiting itself, once logged on
  entry, does not re-qualify
  as a fresh `POLICY_DECISION` every subsequent cycle it merely
  continues** — consistent with the "log transitions, not every bar" rule
  (section 9).
- **`NO_CHANGE` is unused** — proposal assembly is discrete per candidate,
  not a continuously-maintained recommendation (unlike, say, a future
  `TrailingManager`'s stop level).
- **`PROPOSED_ACTION`** — used **only** when the record is complete *with
  respect to this contract's own scope* (candidate and stop geometry,
  correctly correlated per section 3), fully validated (section 6), and
  safe to forward. "Complete" here means "everything this layer owns has
  been assembled and validated" — for the v1 live path, that is genuinely
  sufficient: the Mode Baton Signal Publisher needs nothing further from
  this Composer that isn't already published (section 8). This is the
  Composer's entire reason for existing — the first and only point in
  this pipeline where `PROPOSED_ACTION` is ever produced.

**The Composer never emits `EXECUTABLE_INSTRUCTION`.** That disposition
does not exist anywhere in the v1 live path — there is no Order Management
component to produce it. The Mode Baton Signal Publisher's own output
(writing `PV51`-`PV74`) is not a disposition in the architecture's sense;
it is this pipeline's terminal action, per
`Chakra_ModeBatonSignalPublisher_Interface_Contract.md`.

**Once published, this Composer's own decision record is never
retroactively rewritten.** The Composer's `ComposerDecisionSeq`/
`ProposedActionSeq` records describe what the Composer decided at the time
it decided it. Nothing downstream reports a disposition back to this
Composer (there is no downstream disposition in the v1 live path) — the
Composer's record is simply the final word on what the Composer itself
did.

------------------------------------------------------------------------

# 12. Downstream consumption

**As of Revision 6, `Chakra_ModeBatonSignalPublisher_Interface_Contract.md`
— not an Entry Authority — consumes `ProposedActionSeq`, using the
established event-style discipline, unmodified:**

- **First-attachment and rewiring baselining** — the unified
  `slotBaselined` mechanism, applied by the publisher to its own
  consumption of this Composer's output, exactly as every event-pattern
  consumer in this framework already does.
- **No replay of an old sticky proposal** — attachment/rewiring/the
  publisher's own recalc/disable-reenable all baseline without consuming,
  never treating an already-existing `ProposedActionSeq` as freshly
  actionable merely because the publisher just started watching it.
- **Sequence-gap detection**, capturing the prior sequence before mutating
  the tracking field, per the envelope's corrected Revision 7 pattern.
- **Backward-sequence reset handling** — logged distinctly from a forward
  gap, treated as a probable storage reset, per the same pattern.
- **Expiry** — `ProposedActionValidUntilDateTime` checked against the
  publisher's own evaluation time before acting.
- **Consume-and-drop** — the default failure policy, same as every other
  event-pattern consumer in this framework; per
  `Chakra_ModeBatonSignalPublisher_Interface_Contract.md` section 9, the
  publisher does not retry, republish, or monitor after commit.
- **Independent lifecycle and order-state revalidation** — not applicable
  in the same sense as the superseded Entry Authority design: the
  publisher performs no order-state check of its own (it never touches an
  order), but it does independently re-run its own price/candidate
  revalidation immediately before writing the payload, the same
  "independent re-check at every stage" discipline this entire pipeline
  has required since EntryGate's Revision 3.

**Latest-wins is explicit**: if a second `ProposedActionSeq` publishes
before the publisher consumes the first (section 10), the publisher sees
the sequence-gap and knows an intermediate proposal was dropped —
consistent with every other single-slot delivery record in this
framework. **Guaranteed delivery and unbounded queueing remain out of
scope** — this document does not ask the publisher to buffer proposals
any differently than any other event-pattern consumer already does.

------------------------------------------------------------------------

# 13. Recalculation and replay

**During full recalculation:**

- Historical diagnostic subgraphs may be rebuilt if useful (the same
  hidden Occurrence/Status subgraph pair pattern already established
  throughout this framework) — entirely separate from, and never driving,
  the live sequences below.
- **`ComposerDecisionSeq` never increments live.**
- **`ProposedActionSeq` never increments live.**
- **No live proposals are created**, under any circumstance.
- **The Mode Baton Signal Publisher is never invoked during recalculation.**
- **Private trackers baseline at all four lifecycle boundaries**
  (attachment, rewiring, the Composer's own recalc, the Composer's own
  disable/re-enable), following one safe rule at recalc-entry specifically:

  ```text
  recalc entry
      -> abandon any in-flight pending assembly
      -> baseline current source sequences:
           LastConsumedEntryCandidateSeq = current EntryCandidate.CandidateSeq
           LastObservedStopGeometrySeq   = current StopGeometry.StopGeometrySeq
      -> clear PendingCandidateSeq and all matched-geometry state to neutral
      -> do NOT consume whatever is currently sitting in either sticky slot
         at the moment of baselining
      -> wait for a genuinely newer CandidateSeq (or, for StopGeometry, a
         genuinely newer, still-correlated record) via the normal live
         gates (section 2/4) before assembling anything
  ```

  **A candidate already sitting in a producer's sticky slot at the moment
  of recalc-entry baselining is never treated as newly actionable** — this
  is the same "baselining and consuming are never the same call" rule
  already governing every other attachment/rewiring boundary in this
  framework, applied to recalc-entry specifically. Revision 1 said both
  that trackers "re-baseline" (implying the gate would see no difference
  and therefore report nothing new) and that an in-progress candidate is
  "re-observed as a fresh candidate" after recalculation (implying the
  opposite) — those cannot both be true, and Revision 1 never resolved
  which one governs. This revision resolves it: baselining wins. Nothing
  already present at the recalc-entry boundary is ever assembled by this
  Composer instance after that boundary — only a genuinely newer sequence
  value, arriving through the normal live path afterward, can be.

**Replay's forward evaluation uses the normal master-cycle path** — no
special-casing beyond correctly following the rules above. **Proposals
must not be duplicated across a recalculation/replay transition**: because
every private tracker baselines to the *current* value at recalc-entry
(above) rather than resuming mid-assembly, a candidate whose assembly was
in progress before a recalculation boundary is never resurrected from
stale pre-recalc pending state. Replay's forward simulation can and does
generate genuinely new forward `CandidateSeq`/`StopGeometrySeq` values
through the normal path once it resumes live
evaluation — those are picked up exactly like any other live-observed
value. **A candidate already sitting in the slot at the moment of
baselining is not one of them**, precisely because baselining consumed
nothing.

**Historical records use availability-indexed data and historical
validity signals** — the three-part historical-validity check (array
coverage, historical `Status == VALID`, finite/in-range value) already
established since the Sensor contract's Revision 3, applied here to any
historical diagnostic subgraph this Composer chooses to maintain.

------------------------------------------------------------------------

# 14. Diagnostics and reason ownership

```cpp
enum K_CHAKRA_ENTRYCOMPOSER_DECISION
{
    K_CHAKRA_ENTRYCOMPOSER_DECISION_NONE                  = 0,
    K_CHAKRA_ENTRYCOMPOSER_DECISION_PROPOSAL_APPROVED     = 1,
    // 2 retired in Revision 7 -- was WAITING_FOR_SIZE; removed, not
    // reassigned, because SizedCandidate is no longer an input (see
    // "Changes from Revision 6")
    K_CHAKRA_ENTRYCOMPOSER_DECISION_WAITING_FOR_GEOMETRY  = 3,
    K_CHAKRA_ENTRYCOMPOSER_DECISION_CORRELATION_MISMATCH  = 4,
    K_CHAKRA_ENTRYCOMPOSER_DECISION_SKIP_EXPIRED          = 5,
    K_CHAKRA_ENTRYCOMPOSER_DECISION_SKIP_LIFECYCLE        = 6,
    K_CHAKRA_ENTRYCOMPOSER_DECISION_PRICE_DRIFT_EXCEEDED  = 7,
    K_CHAKRA_ENTRYCOMPOSER_DECISION_DUPLICATE_SUPPRESSED  = 8,
    K_CHAKRA_ENTRYCOMPOSER_DECISION_SKIP_UNDECIDABLE      = 9,
    K_CHAKRA_ENTRYCOMPOSER_DECISION_SUPERSEDED            = 10
};

enum K_CHAKRA_ENTRYCOMPOSER_REASON_CODE
{
    K_CHAKRA_ENTRYCOMPOSER_REASON_NONE                          = 0,   // = K_CHAKRA_REASON_NONE
    K_CHAKRA_ENTRYCOMPOSER_REASON_NO_PENDING_ASSEMBLY            = 1,
    // 2 retired in Revision 7 -- was MISSING_MATCHING_SIZE; removed, not
    // reassigned, same reason as DECISION value 2 above
    K_CHAKRA_ENTRYCOMPOSER_REASON_MISSING_MATCHING_GEOMETRY       = 3,
    // 4 retired in Revision 7 -- was CANDIDATE_SIZE_MISMATCH; same reason
    K_CHAKRA_ENTRYCOMPOSER_REASON_CANDIDATE_GEOMETRY_MISMATCH      = 5,
    K_CHAKRA_ENTRYCOMPOSER_REASON_SIDE_MISMATCH                   = 6,
    K_CHAKRA_ENTRYCOMPOSER_REASON_TRIGGER_PROVENANCE_MISMATCH      = 7,
    K_CHAKRA_ENTRYCOMPOSER_REASON_CANDIDATE_EXPIRED               = 8,
    // 9 retired in Revision 7 -- was SIZE_EXPIRED; same reason
    K_CHAKRA_ENTRYCOMPOSER_REASON_GEOMETRY_EXPIRED                = 10,
    K_CHAKRA_ENTRYCOMPOSER_REASON_LIFECYCLE_STALE                 = 11,
    // 12 retired in Revision 4 -- was INSTRUMENT_ACCOUNT_MISMATCH; removed,
    // not reassigned, because no upstream record publishes a comparable
    // instrument/account value for this Composer to mismatch against (see
    // "Changes from Revision 3" and section 3)
    // 13 retired in Revision 7 -- was INVALID_QUANTITY; no quantity exists
    // in this document's own scope any longer
    K_CHAKRA_ENTRYCOMPOSER_REASON_INVALID_STOP_GEOMETRY            = 14,
    K_CHAKRA_ENTRYCOMPOSER_REASON_PRICE_DRIFT_BEYOND_TOLERANCE     = 15,
    K_CHAKRA_ENTRYCOMPOSER_REASON_PRODUCER_SCHEMA_IDENTITY_MISMATCH = 16,
    K_CHAKRA_ENTRYCOMPOSER_REASON_PRODUCER_STORAGE_RESET           = 17,
    K_CHAKRA_ENTRYCOMPOSER_REASON_PROPOSAL_APPROVED                = 18,
    K_CHAKRA_ENTRYCOMPOSER_REASON_DUPLICATE_SUPPRESSED             = 19,
    K_CHAKRA_ENTRYCOMPOSER_REASON_CANDIDATE_SUPERSEDED             = 20
    // extend per concrete Entry Composer policy as needed, numbered above these common ones
};
```

**Mapping** (several reasons deliberately share a `DecisionType`,
disambiguated by `ReasonCode`, the same pattern every prior layer uses):

| `ReasonCode` | `ComposerDecisionType` |
|---|---|
| `MISSING_MATCHING_GEOMETRY` | `WAITING_FOR_GEOMETRY` |
| `CANDIDATE_GEOMETRY_MISMATCH`, `SIDE_MISMATCH`, `TRIGGER_PROVENANCE_MISMATCH` | `CORRELATION_MISMATCH` |
| `CANDIDATE_EXPIRED`, `GEOMETRY_EXPIRED` | `SKIP_EXPIRED` |
| `LIFECYCLE_STALE` | `SKIP_LIFECYCLE` |
| `INVALID_STOP_GEOMETRY`, `PRODUCER_SCHEMA_IDENTITY_MISMATCH`, `PRODUCER_STORAGE_RESET` | `SKIP_UNDECIDABLE` |
| `PRICE_DRIFT_BEYOND_TOLERANCE` | `PRICE_DRIFT_EXCEEDED` |
| `PROPOSAL_APPROVED` | `PROPOSAL_APPROVED` |
| `DUPLICATE_SUPPRESSED` | `DUPLICATE_SUPPRESSED` |
| `CANDIDATE_SUPERSEDED` | `SUPERSEDED` |

**`MISSING_MATCHING_GEOMETRY`** describes geometry simply not having
arrived yet — nothing correlating to `PendingCandidateSeq` currently
exists. **`CANDIDATE_GEOMETRY_MISMATCH`** describes the narrower, distinct
case where a record *is* currently active but correlates to a
`CandidateSeq` this Composer has already terminally resolved (section 3's
disposition rule) — a materially different diagnostic signal (a producer
publishing something this Composer can never use again, versus one that
just hasn't published yet).

**Reason codes describe only what the Entry Composer itself observed or
decided.** No member of this enum describes any downstream disposition —
there is no downstream disposition to describe in the v1 live path (no
Order Management, no Entry Authority); the Mode Baton Signal Publisher's
own diagnostics, and Mode Baton's own inherited existing behavior, are
each documented separately.

**Log meaningful transitions and decisions, not every master bar** —
section 9's logging discipline restated here for the diagnostics section
specifically, since this is the enum that discipline governs.

------------------------------------------------------------------------

# 15. Simultaneous candidates and arbitration

**Addressed explicitly, not silently ignored:**

- **One Composer instance per candidate family/source** (section 1) is
  this contract's own chosen resolution for the ordinary case — a single
  Composer never has more than one `EntryCandidate` source to reconcile,
  so there is no simultaneous-candidate problem *within* one instance
  beyond the single-slot supersession already handled by section 4.
- **The Composer must never silently merge two different EntryGate
  sources' candidates**, even if a future deployment somehow wires more
  than one EntryGate to the same Composer instance — that configuration is
  **not supported** by this contract; section 1's one-pair assumption is a
  hard requirement, not a recommendation.
- **Where arbitration belongs, if multiple *independent* Composer
  instances (each with its own EntryGate/StopManager pair, each with its
  own Mode Baton Signal Publisher) could simultaneously propose entries**
  — e.g. two unrelated strategies, possibly opposing sides, both
  completing assembly in the same master cycle — **this document does not
  itself resolve it, and, as of Revision 7, nothing downstream resolves it
  either.** The superseded Order Management Handoff design's own section 6
  (fixed configuration-order, first-manager-ready-wins arbitration) is not
  being built — see that document's own superseded banner. A
  portfolio-level arbitration component remains **excluded outright** by
  this project's own scope boundary (Chakra does not own portfolio or
  account-level risk management), not merely undesigned. One place this
  can still legitimately live:
  1. A deterministic precedence rule declared within each Composer
     instance (e.g. "defer to the other side if opposing exposure would
     result") — narrow, local, and only handles pairwise conflicts a given
     instance is explicitly configured to know about.
  2. **If multiple Composer instances publish through separate Mode Baton
     Signal Publishers into the *same* Mode Baton instance's multiple
     master-chart inputs** (`K_Util_ModeBaton_V1.cpp` supports up to three
     configured masters), Mode Baton's own existing, proven arbitration
     applies unmodified: fixed configuration order across its three master
     inputs, first-eligible-signal-wins per its own `seekingTrade`
     state machine (source-audited, not designed here). This is not a
     Chakra-designed mechanism — it is Mode Baton's own pre-existing
     behavior, inherited as-is.
- **If neither a local precedence rule nor Mode Baton's own multi-master
  handling covers a given deployment's needs (e.g. two fully independent
  Mode Baton instances, each with its own publisher, both eligible to
  trade the same instrument/account), there is currently no arbitration at
  all** — this is a genuine, unresolved gap for that specific deployment
  shape, not a silently-assumed-safe default. See open question 4.

------------------------------------------------------------------------

# 16. Worked examples

1. **Ordinary path — both arrive together.** `EntryCandidate` and
   `StopGeometry` both correlate to `CandidateSeq = 42` and are both
   observed as matched on the same cycle. Final validation (section 6) and
   the drift check (section 7) both pass. `ComposerDecisionType =
   PROPOSAL_APPROVED`, `ComposerDecisionSeq` and `ProposedActionSeq` both
   increment.
2. **Geometry arrives first.** `StopGeometry` for candidate 42 is observed
   before `EntryCandidate` 42 itself is (its own producer evaluated
   earlier this cycle, or on an earlier cycle while candidate 42 hadn't
   yet been resolved by EntryGate). Once `EntryCandidate` 42 is observed,
   `PendingCandidateSeq` is set to 42, and the already-current
   `StopGeometry` is found to already match — no waiting needed for that
   input. The Composer does not drop or ignore geometry that arrived
   "too early" relative to its own candidate observation.
3. **Cross-candidate mismatch — no proposal.** `StopGeometry` currently
   correlates to `CandidateSeq = 41` (a different, already-resolved
   candidate); the currently pending `EntryCandidate` is 42. The input
   does not match `PendingCandidateSeq = 42`. `ComposerDecisionType =
   WAITING_FOR_GEOMETRY` — logged once on entry, not merged from the
   non-matching 41 record under any circumstance.
4. **Composite-identity mismatch — numerically equal `CandidateSeq`, two
   different EntryGate producers.** This Composer's own EntryGate source
   was rewired at some point — old wiring: chart 5/study 12; new wiring:
   chart 5/study 19, both eventually reaching `CandidateSeq = 42`
   independently. `StopGeometry` still holds a stale correlation to the
   *old* producer (`StopGeometrySourceEntryGateStudyID = 12`,
   `...CandidateSeq = 42`); the currently pending `EntryCandidate` is from
   the *new* producer (study 19, `CandidateSeq = 42`). A bare
   `CandidateSeq` comparison (`42 == 42`) would wrongly report a match.
   Under the composite check (section 3), `StopGeometrySourceEntryGateStudyID
   (12) !=` this Composer's currently-wired EntryGate `StudyID (19)` — no
   match. `ComposerDecisionType = WAITING_FOR_GEOMETRY`, `ReasonCode =
   MISSING_MATCHING_GEOMETRY` (the old-producer record is simply not
   usable for the new candidate, not logged as a mismatch every cycle) —
   the Composer never assembles a proposal from a mismatched producer
   merely because its sequence number happened to coincide.
5. **Producer storage reset — backward sequence movement on the wired
   EntryGate source.** `LastConsumedEntryCandidateSeq = 50` from normal
   operation; the EntryGate producer's own persistent storage is reset
   (e.g. a chart reload) and its `CandidateSeq` restarts from a lower
   value, say 3. The section 4 gate (`CandidateSeq !=
   LastConsumedEntryCandidateSeq`) correctly fires (`3 != 50`); the branch
   logic detects `3 < 50` (backward) rather than treating it as 47
   candidates' worth of forward gap. Any in-flight `PendingCandidateSeq`
   is abandoned, the event is logged distinctly as a probable storage
   reset (not an ordinary sequence gap), and
   `LastConsumedEntryCandidateSeq` re-baselines to `3` **without**
   treating candidate 3 as newly actionable this same evaluation — it
   only becomes eligible for assembly once a *later* cycle observes a
   `CandidateSeq` that once again differs from the just-rebaselined value.
6. **Candidate expires while waiting.** Candidate 42 is pending;
   `EntryCandidateValidUntilDateTime` passes before matching
   `StopGeometry` ever arrives. `ComposerDecisionType = SKIP_EXPIRED`,
   `ReasonCode = CANDIDATE_EXPIRED`. `LastConsumedEntryCandidateSeq`
   advances to 42; `PendingCandidateSeq` clears to 0.
7. **Lifecycle becomes ineligible while waiting.** A working entry order
   from an unrelated path appears while candidate 42 is still pending
   assembly. `ComposerDecisionType = SKIP_LIFECYCLE`, `ReasonCode =
   LIFECYCLE_STALE`. Consumed and dropped, matching every upstream layer's
   own lifecycle-staleness handling.
8. **Price drift exceeds tolerance.** Both inputs match (composite
   identity included) and pass final validation; the section 7 drift check
   finds the Composer's current price reference diverges from
   `StopGeometryReferencePrice` beyond the configured tolerance. Default
   disposition: `ComposerDecisionType = PRICE_DRIFT_EXCEEDED`, `ReasonCode
   = PRICE_DRIFT_BEYOND_TOLERANCE`. No proposal emitted; candidate 42 is
   consumed and dropped (the default reject disposition, section 7) unless
   this Composer instance has explicitly documented the wait-for-refresh
   alternative instead.
9. **A later rejection does not overwrite an earlier unconsumed
   proposal.** Candidate 42's proposal is approved and published
   (`ProposedActionSeq` increments); before the Mode Baton Signal
   Publisher consumes it, a later, unrelated candidate observation (e.g. a
   stale/mismatched input for a different, already-superseded candidate)
   produces a `ComposerDecisionSeq` advance with `ComposerDecisionType =
   CORRELATION_MISMATCH`. The sticky proposed-action record for candidate
   42 is untouched — `ComposerDecisionSeq` moving never touches
   `ProposedActionSeq` (section 10).
10. **A later successful proposal overwrites under latest-wins.**
    Candidate 42's proposal is approved and published, not yet consumed by
    the publisher; a wholly new candidate (43) later completes its own
    assembly and is also approved. `ProposedActionSeq` advances again, now
    describing candidate 43; candidate 42's proposal is gone. The
    publisher's own sequence-gap check (section 12) detects the jump and
    logs the dropped intermediate proposal — the same latest-approved-wins
    discipline every upstream delivery record already uses.
11. **Duplicate inputs do not emit the same proposal twice.** Both inputs
    for candidate 42 remain unchanged (same `CandidateSeq`/
    `StopGeometrySeq`) across several master cycles after a proposal was
    already approved and published for it. `LastConsumedEntryCandidateSeq`
    already covers 42 (section 4) — the gate in section 2 finds no new
    `CandidateSeq`, so no re-evaluation, no second decision, and no second
    `ProposedActionSeq` increment occur. If some other path caused
    re-evaluation of an already-terminally-resolved candidate,
    `ComposerDecisionType = DUPLICATE_SUPPRESSED`, `ReasonCode =
    DUPLICATE_SUPPRESSED` is the explicit safety net.
12. **Recalculation rebuilds diagnostics but emits no proposal, and does
    not resume a pre-recalc pending candidate.** Before
    `IsFullRecalculation` begins, candidate 47 was pending, waiting on
    `StopGeometry`. At recalc-entry, `LastConsumedEntryCandidateSeq`
    baselines to whatever `CandidateSeq` the (possibly historical-replay)
    EntryGate producer currently shows — including 47 itself, if that's
    what's sitting in the slot — and `PendingCandidateSeq` clears to 0
    **without** consuming it. A historical diagnostic subgraph may
    separately record that candidate 47 would have assembled into a valid
    proposal at a given historical bar — `ComposerDecisionSeq`/
    `ProposedActionSeq` never increment live, the Mode Baton Signal
    Publisher is never invoked, and this historical fact is entirely
    separate machinery from the live sequences (section 13). Candidate 47
    is only ever picked up live again if a *later* cycle, after
    recalculation ends, observes a `CandidateSeq` that once again differs
    from the rebaselined value.
13. **Invalid design.** An Entry Composer that calls an order-submission
    function directly, or that itself writes into Mode Baton's `PV51`-
    `PV74` record. Either is the Mode Baton Signal Publisher's job wearing
    the Composer's name — the Composer's only legitimate output is a
    composition decision and, on approval, a bare `PROPOSED_ACTION`.
14. **Actual fill differs after submission — not this document's
    problem, and not anyone else's Chakra-designed problem either.** The
    Mode Baton Signal Publisher publishes from candidate 42's proposal;
    Mode Baton submits the entry and the actual fill lands materially away
    from `ProposedActionStopReferencePrice`. No Chakra component
    reconciles this (section 7's corrected note) — Mode Baton's own
    existing, inherited behavior is whatever it already does, unmodified.
    The Composer's own published proposal is untouched, and the Composer
    itself never re-evaluates or reconciles anything once
    `ProposedActionSeq` has been consumed.

------------------------------------------------------------------------

# Unresolved design questions — deliberately left open

**Renumbered in Revision 7** — the former open questions 3, 5, and 7 (entry
style, post-fill remediation, Composer-to-candle-manager relationship) were
each "resolved" only by pointing at the now-superseded Order Management
Handoff design. They are not carried forward as resolved; see the notes
under questions 2 and 4 below for what actually replaced each. Only
questions that affect real implementation. Envelope, Sensor, MCtx,
TriggerStr, EntryGate, StopManager, and base-architecture decisions already
settled are not reopened here.

1. **What is the Composer's own current-price reference source, and what
   is the drift tolerance** (section 7)? Genuinely new territory — no
   prior Chakra contract has needed a live, ungated, non-Sensor price
   read. **Until resolved, the price-drift hard gate cannot be concretely
   implemented** — a concrete Composer must pick and document a specific
   ACSIL price source and a specific tolerance constant before shipping,
   but this document does not mandate which.
2. **Does the "wait for a newly-correlated refresh" drift disposition
   (section 7) actually work**, given that `StopManager`'s own contract
   does not currently specify automatic re-evaluation/republication for
   an already-approved candidate? **Until resolved, only the default
   reject disposition is safely usable.** (Entry style is no longer a
   related open question, since Mode Baton's mode 25 is always a market
   order regardless of what this document publishes — source-audit
   finding, "Changes from Revision 5.")
3. **How are simultaneous proposals from independent Composer instances
   arbitrated** (section 15)? **Genuinely open as of Revision 7, not
   resolved.** The previous "resolution" pointed at the superseded Order
   Management Handoff design's own arbitration mechanism, which is not
   being built. What actually exists: a local per-instance precedence
   rule (available but not designed here), and — only for the specific
   case of multiple Composers feeding the same Mode Baton instance through
   separate publishers — Mode Baton's own existing multi-master handling
   (source-audited, fixed configuration order across up to three
   configured masters). For any other deployment shape (e.g. two fully
   independent Mode Baton instances eligible to trade the same
   instrument/account), **there is currently no arbitration at all** — a
   real, unresolved gap, not a silently-assumed-safe default.
4. **Must profit targets be present before an entry proposal is
   considered complete, or can targets attach later?** This contract
   itself still carries no target field (core responsibility, above) —
   but as of the Mode Baton Signal Publisher's Revision 1, target
   *prices* for `PV56`/`PV57` are computed by the publisher directly from
   this Composer's own `StopGeometryReferencePrice` plus a configured
   tick offset (publisher contract section 4), not by a future
   `ProfitManager`-style component. This resolves the practical blocker
   (mode 25 does need target prices, and now has a source) without this
   Composer's own scope changing — the publisher's computation is a new
   responsibility one layer downstream, not a fourth correlated input
   here.
5. **What is the exact physical relationship between the Entry Composer
   and the candle baton manager** (`K_Util_Candle_Baton_Manager_V1`)?
   **Resolved, superseding the prior "Entry Authority" answer**: this
   Composer's `ProposedActionSeq` feeds the Mode Baton Signal Publisher,
   which writes directly into Mode Baton's own pre-existing `PV51`-`PV74`
   signal record — the same record `K_Strat_Bhaskar_V1.cpp`/`V2.cpp`
   already write. `K_Util_ModeBaton_V1.cpp` remains unmodified and
   `K_Util_Candle_Baton_Manager_V1` learns of a trade only through Mode
   Baton's own existing master-handoff publication after entry, exactly
   as it already does for Bhaskar-originated signals — no new physical
   relationship exists between this Composer and either existing study
   beyond publishing into a PV record they already read/write.

Do not hide an implementation blocker inside prose above. **Three of these
five remain genuinely open** (1, 2, 3) and each states explicitly what
functionality remains unavailable until resolved. **Two (4, 5) are
resolved**, by the Mode Baton Signal Publisher's own design, and are kept
in this list so a reader scanning this document alone sees the resolution
and its source directly.

None of these are silently resolved by giving the Entry Composer
responsibilities belonging to `EntryGate`, `StopManager`,
`TrailingManager`, `ProfitManager`, or any execution component — none of
which exist in the v1 live path except `EntryGate` and `StopManager`
themselves.
