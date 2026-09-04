# Chakra — Trigger Strategy (TriggerStr) Interface Contract (Draft, Revision 3)

Status: **design only, no ACSIL implementation yet.**

Builds directly on:
- `Algo_Overlay_Framework_Architecture.md`
- `Chakra_Common_Interface_Envelope_Contract.md` (approved, Revision 8)
- `Chakra_Sensor_Interface_Contract.md` (Revision 4)
- `Chakra_MCtx_Interface_Contract.md` (Revision 5)

This document does not redefine anything from those — it specifies how the
TriggerStr layer fills in the envelope's **event pattern** (the machinery
built and repeatedly hardened across the envelope's own revision history
specifically for this kind of component), how it consumes Sensors and
MCtx, and what it must and must not do. Read the three prerequisite
documents first.

**Definition (from `Algo_Overlay_Framework_Architecture.md`).** A Trigger
Strategy answers exactly one question: *"Has my setup occurred?"* It
detects executable opportunities. It is deliberately unaware of risk,
sizing, trailing, exits, other strategies, and — critically — of whether
the setup it just detected is actually *worth taking*. That judgment
belongs entirely to Trade Policy (`EntryGate` and its siblings), which
hasn't had its own contract pass yet but is referenced throughout this
document as TriggerStr's primary consumer, so its expectations aren't
rediscovered later the way MCtx's transition-consumption rules had to be
retrofitted onto this contract's own earlier drafts.

Naming: `K_Chakra_TriggerStr_<DescriptiveName>_V<n>`.

**TriggerStr is event-style, not state-style — the first Chakra layer that
actually is.** Everything the envelope contract built and repeatedly fixed
for the event pattern (`EventSeq`, sticky payload, `EventType == NONE`,
latest-event-wins with gap logging, consume-and-drop, lifecycle
baselining) exists specifically for this layer. Getting it right here is
the payoff for all six envelope revisions; getting it wrong here after all
that work would be a much worse failure than a fresh mistake.

------------------------------------------------------------------------

# Changes from Revision 1

Three material event-layer issues from a review pass:

1. **The consumer gate could hide or replay a sticky event, contradicting
   this document's own permanence rule (blocking).** Revision 1's gate
   required `Status == VALID` at consumption time. Worked example: event 12
   fires (`Status = VALID`); `EntryGate` doesn't evaluate that call;
   TriggerStr then enters cooldown or hits a transient dependency hiccup and
   publishes `NOT_READY`/`ERROR`; `EntryGate` now evaluates and *cannot*
   consume event 12, because current `Status != VALID` — even though
   `EventType`/`EventSeq` still correctly hold it. If `Status` later returns
   to `VALID` with the record untouched, `EntryGate` consumes it *late*; if
   whatever caused the transition also cleared `EventType` (as recalc-entry
   legitimately does), the event is lost *entirely* — directly contradicting
   section 14's "a fired event remains historically true" rule. Fixed by
   separating two concepts that Revision 1 conflated: current producer
   *health* (`Status`) versus validity of the *last fired event record*
   (`EventType`/`EventSeq`/payload). The mandatory gate no longer includes
   `Status == VALID` — `EventType != NONE` already correctly reflects
   "is there an undeleted fired record," since only two documented
   boundaries (disable-entry, recalc-entry) are allowed to clear it, and
   nothing else. See sections 2, 6, 11, 14.
2. **Bidirectional triggers need independent per-side edge and re-arm
   state (blocking).** A single composite edge tracker cannot correctly
   handle a trigger where long and short conditions can each independently
   go active: if long is already active when short *independently* becomes
   active, a composite "was anything active" tracker sees no edge, and the
   short occurrence is silently missed. Section 13's same-call precedence
   rule doesn't cover this — it's a cross-call problem. Fixed by requiring
   independent per-side edge/re-arm state by default, with a mutually-
   exclusive single-tracker design allowed only as an explicit, proven,
   documented exception. See section 6.
3. **Sticky delivery and latest-event-wins never answered whether a fired
   event is still actionable by the time a consumer sees it.** A delayed
   `EntryGate` could receive a breakout event several bars after it
   occurred — historically real, per section 14, but no longer a genuine
   entry opportunity. Fixed by requiring every concrete trigger to declare
   an actionable-age/expiry model, checked by the consumer against the
   event's *availability* time — this does not retroactively un-fire the
   event, it separates "this occurred" from "this is still actionable now."
   See new section 15 (sections 16 onward renumbered accordingly).

------------------------------------------------------------------------

# Changes from Revision 2

Two material interface problems from a further review:

1. **Actionable expiry could not remain an unpublished, trigger-specific
   code constant (blocking).** Revision 2's section 15 allowed a fixed
   expiry tolerance to live purely as a documented code constant, with the
   consumer expected to separately "know" it. That couples `EntryGate` to
   each trigger's private implementation — a genuinely *generic* `EntryGate`
   cannot discover whether a given trigger expires after 30 seconds, three
   bars, or session close without duplicating trigger-specific knowledge
   inside itself, which defeats the point of a generic event interface.
   Fixed: **`EventValidUntilDateTime` is now a required field, published
   and made sticky with every fired event**, resolved to a concrete
   timestamp at fire time regardless of expiry model (including bar-based,
   via a documented conservative-estimate approach) so a consumer needs
   exactly one generic rule: `consumer evaluation time <=
   EventValidUntilDateTime`. This also removes the cross-chart ambiguity a
   bar-*count*-based boundary would have had, since a producer's bar index
   has no defined meaning on a consumer's own chart. See sections 2 and 15.
2. **The `ORBreakout` worked example leaked `EntryGate` behavior into
   TriggerStr — the exact mistake this contract's own litmus test exists to
   catch, missed in this contract's own example.** The example wired an
   *optional* `WeeklyExpansion` MCtx that "narrows which breakouts count as
   valid setups... more permissive if unwired" — meaning the same trigger
   study would answer a *different* question depending on optional wiring,
   which is an entry filter hidden inside TriggerStr, not a fixed setup
   definition. Fixed: added a general rule (section 5) — any dependency
   that changes the trigger's truth condition is required and part of that
   named setup's definition; an optional dependency may enrich payload/
   provenance but must never suppress or create an event. The worked
   example (section 19) is corrected accordingly.

------------------------------------------------------------------------

# 1. Identity and naming

- **`ComponentKind`** is always `K_CHAKRA_KIND_TRIGGERSTR`.
- **`ProducerTypeID`** is a TriggerStr-layer-local enum, one member per
  distinct setup concept, no central registry, explicit `NONE = 0` — same
  pattern as Sensor/MCtx.
  ```cpp
  enum K_CHAKRA_TRIGGERSTR_TYPE
  {
      K_CHAKRA_TRIGGERSTR_TYPE_NONE              = 0,
      K_CHAKRA_TRIGGERSTR_TYPE_OR_BREAKOUT       = 1,
      K_CHAKRA_TRIGGERSTR_TYPE_BULLISH_ENGULFING = 2,
      K_CHAKRA_TRIGGERSTR_TYPE_TREND_TRANSITION  = 3,
      // one member per concrete trigger, assigned as each is built
  };
  ```
- **`PayloadSchemaVersion`** (int32 PV 20) scoped per `ProducerTypeID`, same
  convention as Sensor/MCtx.
- **One study instance detects one clearly defined setup concept — no
  bundling.** `OR Breakout`, `Bullish Engulfing`, and `Trend Transition` are
  three separate studies, matching the base architecture spec's own
  Trigger Strategy examples list (OR Breakout, Bullish Engulfing, Bearish
  Engulfing, HVN Breakout, DOT Acceptance, Compression Breakout — each its
  own concept). TriggerStr gets **no** Sensor-style "coherent family"
  exception — a setup either is this specific pattern or it isn't; there's
  no equivalent of "different unit expressions of one measurement" here.
- Every TriggerStr defines its own `EventType` enum with an explicit
  `NONE = 0` member — see section 2.

------------------------------------------------------------------------

# 2. Event payload shape: durable `EventSeq`, sticky payload, `EventType == NONE`

This is a direct application of the envelope contract's event pattern —
not a new mechanism, a filled-in one:

- **`EventSeq`** (int64, payload-owned, conventionally int64 PV 10 per the
  envelope's established convention) increments **only** when a genuine
  new setup occurrence is detected — never on a status transition,
  disable/re-enable, or recalc-entry (envelope Revision 5's fix, inherited
  unmodified). **`PublicationSeq` is not `EventSeq` and is never used as
  the dedupe key** — this is precisely the bug the envelope contract had at
  Revision 4 and fixed at Revision 5; restating it here because TriggerStr
  is the layer where getting this wrong would actually matter in practice,
  not just in the abstract.
- **`EventType`** (int32, conventionally PV 21) — this trigger's own enum,
  explicit `NONE = 0` required, mirroring `K_CLOSE_EVENT_NONE` in the
  close-protect contract. A `VALID`-but-never-fired TriggerStr must never
  be mistaken for one that has fired: the event gate always requires both
  `EventType != NONE` **and** `EventSeq != LastConsumedEventSeq` together,
  never either alone (envelope rule, load-bearing here specifically because
  recalc-entry clears `EventType` to `NONE` while leaving `EventSeq`
  untouched — weakening the gate to an `EventSeq`-only check would break
  that).
- **`Applicability`** (int32, conventionally PV 22) — see section 3.
- **The complete event payload is rewritten together with `EventSeq`, and
  stays sticky between increments** — same discipline as every other event
  payload in this framework. Commit order:
  `envelope + event payload (including EventType/Applicability/setup
  details) -> EventSeq -> PublicationSeq`, per the envelope's event
  publication invariants.
- **Reserved payload slots, by convention** (fixed across every TriggerStr,
  so generic tooling can rely on them):

  | Field | Namespace/PV | Purpose |
  |---|---|---|
  | `PayloadSchemaVersion` | Int32 PV 20 | Envelope convention |
  | `EventType` | Int32 PV 21 | This section |
  | `Applicability` | Int32 PV 22 | Section 3 |
  | `PertainsToBarIndex` | Int32 PV 23 | Section 4, retrospective triggers only |
  | `ExpiryMode` | Int32 PV 24 | Section 15 |
  | `EventSeq` | Int64 PV 10 | This section |
  | `PertainsToDateTime` | Datetime PV 10 | Section 4, retrospective triggers only |
  | `EventValidUntilDateTime` | Datetime PV 11, **required** | Section 15 |

  Everything else (which specific setup details a trigger needs to
  publish) uses whatever remaining slots in the trigger's own payload
  ranges make sense, documented per trigger.

**Exactly two boundaries clear the sticky event record — nothing else may,
ever (corrected from Revision 1; see "Changes from Revision 1," item 1):**

- **Disable-entry** (`Status -> DISABLED`).
- **Recalc-entry** (`IsFullRecalculation` first call, section 8).

Both are explicit, deliberate, user- or lifecycle-driven transitions with
well-understood real-world meaning ("this study is off" / "this is a fresh
recalculation"), so clearing the record there is intentional and
documented, not an accident. **Every other `Status` change — entering
cooldown (`NOT_READY`/diagnostic), a transient dependency hiccup (`ERROR`),
or anything else — must leave `EventType`/`EventSeq`/the event payload
completely untouched.** This is what makes `EventType != NONE` alone a
correct and sufficient signal for "is there a currently-deliverable fired
record," without needing to also check current `Status` — see section 11's
corrected consumer gate.

------------------------------------------------------------------------

# 3. Trigger direction / applicability

```cpp
enum K_CHAKRA_TRIGGERSTR_APPLICABILITY
{
    K_CHAKRA_TRIGGERSTR_APPLICABILITY_NONE  = 0,
    K_CHAKRA_TRIGGERSTR_APPLICABILITY_LONG  = 1,
    K_CHAKRA_TRIGGERSTR_APPLICABILITY_SHORT = 2,
    K_CHAKRA_TRIGGERSTR_APPLICABILITY_BOTH  = 3   // LONG | SHORT
};
```

Every fired event carries an `Applicability` — which side this specific
occurrence applies to. This is the same concept the close-protect
contract's `SideMask` already captures (`K_CLOSE_PROTECT_SIDE_MASK`),
scoped to this layer per the envelope's "Direction and Applicability"
deferred design point (Revision 3): TriggerStr owns its own authoritative
applicability field rather than sharing a common-envelope one.

A trigger may be inherently single-sided (e.g. an input like
`AdverseSideMode` restricting it to Long-only or Short-only by design,
mirroring `K_CloseProtect_DeltaSMA_BodyPct_Decision`'s existing pattern),
or inherently bidirectional (e.g. an `OR Breakout` that can fire either
direction depending on which side breaks first). Either is fine — document
which, per trigger.

------------------------------------------------------------------------

# 4. Setup bar/time, availability bar/time, and retrospective `PertainsTo*`

For a **non-retrospective** trigger (the common case — the setup condition
and its detection happen on the same bar), "setup bar" and "availability
bar" are the same thing, and the envelope's `EvalBarIndex`/`EvalDateTime`
already capture both: they mean **when this event became
available/knowable**, full stop, no special case.

For a **retrospective** trigger (e.g. one built on a Renko ring/dot
confirmation, where the pattern's actual pivot is an earlier bar than the
bar where the second confirming brick closes), the same split the Sensor
contract established (Revision 4) applies here, for the identical reason —
preventing lookahead bias:

- `EvalBarIndex`/`EvalDateTime` (envelope) — **availability**: the bar
  where the setup was *confirmed and became actionable*. `EventSeq`
  increments here, not earlier.
- `PertainsToBarIndex`/`PertainsToDateTime` (payload) — **semantic
  anchor**: the earlier bar the underlying pattern is actually about (e.g.
  the ring's pivot extreme). Descriptive only, never used to gate
  `EventSeq`.

**A retrospective trigger becomes consumable only at availability time —
never earlier, live or historical.** This is not just a historical-replay
safety rule (section 8 covers that separately); it applies to live
evaluation too: a two-brick-confirmation setup must not increment
`EventSeq` on the probe brick, only on the confirming one, exactly matching
this project's established Renko ring/dot two-brick rule
(`reference_acsil_skill.md`).

------------------------------------------------------------------------

# 5. Sensor and MCtx dependency contract

TriggerStr may depend on Sensors, MCtx contexts, or both — a setup
condition can be defined directly on raw measurements (e.g. a Delta
threshold breach) and/or on a classified context (e.g. "only a valid OR
Breakout if `Trend` context isn't already `RANGE`-exhausted" — which, per
the rule below, makes `Trend` a **required** dependency of that specific
trigger, not an optional refinement). All dependencies follow the identical
discipline MCtx already established for its own Sensor dependencies (MCtx
contract section 4), reused here without modification:

- Whether each dependency is **required or optional**, with the same
  required-unwired-is-not-silent rule (`NOT_READY`, neutral payload,
  deterministic `ReasonCode`, logged on transition — never silent for a
  required dependency).
- **Any dependency that changes the trigger's truth condition is required
  and part of that named setup's own definition — never optional.**
  (New in Revision 3; see "Changes from Revision 2," item 2.) An *optional*
  dependency may enrich the fired event's payload/provenance (e.g.
  publishing "what `WeeklyExpansion` said at fire time" as a descriptive,
  non-gating field) but **must never suppress or create an event** — if
  wiring or unwiring a dependency would change whether the same underlying
  price action counts as a setup, that dependency isn't optional, it's part
  of the setup's own condition (section 6), and the trigger needs either a
  required dependency or a distinct, separately-named trigger concept
  (section 1) rather than an optional toggle. A single study secretly
  answering two different setup questions depending on how it's wired
  violates the same "one study, one setup concept" rule as bundling two
  unrelated triggers together.
- Exact expected `ProducerTypeID`, envelope `SchemaVersion`, and
  `PayloadSchemaVersion`, validated in the envelope's full identity order
  before any payload is read.
- Which specific fields are read — never "everything the dependency
  happens to publish."
- The freshness tolerance TriggerStr itself chooses for each dependency
  (consumer decides staleness, unchanged principle).
- **Both Sensors and MCtx are consumed via the state pattern exclusively**
  — read the current valid snapshot repeatedly, no event-style
  deduplication, since both are state-style producers. This is true
  regardless of what TriggerStr itself is (event-style) — a producer's own
  style has no bearing on how it consumes *its* dependencies.
- **Live-versus-historical access, including the retrospective-availability
  rule**: TriggerStr's own historical recalculation (rebuilding its private
  edge-detection state — never a live `EventSeq`, section 8) reads a
  Sensor or MCtx dependency's **historical subgraph arrays**, never their
  persistent payload — and if that dependency is itself retrospective, TriggerStr
  reads its **availability-indexed** machine subgraph, never its
  pertains-to-indexed visual one, for the identical lookahead-bias reason
  established in the Sensor and MCtx contracts. Getting this wrong here
  would be the third occurrence of the same mistake across this framework.
- **Historical validity, not just array coverage**: the same three-part
  check MCtx now requires (`mapped index exists AND historical Status ==
  VALID AND value finite/in-range`) applies identically here.

------------------------------------------------------------------------

# 6. Exact trigger conditions, edge gating, confirmation, and re-arm behavior

Every trigger must document, in writing, with the same rigor MCtx requires
of its classification rules:

- **Exact inputs and units** — which Sensor/MCtx field(s), or which raw
  price/volume data, and in what units.
- **Exact condition(s)** — the precise boolean expression that constitutes
  "setup occurred," including inclusive/exclusive boundary behavior.
- **Thresholds, confirmation windows, and re-arm cooldowns are code
  constants** — not ACSIL `Input`s, per the base architecture spec's
  versioning rule and MCtx's own corrected boundary (MCtx Revision 2):
  wiring/display/meaning-preserving-cadence may be inputs, the setup
  *definition itself* may not. A threshold change is a code change and a
  commit.
- **Edge gating, not level gating.** A trigger fires only on the
  transition from condition-inactive to condition-active — never on every
  call where the condition happens to still be true. This is the existing
  established pattern in this codebase
  (`K_Template_CloseProtect_DecisionStudy`'s `lastTriggerBarIndex`/
  `lastTriggerWasActive` private edge state) and TriggerStr studies follow
  it identically, with the private edge state living in the study's own
  50-89 PV range.
- **Confirmation, if the setup requires it** — e.g. a minimum-duration or
  N-brick requirement before a probe is accepted as a real setup. Document
  the exact rule; a probe that fails to confirm is noise and must never
  increment `EventSeq` (same principle as the Renko ring/dot two-brick
  rule).
- **Re-arm behavior** — after firing, when may the same trigger fire
  again, for the same or the opposite side? Edge gating alone guarantees
  "not while still active," but a trigger may additionally require the
  condition to have gone fully inactive for some duration, or an explicit
  cooldown, before re-arming. This must be an explicit, documented,
  code-constant policy per trigger, never implicit.
- **Being in a cooldown or awaiting-confirmation state leaves `Status =
  VALID`.** (Corrected from Revision 1.) `NOT_READY` is reserved for
  genuine warmup (insufficient historical data to evaluate at all) — a
  trigger that has fully warmed up but is simply waiting for its own
  re-arm cooldown to elapse, or waiting for a probe to confirm, is
  operationally healthy and reports `Status = VALID` with
  `ReasonCode = AWAITING_REARM`/`AWAITING_CONFIRMATION` (section 18) for
  diagnostic detail. This matters because it's what keeps a still-sticky,
  not-yet-consumed fired event deliverable while the trigger is quietly
  between occurrences — see section 2's two-clear-boundaries rule.

## Bidirectional triggers need independent per-side edge and re-arm state (new in Revision 2)

**A single composite "was the condition active" tracker is insufficient
for any trigger where long and short conditions can each independently
become active.** Worked example of the bug this prevents: long condition
becomes active (fires `LONG`); long remains active; short condition
*independently* becomes active while long is still active. A composite
tracker was already `true` (because long was active), so it sees no edge
for the combined signal — the short occurrence is silently missed, even
though it just newly turned on. This is a cross-*call* problem, distinct
from section 13's same-*call* simultaneous-detection precedence rule,
which does not solve it.

Every bidirectional trigger must choose one of:

- **Independent per-side edge and re-arm state** (the default expectation):
  separate `lastLongWasActive`/`lastLongTriggerBarIndex` and
  `lastShortWasActive`/`lastShortTriggerBarIndex` private trackers (and
  separate re-arm/cooldown state per side, if re-arm cooldowns are used at
  all), so long and short can each independently edge-detect, fire, and
  re-arm without interfering with each other. **A long fire must not
  accidentally disarm or re-arm the short side, or vice versa, unless that
  coupling is an explicit, documented setup rule** — e.g. a trigger whose
  own semantics genuinely mean "only one side can be looking for a setup
  at a time" may deliberately couple them, but this must be stated, not
  incidental to how the tracker happened to be implemented.
- **A documented, proven mutually-exclusive state machine**, only where a
  trigger's own condition logic makes long and short *literally* unable to
  be active simultaneously by construction (e.g. a single signed indicator
  that is never both positive and negative at once) — in which case one
  composite tracker is provably sufficient and the proof belongs in that
  trigger's own documentation, not assumed by default.

------------------------------------------------------------------------

# 7. Lifecycle baselining — every private tracker, no exceptions

TriggerStr sits at the intersection of every lifecycle-baselining pattern
this framework has established so far, and needs all of them simultaneously:

- **As a producer of its own edge-gating state** (section 6): baseline at
  its own `IsFullRecalculation` entry, per the envelope's recalc-entry
  rules — unchanged, standard.
- **As a consumer of its Sensor/MCtx dependencies** (section 5): the
  state-pattern dependency-rewiring/attachment rules already established
  for MCtx apply identically — baseline on first attachment, not "inert by
  default," per the envelope's own Revision 6/7 fix.
- **If a trigger detects an MCtx *transition*** (e.g. `RANGE -> TREND` as
  part of its setup condition): it must follow MCtx's own
  lifecycle-baselining rule for private transition detection (MCtx
  Revision 4, section 7) **exactly** — baseline private previous-state to
  the MCtx's current valid state on attachment/rewiring/own-recalc/own-
  disable-re-enable, never fire on the baselining evaluation itself,
  invalidate-and-silently-rebaseline across any period where the MCtx
  itself is invalid. This is the case MCtx's contract explicitly flagged
  as "especially important once TriggerStr exists" — this is that moment;
  the rule is inherited, not reinvented.
- **As a producer whose own events get consumed downstream** (by
  `EntryGate` or similar): the consumer's obligations are the mirror image
  of the above, and are TriggerStr's responsibility to document clearly
  enough that a downstream consumer can implement them correctly — see
  section 11.
- **Never treat first attachment, or recovery from an invalid `Status`
  period, as a new edge, at any of the layers above.** This single
  instruction covers every one of these cases: an edge/transition/event is
  only real if it follows a proper baseline, never merely because
  something started reading valid data for the first time.

------------------------------------------------------------------------

# 8. No historical `EventSeq` increments during full recalculation

This is stricter than the Sensor/MCtx "at most two publications per sweep"
rule, and deliberately so: a Sensor or MCtx's *current state* is legitimately
meaningful to publish once at the end of a recalc sweep, but TriggerStr has
no equivalent concept — a historical setup is not a "current opportunity,"
it is a past occurrence with nothing to act on now.

- **`EventSeq` never increments during `IsFullRecalculation`, under any
  circumstance, including at the final bar.** Not once, not even for "the
  most recent" historical setup. This is the envelope's "Event-style
  producers must not manufacture live events during recalculation" rule
  (Revision 6), stated here as TriggerStr's primary law rather than an
  inherited footnote, because this is the layer it was written for.
- The recalc-entry `Status -> NOT_READY` publication still happens
  normally (that's an envelope-level, `PublicationSeq`-driven mechanic,
  unaffected by anything above).
- **Private edge-detection state baselines silently against history** —
  exactly the existing `K_Template_CloseProtect_DecisionStudy` recalc
  branch pattern (baseline `lastTriggerBarIndex`/`lastTriggerWasActive`
  against the last bar, return, never touch `EventSeq`).
- **Historical subgraphs may still mark where a setup would have occurred**
  (section 16) — this is a completely separate, parallel mechanism from
  the persistent `EventSeq`, used for charting/backtest visualization and
  historical machine consumption by other studies, never for live event
  delivery.

------------------------------------------------------------------------

# 9. Replay-safe event generation through the normal forward path

Sierra Chart's replay mode primes historical state up to the replay's
start point (hitting the recalc path above, silently) and then advances
forward bar-by-bar or tick-by-tick using the **normal** (non-recalc)
evaluation path — the same path live trading uses. A genuinely new setup
detected as replay advances forward correctly increments `EventSeq` exactly
as it would live, because it goes through the normal edge-gated evaluation
path, not the recalc path. No special-casing is needed for replay beyond
following sections 6-8 correctly — this is the envelope's own Revision 6
clarification, restated here because TriggerStr is where replay validity
actually gets tested (backtesting a trigger's historical performance is
close to this layer's entire reason for existing).

------------------------------------------------------------------------

# 10. Latest-event-wins semantics and sequence-gap logging

**Explicit, deliberate policy, not an accident:** if a trigger fires
several occurrences between two consumer evaluations, the sticky
single-slot design (one current `EventSeq` + one current event payload)
means only the *latest* occurrence is visible — the consumer detects that
something was missed (via a sequence jump greater than 1) but cannot
recover the earlier occurrence's payload. This is the envelope's default
and only currently-defined delivery semantic, inherited unmodified.

- Gap detection uses the corrected pattern from the envelope contract
  (Revision 7): capture `previousSeq` **before** updating
  `LastConsumedEventSeq`, then branch — forward gap (`> 1`) logs dropped
  occurrences; backward movement logs distinctly as a possible
  full-storage-reset and re-baselines, never a raw arithmetic difference.
- **If a specific trigger genuinely needs every occurrence delivered, not
  just the latest** (e.g. a scalping-style trigger where missing an
  intermediate occurrence is materially different from missing the
  latest), that trigger must design its own guaranteed-delivery mechanism
  explicitly, in its own documentation — the common envelope does not
  provide one, and TriggerStr does not get a different default from every
  other event-style layer just because "missing a trade" feels more
  consequential than missing an intermediate Sensor reading. State the
  choice explicitly per trigger; do not leave it as an unexamined
  accident either way.

------------------------------------------------------------------------

# 11. Consume-and-drop expectations for downstream consumers

TriggerStr's primary consumer is `EntryGate` (or a similar Trade Policy
sub-component), which hasn't had its own contract pass yet — these
expectations are documented here so they aren't rediscovered later, the
same courtesy MCtx extended to this contract:

- **A consumer gates on `identityValid && EventType != NONE && EventSeq !=
  LastConsumedEventSeq` — `Status` is deliberately not part of this
  check** (corrected from Revision 1; see "Changes from Revision 1," item
  1, and section 2's two-clear-boundaries rule). `EventType != NONE`
  already correctly means "there is an undeleted fired record," because
  only disable-entry and recalc-entry are ever allowed to clear it. A
  consumer *may* still read `Status` separately for its own diagnostic
  purposes (e.g. logging that the producer is currently in cooldown), but
  must never require `Status == VALID` to consume an otherwise-valid
  sticky event — doing so is what caused Revision 1's late-consumption/
  lost-event bug.
- **Default failure policy is consume-and-drop, not retry.** If a consumer
  observes a fired event but cannot usefully act on it this call — e.g.
  some other dependency needed to evaluate the setup further is itself
  stale — the default is to still advance `LastConsumedEventSeq` and log
  the drop, not leave the event pending in hopes of a later retry. This is
  the candle-baton "decide event meaning once, at first eligible
  observation" lesson, inherited via the envelope contract, restated here
  because it directly governs how `EntryGate` must be built once that
  contract exists.
- A consumer must baseline `LastConsumedEventSeq` to TriggerStr's *current*
  `EventSeq` on its own first attachment, rewiring, recalc, and
  disable/re-enable — never to 0, never assumed inert by default (section
  7, restated from the consumer's side).
- **A consumer must check `consumer evaluation time <=
  EventValidUntilDateTime` before acting on it** — section 15. This is the
  *only* actionability check a generic consumer needs; the boundary always
  travels with the event as published data, never as trigger-specific
  knowledge the consumer must separately possess. Consuming (advancing
  `LastConsumedEventSeq`) and acting on an event are not the same decision;
  an expired event is still consumed-and-dropped per the failure policy
  above, just with a distinct stale-event reason.

------------------------------------------------------------------------

# 12. Intrabar versus closed-bar triggers and provisional-event policy

Every trigger declares exactly one cadence, matching the Sensor contract's
cadence-declaration discipline:

- **Closed-bar (strong default recommendation).** The trigger only
  evaluates/fires once a bar closes. Safest choice: a closed-bar
  measurement cannot reverse after the fact, so a fired event can never
  turn out to have been based on data that changed moments later.
- **Intrabar (opt-in, must be explicitly justified).** A trigger firing on
  still-forming bar data is materially riskier than a Sensor publishing a
  provisional intrabar *value* (Sensor contract, open question 2): a
  Sensor's provisional value is simply overwritten by the next update, but
  a TriggerStr event **may already have been consumed and acted on** by
  `EntryGate` before the bar closes and the condition reverses. **There is
  no "un-firing" a `TriggerStr` event** — see section 14, trigger
  invalidation. Any trigger choosing intrabar cadence must document why
  the latency benefit is worth this risk, and what (if anything) mitigates
  it for that specific setup.

This directly extends, rather than resolves, the Sensor contract's still-open
question 2 about provisional intrabar values — for TriggerStr specifically,
the contract takes a position (closed-bar is the default, intrabar is an
explicit, justified exception) rather than leaving it symmetric the way
the Sensor contract did, because the consequence of getting it wrong is
categorically worse at this layer.

------------------------------------------------------------------------

# 13. Side changes and simultaneous long/short occurrences

**This section covers detecting both sides within one evaluation call.**
For a bidirectional trigger, detecting each side correctly *across* calls
— so that one side going active doesn't mask the other side's independent
edge — is section 6's per-side edge/re-arm state requirement, which this
section does not substitute for.

- **A trigger's individual occurrences may vary in `Applicability`** even
  if the trigger itself isn't restricted to one side — an `OR Breakout`
  trigger firing `LONG` on one occurrence and `SHORT` on a later, unrelated
  occurrence is normal and expected; each event's own `Applicability`
  field says which side that specific occurrence was for.
- **`EventSeq` increments at most once per evaluation call.** If a
  trigger's condition logic could otherwise detect both a long and a short
  setup within the same call (a genuinely unusual case — e.g. a symmetric
  volatility-expansion pattern breaking both directions at once), the
  trigger must have an explicit, documented precedence/tie-break rule
  deciding which one this call's single event represents, **or** publish a
  single event with `Applicability = BOTH` if the setup's own semantics
  genuinely mean "conditions favor a breakout in either direction" rather
  than two independent occurrences. This mirrors MCtx's "precedence if
  several state conditions match" rule (MCtx section 6), applied to trigger
  conditions instead of classification states. Never publish two `EventSeq`
  increments in one call to represent two simultaneous sides — that would
  violate "at most once per evaluation call" and complicate every
  consumer's gate for no documented benefit.

------------------------------------------------------------------------

# 14. Trigger invalidation: a fired event is a permanent historical fact

**This section distinguishes two things that Revision 1 conflated:**
*historical occurrence* (the fact that a setup happened) is permanent,
full stop — nothing ever erases it. The *latest sticky delivery record*
(the live `EventSeq`/`EventType`/payload a consumer reads right now) is a
**delivery mechanism**, not the historical record itself, and delivery
mechanisms have documented lifecycle rules (section 2's exactly-two-clear-
boundaries rule) governing when they get superseded or cleared. Confusing
the two is what produced Revision 1's contradiction (this section said
fired payload "does not change," while the recalc-entry rule elsewhere
cleared `EventType` — both were true, they just described different
things):

- **Historical occurrence, captured permanently in the historical
  Occurrence/Status subgraphs (section 16):** never cleared, never
  overwritten, by anything — this is what "remains historically true"
  actually refers to.
- **The latest sticky delivery record:** persists unchanged across
  ordinary operation (including cooldown/transient-`ERROR` `Status`
  changes, section 2), gets superseded the instant a *newer* event fires,
  and gets explicitly cleared only at disable-entry or recalc-entry. A
  consumer that hasn't yet consumed a given `EventSeq` before it's
  superseded by a newer one simply never sees the older one — that's the
  latest-event-wins policy (section 10), not an invalidation.

**Once `EventSeq` increments and is published, that occurrence remains
historically true — it is never retroactively invalidated by a
dependency changing its mind later, live or in a subsequent recalculation
under the same code version.** This follows directly from the "decide
event meaning once, at first eligible observation" principle already
governing consumer-side behavior (section 11), applied here to the
producer side:

- If a Sensor or MCtx dependency that contributed to a fired event later
  changes state (even moments later, same session, no code change), the
  **already-fired event's payload does not change** — it was captured
  atomically at the moment of firing (section 2's commit-order rule) and
  reflects what was true then, not what's true now. TriggerStr does not
  attempt to "walk back" a past fire.
- A **subsequent recalculation under a *different* code version** (a
  changed threshold, a bugfix) will naturally produce different historical
  results — that's expected and correct per the base architecture's
  versioning philosophy (git hash is the version), not a retroactive
  invalidation of the *previous* run's facts. Replay under the *same* code
  version must reproduce the *same* events (section 9).
- This is what makes intrabar firing (section 12) genuinely risky rather
  than merely inconvenient: an intrabar fire that gets consumed by
  `EntryGate` before the underlying condition reverses cannot be undone by
  TriggerStr publishing anything afterward — the framework has no
  "cancel event" concept, deliberately, because introducing one would
  reopen exactly the kind of event-mutation risk the candle-baton lessons
  already warned against.

------------------------------------------------------------------------

# 15. Actionable age and event expiry

Sticky delivery (section 2) and latest-event-wins (section 10) answer
"will a consumer eventually see this event" — neither answers "is it still
worth acting on by the time they do." A delayed `EntryGate` receiving a
breakout event several bars after it actually occurred has a historically
real event (section 14) that is no longer a genuine entry opportunity —
price has likely moved past the level the breakout was about. This is a
**separate question from historical validity**, and this contract does not
retroactively "un-fire" an expired event to answer it — it just gives a
consumer the means to recognize staleness and decline to act.

**Expiry is part of the event delivery contract and travels with the
event — it is never a private, unpublished trigger-specific constant.**
(Corrected from Revision 2; see "Changes from Revision 2," item 1.) A
generic `EntryGate` that can consume any TriggerStr without duplicating
trigger-specific knowledge inside itself needs the boundary published as
data, not documented only in that trigger's own source.

- **`EventValidUntilDateTime` (Datetime PV 11) is a required field on every
  fired event, sticky with that event's record** — set once, atomically,
  at the moment `EventSeq` increments (same commit-order discipline as the
  rest of the event payload, section 2), and never touched again until the
  next event supersedes it. A generic consumer's entire actionability check
  is exactly one comparison: **`consumer evaluation time <=
  EventValidUntilDateTime`** — using the same live/historical "now" rules
  the envelope's "Age and 'now'" section already establishes
  (`sc.GetCurrentDateTime()` live/paced-replay, `sc.BaseDateTimeIn[EvalIndex]`
  historical). No trigger-specific knowledge required on the consumer side.
- **`EventValidUntilDateTime` is always resolved to a concrete timestamp at
  fire time, whatever the underlying expiry model** — each TriggerStr
  documents which model it used (`ExpiryMode`, Int32 PV 24, required —
  diagnostic/traceability metadata, not itself part of the consumer's
  comparison):
  - **Time-based**: `EventValidUntilDateTime = EvalDateTime + fixed
    duration` — exact.
  - **Bar-based**: estimated using the chart's known/typical bar interval —
    inherently approximate across session gaps or weekends, so a trigger
    using this model must round generously (choose a safely later boundary,
    never a precise-but-fragile one) rather than assume calendar time maps
    cleanly onto a bar count. **Never publish a raw bar-count boundary as
    the primary field** (e.g. no `EventValidUntilBarIndex`) — a bar index
    has no defined meaning on a consumer's own chart (the envelope's
    long-standing cross-chart warning), so a bar-based model must still
    resolve down to a concrete, chart-agnostic timestamp.
  - **Same-evaluation-only**: `EventValidUntilDateTime` set at or an
    infinitesimal instant past `EvalDateTime` itself — any evaluation after
    the firing call is already past it, correctly modeling the strictest
    case as an ordinary comparison rather than a special one.
  - **Session-bound**: `EventValidUntilDateTime` = that session's close.
- **Downstream behavior on expiry: consume-and-drop, with a distinct
  stale-event reason.** An expired event is still consumed (advance
  `LastConsumedEventSeq`, per section 11's failure policy) — it is not left
  pending, and it is not treated as never-having-happened (section 14). The
  consumer simply declines to act and logs why, using its own diagnostics
  (section 18 — `ReasonCode` is producer-self-referential, never a
  consumer's judgment written into TriggerStr's own storage).

This does not conflict with section 14: "this setup genuinely occurred" and
"this occurrence remains actionable now" are different questions, and a
`NO`-answer to the second never changes the answer to the first.

------------------------------------------------------------------------

# 16. Historical event subgraphs — separate from the latest sticky payload

Mirrors the Sensor contract's subgraph/payload split (section 9 there),
extended with TriggerStr's own occurrence-and-validity shape:

- **A hidden historical "Occurrence" subgraph** — one value per bar
  (`EventType`, or a simpler 0/1 flag if a trigger has only one event
  type), marking which historical bars had a confirmed setup, indexed at
  **availability** (matching section 4 — never at `PertainsTo` for a
  retrospective trigger, for the same lookahead-bias reason established
  throughout this document).
- **A hidden historical "Status" subgraph**, `K_CHAKRA_STATUS`-valued, one
  value per bar, written in the same pass as the Occurrence subgraph —
  exactly the Sensor contract's Revision 3 historical-validity mechanism,
  extended here. A downstream historical consumer (another study reading
  this trigger's history) requires the same three-part check: array
  coverage, historical `Status == VALID`, and the value being meaningful —
  never array coverage alone.
- **An optional visual subgraph at `PertainsTo`**, for retrospective
  triggers wanting the marker to appear at the pattern's actual pivot on
  the chart — purely a display concern, entirely separate from the
  machine-readable Occurrence/Status pair above.
- **None of this drives or is driven by the persistent sticky
  `EventSeq`/payload**, which reflects only the single latest fired event
  (section 10) — the historical subgraphs and the live event interface are
  two genuinely separate representations of related but distinct
  information, exactly as the Sensor contract already established for
  measurements.

------------------------------------------------------------------------

# 17. Explicit prohibitions

A TriggerStr must **not**:

- Approve, reject, or otherwise gate whether a detected setup should be
  *taken* — that is `EntryGate`'s job entirely. A trigger answering "has
  my setup occurred" has already finished its job the moment it answers;
  it does not additionally ask "and is this a good idea."
- Size a position.
- Choose, place, or manage a stop.
- Manage a trailing stop or a target/profit-taking decision.
- Reconcile or arbitrate between multiple MCtx contexts it depends on — if
  it uses MCtx as part of its condition, it uses whatever that MCtx
  currently says; resolving disagreements between multiple contexts (if a
  trigger even depends on more than one) is a Trade Policy problem, per
  the base architecture spec's already-settled conflict-handling principle
  (MCtx section 10), not something a trigger's own condition logic
  attempts to sort out.
- Know about positions, trades, account state, or risk in any way.

**Litmus test:** *Could two differently-configured `EntryGate` instances
legitimately respond differently to the same fired event, without
TriggerStr changing at all?* If yes, the event is a valid, purely
descriptive "setup occurred" fact. If the payload already encodes an
accept/reject/size/risk judgment, policy has leaked into TriggerStr and
needs to move to `EntryGate` instead. A trigger **may use context to
define what the setup *is*** (e.g. "only count an OR breakout as a setup
if `Trend` context isn't `RANGE`-exhausted" is a legitimate part of the
setup's own definition) — it must never use context to decide **whether
taking that setup is desirable**, which is a different question belonging
one layer up.

------------------------------------------------------------------------

# 18. Diagnostics

```cpp
enum K_CHAKRA_TRIGGERSTR_REASON_CODE
{
    K_CHAKRA_TRIGGERSTR_REASON_NONE                   = 0,   // = K_CHAKRA_REASON_NONE
    K_CHAKRA_TRIGGERSTR_REASON_DEPENDENCY_UNWIRED      = 1,
    K_CHAKRA_TRIGGERSTR_REASON_DEPENDENCY_SCHEMA_MISMATCH = 2,
    K_CHAKRA_TRIGGERSTR_REASON_DEPENDENCY_UNAVAILABLE  = 3,
    K_CHAKRA_TRIGGERSTR_REASON_DEPENDENCY_STALE        = 4,
    K_CHAKRA_TRIGGERSTR_REASON_WARMUP_INCOMPLETE       = 5,
    K_CHAKRA_TRIGGERSTR_REASON_AWAITING_CONFIRMATION   = 6,   // probe seen, not yet confirmed
    K_CHAKRA_TRIGGERSTR_REASON_AWAITING_REARM          = 7,   // condition still active / cooling down since last fire
    K_CHAKRA_TRIGGERSTR_REASON_CONFIGURATION_INVALID   = 8,
    K_CHAKRA_TRIGGERSTR_REASON_INVALID_MEASUREMENT     = 9    // NaN/Inf or otherwise non-finite input
    // extend per concrete trigger as needed, numbered above these common ones
};
```

`K_CHAKRA_TRIGGERSTR_REASON_AWAITING_REARM` and
`..._AWAITING_CONFIRMATION` are diagnostic-only, published alongside
`Status = VALID` (section 6, corrected from Revision 1 — neither is a
`NOT_READY` condition), not errors. **Log transitions or first occurrence
of a given `ReasonCode`, not every call**, same convention as every other
layer. A dropped-occurrence gap (section 10) should be logged distinctly
from these, since it's about a downstream consumer's own bookkeeping, not
TriggerStr's own diagnostic state.

**A stale/expired event (section 15) is diagnosed by the *consumer*, not
TriggerStr.** `ReasonCode` is strictly self-referential per the envelope
contract — a producer's own diagnostics, never a judgment about how a
consumer used its output. `EntryGate` logs its own "declined, event
expired" reason using its own diagnostics, not by writing into
TriggerStr's `ReasonCode`.

------------------------------------------------------------------------

# 19. Worked examples

**Normal trigger — `K_Chakra_TriggerStr_ORBreakout_V1`.** Non-retrospective,
closed-bar, bidirectional. Fires on **every** confirmed break of the
opening range, `Applicability = LONG` or `SHORT` depending on which side
broke. `EventType` values: `NONE=0`, `BREAKOUT=1`. Depends only on an
OR-boundary Sensor (required) — the setup condition is exactly "the range
broke," nothing more, so `EntryGate` decides separately (using its own
`WeeklyExpansion` MCtx dependency) whether *this particular* breakout is
worth taking. `EventValidUntilDateTime` set to a short, time-based
tolerance past the breakout bar's close (section 15).

**Corrected from Revision 2** (see "Changes from Revision 2," item 2):
Revision 2's version of this example wired `WeeklyExpansion` as an
*optional* dependency that "narrowed which breakouts count as valid
setups... more permissive if unwired" — meaning the same study answered a
different question depending on how it was wired, an entry filter smuggled
into TriggerStr. The corrected design above fixes it by moving that
judgment entirely to `EntryGate`. If a genuinely *different*, narrower
setup concept is wanted — "only OR breakouts during an under-expanded
week" — that's `K_Chakra_TriggerStr_UnderExpandedORBreakout_V1`, a
**separate, distinctly-named trigger** with `WeeklyExpansion` as a
**required** dependency (section 5), not the same study with an optional
toggle.

**MCtx-transition trigger — `K_Chakra_TriggerStr_TrendTransition_V1`.**
Fires when the `Trend` MCtx transitions `RANGE -> TREND_UP` or `RANGE ->
TREND_DOWN`. Exercises section 7's inherited MCtx-transition
lifecycle-baselining rule directly: on attachment, baselines its private
previous-`Trend`-state to the MCtx's *current* state without firing, and
only reports a transition on a genuinely subsequent state change.
`Applicability` derives from which direction the transition was into.

**Retrospective trigger — `K_Chakra_TriggerStr_RenkoReversalAccepted_V1`.**
Built on a Renko ring/dot study. The underlying pivot is at bar 103; the
second confirming brick closes at bar 105. `EventSeq` increments at bar
105 (availability); `PertainsToBarIndex = 103` in the payload. A replay
consumer evaluating up through bar 104 never sees this event — exactly the
lookahead-bias protection sections 4, 5, and 16 all exist to guarantee.

**Invalid — policy leakage:** a trigger publishing
`TakeThisTrade = true`, or sizing/risk fields, or an `AllowEntry` flag
alongside its setup detection. That's `EntryGate` wearing a TriggerStr's
name — the setup fact (`EventType = BREAKOUT`, `Applicability = LONG`) is
the legitimate payload; any judgment about whether to act on it is not.

------------------------------------------------------------------------

# Unresolved design questions — deliberately left open

Only questions that affect real implementation. Envelope, Sensor, and MCtx
decisions already settled are not reopened here.

1. **Is a guaranteed-delivery (non-latest-wins) mechanism ever actually
   needed for a real trigger**, or does every realistic setup tolerate
   "only the latest occurrence between evaluations matters"? Section 10
   allows a trigger to define one explicitly but doesn't yet have a
   concrete case demonstrating it's required.
2. **Should intrabar/provisional trigger firing be allowed at all**, or
   should the contract flatly require closed-bar-only given the "no
   undo" consequence (section 14)? Section 12 currently allows it as a
   documented exception; a stricter reading would forbid it outright.
3. **How should re-arm cooldowns be declared** — always a bare code
   constant (a fixed bar count), or can a cooldown *duration* legitimately
   be an operational input under MCtx's wiring/display/meaning-preserving-
   cadence carve-out (MCtx section 6), since changing a cooldown doesn't
   change what pattern is being detected, only how often it can re-fire?
   Not resolved here.
4. **Does `EntryGate`'s eventual contract need anything beyond section 11's
   consume-and-drop and actionable-age-check expectations**, or will those
   prove sufficient once Trade Policy's own contract pass happens? Flagged
   for that pass to confirm, not re-derive from scratch.

Resolved by Revision 3, removed from this list: whether
`EventValidUntilDateTime` should be required (yes — section 15) and
whether an optional MCtx dependency may gate whether a trigger fires (no —
section 5).
