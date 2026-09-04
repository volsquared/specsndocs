# Chakra — Trade Policy: EntryGate Interface Contract (Draft, Revision 8)

Status: **design only, no ACSIL implementation yet.**

Builds directly on:
- `Algo_Overlay_Framework_Architecture.md`, including the approved
  "Decision Cycle and Master Clock" amendment — now with a **five**-value
  Trade Policy outcome set (`NOT_APPLICABLE`/`NO_ACTION`/`POLICY_DECISION`/
  `NO_CHANGE`/`PROPOSED_ACTION`), `POLICY_DECISION` having been added
  specifically because this EntryGate contract pass demonstrated it was
  needed — plus `DecisionFrame` and bounded retention, unchanged.
- `Chakra_Common_Interface_Envelope_Contract.md` (approved, Revision 8).
- `Chakra_Sensor_Interface_Contract.md` (Revision 4).
- `Chakra_MCtx_Interface_Contract.md` (Revision 5).
- `Chakra_TriggerStr_Interface_Contract.md` (Revision 3).

**This document designs only the EntryGate sub-component of Trade Policy.**
`PositionSizer`, `StopManager`, `TrailingManager`, `ProfitManager`, and the
physical `DecisionFrame` storage layout are explicitly out of scope here —
each needs its own contract pass, and several of the open questions at the
end of this document exist specifically because they depend on decisions
those passes haven't made yet.

**Core responsibility, frozen:**

> EntryGate consumes one fresh TriggerStr occurrence plus the specific
> current context facts required by its local policy rules and decides
> whether that entry candidate is accepted, rejected, or cannot be safely
> decided.

**EntryGate must not:**

- Detect setups (that's TriggerStr's job entirely).
- Calculate quantity (`PositionSizer`).
- Choose stop or target prices (`StopManager`/`ProfitManager`).
- Submit orders.
- Forward an incomplete entry directly to the candle baton manager.
- Manage an existing position.
- Perform global arbitration across unrelated strategies.

Naming: `K_Chakra_TradePol_EntryGate_<DescriptiveName>_V<n>`.

**EntryGate is event/decision-style, like TriggerStr — not state-style,
like Sensor/MCtx.** It publishes discrete decisions, not continuous
descriptive state, and reuses the envelope's event pattern accordingly,
with its own decision-specific sequence field (section 10).

------------------------------------------------------------------------

# Changes from Revision 1

Three material issues from a review pass, one of which required an
amendment to the base architecture document itself:

1. **`PROPOSED_ACTION` was being stretched beyond its frozen architecture
   meaning (blocking; resolved, not left as a flagged tension).** Revision
   1 used `PROPOSED_ACTION` for `ACCEPT`/`REJECT`/`SKIP_EXPIRED`/
   `SKIP_UNDECIDABLE` alike, then flagged the resulting conflict with the
   architecture's "only `PROPOSED_ACTION` is forwarded to Order Management"
   rule as an open tension. That undersold the problem — even `ACCEPT`
   only produces an incomplete `EntryCandidate`, so *none* of EntryGate's
   own outputs were ever `PROPOSED_ACTION` in the architecture's real
   sense. Fixed at the source: `Algo_Overlay_Framework_Architecture.md`'s
   "Decision Cycle and Master Clock" section now defines a fifth generic
   Trade Policy outcome, **`POLICY_DECISION`** — a genuine, durable decision
   that is not yet a complete, manager-ready action. EntryGate's
   `ACCEPT`/`REJECT`/`SKIP_*` are all `POLICY_DECISION` at the generic
   level; `PROPOSED_ACTION` is reserved for whatever eventually assembles a
   complete action (the Entry Composer, section 4) and is never something
   EntryGate itself produces. See section 2.
2. **A lifecycle-ineligible-but-consumed trigger was reported as
   `NOT_APPLICABLE`, contradicting the architecture's own definition of
   that outcome ("did not evaluate at all").** Revision 1 had EntryGate
   advance `LastConsumedEventSeq`, publish `DecisionType = SKIP_LIFECYCLE`,
   and record diagnostics — a durable decision by any reasonable
   definition — while simultaneously calling the cycle `NOT_APPLICABLE`.
   Fixed: draining a trigger while lifecycle-ineligible is now
   `POLICY_DECISION` with `DecisionType = SKIP_LIFECYCLE`, exactly like any
   other skip. `NOT_APPLICABLE` is reserved for a cycle where nothing at
   all was consumed. See sections 2 and 6.
3. **`DecisionSeq` was unsafe as the accepted-candidate delivery sequence
   (blocking).** `DecisionSeq` increments for every decision —
   `ACCEPT`/`REJECT`/`SKIP_*` alike, sharing one sequence. A later
   `REJECT`/`SKIP_*` (a genuinely different, later trigger occurrence)
   would overwrite the sticky payload and silently erase an
   already-accepted, not-yet-consumed candidate from an earlier decision,
   with `PositionSizer` having no way to recover it — nothing in this
   contract guaranteed same-cycle synchronous consumption. This is the
   same class of mistake `PublicationSeq`-as-event-dedupe-key was in the
   envelope contract (Revision 4/5): a general-purpose sequence used as a
   correctness-critical dedupe key for a narrower concern it wasn't
   designed for. Fixed by giving the accepted candidate its **own**
   sequence, `CandidateSeq`, incrementing only on `ACCEPT`, immune to being
   clobbered by unrelated later decisions. See section 10.

------------------------------------------------------------------------

# Changes from Revision 4 (user-directed scope correction, part 1, 2026-07-19 — item 1 corrected in Revision 6, see below)

**Not a review round — the same project-wide scope boundary applied
identically across every upstream TradePol contract in this pass (see
`Chakra_OrderManagement_Handoff_Interface_Contract.md`'s "Changes from
Revision 3" for the full statement).** Chakra does not own portfolio or
account-level risk management; the user configures the chart, instrument,
and trade account manually. This cancels the previously planned
`Chakra_ExecutionContext_AccountInstrument_Interface_Contract.md` and
corrects two spots here:

1. **Open question 5 ("which position-state source is authoritative for
   lifecycle applicability?") named "a Chakra-level position-snapshot
   abstraction sitting in front of" the candle baton manager's own state as
   one of two candidate answers.** That candidate is now excluded — it is
   exactly the kind of general-purpose account/instrument context service
   this project's scope boundary rules out.
   **~~The remaining answer (reading the candle baton manager's own
   existing sync/position state directly) is the only one consistent
   with "Chakra operates within the manually configured chart
   context."~~ — this conclusion was WRONG and is corrected in Revision 6
   below.** `K_Util_Candle_Baton_Manager_V1` is a post-fill follower with
   no order-submission logic and no visibility into anything before a
   trade is confirmed live and handed off to it (per the central finding
   of `Chakra_OrderManagement_Handoff_Interface_Contract.md`, already
   established before this revision was written) — it cannot be, and
   should never have been cited as, an authoritative source for a
   *pre-fill* pending/working-entry check. Excluding the Chakra-level
   abstraction candidate was correct; concluding the candle manager was
   the remaining answer was not — a third answer (direct platform order
   state) was missed. See Revision 6.
2. **Open question 3's "portfolio-level concern" framing is corrected** —
   a portfolio-level arbitration component is excluded by this project's
   scope boundary, not an undesigned future extension; the concrete case
   that matters (multiple Composer sources feeding one Entry Authority
   instance) is resolved in
   `Chakra_OrderManagement_Handoff_Interface_Contract.md` section 6.
   **(This point stands; it is not corrected by Revision 6.)**

Nothing about section 6's lifecycle-applicability rules, the accept/
reject/undecidable decision shape, `CandidateSeq`, or any other settled
part of this contract changes.

------------------------------------------------------------------------

# Changes from Revision 7 (independent review-correction pass, 2026-07-19)

**A real gap found on review, of the same shape as the Revision 7 "pending"
fix — this time the impossible comparison was instrument/account identity,
not platform visibility.** Section 6's position-facts list and section
11's lifecycle-revalidation checklist both still required "instrument/
account identity still matches" — but `EntryCandidate` (and every record
downstream of it) publishes no instrument or account field to compare
against; there is nothing to check. Fixed: both bullets removed. This
EntryGate instance is manually wired to one chart/instrument/account
(section 1's "one instance, one coherent concept" rule) — consistency
comes from that wiring, not a runtime comparison — and the Entry Authority
independently confirms its own manually configured symbol/account
immediately before submission (`Chakra_OrderManagement_Handoff_Interface_Contract.md`
section 8), which is a configuration/platform-state check, not an
identity comparison either. **Nothing about the corrected pending-entry
fix from Revision 7 is reopened.**

------------------------------------------------------------------------

# Changes from Revision 6 (independent review-correction pass, 2026-07-19)

**A real gap found on review of Revision 6's own fix, corrected — Revision
6 corrected *who* reads pre-fill lifecycle state (platform, never the
candle baton manager) but left *what* is actually visible that way
overstated.** Section 6's position-facts list still included "a pending
entry already exists (proposed but not yet submitted/working)" as if it
were a platform-queryable fact — it isn't. The platform has no concept of
an unsubmitted Chakra-internal proposal; only the Entry Authority's own
private in-flight-submission marker tracks that
(`Chakra_OrderManagement_Handoff_Interface_Contract.md` section 10), and
it is not visible to EntryGate or any stage upstream of the Entry
Authority. Fixed: section 6's checklist now lists only what platform
queries actually answer (flat/not-flat, working order state), with an
explicit note that "pending" downstream state is out of scope for this
check by design — EntryGate's own candidate delivery already handles the
"another candidate incoming" case via ordinary latest-wins overwrite
(section 10), not a lifecycle block. The matching "Lifecycle revalidation"
subsection's "no incompatible pending candidate/action from another
source" bullet is corrected the same way. **Nothing about the corrected
source (platform, not the candle manager) from Revision 6 is reopened —
this fix only narrows what that platform-sourced check claims to cover.**

------------------------------------------------------------------------

# Changes from Revision 5 (user-directed correction, 2026-07-19)

**A direct user correction to Revision 4's own item 1, above — the first
pass swapped one wrong answer (a new shared Chakra abstraction) for
another wrong answer (the candle baton manager) instead of the right one.**
The user restated the actual responsibility split:

- **Entry Authority/platform order state owns pre-fill pending and
  working-entry validation.** Before a trade is confirmed live, the only
  authoritative facts available are the platform's own order/position
  state for the manually configured symbol/account (ACSIL position and
  working-order queries) — read directly, not through any Chakra-level
  abstraction and not through `K_Util_Candle_Baton_Manager_V1`, which has
  zero pre-fill visibility by design (its own two design documents state
  this as an explicit non-goal; see the Order Management Handoff
  contract's own central finding).
- **`K_Util_Candle_Baton_Manager_V1` may provide managed-position/trade
  state only *after* a confirmed fill and handoff** — exactly the same
  moment the Order Management Handoff contract's own section 11 already
  defines as when that manager first becomes aware a trade exists at all.
  Citing it as a source for anything before that moment is not a
  simplification, it is describing a query against data that does not yet
  exist there.
- **EntryGate makes only a preliminary lifecycle decision from available
  configured inputs** (section 6, corrected below) — a best-effort,
  non-final read of platform order state, the same standing every
  upstream-of-submission stage's own lifecycle check already has
  throughout this framework (PositionSizer section 5, the Entry Composer).
  **The Entry Authority independently revalidates platform state
  immediately before submission**
  (`Chakra_OrderManagement_Handoff_Interface_Contract.md` section 8) —
  including against its own private in-flight-submission marker that no
  upstream stage, including EntryGate, can see — and that revalidation,
  not EntryGate's own preliminary check, is what's actually authoritative
  at the moment it matters.
- **No new shared Execution Context or position-snapshot service is
  created to support this** — the correction is entirely about *which
  already-available platform facts* EntryGate's own preliminary check
  reads, not a new component. This is consistent with, not a reopening
  of, Revision 4's own exclusion of the Chakra-level abstraction
  candidate.

**Section 6 ("Position/lifecycle snapshot") is corrected below** to
describe platform order state as the source, with EntryGate's own read
explicitly labeled preliminary/non-final. **Open question 5 is corrected**
to no longer claim this is resolved via the candle manager. Nothing about
section 6's ordering rules (determine applicability first; consume-and-
drop while ineligible; never replay a trigger observed mid-trade),
the accept/reject/undecidable decision shape, or `CandidateSeq` changes —
only *which platform facts* "flat / pending / working" reads from.

------------------------------------------------------------------------

# Changes from Revision 3

One small, purely additive correction, made while designing the Entry
Composer's own contract — not a reopening of anything settled here:

1. **`K_CHAKRA_TRADEPOL_CATEGORY` was missing the Entry Composer's own
   category value.** This enum is the canonical, TradePol-layer-wide
   catalog (section 1) — every sub-component's own contract reuses it
   rather than redefining it. The Entry Composer, whose own contract now
   exists (`Chakra_TradePol_EntryComposer_Interface_Contract.md`), is a
   sixth distinct `PolicyCategory` role and was never added when the
   original five were reserved (EntryGate/PositionSizer/StopManager/
   TrailingManager/ProfitManager). Fixed: added
   `K_CHAKRA_TRADEPOL_CATEGORY_ENTRY_COMPOSER = 6` to the enum below.
   Nothing else about the enum, or about EntryGate's own identity/behavior,
   changes — this is strictly additive (a new value at the end, existing
   values unchanged) and required rather than reusing an existing category
   for the Entry Composer, per this framework's standing rule that a
   distinct role always gets its own identity.

------------------------------------------------------------------------

# Changes from Revision 2

Two further material follow-ups from review:

1. **`CandidateSeq`/`CandidateStatus`/expiry alone do not make an accepted
   candidate safe to act on later — lifecycle can go stale independently
   of time.** Worked example: EntryGate accepts candidate 7 while flat;
   before `PositionSizer` gets to it, some other candidate path creates a
   pending or working entry; `PositionSizer` observes candidate 7 — still
   `ACTIVE`, still unexpired — and, without revalidating lifecycle state,
   sizes a now-incompatible second entry. Revision 2 only guarded against
   *time*-based staleness (`CandidateValidUntilDateTime`); it never
   required re-checking whether the world had changed in the meantime.
   Fixed: every consumer of the candidate record — `PositionSizer` **and**
   the eventual Entry Composer, independently, each immediately before its
   own action — must revalidate authoritative lifecycle state, not just
   trust that it was valid when EntryGate accepted it. See section 11
   (retitled).
2. **The base architecture's bounded-retention logging rule never
   mentioned `POLICY_DECISION`.** It still said "log when `PROPOSED_ACTION`
   or `EXECUTABLE_INSTRUCTION` occurs" — read literally, this drops exactly
   the records that explain *why no trade occurred* (an accepted candidate,
   a rejected/expired/undecidable trigger, a lifecycle-consumed trigger).
   `Algo_Overlay_Framework_Architecture.md` is amended: sparse logging now
   explicitly includes *meaningful* `POLICY_DECISION` records, still
   suppressing routine `NOT_APPLICABLE`/`NO_ACTION` and ordinary
   `NO_CHANGE`. Section 13 here already anticipated this in spirit; it's
   now explicitly aligned with the corrected architecture wording.

------------------------------------------------------------------------

# 1. Naming and identity

- **`ComponentKind`** is always `K_CHAKRA_KIND_TRADEPOL`.
- **`PolicyCategory`** (int32, conventionally PV 21 — a **TradePol-layer-wide**
  reserved slot, not EntryGate-specific; every future `PositionSizer`/
  `StopManager`/`TrailingManager`/`ProfitManager` contract reserves the
  same slot for the same field) identifies *which of the five Trade
  Policy roles* a given study plays:
  ```cpp
  enum K_CHAKRA_TRADEPOL_CATEGORY
  {
      K_CHAKRA_TRADEPOL_CATEGORY_NONE             = 0,
      K_CHAKRA_TRADEPOL_CATEGORY_ENTRY_GATE       = 1,
      K_CHAKRA_TRADEPOL_CATEGORY_POSITION_SIZER   = 2,
      K_CHAKRA_TRADEPOL_CATEGORY_STOP_MANAGER     = 3,
      K_CHAKRA_TRADEPOL_CATEGORY_TRAILING_MANAGER = 4,
      K_CHAKRA_TRADEPOL_CATEGORY_PROFIT_MANAGER   = 5,
      K_CHAKRA_TRADEPOL_CATEGORY_ENTRY_COMPOSER   = 6   // added in Revision 4, see "Changes from Revision 3"
  };
  ```
  This exists for the same reason `ComponentKind` exists at the envelope
  level: a generic consumer can filter "give me every EntryGate on this
  chart" without needing to already know the full, ever-growing list of
  specific `ProducerTypeID` values.
- **`ProducerTypeID`** is scoped *per category*, not shared across all of
  Trade Policy — an EntryGate's `ProducerTypeID = 1` and a (future)
  `PositionSizer`'s `ProducerTypeID = 1` are unrelated identities, exactly
  as `PayloadSchemaVersion` is already scoped per `ProducerTypeID`
  elsewhere in this framework. EntryGate's own enum:
  ```cpp
  enum K_CHAKRA_TRADEPOL_ENTRYGATE_TYPE
  {
      K_CHAKRA_TRADEPOL_ENTRYGATE_TYPE_NONE       = 0,
      K_CHAKRA_TRADEPOL_ENTRYGATE_TYPE_ORBREAKOUT = 1,
      // one member per concrete named EntryGate policy, assigned as each is built
  };
  ```
- **`PayloadSchemaVersion`** (int32 PV 20) scoped per `ProducerTypeID`,
  unchanged convention.
- **One EntryGate instance implements one coherent entry policy for one
  expected TriggerStr concept.** It is not a generic "accepts any trigger"
  gate — it is built against a specific, named `TriggerStr` `ProducerTypeID`
  and a specific, documented set of acceptance rules for that trigger.

Example: `K_Chakra_TradePol_EntryGate_ORBreakout_V1`.

**Identity validation order, extended for this layer:** `SchemaVersion` ->
`ComponentKind` -> `PolicyCategory` -> `ProducerTypeID` ->
`PayloadSchemaVersion` -> only then `Status`/payload. `PolicyCategory` is a
new, required link in the chain inserted specifically because Trade Policy
is the first layer with more than one distinct role sharing one
`ComponentKind`.

------------------------------------------------------------------------

# 2. Decision-cycle applicability

EntryGate participates only when all three hold:

- The configured master decision cycle advances (`Algo_Overlay_Framework_Architecture.md`,
  "Decision Cycle and Master Clock" — EntryGate evaluates once per master
  cycle, not on its own independent cadence, unless the whole system is in
  an explicitly named Intrabar Decision Cycle mode).
- The position lifecycle permits considering a new entry (section 6).
- A new TriggerStr event is available for consideration (section 5).

**Generic Trade Policy outcome** (the architecture's five-value set;
EntryGate uses three of them):

- **`NOT_APPLICABLE`** — **nothing was consumed at all this cycle**, for
  any reason (lifecycle-ineligible with no trigger pending, or otherwise
  irrelevant). If EntryGate is lifecycle-ineligible *and* a trigger is
  observed and drained (section 6), that is **not** `NOT_APPLICABLE` — see
  `POLICY_DECISION` below. This is a stricter reading than Revision 1 used
  (see "Changes from Revision 1," item 2): `NOT_APPLICABLE` means a true
  no-op, not merely "no entry resulted."
- **`NO_ACTION`** — flat and eligible, but no new TriggerStr event is
  pending this cycle. Nothing to consume, nothing to decide.
- **`POLICY_DECISION`** — a trigger was consumed and EntryGate produced a
  decision record about it: `ACCEPT`, `REJECT`, or any `SKIP_*`
  (section 3), including the lifecycle-ineligible-drain case above. This is
  EntryGate's normal, expected outcome whenever it has something to say —
  see section 4 for why this is `POLICY_DECISION` and never
  `PROPOSED_ACTION`.

**Neither `NO_CHANGE` nor `PROPOSED_ACTION` is ever produced by EntryGate
itself.** `NO_CHANGE` requires "a real persistent recommendation that can
remain unchanged" — entry decisions are discrete, one-shot events, not a
continuously-maintained recommendation the way a `TrailingManager`'s stop
level would be; it remains available in the generic framework for
sub-components that genuinely have that shape. `PROPOSED_ACTION` is
reserved for a complete, manager-ready action — EntryGate's own output is
never complete on its own (section 4); only the Entry Composer, once it
exists, can produce a genuine `PROPOSED_ACTION` from what EntryGate (and
`PositionSizer`/`StopManager`) contribute.

**A policy-specific decision enum exists independently of the generic
outcome above** — see section 3. The generic outcome says "did EntryGate
do anything this cycle"; the policy-specific enum says "what, exactly."

------------------------------------------------------------------------

# 3. Policy-specific decision results

```cpp
enum K_CHAKRA_ENTRY_GATE_DECISION
{
    K_CHAKRA_ENTRY_GATE_DECISION_NONE              = 0,
    K_CHAKRA_ENTRY_GATE_DECISION_ACCEPT             = 1,
    K_CHAKRA_ENTRY_GATE_DECISION_REJECT             = 2,
    K_CHAKRA_ENTRY_GATE_DECISION_SKIP_EXPIRED       = 3,
    K_CHAKRA_ENTRY_GATE_DECISION_SKIP_UNDECIDABLE   = 4,
    K_CHAKRA_ENTRY_GATE_DECISION_SKIP_LIFECYCLE     = 5
};
```

`NONE = 0` is the required "nothing decided yet / not currently valid"
sentinel, matching every other event-style `...Type` field in this
framework (`EventType`, etc.).

Required cases and how they map (some deliberately share a `DecisionType`
value, disambiguated by `ReasonCode` instead — see section 14):

| Required case | `DecisionType` | Notes |
|---|---|---|
| No trigger pending | *(not this enum — generic `NO_ACTION`, section 2)* | Nothing was consumed, so there is no decision record to produce. |
| Expired trigger | `SKIP_EXPIRED` | Consumed, but past its `EventValidUntilDateTime` (section 5). |
| Accepted trigger | `ACCEPT` | |
| Rejected trigger | `REJECT` | A named local rule (section 9) said no. |
| Required dependency invalid/stale | `SKIP_UNDECIDABLE` | Distinguished from the next row by `ReasonCode`, not `DecisionType`. |
| Unhandled context combination | `SKIP_UNDECIDABLE` | Same `DecisionType` as above — both are "genuinely cannot decide," for different reasons; `ReasonCode` carries the distinction (section 14). |
| Incompatible position/lifecycle state | `SKIP_LIFECYCLE` | |

**Do not collapse `REJECT`, `SKIP_EXPIRED`, or `SKIP_UNDECIDABLE` into the
generic `NO_ACTION` outcome.** All three represent "a trigger was consumed
and a real decision was made about it" — the generic outcome for all of
them is `POLICY_DECISION` (section 2), never `NO_ACTION`. `NO_ACTION` is
reserved strictly for "there was nothing to consume in the first place."

------------------------------------------------------------------------

# 4. Important unresolved composition boundary

**Freeze this explicitly: an accepted EntryGate decision is not yet a
complete order proposal.** It lacks at least:

- Position quantity, from `PositionSizer`.
- Initial stop behavior, from `StopManager`.
- Possibly target/profit instructions.

**Therefore EntryGate publishes a durable, accepted `EntryCandidate` for
downstream Trade Policy components to consume — it never sends
ENTER LONG/SHORT directly to the candle baton manager.** The pipeline:

```text
Trigger event
    -> EntryGate decision (POLICY_DECISION, DecisionSeq)
    -> accepted EntryCandidate (POLICY_DECISION with a candidate, CandidateSeq — section 10)
    -> PositionSizer
    -> initial StopManager/Profit policy
    -> complete proposed entry action (PROPOSED_ACTION, produced by the Entry Composer)
    -> candle baton manager
```

The component that performs the final composition step (`EntryCandidate` +
size + initial stop/profit instructions -> one complete proposed entry
action) is **not designed in this pass** — its own contract comes later,
once `PositionSizer` and `StopManager` exist. It is referred to here only
as **"the Entry Composer"**, a reserved name so this contract can state the
boundary precisely without inventing the component behind it. **This
reservation exists specifically so an incomplete entry can never
accidentally become executable** — there is nowhere for a bare
`EntryCandidate` to go except through the Composer; nothing in this
contract lets EntryGate skip that step.

**Resolved, not left as a flagged tension (corrected from Revision 1 — see
"Changes from Revision 1," item 1).** The base architecture's "Decision
Cycle and Master Clock" section now defines `POLICY_DECISION` as its own
generic Trade Policy outcome, distinct from `PROPOSED_ACTION`: `ACCEPT`,
`REJECT`, and every `SKIP_*` are all `POLICY_DECISION` — genuine, durable
decisions, never something Order Management is asked to validate.
`PROPOSED_ACTION` is reserved exclusively for what the Entry Composer
eventually produces, once an `EntryCandidate` has been fully assembled with
size and an initial stop. EntryGate itself never produces a
`PROPOSED_ACTION`. This closes the apparent conflict Revision 1 identified
but didn't resolve — see section 2.

------------------------------------------------------------------------

# 5. TriggerStr consumption

Full reuse of the TriggerStr contract's own consumer-side rules (its
section 11), applied here without modification:

- **Exact identity validation before any payload is read**: envelope
  `SchemaVersion` -> `ComponentKind` (`K_CHAKRA_KIND_TRIGGERSTR`) ->
  expected TriggerStr `ProducerTypeID` -> TriggerStr `PayloadSchemaVersion`
  -> only then `EventType`/`EventSeq`/payload.
- **The event gate**: `EventType != NONE && EventSeq !=
  LastConsumedEventSeq` — **not** gated on `Status == VALID`, per
  TriggerStr's own corrected model (its Revision 2 fix): only
  disable-entry and recalc-entry on the TriggerStr side ever clear its
  sticky record, so `EventType != NONE` alone is the correct "is there an
  undeleted fired record" signal.
- **Event applicability** — the consumed event's `Applicability` must
  contain the side EntryGate is considering (section 7).
- **`EventValidUntilDateTime`** — checked against the master cycle's own
  evaluation time (section 2), using availability time, **never**
  `PertainsToDateTime` (TriggerStr section 15).
- **Lifecycle baselining, all four boundaries, the same unified mechanism
  TriggerStr and MCtx already established**: baseline
  `LastConsumedEventSeq` to the TriggerStr's *current* `EventSeq` on
  EntryGate's own initial attachment to that TriggerStr, on rewiring, on
  EntryGate's own `IsFullRecalculation`, and on EntryGate's own
  disable -> re-enable. **Never replay an existing sticky event at any of
  these boundaries** — baselining and consuming never happen in the same
  evaluation.
- **Default failure policy is consume-and-drop**, per TriggerStr's own
  stated consumer expectation.
- **Gap detection captures `previousSeq` before mutating
  `LastConsumedEventSeq`**, per the envelope's corrected pattern (its
  Revision 7), branching forward-gap-logged versus backward-movement-
  logged-as-probable-storage-reset, never a raw arithmetic difference.
- **A consumed trigger must produce exactly one deterministic decision
  record, even when rejected or dropped.** There is no "consumed but
  produced nothing" state — every consumption maps to some `DecisionType`
  value (section 3).

------------------------------------------------------------------------

# 6. Position/lifecycle snapshot

**Source, corrected in Revision 6 (see "Changes from Revision 5"): these
are pre-fill facts, read directly from platform order/position state for
the manually configured symbol/account (ACSIL position and working-order
queries) — never from `K_Util_Candle_Baton_Manager_V1`, which has no
visibility into anything before a trade is confirmed live and handed off
to it.** EntryGate's own read here is a **preliminary, best-effort check**
— the same standing every pre-submission stage's own lifecycle check has
throughout this pipeline; the Entry Authority performs the final,
authoritative revalidation immediately before actually submitting an order
(`Chakra_OrderManagement_Handoff_Interface_Contract.md` section 8),
including against its own private in-flight-submission marker that
EntryGate cannot see.

EntryGate requires these position facts before consuming a trigger:

- Flat / not flat (platform position query).
- A working entry order already exists, submitted and awaiting fill
  (platform order query).
- The current decision-cycle identity (section 10).

**Not instrument/account identity (corrected on review) — there is no
platform-visible fact to check here, and no field to check it against.**
This EntryGate instance is manually wired to one specific chart/
instrument/account (section 1); consistency comes from that wiring, not
a runtime comparison. The Entry Authority independently confirms its own
manually configured symbol/account immediately before submission
(`Chakra_OrderManagement_Handoff_Interface_Contract.md` section 8).

**Explicitly not one of these, and not checkable via platform state at
all (corrected on review — a real gap in the Revision 6 lifecycle-source
fix, which corrected *who* reads pre-fill state but not *what* is
actually visible that way): whether a proposal is already "pending"
further downstream** — already composed and in flight at the Entry
Authority, or awaiting composition by `PositionSizer`/`StopManager`/the
Entry Composer. The platform has no concept of an unsubmitted
Chakra-internal proposal; only the Entry Authority's own private
in-flight-submission marker tracks that
(`Chakra_OrderManagement_Handoff_Interface_Contract.md` section 10), and
it is not visible to EntryGate or any earlier stage. EntryGate does not
need a check for this: its own candidate delivery is a single sticky slot
under latest-wins overwrite (section 10) — a new accepted candidate while
an earlier one is still unconsumed simply supersedes it, detected
downstream via sequence-gap logging, not blocked here.

**Crucial ordering, exactly as specified:**

1. Determine whether EntryGate is lifecycle-applicable (flat, no working
   entry) **first**, before doing anything else.
2. If not applicable, decide whether the trigger should remain unseen (not
   yet consumed) or be consumed-and-dropped.
3. **Never allow a trigger observed while already in a trade to replay
   later once the position becomes flat again.**

**Default recommendation: consume-and-drop newly observed entry triggers
while lifecycle-ineligible, with a reason, so they cannot backlog and
surface as stale entries later.** Concretely: even while EntryGate's
*substantive* accept/reject logic never runs (lifecycle-ineligible), the
TriggerStr consumption bookkeeping from section 5 still runs —
`LastConsumedEventSeq` still advances past any observed trigger. **This
produces a real decision record** — `DecisionType = SKIP_LIFECYCLE`,
`DecisionSeq` increments — and the cycle's generic outcome is therefore
`POLICY_DECISION`, not `NOT_APPLICABLE` (corrected from Revision 1, whose
claim that this was still `NOT_APPLICABLE` while producing a durable
record was an internal contradiction — see "Changes from Revision 1," item
2, and section 2). This exists for exactly one reason: without it, a
trigger that fired while EntryGate was ineligible would still sit
un-consumed, and could resurface as if freshly fired the moment the
position becomes flat again — rule 3, violated. `NOT_APPLICABLE` is
reserved for the case where lifecycle-ineligibility coincides with no
trigger being observed at all — a true no-op, nothing to drain, nothing to
record.

------------------------------------------------------------------------

# 7. Direction and side safety

- **TriggerStr `Applicability` must contain exactly the side being
  considered.** EntryGate does not widen or narrow it.
- **`EntryCandidate` preserves long/short applicability unchanged** from
  the consumed trigger event, once resolved (below).
- **`NONE` is invalid.** A consumed event with `EventType != NONE` but
  `Applicability == NONE` is a producer inconsistency EntryGate must not
  paper over — treat as `SKIP_UNDECIDABLE`, never guess a side.
- **`BOTH` must be resolved by an explicit, documented, code-constant
  EntryGate rule before an accepted candidate is ever published — never
  let an ambiguous side reach `PositionSizer` or Order Management.** E.g.
  "if `Applicability == BOTH`, accept only the side matching the current
  `Trend` MCtx bias; if `Trend` doesn't clearly favor either side, `REJECT`."
  Whatever the rule, it must be stated in that EntryGate's own
  documentation, not left implicit.
- **Current position/order state must be checked for incompatible or
  opposing exposure** — an accepted candidate for one side while
  incompatible exposure already exists on the other is a lifecycle
  violation (section 6), not a direction question.

**Do not infer direction from context bias when the trigger event already
owns authoritative applicability.** An MCtx's `Direction`/`Bias` field is
purely descriptive (MCtx contract, section 8) and may be used as one
*input* to an accept/reject rule (section 9) — e.g. "reject if `Trend`
bias contradicts the trigger's side" — but it is never the *source* of
which side to trade. TriggerStr's `Applicability` is authoritative for
that, full stop.

------------------------------------------------------------------------

# 8. Sensor and MCtx dependencies

Each dependency declaration specifies, reusing the discipline already
established for MCtx's own Sensor dependencies (MCtx contract, section 4)
without modification:

- Required versus optional.
- Expected producer (`ProducerTypeID`) and `PayloadSchemaVersion`.
- Which specific fields are read.
- Freshness tolerance EntryGate itself chooses.
- Live state access via the state pattern (state-style dependencies read
  repeatedly, no event-style deduplication).
- Historical subgraph mapping, **if** EntryGate maintains historical
  decision output (section 12) — same live-versus-historical split and
  availability-indexed-subgraph rule already established for MCtx and
  TriggerStr.
- Source availability/provenance where the dependency is itself
  cross-chart (Sensor contract, section 4 — `SourceDataAsOfDateTime`, not
  `SourcePeriodStartDateTime`, for freshness).

**Policy-specific rule (EntryGate's own version of the "truth condition"
rule already established for TriggerStr, section 5 there):**

> Any dependency capable of changing ACCEPT versus REJECT is required for
> that EntryGate policy. Optional dependencies may add diagnostics or
> provenance but cannot change the decision.

If a required dependency is invalid, stale, or unwired: **do not guess.**
Consume-and-drop the trigger, publish `SKIP_UNDECIDABLE`, record a
deterministic reason (section 14).

------------------------------------------------------------------------

# 9. Local context reconciliation

EntryGate owns reconciliation only for the contexts it itself consumes —
this is the base architecture's already-settled Conflict Resolution
principle (contexts don't arbitrate against each other; reconciliation is
a **local policy problem**; no global registry), now given its first real
implementation.

Every EntryGate policy requires declarative, code-versioned rules
specifying:

- Exact context states considered.
- Precedence, if more than one rule could apply.
- Exceptions/`UNLESS` clauses.
- Directional handling (how a rule interacts with section 7's side
  resolution).
- Default behavior for unhandled combinations.

```text
EntryGate_ORBreakout — Rule A

IF WeeklyExpansion == UNDER_EXPANDED
THEN ACCEPT
UNLESS TrendStretch == EXTREME
```

**Follow the architecture's established fallback exactly:**

```text
Unhandled combination
    -> do not guess
    -> consume trigger
    -> SKIP_UNDECIDABLE
    -> log contexts, trigger, cycle, and reason
```

No global conflict registry, ever. Thresholds and decision rules remain
code constants / versioned code — never hot-editable runtime calibration,
per the base architecture's versioning rule and MCtx's own corrected
wiring/display/meaning-preserving-cadence boundary (MCtx section 6),
applied identically here: an EntryGate's *acceptance criteria* are never
ACSIL `Input`s.

------------------------------------------------------------------------

# 10. Output contract

EntryGate publishes **two separate sticky records**, not one, because they
have different consumers and different correctness requirements —
collapsing them into a single sequence is a real bug, not a simplification
(see "Changes from Revision 1," item 3):

- **The decision record** (`DecisionSeq`-keyed) — the audit trail. Every
  decision, of every type, increments this. Consumed by diagnostics,
  logging, and `DecisionFrame`.
- **The candidate record** (`CandidateSeq`-keyed) — the correctness-critical
  delivery mechanism to `PositionSizer`. Increments **only** on `ACCEPT`.

## Decision record

**`DecisionSeq` occupies the same conventional slot TriggerStr's `EventSeq`
uses (int64 PV 10) — same structural role (payload-owned, monotonic,
never deliberately reset, the dedupe anchor for the event pattern), but a
distinct name and distinct semantics.** `DecisionSeq` increments once per
EntryGate *decision* (`ACCEPT`/`REJECT`/any `SKIP_*`), not once per raw
TriggerStr firing — the two sequences live on two different producers and
must never be confused with each other by a downstream consumer, even
though they occupy the same PV number by convention.

| Field | Namespace/PV | Purpose |
|---|---|---|
| `PayloadSchemaVersion` | Int32 PV 20 | Envelope convention |
| `PolicyCategory` | Int32 PV 21 | Section 1, TradePol-layer-wide |
| `DecisionType` | Int32 PV 22 | Section 3 |
| `ResolvedSide` | Int32 PV 23 | Section 7 — `LONG`/`SHORT` only in output, never `BOTH`/`NONE` |
| `SourceTriggerChartNumber` | Int32 PV 24 | Provenance |
| `SourceTriggerStudyID` | Int32 PV 25 | Provenance |
| `SourceTriggerProducerTypeID` | Int32 PV 26 | Provenance |
| `SourceTriggerEventType` | Int32 PV 27 | The consumed trigger's own `EventType` |
| `DecisionSeq` | Int64 PV 10 | This section |
| `SourceTriggerEventSeq` | Int64 PV 11 | Which specific trigger occurrence this decision is about |
| `MasterCycleSeq` | Int64 PV 12 | The decision cycle this decision belongs to |
| `SourceTriggerAvailabilityDateTime` | Datetime PV 10 | Propagated from the trigger's `EvalDateTime` |
| `SourceTriggerEventValidUntilDateTime` | Datetime PV 11 | Propagated from the trigger's own expiry |

Sticky until superseded by a newer decision or cleared at EntryGate's own
disable-entry/recalc-entry — TriggerStr's own rule (its section 2),
applied here without modification.

## Candidate record (new in Revision 2)

**Why a separate sequence is required, not optional:** `DecisionSeq`
increments on `REJECT`/`SKIP_*` just as much as on `ACCEPT`. If
`PositionSizer` used `DecisionSeq` as its own dedupe key, a later
`REJECT`/`SKIP_*` — a different, later trigger occurrence entirely — would
overwrite the sticky payload and silently erase an already-accepted,
not-yet-consumed candidate, with no way to recover it. Nothing in this
framework guarantees `PositionSizer` consumes an accepted candidate within
the same cycle it was published (that would require designing a physical,
synchronous cross-study coordinator, which this pass deliberately does not
attempt — see open questions). A separate `CandidateSeq` sidesteps the
problem entirely rather than requiring that guarantee: this is the same
lesson as `PublicationSeq` being unsafe as the envelope's own event dedupe
key (envelope Revision 4/5), applied here one layer up.

- **`CandidateSeq`** (int64, PV 13) — increments **only** on `ACCEPT`.
  Never touched by `REJECT`/`SKIP_*`. This is `PositionSizer`'s dedupe
  anchor, and the *only* field it needs to determine "is there a new
  candidate."
- **`CandidateStatus`** (int32, PV 28) — `NONE = 0` / `ACTIVE = 1`,
  mirroring the `EventType`-style "is there a live record" sentinel. Set to
  `ACTIVE` on `ACCEPT`; cleared to `NONE` only at EntryGate's own
  disable-entry/recalc-entry (the same two exclusive clear boundaries as
  everywhere else in this framework — **never** as a side effect of a
  later `REJECT`/`SKIP_*` decision, which is about a different, later
  trigger occurrence entirely and must not retroactively affect a
  previously-accepted, still-live candidate).
  ```cpp
  enum K_CHAKRA_ENTRYGATE_CANDIDATE_STATUS
  {
      K_CHAKRA_ENTRYGATE_CANDIDATE_NONE   = 0,
      K_CHAKRA_ENTRYGATE_CANDIDATE_ACTIVE = 1
  };
  ```
- **Self-contained payload** — the candidate record duplicates what
  `PositionSizer` needs rather than requiring it to cross-reference the
  decision record:

  | Field | Namespace/PV | Purpose |
  |---|---|---|
  | `CandidateStatus` | Int32 PV 28 | Above |
  | `CandidateResolvedSide` | Int32 PV 29 | Section 7 — self-contained copy, `LONG`/`SHORT` only |
  | `CandidateSeq` | Int64 PV 13 | Above |
  | `CandidateSourceDecisionSeq` | Int64 PV 14 | Links back to the `DecisionSeq` that produced this candidate, for traceability |
  | `CandidateSourceTriggerEventSeq` | Int64 PV 15 | Self-contained copy — which trigger occurrence this candidate came from |
  | `CandidateValidUntilDateTime` | Datetime PV 12 | Section 11 |

- **`PositionSizer`'s gate is the event pattern, one layer further
  downstream**: `identityValid && CandidateStatus != NONE && CandidateSeq
  != LastConsumedCandidateSeq`. This is structurally identical to how
  EntryGate itself consumes TriggerStr (section 5) — same lifecycle
  baselining (attachment/rewiring/`PositionSizer`'s own recalc/disable-
  re-enable), same consume-and-drop default, same gap-detection discipline.
  `PositionSizer`'s own future contract pass should start from this
  section rather than re-deriving it, the same courtesy this contract
  received from TriggerStr and MCtx.

**Provenance for the Sensor/MCtx states actually used** (decision record)
is documented per EntryGate instance rather than given a fixed universal
shape (the *set* of consulted dependencies varies per policy) — the
recommended convention is to publish each consulted dependency's
`ProducerTypeID` and the `PublicationSeq` it was read at, in whatever
remaining slots that instance's payload needs, so an auditor can identify
exactly which snapshot versions fed a given decision.

------------------------------------------------------------------------

# 11. Candidate expiry and lifecycle revalidation

## Time-based expiry

**An accepted candidate can never outlive its source TriggerStr event:**

```text
CandidateValidUntilDateTime <= Trigger.EventValidUntilDateTime
```

`CandidateValidUntilDateTime` lives in the candidate record (section 10),
not the decision record — it governs whether `PositionSizer` may still act
on the candidate `CandidateSeq` identifies, independent of whatever
EntryGate has decided about *later* trigger occurrences in the meantime.

If downstream `PositionSizer`/`StopManager` processing (through the Entry
Composer, section 4) does not complete before `CandidateValidUntilDateTime`:

- Consume/drop the candidate (`PositionSizer` advances
  `LastConsumedCandidateSeq`, section 10) — the candidate record itself is
  **not** cleared by EntryGate on expiry; expiry is always a consumer-side
  check against the published deadline, the same discipline
  `EventValidUntilDateTime` already established (TriggerStr section 15).
- Do not form an executable entry.
- Log the expiry (section 14).

**Use a concrete timestamp, never a producer-chart bar index** — the same
rule TriggerStr's own `EventValidUntilDateTime` follows (section 15
there), for the identical cross-chart-ambiguity reason: a bar index has no
defined meaning outside its producer's own chart.

## Lifecycle revalidation (new in Revision 3 — a distinct, non-time-based staleness dimension)

**`CandidateSeq`/`CandidateStatus`/expiry are not sufficient to make an
accepted candidate safe to act on.** A candidate can be `ACTIVE` and
unexpired while the world it was accepted into has changed — another
candidate path may have created a pending or working entry, exposure may
have changed, the account/instrument mapping may have shifted. This is a
*lifecycle*-staleness dimension, distinct from and in addition to
time-based expiry above, and follows the same established rule that a
valid historical decision does not guarantee it remains actionable now
(the same principle already governing time-based expiry and TriggerStr's
own event-vs-actionability distinction).

**Every consumer of the candidate record must independently revalidate
authoritative lifecycle state immediately before acting on it — not trust
that it was valid when EntryGate accepted it:**

- **`PositionSizer`**, immediately before it sizes anything.
- **The Entry Composer**, immediately before it emits the complete
  `PROPOSED_ACTION` — **a second, independent check, not a reuse of
  `PositionSizer`'s result** — because conditions may change again in the
  time between `PositionSizer`'s evaluation and the Composer's own.

Each revalidation checks, at minimum, against platform state — **never
against another candidate/action "pending" in some other component,
which is not a platform-visible fact and not something this check can
see (corrected on review; see section 6, above, for the full
correction)**:

- Still flat, where the candidate's entry requires flat.
- No working entry order.
- No opposing exposure.
- The candidate is still unexpired (time-based check above, re-applied —
  time keeps passing between stages too).

**Not "instrument/account identity still matches" (corrected on review —
a real gap of the same shape as the "pending" fix directly above):
`EntryCandidate` publishes no instrument or account field to compare
against.** Consistency comes from manual wiring instead — this EntryGate
instance, and every consumer of its candidate, are configured against the
same one chart/instrument/account by construction (section 1's "one
instance, one coherent concept" rule), not verified by a runtime field
comparison that has nothing to compare. The Entry Authority separately
validates its own manually configured symbol/account is what it actually
submits against, immediately before submission
(`Chakra_OrderManagement_Handoff_Interface_Contract.md` section 8) — that
is a configuration/platform-state check, not an identity comparison
either.

**Whether an incompatible submission is already in flight at the Entry
Authority is that component's own idempotency check, immediately before
submission** (`Chakra_OrderManagement_Handoff_Interface_Contract.md`
section 10) — the one place in this pipeline where "in flight" is
actually visible, since only the Entry Authority holds that private
marker.

**If any check fails: consume-and-drop `CandidateSeq` with a
lifecycle-stale reason, form no executable entry, and log it.** This
mirrors section 6's own rule for TriggerStr consumption exactly, one stage
further downstream. The reason code for this belongs to whichever
component detects it (`PositionSizer`'s own diagnostics, or the Entry
Composer's) — not EntryGate's `ReasonCode` enum (section 14), per the
"`ReasonCode` is self-referential" principle already established
throughout this framework (TriggerStr section 18): EntryGate's own
diagnostics describe EntryGate's own condition, never a downstream
consumer's judgment about what it did with EntryGate's output.

------------------------------------------------------------------------

# 12. Recalculation and replay

- **No live decisions or orders during full recalculation** — EntryGate is
  event/decision-style, so this follows TriggerStr's own recalc rule
  exactly (its section 8): no external actions, ever, recalc or not.
- **No public `DecisionSeq` or `CandidateSeq` increments for historical
  decisions**, under any circumstance, matching TriggerStr's absolute rule
  that a historical occurrence is not a current opportunity to report. This
  applies equally to both records (section 10) — a historical `ACCEPT`
  must not produce a live-deliverable candidate any more than it produces
  a live decision.
- **Historical subgraphs may record `ACCEPT`/`REJECT`/`SKIP_*` results at
  availability time** — the same hidden Occurrence/Status subgraph pair
  pattern already established (Sensor Revision 3, TriggerStr section 16),
  entirely separate from and never driving the persistent `DecisionSeq`.
- **Historical dependency access uses availability-indexed Sensor/MCtx/
  TriggerStr machine subgraphs and the three-part historical-validity
  check** (array coverage, historical `Status == VALID`, finite/in-range
  value) — never the persistent payload, never array coverage alone.
- **Private prior-state and source-sequence trackers baseline at all four
  lifecycle boundaries** (attachment, rewiring, EntryGate's own recalc,
  EntryGate's own disable/re-enable) — section 5's rule, restated as a
  recalc/replay-specific reminder because it's the boundary most likely to
  be gotten wrong under load.
- **Replay's forward path produces decisions through the normal
  master-cycle path** — no special-casing beyond following the rules
  above correctly, per the envelope's Revision 6 clarification.

**Historical `ACCEPT` does not become a live `EntryCandidate`.** This is
TriggerStr's permanence-versus-actionability distinction (its section 14)
applied here: a historical `ACCEPT` recorded during a backtest recalc pass
is a fact for analysis, never something a live cycle treats as an
actionable candidate merely because a recalculation happened to touch that
bar.

------------------------------------------------------------------------

# 13. `DecisionFrame` participation

EntryGate contributes only what it actually consumed this cycle, per the
architecture's `DecisionFrame` scoping rule:

- Master cycle identity.
- The source Trigger event actually consumed (if any).
- The position/lifecycle snapshot actually relevant.
- The required Sensor/MCtx snapshots actually used.
- The generic Trade Policy outcome (`NOT_APPLICABLE`/`NO_ACTION`/
  `POLICY_DECISION`).
- The EntryGate-specific `DecisionType` and `ReasonCode`.
- The accepted candidate record (`CandidateSeq`-keyed, section 10), if one
  exists.

**Do not copy every available Chakra state into the frame** — the
architecture's own anti-pattern warning, restated because EntryGate is the
first component with enough dependencies (multiple Sensors, multiple
MCtx contexts, one TriggerStr) that the temptation to over-capture is real.

**Accepted candidates, rejected/expired/undecidable triggers, and
lifecycle-consumed triggers are all meaningful `POLICY_DECISION` records
and qualify for sparse decision logging** — the architecture's
bounded-retention rule now says so explicitly (amended alongside
`POLICY_DECISION`'s own introduction, see "Changes from Revision 2"),
precisely because these are the records that explain *why no trade
occurred*. None of them are treated as equivalent to a quiet
`NOT_APPLICABLE`/`NO_ACTION` cycle for logging purposes, even though only
an accepted candidate produces any further downstream output.

------------------------------------------------------------------------

# 14. Diagnostics

```cpp
enum K_CHAKRA_ENTRYGATE_REASON_CODE
{
    K_CHAKRA_ENTRYGATE_REASON_NONE                       = 0,   // = K_CHAKRA_REASON_NONE
    K_CHAKRA_ENTRYGATE_REASON_NO_NEW_TRIGGER              = 1,
    K_CHAKRA_ENTRYGATE_REASON_TRIGGER_EXPIRED             = 2,
    K_CHAKRA_ENTRYGATE_REASON_TRIGGER_SEQUENCE_GAP        = 3,
    K_CHAKRA_ENTRYGATE_REASON_TRIGGER_STORAGE_RESET       = 4,
    K_CHAKRA_ENTRYGATE_REASON_TRIGGER_SCHEMA_MISMATCH     = 5,
    K_CHAKRA_ENTRYGATE_REASON_INVALID_APPLICABILITY       = 6,
    K_CHAKRA_ENTRYGATE_REASON_LIFECYCLE_INELIGIBLE        = 7,
    K_CHAKRA_ENTRYGATE_REASON_ENTRY_ALREADY_PENDING       = 8,
    K_CHAKRA_ENTRYGATE_REASON_SENSOR_UNAVAILABLE_STALE    = 9,
    K_CHAKRA_ENTRYGATE_REASON_MCTX_UNAVAILABLE_STALE      = 10,
    K_CHAKRA_ENTRYGATE_REASON_UNHANDLED_CONTEXT_COMBINATION = 11,
    K_CHAKRA_ENTRYGATE_REASON_ACCEPTED                    = 12,
    K_CHAKRA_ENTRYGATE_REASON_REJECTED_LOCAL_RULE         = 13
    // extend per concrete EntryGate policy as needed, numbered above these common ones
};
```

`K_CHAKRA_ENTRYGATE_REASON_SENSOR_UNAVAILABLE_STALE` and `..._MCTX_..._STALE`
both resolve to `DecisionType = SKIP_UNDECIDABLE`, disambiguated here;
likewise `..._UNHANDLED_CONTEXT_COMBINATION`. **Log on decisions/
transitions, not repeatedly every master bar** — same convention as every
other layer in this framework.

**Removed in Revision 3: `CANDIDATE_EXPIRED_DOWNSTREAM`.** It described a
downstream consumer's own observation (the candidate expired, or went
lifecycle-stale, by the time `PositionSizer`/the Entry Composer looked at
it) — not something EntryGate itself detects or should report on. Per
section 11's `ReasonCode`-is-self-referential rule, that diagnostic belongs
to `PositionSizer`'s or the Entry Composer's own future `ReasonCode` enum,
never EntryGate's.

------------------------------------------------------------------------

# 15. Worked examples

1. **Accepted.** `K_Chakra_TradePol_EntryGate_ORBreakout_V1` consumes a
   fresh `BREAKOUT` event from `K_Chakra_TriggerStr_ORBreakout_V1`
   (`Applicability = LONG`) while `WeeklyExpansion.State =
   UNDER_EXPANDED`. Rule A (section 9) applies, no `UNLESS` triggers.
   Generic outcome `POLICY_DECISION`; `DecisionType = ACCEPT`, `ReasonCode
   = ACCEPTED`, `DecisionSeq` increments. Because this is an `ACCEPT`,
   `CandidateSeq` *also* increments and a candidate record is published
   (section 10) — `CandidateStatus = ACTIVE`, `CandidateResolvedSide =
   LONG`.
2. **Rejected.** Same trigger, same cycle shape, but a locally defined
   Rule B applies instead: `IF WeeklyExpansion == OVER_EXPANDED THEN
   REJECT`. `DecisionType = REJECT`, `ReasonCode = REJECTED_LOCAL_RULE`. No
   `EntryCandidate`.
3. **Expired.** The trigger fired several master cycles ago (a slow
   master clock, per the architecture's own documented consequence); by
   the time this cycle evaluates it, `consumer evaluation time >
   EventValidUntilDateTime`. `DecisionType = SKIP_EXPIRED`, `ReasonCode =
   TRIGGER_EXPIRED`. Consumed and dropped, per section 5.
4. **Lifecycle-ineligible.** A working entry order already exists when a
   fresh trigger fires. `LastConsumedEventSeq` advances (section 6),
   producing a real decision — `DecisionType = SKIP_LIFECYCLE` /
   `ReasonCode = ENTRY_ALREADY_PENDING`, `DecisionSeq` increments — so the
   generic outcome is `POLICY_DECISION`, **not** `NOT_APPLICABLE`
   (corrected from Revision 1). `CandidateSeq` is untouched (only `ACCEPT`
   moves it). The trigger is gone for good — it cannot resurface once flat
   again.
5. **Undecidable — dependency.** A required MCtx context is stale beyond
   its tolerance when the trigger fires. `DecisionType =
   SKIP_UNDECIDABLE`, `ReasonCode = MCTX_UNAVAILABLE_STALE`. Consumed and
   dropped — never guessed.
6. **Undecidable — unhandled combination.** `WeeklyExpansion = NORMAL` and
   `TrendStretch = EXTREME` occur together, and this EntryGate's rule set
   has no rule covering that combination. `DecisionType =
   SKIP_UNDECIDABLE`, `ReasonCode = UNHANDLED_CONTEXT_COMBINATION`, logged
   with the exact context states, trigger, and cycle per section 9's
   fallback.
7. **Invalid — policy leakage.** An EntryGate that computes a position
   size, picks a stop price, or calls an order-submission function
   directly. That's `PositionSizer`/`StopManager`/Order Management wearing
   an EntryGate's name — EntryGate's only legitimate output is an
   accept/reject decision and, on accept, a bare `EntryCandidate`.
8. **Staged pipeline in action.** Example 1's accepted candidate
   (`CandidateSeq`, `CandidateStatus = ACTIVE`) is consumed by
   `PositionSizer` (not yet designed) via the same event-pattern gate
   TriggerStr's own consumers use (section 10) — at this point it has
   **not** reached Order Management, has no quantity, and has no stop.
   Even if EntryGate's *decision* record moves on to a later `REJECT`/
   `SKIP_*` for a subsequent trigger in the meantime, this candidate stays
   exactly as published until `PositionSizer` consumes it or it expires
   (section 11) — `DecisionSeq` moving does not touch `CandidateSeq`. Only
   once the (not-yet-designed) Entry Composer assembles the candidate +
   size + initial stop into one complete proposed entry action does
   anything reach the candle baton manager.

------------------------------------------------------------------------

# Unresolved design questions — deliberately left open

Only questions that affect real implementation. Envelope, Sensor, MCtx,
TriggerStr, and base-architecture decisions already settled are not
reopened here.

1. **What component composes an accepted candidate + size + initial
   stop/profit instructions into one complete proposed entry action?**
   Referred to here only as "the Entry Composer" (section 4) — its own
   identity, contract, and relationship to `PositionSizer`/`StopManager`
   is undesigned. Its consumption of EntryGate's candidate record should
   follow section 10's `CandidateSeq` pattern, but the Composer's own
   lifecycle (does it run once per cycle like EntryGate, or is it invoked
   differently?) is unaddressed here.
2. **Is one EntryGate instance restricted to one TriggerStr source?**
   Section 1 assumes yes ("one expected TriggerStr concept") but this
   hasn't been stress-tested against a policy that might legitimately want
   to accept setups from more than one related trigger.
3. **How are multiple accepted candidates in the same master cycle
   arbitrated** — e.g. two different EntryGate instances both accepting
   entries (possibly opposing sides, possibly the same side) in one cycle?
   Not a "context conflict" in the section 9 sense. **Partially resolved**:
   a portfolio-level arbitration component above independently-configured
   instances is excluded by this project's scope boundary (see "Changes
   from Revision 4"), not an undesigned future extension; the concrete
   case that matters is resolved in
   `Chakra_OrderManagement_Handoff_Interface_Contract.md` section 6.
4. **Does rejected/skipped EntryGate output need durable downstream
   delivery to any other component, or only audit/`DecisionFrame`
   recording** (section 13)? Currently assumed to be recording-only, but
   not confirmed against every plausible consumer.
5. **Which position-state source is authoritative for lifecycle
   applicability** (section 6)? **Resolved in Revision 6** (correcting
   Revision 4's own wrong answer — see "Changes from Revision 5"):
   pre-fill, platform order/position state is read directly; the candle
   baton manager is never a source before a trade is confirmed live and
   handed off to it; a separate Chakra-level position-snapshot abstraction
   remains excluded. What is left open is only the concrete access
   mechanism (which specific ACSIL position/order-query calls a given
   implementation uses), not the architecture choice.

Resolved by this revision, removed from this list: whether EntryGate's own
output could ever legitimately be `PROPOSED_ACTION` (no — architecture
amended to add `POLICY_DECISION`, section 2) and whether `DecisionSeq`
alone was safe as the candidate delivery mechanism (no — `CandidateSeq`
added, section 10).

None of these are silently resolved by giving EntryGate extra
responsibilities beyond its frozen core boundary.
