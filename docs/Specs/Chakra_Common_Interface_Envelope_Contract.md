# Chakra — Common Interface Envelope Contract (Draft, Revision 9)

Status: **design only, no ACSIL code created or modified for this contract yet.**

Companion to `Algo_Overlay_Framework_Architecture.md`. That document defines
the Sensor -> Market Context -> Trigger Strategy -> Trade Policy -> Order
Management pipeline; this document defines the envelope every Sensor, MCtx,
TriggerStr, and TradePol study publishes on its own persistent storage, so any
consumer (same layer or a layer above) can read it generically before it ever
looks at layer-specific payload.

This is the Chakra equivalent of `K_BatonDecisionContract.h` and the
`K_Template_CloseProtect_DecisionStudy` / `K_Template_CloseProtect_Aggregator`
pair in the candle baton manager's close/protect contract. Order Management
(`K_Util_Candle_Baton_Manager_V1`) is unaffected by this document — it keeps
`K_BatonDecisionContract.h` as-is.

Layer-specific payload fields (what a `WeeklyATR` sensor actually reports,
what an `EntryGate` actually decides) are explicitly **out of scope here** —
those get their own per-layer contract pass once this envelope is settled.

Study naming for anything implementing this contract:
`K_Chakra_<LayerTag>_<DescriptiveName>_V<n>`, layer tags `Sensor` / `MCtx` /
`TriggerStr` / `TradePol`.

------------------------------------------------------------------------

# Changes from Revision 8

Purely additive — nothing about the approved Revision 8 fields, rules, or
invariants is modified or reopened. Found during the Chakra-to-Order-
Management handoff contract pass (`Chakra_OrderManagement_Handoff_Interface_Contract.md`):
`K_CHAKRA_COMPONENT_KIND` never reserved a value for Order Management, since
no prior contract pass needed one — every layer through Trade Policy
produces `PROPOSED_ACTION` and stops there. The handoff contract's own
disposition record (`NO_INSTRUCTION` / `EXECUTABLE_INSTRUCTION` / declined,
correlated back to a specific `ProposedActionSeq`) is exactly the kind of
fact the architecture's own `DecisionFrame` needs to capture ("resulting
executable changes"), and giving it a standard Chakra envelope — rather
than a bespoke, ad hoc reporting mechanism — lets every existing consumer
discipline (identity validation, event pattern, baselining) apply to it
unmodified, the same reasoning that has justified every other additive
enum extension in this project. Fixed: added `K_CHAKRA_KIND_ORDERMGMT = 5`
to `K_CHAKRA_COMPONENT_KIND` (below), purely additive — no existing value
renumbered or reinterpreted. This does **not** mean the existing candle
baton manager (`K_Util_Candle_Baton_Manager_V1`) itself becomes a Chakra
producer — see the handoff contract's own scope section for why that
component and the new Order Management disposition producer are not the
same physical persistent-storage instance.

------------------------------------------------------------------------

# Changes from Revision 3

Revision 3 was reviewed a third time before any code was written. Two of the
three findings were blocking logic bugs, not style issues:

1. **The universal consumer gate was wrong for state dependencies
   (blocking).** Revision 3's single gate (`sawChange && Status == VALID`)
   treated "may I use this data" as conditional on a fresh publication having
   just arrived. That's correct for a discrete event, but wrong for reading a
   Sensor's or MCtx's *current* state: a TradePol component deciding based on
   MCtx's `TREND` context should be able to read it on every evaluation it
   needs to, not only on the evaluation where `TREND` happened to change.
   Fixed by defining **two distinct consumption patterns** instead of one
   universal gate — see below. This also simplifies what Revision 2/3 called
   the "two-step gate": since `PublicationSeq` no longer ever resets in
   ordinary operation (fixed in Revision 2), the event pattern no longer
   needs a separate unconditional-observe step to survive a reset-collision
   that can no longer happen — a single-step gate is now sufficient and
   correct for events too.
2. **Recalc-entry "publish once" did not actually fall out of the
   publish-on-change rule (blocking).** For an AutoLoop producer, ACSIL
   invokes the study function once per bar with `IsFullRecalculation`
   remaining true for the whole sweep. If the producer's recalc-entry logic
   runs unconditionally on every such call rather than detecting the first
   call specifically, each bar's call sees the *previous bar's* settled
   `VALID` state as "current," writes `NOT_READY` as a real change from
   that, recomputes, and writes `VALID` again — producing two publications
   per bar for the entire historical sweep, not one publication for the
   whole transition. "Publish only on change" does not prevent this by
   itself, because each per-bar toggle genuinely is a change relative to
   the immediately preceding call. Fixed by requiring producers to detect
   the actual first call of a recalculation sweep, with concrete guidance
   per looping model (below) rather than relying on snapshot-equality alone.
3. **The commit-order convention was described as stronger than it is.**
   Writing `PublicationSeq` last, and having a consumer read it before and
   after the snapshot, does not detect every torn read: a consumer can read
   the old sequence, read a partially-updated snapshot mid-write, and read
   the *same* old sequence again if its second read races ahead of the
   producer's own final sequence write. Both reads matching does not
   guarantee no tear occurred. Fixed by no longer calling this "atomicity"
   or a guarantee — it's a commit-order convention that is expected to be
   sufficient given ACSIL's serialized per-study calculation model, not a
   proven guarantee under any hypothetical concurrent execution. A true
   seqlock (writer marks an in-progress state before touching any field,
   readers retry on an odd/in-progress marker as well as on mismatch) would
   close the gap completely but is deliberately not adopted here — judged
   unnecessary complexity for a single-threaded-per-study environment.

Plus three smaller fixes:

4. Cached consumer-side state must be treated as immediately unusable the
   moment identity validation or `Status` fails — never silently reused
   "just this once more" from the last known-good read.
5. Default failure policy for events a consumer can observe but can't act on
   (e.g. because some *other* dependency needed to interpret it is stale):
   **consume-and-drop**, not retry — matching the candle-baton lesson
   "decide event meaning once, at first eligible observation, not via
   implicit retry," which exists specifically to prevent a stale event
   mutating into a different action later or backlogging.
6. `PayloadSchemaVersion` added to the formal identity-validation order,
   immediately after `ProducerTypeID`.

------------------------------------------------------------------------

# Changes from Revision 4

One blocking issue found on a fourth review:

1. **`PublicationSeq` cannot also serve as an event dedupe key (blocking).**
   `PublicationSeq` increments on *every* snapshot change — including
   `VALID -> DISABLED`, recalc entry to `NOT_READY`, re-enable back to
   `VALID` with no new setup detected, and any diagnostic-only republish.
   The Revision 4 event pattern used `PublicationSeq != LastConsumedSeq` as
   its dedupe key, which manufactures a false event: a trigger fires at
   `PublicationSeq = 5` and gets consumed; the producer is then disabled
   (`seq 6`) and re-enabled with no new trigger (`seq 7`); the consumer sees
   `7 != 5` and `Status == VALID` and fires on a nonexistent trigger. The
   same failure mode occurs when a TriggerStr returns to `VALID` after a
   recalculation without detecting a new setup. Fixed by giving event-style
   layer payloads their **own** durable `EventSeq` (int64, payload-owned,
   not a common-envelope field), separate from `PublicationSeq`, that
   increments **only** when a genuine event/decision occurs — never on a
   status transition, disable/re-enable, recalc-entry, or diagnostic-only
   republish. `PublicationSeq` reverts to meaning exactly one thing,
   unconditionally: the general snapshot commit marker / revision counter.
   It is never an event dedupe key for any layer. See the rewritten "Event
   pattern" below.

   Naming note: this reintroduces a field called `EventSeq`, which existed
   under that name before the Revision 1 -> 2 rename to `PublicationSeq`.
   It is **not** the same field being un-renamed — `PublicationSeq` stays
   exactly as Revision 2-4 defined it (general, envelope-level, never an
   event key). The new `EventSeq` is a distinct, narrower, payload-owned
   field that only event-style layers define at all.

------------------------------------------------------------------------

# Changes from Revision 5

A fifth, deeper review found three further blocking gaps and several
corrections that strengthen invariants already in place rather than
reversing them:

1. **No rule for the consumer's *own* lifecycle baselining (blocking).**
   Revision 5 covered baselining when a *dependency* gets rewired, but never
   said what a consumer must do at its *own* lifecycle boundaries (its own
   `IsFullRecalculation`, its own disable/re-enable, initial sync, or a new
   trade/session scope for a trade-bound consumer). Worked example: consumer
   has consumed `EventSeq = 8`; the *consumer itself* reloads (its own
   `IsFullRecalculation`); on resuming, `LastConsumedEventSeq` is still 8
   from before the reload; if the producer is meanwhile sitting at
   `EventSeq = 12`, the consumer treats a possibly long-past occurrence as a
   fresh live event the instant it resumes. Fixed by adding an explicit
   "Consumer's own lifecycle baselining" rule, and a companion rule that a
   consumer must never execute an external/live action while its own
   `IsFullRecalculation` is true. This is exactly the "baseline at lifecycle
   boundaries" lesson this contract already claims to carry over from the
   candle-baton contract — it just was never actually applied to the
   consumer's own lifecycle, only to a dependency's identity changing.
2. **Historical recalculation could manufacture live events (blocking).**
   Nothing prevented an event-style producer from incrementing the public
   `EventSeq` for every historically-detected occurrence while replaying
   history during its own `IsFullRecalculation` — which would both leave the
   sticky payload holding an arbitrary historical event, and let a consumer
   mistake it for something that just happened. Fixed by an explicit rule:
   during `IsFullRecalculation`, an event-style producer baselines its
   *private* edge-detection state against history but must not increment the
   public `EventSeq` for anything it detects there — only a genuinely new
   occurrence found during normal (post-recalc) evaluation may increment it.
   This is exactly what `K_Template_CloseProtect_DecisionStudy.cpp` already
   does in this codebase (its recalc branch baselines `lastTriggerBarIndex`/
   `lastTriggerWasActive` and returns, never touching `sourceEventSeq`) — the
   Chakra contract now states this explicitly rather than leaving it to be
   inferred from precedent.
3. **Sticky single-slot events don't guarantee delivery of every occurrence
   (blocking, resolved as an explicit policy, not a mechanism change).** If
   a producer fires several events between two consumer evaluations, the
   sticky single-slot design means only the latest is visible — the sequence
   jump correctly signals detection ("something was missed") but not
   recovery. Resolved by declaring **latest-event-wins/coalescing as this
   contract's default and only defined delivery semantic** — a consumer must
   treat a sequence jump greater than 1 as a signal that intermediate
   events were dropped, and log it, rather than silently ignoring it. A
   guaranteed-delivery mechanism (queue/ring-buffer/aggregator) is
   explicitly out of scope for the common envelope; a specific layer that
   genuinely needs one must design it itself.

Plus corrections that tighten existing rules:

4. **Schema validation must be exact-match, not just nonzero.** `0` means
   "no compatible producer," but a *nonzero, incompatible* version is also
   invalid and was previously unhandled. Consumers now compare
   `SchemaVersion == K_CHAKRA_ENVELOPE_SCHEMA_VERSION_CURRENT` and
   `PayloadSchemaVersion == <layer's expected value>` exactly.
5. **Event publication invariants formalized**, including a corrected commit
   order (envelope + event payload -> `EventSeq` -> `PublicationSeq`, not
   just envelope -> `PublicationSeq`), and an explicit statement that the
   Revision 5 recalc-payload-clearing behavior (`EventType -> NONE` while
   `EventSeq` stays put) is only safe because the event gate always requires
   both together — that invariant must never be weakened to an `EventSeq`
   check alone.
6. **AutoLoop recalc-entry detection tightened** to `sc.Index == 0` only —
   the Revision 3-5 wording offering `sc.DataStartIndex` as an alternative
   conflated "where valid output begins" with "where ACSIL first invokes the
   function," which are different things and could invite incorrect
   initialization logic.
7. **"Replay now" substantially narrowed**, no longer fully open — see
   "Age and 'now'" below. `sc.GetCurrentDateTime()` and
   `sc.BaseDateTimeIn[EvalIndex]` (both confirmed present in the installed
   `sierrachart.h`) are used for different evaluation contexts rather than
   one universal answer.
8. **`ReasonCode` invariants strengthened**: `VALID` with nothing to report
   must be `NONE`; `NOT_READY`/`DISABLED`/`ERROR` should normally carry a
   nonzero, appropriate code; a `ReasonCode` must never describe an earlier
   snapshot than the one it's published alongside.
9. **Identity fields must survive invalid `Status`.** `SchemaVersion`,
   `ComponentKind`, `ProducerTypeID`, and `PayloadSchemaVersion` are stable
   identity facts, not condition payload — they must never be cleared or
   neutralized on `NOT_READY`/`DISABLED`/`ERROR`, or a consumer loses the
   ability to distinguish "a disabled but otherwise-compatible producer"
   from "an absent or unrelated study."

------------------------------------------------------------------------

# Changes from Revision 6

A sixth review approved Revision 6's architecture and found one operational
bug worth fixing before implementation, plus one logging bug:

1. **Initial attachment is not inert by default — this was a real bug, not
   just imprecise wording.** Revision 6 claimed first-ever attachment to a
   dependency was "already inert" because both a fresh producer and a fresh
   consumer default to 0. That's only true if the producer is *also* fresh.
   The far more common case is a consumer attaching to a producer that has
   already been running and already has a nonzero `EventSeq` (say, 8) — the
   new consumer's zero-default `LastConsumedEventSeq` sees `8 != 0` and
   `EventType != NONE`, and immediately replays event 8 as if it just
   happened, even though it may have fired long before this consumer
   existed. Fixed by folding first-ever attachment into the **same explicit
   mechanism already used for dependency rewiring**, rather than treating it
   as a separate case that happens to be safe by coincidence: a private
   per-slot "baselined" flag (or equivalently, treating unset stored
   chart/study identity as an automatic mismatch) forces an explicit
   baseline-to-current-value-without-consuming step on first attachment,
   exactly like rewiring. The same explicit mechanism — not reliance on zero
   defaults — is required for the consumer's own disable -> re-enable
   transition too.
2. **Gap-detection logging compared a value against itself.** The Revision 6
   wording said to detect `producer.EventSeq - consumer.LastConsumedEventSeq
   > 1` "after a successful consume" — but if `LastConsumedEventSeq` has
   already been updated to the new value by that point, the difference is
   always 0. Fixed by capturing the previous value in a local before
   updating, and by explicitly branching on forward-vs-backward movement
   rather than computing a raw difference: a backward jump
   (`EventSeq < previousSeq`) is not a "some events were coalesced" gap —
   given the documented full-storage-lifecycle-reset residual risk (open
   questions), it's a distinct signal that persistent storage may have
   reset, and must be logged as such and re-baselined, not silently absorbed
   by arithmetic that happens to not trip the `> 1` check.

------------------------------------------------------------------------

# Changes from Revision 7

Purely additive — nothing about the approved Revision 7 fields, rules, or
invariants is modified or reopened. Found during the Sensor Interface
Contract pass: sensor payloads routinely need float precision (price, ATR,
ratios), which the common envelope never provisioned because none of its
own 10 fields needed it. The Float/Double persistent-variable accessor
namespaces are added to the PV ownership table below, following the exact
same public/private range shape already used for int32/int64/SCDateTime, so
this is a framework-wide convention available to any layer's payload — not
a Sensor-specific rule. See "Public versus private persistent-variable
ownership."

------------------------------------------------------------------------

# Lessons carried over (non-negotiable, unchanged since Revision 2)

1. **Sequence inequality, not ordering.** Compare `!=`, never `>`.
2. **Never rely on a transient one-call pulse.** Every field that matters to
   a consumer must be sticky.
3. **Private edge-gating state must never occupy a slot adjacent to or
   interleaved with public contract slots.** Non-adjacent ranges reserved up
   front.
4. **Decide event meaning once, at first eligible observation, not via
   implicit retry.** Reinforced by fix #5 above.

------------------------------------------------------------------------

# The publication model and commit-order convention

A **publication** is the producer writing its **complete public snapshot —
common envelope fields and layer-specific payload fields together**, then
writing `PublicationSeq` last as the commit marker.

- **What counts as a change.** A publication happens whenever *any* field in
  the complete public snapshot would differ from what is currently
  published — envelope, payload, or both. Landing on fully identical state
  on re-evaluation is a no-op: nothing is written, nothing increments.
- **Commit order.** A producer writes every field except `PublicationSeq`
  first (any order relative to each other), then writes/increments
  `PublicationSeq` last.
- **This is a convention, not a proven atomicity guarantee.** Writing the
  sequence last, and having a consumer optionally read it before and after
  reading the snapshot, catches the common case of a consumer racing a
  full producer write — but it does **not** catch every possible
  interleaving (a consumer's second sequence read can itself race ahead of
  the producer's final write and still match the first read while the
  snapshot in between was torn). This contract relies explicitly on ACSIL's
  serialized per-study calculation model to make that residual case
  impractical, not on a formal guarantee. A full seqlock (writer sets an
  in-progress marker before touching any field; readers retry on an
  odd/in-progress marker in addition to a before/after mismatch) would
  close this completely and was considered, but is deliberately not adopted
  — judged unnecessary complexity here.
  ```
  // Best-effort sanity check, not a formal guarantee (see above):
  seqBefore = readPublicationSeq();
  readEnvelopeAndPayloadSnapshot();
  seqAfter = readPublicationSeq();
  if (seqBefore != seqAfter)
      // discard this pass's snapshot, retry on the next evaluation
  ```
- `PublicationSeq` starts at 0 and is **never deliberately reset** by the
  producer's own code — not on `IsFullRecalculation`, not on
  disable/re-enable. Monotonic for the life of the current persistent
  storage (see the residual full-lifecycle-reset risk in open questions).

## Recalc-entry publication must happen exactly once per sweep — detection is explicit, not implicit

"Publish only on change" does **not** by itself guarantee a producer
publishes its recalc-entry transition once. A producer must explicitly
detect the first call of a recalculation sweep and only treat that call as
the entry transition:

- **AutoLoop producers:** ACSIL invokes the study function once per bar,
  in increasing index order, with `IsFullRecalculation` true for the whole
  sweep. The first call of the sweep is `sc.IsFullRecalculation && sc.Index
  == 0` — the established Sierra pattern. Do **not** substitute
  `sc.DataStartIndex` for this check: `DataStartIndex` describes where a
  study's output is considered valid (a warmup-length concept), not where
  ACSIL first invokes the function, which for AutoLoop is unconditionally
  index 0 regardless of any `DataStartIndex` the producer sets. Conflating
  the two invites a producer to skip its recalc-entry publication on early
  bars it considers "before valid output," which is a different concept
  entirely. Only the `sc.Index == 0` call performs the recalc-entry
  publication (`Status -> NOT_READY`, etc.); every subsequent call in the
  same sweep proceeds straight to normal per-bar computation, which may
  itself publish additional times as real values become available — that's
  expected and fine, it's the spurious *re-entry* toggle that must not
  happen.
- **Manual-loop producers (`sc.AutoLoop = 0`):** this is the established
  convention for anything needing cross-study reads or complex persistent
  state per this project's ACSIL skill, and the existing
  `K_Template_CloseProtect_DecisionStudy`/`Aggregator` pattern already
  satisfies "exactly once" for free, because ACSIL invokes a manual-loop
  study's function once for the whole recalculation event rather than once
  per bar — there is no second call within the same sweep to spuriously
  re-trigger from. Where a manual-loop producer's own internal design
  doesn't naturally guarantee single-invocation-per-sweep (uncommon, but not
  ruled out), fall back to a private edge-latch (`hasPublishedRecalcEntry`,
  PV in the 50-89 private range) checked and set on the first call, cleared
  once `IsFullRecalculation` is next observed false, so a genuinely new
  future sweep is detected fresh. `sc.IsFullRecalculation &&
  sc.UpdateStartIndex == 0` is a commonly-correct proxy for "start of this
  pass" if a producer needs one without a full latch, but is not universal
  — this is what the reviewer flagged as needing producer-specific handling,
  not a single rule for every manual-loop producer.

## Event-style producers must not manufacture live events during recalculation

`IsFullRecalculation` replays history so a producer can rebuild its own
state — it is not a live event bus playback. An event-style producer
(TriggerStr, or an event-style TradePol output) must:

- **Baseline its private edge-detection state against history** during
  `IsFullRecalculation`, however it needs to (this is the producer's own
  internal business, out of scope for this contract) — but
- **Never increment the public `EventSeq` for anything detected during that
  replay.** Only a genuinely new occurrence detected during normal
  (post-recalc) evaluation may increment `EventSeq`.

This is not a new mechanism — it is exactly what
`K_Template_CloseProtect_DecisionStudy.cpp` already does in this codebase:
its `IsFullRecalculation` branch baselines `lastTriggerBarIndex`/
`lastTriggerWasActive` against the last bar and returns, without ever
touching `sourceEventSeq`; `sourceEventSeq` only increments in the live
evaluation path below. Every Chakra event-style producer must follow the
same shape.

A producer *may* still write historical markers to a subgraph for charting
or backtest-visualization purposes during recalculation — that is a display
concern, entirely orthogonal to the public envelope/payload contract, and is
not affected by this rule.

**Replay is different from an ordinary full recalculation and does not need
special-casing here.** Sierra Chart's replay mode primes historical state up
to the replay's start point (which hits the `IsFullRecalculation` path
above, silently) and then advances forward bar-by-bar or tick-by-tick using
the *normal* (non-recalc) evaluation path — the same path live trading uses.
A genuinely new occurrence detected as replay advances forward correctly
increments `EventSeq` exactly as it would live, because it goes through the
normal path, not the recalc path. No additional mechanism is needed for
replay to correctly simulate live events once the rule above is followed.

------------------------------------------------------------------------

# Two consumption patterns — not one universal gate

`PublicationSeq` means exactly one thing, always: a general snapshot commit
marker (see "The publication model" above). It is never the basis for "may I
use this data" (state pattern) or "did a genuine event occur" (event
pattern) — those are answered by different mechanisms entirely, and a
consumer must use the pattern that matches what it depends on. Conflating
the two was the Revision 3 bug (gating state reads on a sequence change) and
the Revision 4 bug (using the general sequence as an event dedupe key) —
both are fixed below.

## State pattern — Sensor / MCtx, and any TradePol output that represents continuous "current condition" rather than a discrete decision

```
if (identityValid && producer.Status == K_CHAKRA_STATUS_VALID && freshnessValid)
{
    // May read and use the current snapshot on every evaluation that needs
    // it, regardless of whether PublicationSeq changed since it was last
    // checked. A valid, fresh snapshot does not expire just because nothing
    // new has been published.
}
```

`PublicationSeq` is **optional** here and purely a cache-invalidation hint:
a consumer that caches an expensive derived computation from this snapshot
may track its own `LastObservedSeq` and compare `PublicationSeq !=
LastObservedSeq` to decide whether to recompute — but a mismatch is never a
precondition for reading the current value, only a hint that a cached
derivation may now be stale. `identityValid` includes `PayloadSchemaVersion`
(see validation order below). `freshnessValid` is the `Age`-vs-tolerance
check from "Consumer baselining," below.

**If `identityValid` or `Status == VALID` fails on a given evaluation, any
previously cached snapshot or derived value from that dependency must be
treated as immediately unusable** — not silently reused from the last
known-good read. A consumer that wants graceful degradation must decide that
explicitly (e.g. an explicit fallback policy), not by accident via a stale
cache.

## Event pattern — TriggerStr, and any TradePol output that represents a discrete decision/occurrence

Event-style layers own a **separate, payload-level `EventSeq`** (int64,
reserved by convention at int64 PV 10 — the first slot of that layer's
payload int64 range, paralleling how `PayloadSchemaVersion` reserves int32
PV 20). `EventSeq` increments **only** when the producer's own detection
logic finds a genuine new event/decision — never as a side effect of a
status transition, disable/re-enable, recalc-entry, or diagnostic-only
republish. It is monotonic for the life of the current persistent storage,
same caveat as `PublicationSeq` (see open questions). The event payload
itself (whatever fields describe *what* happened) is rewritten together with
`EventSeq` and stays sticky between increments — same discipline as
`PublicationSeq`'s snapshot, just scoped to this narrower field.

Every event-style layer also defines an `EventType`-or-equivalent payload
field with an explicit `NONE = 0` member (mirroring
`K_CLOSE_EVENT_NONE` in the close-protect contract), so a producer that is
`VALID` but has never fired an event cannot be mistaken for one that has.
**This is a load-bearing invariant, not a convenience:** Revision 5's
recalc-entry behavior clears event payload to neutral (`EventType -> NONE`)
while leaving `EventSeq` untouched — that is only safe because the event
gate below always checks `EventType != NONE` **together with** the
`EventSeq` comparison. A future implementation must never weaken the gate to
an `EventSeq`-inequality check alone.

```
if (identityValid
    && producer.Status == K_CHAKRA_STATUS_VALID
    && producer.EventType != <layer's NONE value>
    && producer.EventSeq != consumer.LastConsumedEventSeq[slot])
{
    consumer.LastConsumedEventSeq[slot] = producer.EventSeq;
    consumeEvent();   // decide meaning now; see failure-policy note below
}
```

`PublicationSeq` is **not** part of this gate — it is never the event
dedupe key, for any layer (see "Changes from Revision 4"). A consumer may
still use `PublicationSeq` for the general torn-read sanity check when
reading the snapshot, exactly as in the state pattern, but `EventSeq` alone
determines whether a genuinely new event has occurred.

**Event publication invariants.** Whenever a producer increments `EventSeq`,
all of the following must hold in that same publication:

- `Status == K_CHAKRA_STATUS_VALID` — an invalid/disabled/error publication
  must never increment `EventSeq`.
- `EventType != <layer's NONE value>`.
- The event payload is complete — no partially-written event fields.
- `EvalBarIndex` and `EvalDateTime` identify *that event* specifically (the
  bar/time the occurrence was detected at), not a later unrelated
  re-evaluation.
- **Commit order:** envelope fields and event payload (including the new
  `EventType`/event fields) are written first, **then** `EventSeq` is
  incremented, **then** `PublicationSeq` is written last, as the overall
  commit marker — i.e. `envelope+payload -> EventSeq -> PublicationSeq`,
  not the two-stage `envelope -> PublicationSeq` order used for a
  non-event publication.

**Delivery semantic: latest-event-wins, not guaranteed delivery.** The
sticky single-slot design (one current `EventSeq` + one current event
payload) guarantees a consumer *detects* that something changed, not that it
*receives every occurrence*. If a producer fires several events between two
consumer evaluations, only the latest is visible. This is the contract's
deliberate default. Gap detection must capture the previous value **before**
updating it, and must branch on the direction of movement rather than
computing a raw difference (a raw `EventSeq - LastConsumedEventSeq` after
`LastConsumedEventSeq` has already been updated is always 0 — comparing a
value against itself is not a gap check):

```
const int64 previousSeq = consumer.LastConsumedEventSeq[slot];   // capture first

if (producer.EventSeq != previousSeq)
{
    if (producer.EventSeq > previousSeq)
    {
        if (producer.EventSeq - previousSeq > 1)
            LogDroppedIntermediateEvents(slot, previousSeq, producer.EventSeq);
    }
    else   // producer.EventSeq < previousSeq -- backward movement
    {
        // Not a coalescing gap -- see the full-storage-lifecycle reset
        // residual risk in open questions. Log distinctly, then re-baseline
        // rather than computing a nonsensical negative "events dropped" count.
        LogSequenceWentBackward(slot, previousSeq, producer.EventSeq);
    }

    consumer.LastConsumedEventSeq[slot] = producer.EventSeq;
    consumeEvent();
}
```

If a specific layer genuinely needs guaranteed delivery of every occurrence
(not just the latest), that layer must design its own queue or aggregator
mechanism explicitly in its own contract pass; the common envelope does not
provide one, to avoid building a general mechanism before a concrete need
demonstrates it's required.

**Failure policy (default: consume-and-drop).** If an event is observed
(the `if` above is true) but the consumer cannot usefully act on it this
call — e.g. some *other* dependency needed to interpret it is itself stale
or unavailable — the default behavior is to still update
`LastConsumedEventSeq` (consuming it) and log the drop, not leave it
unconsumed hoping to retry successfully later. This matches the candle-baton
lesson: deciding an event's meaning once, at first eligible observation,
prevents a stale event from later mutating into a different action, or
backlogging behind a consumer that's persistently unable to act. A specific
consumer's own contract pass may deliberately choose retry semantics
instead, but must do so explicitly and for a stated reason — it is not the
default.

TradePol sub-components must declare, in their own contract pass, which of
these two patterns each of their outputs uses — a policy decision (discrete)
and a policy-maintained recommendation (continuous, e.g. "current suggested
stop distance") are not the same shape and should not share one pattern by
default.

------------------------------------------------------------------------

# Producer identity and validation order

A consumer must validate a dependency in this exact order, and must not read
or trust `Status`/payload until all four prior checks pass:

1. **`SchemaVersion`.** Must equal `K_CHAKRA_ENVELOPE_SCHEMA_VERSION_CURRENT`
   **exactly** — `0` means "no compatible producer" (unwired, or not a
   Chakra producer at all), but a *nonzero, non-matching* value is equally
   invalid and must not be treated as "close enough because it's nonzero."
   Distinct from `Status == NOT_READY`. If backward-compatible version
   ranges are ever supported, that must be a deliberate, explicitly-defined
   rule added later — exact match is the contract until then.
2. **`ComponentKind`.** `K_CHAKRA_KIND_NONE` (0) or a mismatch against the
   expected layer → skip, log as misconfiguration.
3. **`ProducerTypeID`.** Mismatch against the specific producer expected →
   skip, log.
4. **`PayloadSchemaVersion`** (layer-payload-range PV 20, by convention —
   see PV ownership). Must equal the consumer's expected value for that
   specific producer **exactly**, same reasoning as step 1 — a compatible
   common envelope never implies a compatible payload layout; mismatch →
   skip, log.
5. **Only now** proceed to the matching consumption pattern above.

These four identity fields (`SchemaVersion`, `ComponentKind`,
`ProducerTypeID`, `PayloadSchemaVersion`) are stable facts about *what this
producer is*, not part of its current condition — a producer must keep them
populated with their correct, unchanging values regardless of `Status`
(`NOT_READY`/`DISABLED`/`ERROR`/`VALID`). They must never be cleared or
neutralized alongside condition payload. Without this, a consumer cannot
tell "a disabled but otherwise-compatible producer" apart from "an absent or
unrelated study" — both would read as unusable, but only one is actually
mis-wired.

------------------------------------------------------------------------

# Envelope fields (10 total, across three accessor namespaces)

| Field | ACSIL storage | Namespace | Purpose |
|---|---|---|---|
| `SchemaVersion` | persistent int | int32 | Versions the **common envelope only**. `0` = no compatible producer. |
| `Status` | persistent int (`K_CHAKRA_STATUS`) | int32 | `NOT_READY` / `VALID` / `DISABLED` / `ERROR`. |
| `ComponentKind` | persistent int (`K_CHAKRA_COMPONENT_KIND`) | int32 | Self-check: confirms the wired chart/study is the expected layer. `NONE = 0` is a real enum member. |
| `ProducerTypeID` | persistent int | int32 | Which specific producer within its layer. Locally enumerated per layer. |
| `EvalBarIndex` | persistent int | int32 | Bar index the published snapshot was computed from. |
| `DecayClass` | persistent int (`K_CHAKRA_DECAY_CLASS`) | int32 | `STATIC_SESSION` / `SLOW` / `FAST`. |
| `ReasonCode` | persistent int | int32 | Numeric only, decoded to text only at log time. Strictly self-referential. Must be explicitly written to `K_CHAKRA_REASON_NONE (0)` on any publication with no active diagnostic condition. |
| `PublicationSeq` | persistent **int64** | int64 (separate namespace) | Monotonic for the life of the current persistent storage; never deliberately reset. **Always** a general snapshot commit marker / revision counter — increments on any change to envelope or payload, including status transitions, disable/re-enable, and recalc-entry. Never an event dedupe key for any layer; see the payload-owned `EventSeq` in the event pattern for that. |
| `EvalDateTime` | persistent SCDateTime | datetime (separate namespace) | Wall/session time the snapshot corresponds to. |
| `ValidUntilDateTime` | persistent SCDateTime, sentinel = 0 | datetime (separate namespace) | Producer-resolved concrete boundary. Only meaningful when `DecayClass == STATIC_SESSION` and nonzero. |

`ApplicabilityScope` remains removed (Revision 3). `PayloadSchemaVersion` is
not a common-envelope field — it is a required-by-convention layer-payload
field at PV 20, one per layer contract.

------------------------------------------------------------------------

# Status semantics (unchanged)

```cpp
enum K_CHAKRA_STATUS
{
    K_CHAKRA_STATUS_NOT_READY = 0,
    K_CHAKRA_STATUS_VALID     = 1,
    K_CHAKRA_STATUS_DISABLED  = 2,
    K_CHAKRA_STATUS_ERROR     = 3
};
```

------------------------------------------------------------------------

# Full recalculation, reload, disable/re-enable, replay, upstream transient invalidation

- **`IsFullRecalculation`:** public envelope behavior on the detected entry
  call only — `Status -> NOT_READY`, *condition* payload fields to their
  neutral values (event-style: `EventType -> NONE`, `EventSeq` untouched —
  see "Event-style producers must not manufacture live events" above), one
  publication. Identity fields (`SchemaVersion`, `ComponentKind`,
  `ProducerTypeID`, `PayloadSchemaVersion`) are never touched — see
  "Producer identity and validation order." See the explicit
  per-looping-model detection requirement above; this is not automatic. How
  a producer rebuilds its own private multi-bar state is out of scope for
  this contract.
- **Disable -> re-enable:** each is one publication. `PublicationSeq` is
  never touched except by its normal per-publication increment. Identity
  fields are unaffected, same as recalc.
- **Replay:** publication can happen on any cadence a producer chooses. What
  "now" means for computing `Age` during replay versus live is no longer
  fully open — see "Age and 'now'" immediately below.
- **Upstream transient invalidation** (a depended-upon producer publishes
  `NOT_READY`/`DISABLED`/`ERROR`, e.g. mid-recalculation, or cycles through
  disable/re-enable): requires no special-case consumer code under either
  consumption pattern. The state pattern re-checks `Status` fresh every
  call by design. The event pattern's gate stays correctly inert while
  `Status != VALID` **and** is immune to the transition itself creating a
  false event, because `EventSeq` is untouched by status transitions,
  disable/re-enable, or recalc-entry — it only moves when the producer's own
  event-detection logic finds a genuine new occurrence. A `VALID -> DISABLED
  -> VALID` cycle with no real event in between leaves `EventSeq` exactly
  where it was, so the event pattern's gate correctly stays false throughout
  (this is precisely the bug fixed in Revision 5 — see "Changes from
  Revision 4").

------------------------------------------------------------------------

# Age and "now"

`Age = now - EvalDateTime` needs a definition of "now" that's correct in
both live and historical contexts — a single universal answer isn't right,
because system clock time is not the same thing as the time a historical bar
was evaluated at. Confirmed present in the installed `sierrachart.h`:
`sc.GetCurrentDateTime()` and `sc.BaseDateTimeIn[BarIndex]`. Use:

- **Live evaluation, and live/paced replay:** `now = sc.GetCurrentDateTime()`
  — returns replay time while a replay is actively running, system time
  otherwise.
- **Historical bar evaluation** (computing what `Age` was *as of* a specific
  historical bar, e.g. during a backtest pass or when reasoning about a
  past bar's decay tolerance): `now = sc.BaseDateTimeIn[EvalIndex]` — the
  bar's own timestamp, **not** wall-clock/system time, which would be
  meaningless relative to bars that happened in the past.

A consumer must pick whichever of these matches the evaluation context it is
actually in — `Age` is always computed in the consumer's *evaluation-time
domain*, never blindly against wall-clock time regardless of context. This
substantially narrows what was fully open through Revision 5; it does not
yet address every edge case (e.g. precisely how a producer's own
`EvalDateTime` should be set during its own historical replay pass), which
is why "replay now" is downgraded rather than fully removed from open
questions.

------------------------------------------------------------------------

# Consumer baselining: own lifecycle and dependency rewiring

## The consumer's own lifecycle (not just a dependency's)

Every rule below about baselining `LastConsumedEventSeq`/`LastObservedSeq`
applies just as much when the *consumer itself* goes through a lifecycle
transition, not only when a dependency is rewired. This was missing through
Revision 5 despite being one of the candle-baton lessons this contract
claims to carry over. Worked example of the gap: a consumer has consumed
`EventSeq = 8` from some TriggerStr; the consumer itself then reloads (its
own `IsFullRecalculation`); on resuming normal evaluation,
`LastConsumedEventSeq` is still 8 from before the reload; if the producer is
by then at `EventSeq = 12` (whether from genuinely new occurrences or, prior
to the Revision 6 fix above, from replaying history), the consumer
immediately treats event 12 as a fresh live occurrence the instant it
resumes — potentially firing a live action (an order, an alert) purely as a
side effect of the consumer's own reload.

A consumer must re-baseline **every** `LastConsumedEventSeq[slot]` and
`LastObservedSeq[slot]` it owns, to each configured dependency's *current*
value (never to 0, same reasoning as dependency rewiring below), at all of
the following consumer-side boundaries:

- The consumer's own `IsFullRecalculation`.
- The consumer's own disable -> re-enable transition.
- **Initial sync/attachment.** This is **not** inert by default — a common
  misreading is that a fresh consumer's zero-default `LastConsumedEventSeq`
  is automatically safe because it starts at 0, but the *producer* is
  usually not fresh: it may already be sitting at `EventSeq = 8` from
  activity before this consumer ever attached, and a naive gate would see
  `8 != 0` and immediately replay event 8 as if it just happened. First
  attachment must be detected and baselined through the **same explicit
  mechanism as dependency rewiring** (below) — never assumed safe from zero
  defaults alone.
- For any consumer whose own lifecycle is scoped to something narrower than
  "for as long as the study exists" — e.g. a TradePol component bound to one
  specific trade — the start of each new trade/session scope. This directly
  mirrors the candle baton manager's own PV161/162 pattern: baseline at
  sync, never hard-reset to 0.

Every one of these boundaries — including the consumer's own disable ->
re-enable — must go through the same explicit per-slot mechanism described
under "Dependency rewiring" below, not an implicit assumption that defaults
happen to line up safely.

**A consumer must never execute an external/live action (an order, an
alert, anything with a real-world side effect) while its own
`IsFullRecalculation` is true.** This is a blanket safety rule on top of the
baselining rule above, not a substitute for it — baselining prevents a
*stale* event from firing after reload; this rule prevents *any* action,
stale or fresh, from firing *during* a reload/backtest pass in the first
place.

## Dependency rewiring, first attachment, and the consumer's own disable/re-enable (one explicit mechanism, not three)

- An event-pattern consumer keeps a **private** `LastConsumedEventSeq`
  (int64) per upstream dependency (PV range 50-89), baselined against that
  producer's payload-owned `EventSeq` — **not** `PublicationSeq`. A
  state-pattern consumer may optionally keep a `LastObservedSeq` (int64)
  purely for cache invalidation, baselined against `PublicationSeq`. The two
  are distinct concepts, track distinct fields, and must not share a slot or
  a name in an implementation — one is a correctness-critical dedupe anchor,
  the other is a performance optimization a consumer is free to skip
  entirely.
- **One explicit trigger condition covers all three cases below** — first
  attachment, rewiring, and the consumer's own disable/re-enable are not
  three separate mechanisms, they are three ways of reaching the same
  "baseline before consuming" state. ACSIL has no attachment event, so each
  consumer stores, per dependency slot, a private `slotBaselined` flag
  (int, defaults to 0/false via ACSIL's normal persistent-int zero-default —
  this default is legitimate to rely on here, since it tracks "have I
  explicitly baselined," not a coincidental value match) plus the chart
  number and study ID it last evaluated against. Every call, before running
  either consumption pattern's gate:
  - If `slotBaselined == 0` (covers first-ever attachment, and any point a
    consumer explicitly clears it — its own recalc, its own disable ->
    re-enable, or a new trade/session scope), **or** the currently-configured
    chart/study differs from the stored value (rewiring):
    - **Event pattern:** set `LastConsumedEventSeq[slot] = producer.EventSeq`
      (whatever it currently is, even if nonzero) — never to 0, which would
      make the very next check look like "something changed" and replay a
      stale or historical sticky event as if it just happened. Do **not**
      consume the existing sticky event on this baselining call.
    - **State pattern:** set `LastObservedSeq[slot] = producer.PublicationSeq`
      (if tracked), and immediately read the producer's current snapshot on
      this same call — a newly-baselined consumer wants current state, not
      "wait for it to next change."
    - Set `slotBaselined = 1` and update the stored chart/study values.
    - Skip the normal consumption gate for this call — baselining and
      consuming never happen in the same evaluation.
  - Otherwise (already baselined, same identity): proceed with the normal
    consumption pattern gate as described above.
- A consumer must explicitly reset `slotBaselined -> 0` for every dependency
  slot it owns at its own `IsFullRecalculation` entry and its own disable ->
  re-enable transition (per "The consumer's own lifecycle," above) — this is
  what routes those boundaries through the same mechanism rather than
  needing separate handling.
- **Missing dependency** (`(0,0)`, deliberately unwired): skip silently, no
  log.
- **Wired but unusable dependency** (fails identity validation, or `Status
  != VALID`): skip, log via the consumer's own diagnostics — never by
  writing into the producer's storage. Per the state-pattern rule above, any
  previously cached value from that dependency is immediately unusable, not
  silently reused.
- **Stale dependency:** `Age = now - producer.EvalDateTime`, compared
  against the consumer's own tolerance for that dependency's `DecayClass`.

------------------------------------------------------------------------

# `ValidUntilDateTime` sentinel rule (unchanged)

Sentinel is `0`. A consumer only reads/compares it when `DecayClass ==
K_CHAKRA_DECAY_STATIC_SESSION` **and** it is nonzero.

------------------------------------------------------------------------

# Public versus private persistent-variable ownership

```
int32 PVs      1-19   Common envelope (public) — 7 assigned
               20-49   Layer-specific public payload — PV 20 reserved by convention for
                       PayloadSchemaVersion; rest not yet designed
               50-89   Private internal state (edge-gating latches, dependency-rewiring
                       chart/study bookkeeping, per-slot slotBaselined flags, etc.) —
                       NEVER read cross-study, ever
               90+     Uncommitted / future use

int64 PVs      1-9     Common envelope (public) — 1 assigned (PublicationSeq)
               10-29    Layer-specific public payload — int64 PV 10 reserved by convention for
                        that layer's own EventSeq, for event-style layers only (see event pattern)
               30-59    Private internal state (e.g. a consumer's own LastConsumedEventSeq/
                        LastObservedSeq per dependency)
               60+      Uncommitted / future use

SCDateTime PVs 1-9     Common envelope (public) — 2 assigned
               10-29    Layer-specific public payload
               30+      Private internal state

Float PVs      (no common-envelope fields use this namespace)
               1-49     Layer-specific public payload (measurement values —
                        price, ratios, anything needing float precision)
               50-89    Private internal state
               90+      Uncommitted / future use

Double PVs     (no common-envelope fields use this namespace)
               1-49     Layer-specific public payload (rare — only if float
                        precision is insufficient)
               50-89    Private internal state
               90+      Uncommitted / future use
```

Float and Double are framework-wide payload namespaces, available to any
layer (Sensor, MCtx, TradePol, ...) that needs them — not exclusive to any
one layer's contract, even though the Sensor Interface Contract was the
first to need them.

Nothing downstream should ever call a cross-study persistent accessor on a
PV in a private range against another study's storage.

------------------------------------------------------------------------

# `ReasonCode` ownership and invariants

Strictly self-referential — a producer's own diagnostics, never a
consumer's judgment about someone else.

- `Status == VALID` with nothing to report → `ReasonCode =
  K_CHAKRA_REASON_NONE` (0), written explicitly, every time — never left
  stale from a prior condition.
- `Status` of `NOT_READY`, `DISABLED`, or `ERROR` → `ReasonCode` should
  normally carry a nonzero, appropriate code, not just default to `NONE` —
  a diagnostic-capable field that's usually empty defeats its own purpose.
- `ReasonCode` must never describe an earlier snapshot than the one it is
  published alongside, in either direction — a fresh publication's
  `ReasonCode` always describes *that* publication's condition, full stop.

------------------------------------------------------------------------

# Direction and Applicability (unchanged since Revision 3)

MCtx owns its own descriptive `Bias`/`Direction` payload field; TriggerStr
and TradePol own their own prescriptive `Applicability` payload field. No
shared field in the common envelope attempts to serve both. Deferred design,
not implemented here.

------------------------------------------------------------------------

# Proposed enums (draft — not yet in any `.h` file)

```cpp
// Exact-match version a producer implementing THIS revision of the common
// envelope writes into its own SchemaVersion field. A consumer compares
// against this (or whatever version it was itself built against) exactly
// -- see "Producer identity and validation order."
const int K_CHAKRA_ENVELOPE_SCHEMA_VERSION_CURRENT = 1;

enum K_CHAKRA_COMPONENT_KIND
{
    K_CHAKRA_KIND_NONE       = 0,
    K_CHAKRA_KIND_SENSOR     = 1,
    K_CHAKRA_KIND_MCTX       = 2,
    K_CHAKRA_KIND_TRIGGERSTR = 3,
    K_CHAKRA_KIND_TRADEPOL   = 4,
    K_CHAKRA_KIND_ORDERMGMT  = 5   // Revision 9 -- the Order Management disposition
                                   // producer defined in
                                   // Chakra_OrderManagement_Handoff_Interface_Contract.md.
                                   // NOT the existing K_Util_Candle_Baton_Manager_V1 itself --
                                   // see that contract's own scope section.
};

enum K_CHAKRA_STATUS
{
    K_CHAKRA_STATUS_NOT_READY = 0,
    K_CHAKRA_STATUS_VALID     = 1,
    K_CHAKRA_STATUS_DISABLED  = 2,
    K_CHAKRA_STATUS_ERROR     = 3
};

enum K_CHAKRA_DECAY_CLASS
{
    K_CHAKRA_DECAY_STATIC_SESSION = 0,
    K_CHAKRA_DECAY_SLOW           = 1,
    K_CHAKRA_DECAY_FAST           = 2
};

enum K_CHAKRA_REASON_CODE_COMMON
{
    K_CHAKRA_REASON_NONE = 0
    // layer-specific reason codes defined in each layer's own contract, numbered above 0
};

enum K_CHAKRA_ENVELOPE_PV_INT   // persistent-int (int32) slots 1-7
{
    K_CHAKRA_PV_SCHEMA_VERSION   = 1,
    K_CHAKRA_PV_STATUS           = 2,
    K_CHAKRA_PV_COMPONENT_KIND   = 3,
    K_CHAKRA_PV_PRODUCER_TYPE_ID = 4,
    K_CHAKRA_PV_EVAL_BAR_INDEX   = 5,
    K_CHAKRA_PV_DECAY_CLASS      = 6,
    K_CHAKRA_PV_REASON_CODE      = 7
};

enum K_CHAKRA_ENVELOPE_PV_INT64   // separate accessor namespace
{
    K_CHAKRA_PV64_PUBLICATION_SEQ = 1
};

enum K_CHAKRA_ENVELOPE_PV_DATETIME   // separate accessor namespace
{
    K_CHAKRA_PV_EVAL_DATETIME        = 1,
    K_CHAKRA_PV_VALID_UNTIL_DATETIME = 2
};

// Layer-payload conventions (not common-envelope PVs, documented here for visibility):
//   int32 PV 20 in every layer's own payload range = that layer's PayloadSchemaVersion
//   int64 PV 10 in every EVENT-STYLE layer's own payload range = that layer's EventSeq
//     (never assigned for pure state-style layers, e.g. a plain Sensor with no discrete
//     event concept -- only TriggerStr and event-style TradePol outputs need this)
//   Every event-style layer also defines its own EventType-equivalent enum with an
//   explicit NONE = 0 member, e.g.:
//
//   enum K_CHAKRA_<LAYER>_EVENT_TYPE
//   {
//       K_CHAKRA_<LAYER>_EVENT_NONE = 0,   // required -- VALID-but-never-fired must not
//                                          // be mistaken for a fired event
//       K_CHAKRA_<LAYER>_EVENT_...  = 1,   // layer-specific event types from here
//   };
```

------------------------------------------------------------------------

# Unresolved design questions — deliberately left open

Questions resolved by Revision 2-6 are removed. Remaining:

1. **Producer-side `EvalDateTime` during historical replay.** "Age and
   'now'" (above) resolves what a *consumer* uses as "now." Still open: what
   should a *producer* write into its own `EvalDateTime` while it is itself
   being replayed/recalculated over history — presumably
   `sc.BaseDateTimeIn[EvalIndex]` for the bar being processed, matching the
   consumer-side historical rule, but this hasn't been stated as a producer
   requirement yet.
2. **Full-storage-lifecycle reset residual risk.** `PublicationSeq` restarting
   at 0 after a DLL unload / study remove-and-readd / chart reconstruction is
   not fully closed by dependency-rewiring detection alone, since this
   document cannot assert ACSIL's StudyID-reuse behavior across such resets
   without verifying against the platform.
3. **`ProducerTypeID` scoping.** Local-per-layer-enum confirmed, no central
   registry — but does Chakra want a lightweight, non-coupling *catalog*
   purely for log decoding/tooling later?
4. **`SchemaVersion` granularity beyond the envelope/payload split already
   made.** Is one envelope version number enough indefinitely?
5. **`ReasonCode` catalog shape.** One shared numeric space across all layers,
   or fully independent per-file numbering like the existing `CPA`-prefixed
   convention?
6. **Is chart+study sufficient producer identity in every case?** Flagging in
   case a future producer design wants one physical study file to publish
   multiple logical sub-producers.
