# Chakra — Mode Baton Signal Publisher Interface Contract (Draft, Revision 1)

Status: **design only, no ACSIL implementation yet.**

Supersedes, for the version-one live path:
- `Chakra_OrderManagement_Handoff_Interface_Contract.md` (Revision 8) — see
  that document's own new "Superseded" banner. Its Entry Authority,
  correlation-identity, idempotency, rejection/partial-fill, stop-attachment-
  failure, and emergency-flatten design is preserved there as a historical
  record and is **not** carried forward here — **not because Mode Baton
  already does that work (a direct source audit of `K_Util_ModeBaton_V1.cpp`
  found it does not: no partial-fill handling, no stop-attachment-failure
  detection, and no emergency remediation beyond ordinary signal-driven
  profit-taking exits), but because Chakra is not in the business of adding
  it either. Chakra deliberately inherits Mode Baton's existing tested
  behavior and adds no execution-safety machinery of its own** — whatever
  gaps exist in that inherited behavior are accepted as-is, not silently
  papered over by claiming they're already covered.

Builds directly on:
- `Algo_Overlay_Framework_Architecture.md`.
- `Chakra_Common_Interface_Envelope_Contract.md` (approved, Revision 8) —
  for the **input** side only (see section 1).
- `Chakra_TradePol_EntryComposer_Interface_Contract.md` (amended to
  Revision 6 alongside this document) — this publisher is that document's
  own terminal step, not a separate component.
- `Chakra_TradePol_StopManager_Interface_Contract.md` (amended to Revision
  9 alongside this document) — source of the stop-geometry fields converted
  here.
- Direct source audit of `K_Strat_Bhaskar_V1.cpp`, `K_Strat_Bhaskar_V2.cpp`,
  and `K_Util_ModeBaton_V1.cpp` (conversation record, not a separate
  document) — this contract's field map, unit table, and lifecycle rules
  are grounded in that audit, not in the superseded greenfield design.

**Central architecture decision, stated once, governing every section
below: there is no separate Entry Authority, adapter, or execution
component.** Chakra's Trade Policy pipeline terminates by writing directly
into the existing Mode Baton master-signal persistent-variable record
(`PV51`-`PV74`, on whichever chart/study slot a Mode Baton instance's
`Main Trade Chart/Study` input is configured to read). This is the exact
same record `K_Strat_Bhaskar_V1.cpp`/`V2.cpp` already write and
`K_Util_ModeBaton_V1.cpp` already reads, unmodified. Wiring a Chakra chart
in as a Mode Baton instance's configured master is configuration, not code
change, on the Mode Baton side.

------------------------------------------------------------------------

# Core responsibility

Populate `PV51`-`PV74` (the fields actually consumed by Mode Baton's
`justTakeTheTrade == 25` — "Simple Baton" — path, per source audit) with
values computed from Chakra's own decision pipeline (EntryGate's accepted
candidate + StopManager's stop geometry + this document's own new
target/invalidation-price logic), then commit by changing `PV51` last.
Nothing else. No order submission, no order tracking, no fill/rejection
handling, no post-entry management, no retry, no monitoring.

------------------------------------------------------------------------

# Non-goals — explicitly removed from the Chakra design

Carried over verbatim from the accepted audit; restated here as this
document's own binding scope boundary, not just historical context:

- A new Entry Authority component or ACSIL study.
- Any general execution adapter.
- Submission-time price-drift policy.
- Execution outcome / disposition records of any kind.
- Rejection and retry protocols.
- Fill and partial-fill tracking.
- Attached-stop confirmation.
- Emergency replacement-stop and emergency-flatten logic.
- `TrailingManager`, `ProfitManager`.
- A post-entry Chakra-to-Order-Management handoff.
- Any change to `K_Util_ModeBaton_V1.cpp` or `K_Util_Candle_Baton_Manager_V1.cpp`.
- Requiring Mode Baton to adopt any Chakra envelope, correlation, fill, or
  disposition semantics.
- Account-equity, monetary-risk, portfolio-risk, or cross-account
  coordination logic of any kind.

If a future requirement genuinely needs one of these, that is new scope
requiring its own explicit decision — it is not reopened by anything in
this document.

------------------------------------------------------------------------

# 1. Identity and scope — input side only

**On the input side** (EntryGate's accepted candidate, StopManager's stop
geometry, and this document's own new price-level computations), standard
Chakra envelope discipline applies unchanged: `ComponentKind`,
`PolicyCategory`, composite correlation identity
(chart/study/`ProducerTypeID`/sequence), the audit/delivery-record
separation, and every lifecycle-revalidation rule already established
through `Chakra_TradePol_EntryComposer_Interface_Contract.md` section 6.
Nothing about that discipline changes here.

**On the output side (`PV51`-`PV74`), there is no envelope, no
`ComponentKind`, no `ProducerTypeID`, no schema version, and no composite
identity of any kind.** This is the pre-existing legacy Mode Baton record,
byte-for-byte identical in shape to what Bhaskar already writes. Mode
Baton is never asked to read, validate, or understand anything
Chakra-specific — it only ever sees the same flat integer/float/double
fields it already reads from Bhaskar today. This is a deliberate,
non-negotiable constraint (not an oversight): **Mode Baton and Candle
Baton are preserved exactly as tested, with zero adaptation to Chakra.**

------------------------------------------------------------------------

# 2. Field map — `PV51`-`PV74`

All units, semantics, and "used by mode 25?" columns are taken directly
from the source audit of `K_Util_ModeBaton_V1.cpp`'s `justTakeTheTrade ==
25` path (both long and short branches), not assumed from field names —
several names are misleading (see notes).

| PV | Field | Type | Used by mode 25? | Chakra source | Unit / meaning |
|---|---|---|---|---|---|
| 51 | event/trade ID | int | trigger (compared via `!=`) | this document, §8 | monotonic counter, **written last** |
| 52 | direction | int | yes | EntryGate resolved side | `0`=none, `1`=long, `2`=short |
| 53 | quantity | int | **no** — confirmed unread by mode 25's sizing path | — | set `0` (§7) |
| 54 | stop distance | double | yes | StopManager | **price-unit distance**, not ticks — see §4 |
| 55 | T1 price | float | **no** — unread by mode 25 | — | set neutral (§7) |
| 56 | T2 price | float | yes | this document, §5 | **absolute price**, despite the field name |
| 57 | T3 price | float | yes | this document, §5 | absolute price |
| 58 | move-to-BE distance | int | no — Bhaskar itself never sets this either | — | set `0` (§7) |
| 59 | move-to-BE X | int | no — same as above | — | set `0` (§7) |
| 60 | move-to-BE Y | int | no — same as above | — | set `0` (§7) |
| 61 | adverse invalidation price | float | yes | this document, §6 | absolute price |
| 62 | ring-adaptive SL trigger | float | no — read but never consumed anywhere in Mode Baton | — | set neutral (§7) |
| 63 | in-direction invalidation price | float | yes | this document, §6 | absolute price |
| 64 | ring price | float | no — read but never consumed anywhere in Mode Baton | — | set neutral (§7) |
| 65 | mode selector | int | — | this document | always `25` |
| 66 | signal candle index | int | conditionally — only if the target Mode Baton instance has candle-index invalidation enabled | this document, §7 | bar index, **on this publisher's own chart** — never mapped to Mode Baton's chart (Mode Baton does that translation itself on read, per §7) |
| 71 | absolute stop reference price | float | yes | StopManager | absolute price |
| 72 | final eligible signal index | int | conditionally, same condition as PV66 | this document, §7 | bar index, same chart basis as PV66 |
| 73 | long breach price | float | yes, long only | this document, §5 | absolute price; `0` on a short signal |
| 74 | short breach price | float | yes, short only | this document, §5 | absolute price; `0` on a long signal |

Fields not in this table (67-70, 77, 80+) are Mode Baton/Bhaskar-internal
bookkeeping outside the mode-25 signal record and are never written by
this publisher.

------------------------------------------------------------------------

# 3. Stop fields — `PV54`, `PV71` (already available, unit conversion only)

No new computation. StopManager already produces both source values:

```text
PV54 = StopGeometryDistanceTicks * StopGeometryTickSize
PV71 = StopGeometryStopPrice
```

`StopGeometryStopPrice` is already risk-safe rounded to the instrument's
tick grid (floor for `LONG`, ceiling for `SHORT` — away from entry) per
`Chakra_TradePol_StopManager_Interface_Contract.md` section 7 — this
publisher does not re-round or re-derive it.

------------------------------------------------------------------------

# 4. Target prices — `PV56`, `PV57` (new logic, genuinely missing upstream)

Verified directly against `Chakra_TradePol_EntryGate_Interface_Contract.md`
and `Chakra_TriggerStr_Interface_Contract.md`: **neither document publishes
a target/take-profit price field anywhere** — both explicitly attribute
target selection to the now-removed `ProfitManager`. This publisher must
compute both fields itself; it is not wiring to something that already
exists.

Modeled directly on Bhaskar's own proven formula (both `V1`/`V2`, long and
short branches identically):

```text
targetPrice = ReferencePrice + (sign) * (TargetOffsetTicks * TickSize)
PV56 = targetPrice
PV57 = targetPrice   -- Bhaskar sets T1/T2/T3 identical; PV57 is the only
                          one of the two mode 25 actually reads as
                          Target2Price, but PV56 is still required (it is
                          read as Target1Price)
```

`ReferencePrice` is StopManager's own `StopGeometryReferencePrice` (the
same reference price the stop distance was computed from — never an
independently invented price). `TargetOffsetTicks` is a new, explicitly
configured value on this publisher (mirroring Bhaskar's own `Tgt in
Ticks` input) — not derived from any existing Chakra field, since none
exists.

------------------------------------------------------------------------

# 5. Breach prices — `PV73`, `PV74` (new logic, genuinely missing upstream)

Also absent from every upstream Chakra document (confirmed: TriggerStr
fires on setup-confirmation itself, with no separate "must cross X before
entry" concept). Mode 25 requires this field to be nonzero and on the
correct side, or its `if (timeCandleUpBreachLevelForEntry > 0 && ...)`
gate simply never opens.

```text
long signal:  PV73 = high of the signal candle (Chakra's own evaluation bar)
              PV74 = 0
short signal: PV74 = low of the signal candle
              PV73 = 0
```

**The opposite-side field is always written as `0`, explicitly, on every
publish** — never left at a stale prior value. This matters because Mode
Baton's own `if (timeCandleUpBreachLevelForEntry > 0 && ...)` and
`if (timeCandleDownBreachLevelForEntry > 0 && ...)` checks are independent
per-field gates; a stale nonzero value on the side not being signaled this
time could arm a breach watch for the wrong side using this publish's other
(possibly incompatible) direction/stop/target fields.

------------------------------------------------------------------------

# 6. Invalidation prices — `PV61`, `PV63` (new logic, genuinely missing upstream)

Also absent from every upstream Chakra document (same audit finding as
section 5: no price field of any kind exists in EntryGate's or
TriggerStr's published output). Mode 25 reads both unconditionally
whenever `justTakeTheTrade != 1` — i.e. on every mode-25 signal — as its
primary abandon-the-attempt gate (`K_Util_ModeBaton_V1.cpp:1535`,
`:2152`), checked before the breach/ring logic ever runs. Getting this
field wrong doesn't just mis-price a trade, it silently prevents the
signal from ever being actable.

**Adopted as the v1 default, exactly matching Bhaskar's own formula** —
using the signal candle's own high/low, **not** `StopGeometryReferencePrice`
(`K_Strat_Bhaskar_V1.cpp:368-369`, `:427-428`, identical in `V2`):

```text
long signal:  PV61 = (signal candle's own low)  - ATR    -- adverse
              PV63 = (signal candle's own high) + 2 * ATR -- in-direction
short signal: PV61 = (signal candle's own high) + ATR
              PV63 = (signal candle's own low)  - 2 * ATR
```

"Signal candle" is this publisher's own evaluation bar — the same bar
whose high/low feed `PV73`/`PV74` (section 5) — read directly from this
publisher's own chart, not from StopManager's `StopGeometryReferencePrice`
(which may be a different price, e.g. current market price rather than
the signal bar's own high/low; substituting one for the other was an
error in this document's own Revision 1 draft, corrected here). `ATR` is
whatever volatility input this publisher is configured to read (mirroring
Bhaskar's own `ATR Study` input). The ATR multiplier/lookback itself
remains configurable tuning, not fixed architecturally — the formula
*shape* above is the fixed v1 default.

------------------------------------------------------------------------

# 7. Candle-index expiry — `PV66`, `PV72` (populate always; documented tension)

**Decision: always populate.** Corrected from Revision 1's draft, which
wrongly had the publisher perform the cross-chart index mapping itself.
Verified directly against source: Bhaskar writes these fields using its
**own chart's raw bar index**, no mapping at all —
`tradeSignalCandleIndex = Index; tradeSignalCandleMaxIndex = Index + 4;`
(`K_Strat_Bhaskar_V1.cpp:364-365`, `:423-424`). The cross-chart
translation (`getRefIndex`, wrapping `GetNearestMatchForDateTimeIndex`) is
performed entirely on the **reading** side, inside Mode Baton itself, when
it maps its own current bar onto the configured master chart's index space
for comparison (`K_Util_ModeBaton_V1.cpp:1549`,
`timedChartIndex = getRefIndex(sc, mainTradeChartAndStudy, Index, 0)`).
**This publisher does the same as Bhaskar**: `PV66` = this publisher's own
evaluation bar index on its own chart, unmodified; `PV72` = `PV66 + N` (a
new, explicitly configured lookahead count, mirroring Bhaskar's own
hardcoded `+4`) — no `GetNearestMatchForDateTimeIndex` call, no awareness
of which chart Mode Baton itself runs on. Mode Baton already handles the
translation on its own side, exactly as it does for Bhaskar-originated
signals.

This is only load-bearing if the specific Mode Baton instance consuming
this publisher has its own `candleIndexBasedBatonInvalidationEnabled`
input turned on — a per-instance configuration fact, not discoverable from
source. Populating it regardless is cheap and harmless if that instance
ignores it, and protective if it doesn't.

**Documented tension, not resolved, flagged for visibility only:**
`Chakra_TradePol_EntryGate_Interface_Contract.md` section 11 and
`Chakra_TriggerStr_Interface_Contract.md` section 15 both explicitly state
a bar-index boundary must never be published as a primary expiry field,
timestamps only, because a bar index "has no defined meaning on a
consumer's own chart." `PV66`/`PV72` are exactly that — accepted here only
because they are Mode Baton's own pre-existing, unmodifiable field
semantics, not a new Chakra-authored interface. This publisher is
intentionally exempt from that principle for these two fields specifically;
the principle still applies to every Chakra-internal delivery record
elsewhere in the pipeline.

------------------------------------------------------------------------

# 8. Fields written neutral on every publish

Written explicitly, not left at whatever a prior publish happened to
leave behind — the same "clear intent over stale accident" reasoning as
section 5's opposite-breach-side rule:

- `PV53` (quantity) = `0` — confirmed unread by mode 25's sizing path
  (`effectPositionSizePV`/local instance input governs size, not this
  field); publishing a real number here would misleadingly imply Chakra
  controls size when it does not (see section 9 of this document and
  `Chakra_TradePol_PositionSizer_Interface_Contract.md`'s own superseded
  banner).
- `PV55` (T1 price) = `0` — unread by mode 25; Bhaskar sets it equal to
  T2/T3 as a harmless convention, but nothing depends on it.
- `PV58`-`PV60` (move-to-BE distance/X/Y) = `0` — Bhaskar's own signal
  block never assigns these either; they carry no live BE policy for mode
  25 today.
- `PV62`, `PV64` (ring-adaptive SL trigger, ring price) = `0` — read by
  Mode Baton but never consumed anywhere in its own code.

------------------------------------------------------------------------

# 9. Publishing lifecycle

**One-shot, not a heartbeat.** This is the single largest deliberate
departure from Bhaskar's own proven pattern, made because the pattern
itself is a liability, not because Bhaskar's pattern is wrong for Bhaskar:

1. Chakra's Composer stage evaluates as normal, every decision cycle.
2. **On a cycle with no genuine new signal: do nothing.** No `PV51`
   increment, no `direction = 0` write, no payload touch of any kind.
   This is the direct fix for the heartbeat-cancellation risk found in the
   source audit — Bhaskar's own unconditional per-closed-bar
   `PV51++`/`direction=0` reset was found to functionally cancel a still-
   pending mode-25 breach watch at Bhaskar's own very next bar close, in
   most cases well before the nominal multi-bar `PV72` window would ever
   matter. A one-shot publisher with no heartbeat cannot cancel its own
   pending signal this way.
3. **On a cycle with a genuine new signal:** write every payload field
   (sections 3-7 above) in full, in any order, **except `PV51`**.
4. **Commit by changing `PV51` last** — the same payload-first,
   sequence-committed-last discipline already established elsewhere in
   this framework, confirmed compatible with Mode Baton's actual read
   pattern (a plain `!=` comparison against its own last-seen value,
   re-reading the full payload only on a change — verified directly in
   `K_Util_ModeBaton_V1.cpp`).
5. **After that commit, this publisher's responsibility ends.** It does
   not re-publish, refresh, retry, poll, or monitor the signal it just
   sent. Mode Baton owns everything from this point: arming
   `seekingTrade`, the invalidation-price gate, the candle-index expiry
   gate, the breach watch, the ring/HHHL structural check, order
   submission, attached stops/targets, and every post-entry concern
   handed to Candle Baton.
6. **No cancellation event exists.** A concrete requirement would need to
   exist before one is designed — see "Unresolved design questions."

------------------------------------------------------------------------

# 10. Recalculation and durability — resolved by an operational invariant, not by new mechanism

**Corrected in this pass: the prior draft first overclaimed durability,
then flagged the resulting collision risk as unresolved. Neither stands —
this is resolved by a v1 operational invariant, not by inventing
persistence guarantees ACSIL doesn't provide or by adding new machinery
(no timestamp-seeded IDs, no durable cross-reconstruction storage, no
feedback channel from Mode Baton back to this publisher, and no change to
Mode Baton's own code).**

**What is reasonably reliable, unchanged from the prior analysis**:
`sc.GetPersistentInt`/`Float`/`Double` values are tied to a specific study
*instance's* own storage slot. An ordinary `IsFullRecalculation` pass on
that same, still-attached instance does not itself zero persistent
variables — only code that explicitly writes `0` does. **This publisher's
own code never explicitly resets `PV51` on `IsFullRecalculation`.**

**What is genuinely not guaranteed**: study removal/re-add, chartbook
reconstruction, or other platform-level actions can create a fresh
instance with zeroed persistent storage, indistinguishable from inside the
study's own code from a legitimate first run. No ACSIL-level mechanism
lets a study detect this boundary or recover a prior counter value across
it. This remains true — it is not solved by anything below.

**The v1 operational invariant that closes the resulting risk**: **if this
publisher's instance is removed, re-added, reconstructed, or rewired, its
consuming Mode Baton instance must also be recalculated/reloaded before
Chakra signal publishing is enabled again.** This is a deployment/operator
discipline, not a runtime mechanism this document builds:

1. **Ordinary full recalculation does not explicitly reset `PV51`** (above)
   — the common case (a routine historical-data refresh on an
   already-attached instance) needs no operator action at all.
2. **No signal publishes during historical/full recalculation** (section
   9's own recalc-entry rule, unchanged) — recalculation, whatever else
   happens to persistent storage, never itself commits a live `PV51`
   change.
3. **A fresh or reconstructed publisher instance starts disabled or
   unarmed** — it does not resume mid-signal or replay a historical
   decision; it waits for a genuinely new live evaluation cycle before
   ever considering a publish, same as every other Chakra event-pattern
   producer's own attachment/recalc-entry baselining.
4. **Operator/chartbook initialization is responsible for resynchronizing
   Mode Baton whenever the publisher instance is reconstructed** — the
   same discipline already required any time a chart/study wiring changes
   in this codebase (e.g. Mode Baton's own `mainTradeChartAndStudy`
   re-pointing). Recalculating/reloading Mode Baton's own instance
   re-baselines its `currentBatonTradeId` tracker (`K_Util_ModeBaton_V1.cpp`'s
   own `IsFullRecalculation` reset, source-audited) to whatever the
   publisher's (now also fresh) `PV51` currently holds — both sides start
   from a consistent point, eliminating the collision window described in
   the prior draft.
5. **The first subsequent live signal still publishes payload-first,
   `PV51`-last** — section 9's commit discipline is unaffected by any of
   the above; resynchronization changes when publishing is safe to resume,
   not how a commit is structured once it does.
6. **Independent publisher reconstruction without a matching Mode Baton
   resynchronization is unsupported configuration, not a runtime case this
   document solves.** If an operator reconstructs the publisher's instance
   and does not also recalculate/reload the Mode Baton instance consuming
   it, the small collision risk described in the prior draft's analysis
   remains real for that specific deployment mistake — this document
   documents the requirement rather than defending against violating it in
   code, the same posture this framework already takes toward other
   manual-wiring invariants (e.g. "one Composer instance is wired to
   exactly one EntryGate source" is enforced by configuration discipline,
   not runtime detection).

------------------------------------------------------------------------

# 11. Worked examples

**Example 1 — long signal, ordinary case.**
EntryGate accepts a long candidate; the evaluation bar has `high=4502.00`,
`low=4498.50`; StopManager publishes `StopGeometryReferencePrice=4500.00`,
`StopGeometryStopPrice=4497.00`, `StopGeometryDistanceTicks=12`,
`StopGeometryTickSize=0.25`. This publisher computes: `PV54 = 12*0.25 =
3.00`, `PV71 = 4497.00`, `PV56=PV57 = 4500.00 + (20*0.25) = 4505.00`
(20-tick target offset, configured), and — with `ATR=3.00` at evaluation
time — `PV61 = 4498.50 - 3.00 = 4495.50` and `PV63 = 4502.00 + 2*3.00 =
4508.00` (section 6's formula, using the evaluation bar's own low/high,
not `StopGeometryReferencePrice`), `PV73 = 4502.00` (the same bar's own
high), `PV74 = 0`, `PV65 = 25`, `PV66` = this publisher's own evaluation
bar index (its own chart, no cross-chart mapping — section 7), `PV72 =
PV66 + 4`.
`PV52 = 1`, `PV53 = 0`, `PV55/58/59/60/62/64 = 0`. All of the above
committed; `PV51` incremented last. Mode Baton picks it up on its own next
closed bar, arms `seekingTrade`, and watches for `currentHigh > PV73`
exactly as it does for a Bhaskar-originated signal.

**Example 2 — no signal this cycle.** Composer evaluates, finds nothing
to propose. No field is touched, `PV51` is not incremented. A Mode Baton
instance still watching a still-valid prior signal continues watching it
undisturbed — the exact failure mode Bhaskar's heartbeat was found to
cause is structurally impossible here.

**Example 3 — publisher undergoes an ordinary recalculation mid-session.**
This publisher's chart runs `IsFullRecalculation` (e.g. a routine
historical-data refresh) while a Mode Baton instance is running live with
`currentBatonTradeId = 118`. This publisher's own code never explicitly
zeroes `PV51`, so its persistent value survives this pass unchanged (still
`118` on the same study instance). Mode Baton's own comparison (`118 !=
118`) sees no change and takes no action. The next genuine signal after
the recalc publishes with `PV51 = 119`, picked up normally.

**Example 4 — publisher's study is removed and re-added (not merely
recalculated).** A materially different case from example 3, per section
10's own honest accounting: the platform creates a fresh study instance
with `PV51` reset to `0` by the platform itself, not by any choice this
document's code makes. **Per section 10's operational invariant, this is a
supported case only if Mode Baton's own instance is also recalculated/
reloaded before publishing resumes** — doing so re-baselines Mode Baton's
`currentBatonTradeId` to the publisher's own (now also fresh) `PV51`
value, so both sides start consistent and the counter never needs to
"count back up through" any prior value at all. If an operator
reconstructs the publisher's chart **without** also resynchronizing Mode
Baton (the unsupported configuration section 10 names explicitly), Mode
Baton's `currentBatonTradeId` would remain at its prior value (e.g.
`118`) while the publisher's counter restarts from `0` — the collision
risk described in earlier drafts of this document applies to exactly that
unsupported case, not to the documented, supported reconstruction
procedure.

------------------------------------------------------------------------

# Unresolved design questions — deliberately left open

1. **Whether a cancellation event is ever needed.** No concrete
   requirement exists today (section 8, item 6). If one emerges, it would
   need its own design: `direction=0` published deliberately, with the
   opposite-side-neutral and payload-first/`PV51`-last discipline above
   still applying, and a decision on whether Mode Baton's existing
   `direction==0 -> seekingTrade=0` handling (already proven, unmodified)
   is sufficient or whether something more is required. Not designed here
   because nothing today requires it.
2. **The `PV66`/`PV72` bar-index-vs-timestamp tension (section 7).**
   Accepted as a deliberate, scoped exception to this framework's own
   timestamp-only expiry principle, because Mode Baton's fields are
   pre-existing and unmodifiable — flagged so a future reviewer doesn't
   mistake this for an oversight or try to "fix" it by switching to a
   timestamp Mode Baton doesn't read.
3. **The adverse/in-direction invalidation-price formula — resolved,
   adopted as the v1 default, not left unresolved.** Section 6 above
   specifies Bhaskar's own exact formula, verified against source
   (`PV61`/`PV63` from the signal candle's own low/high, never
   `StopGeometryReferencePrice`). Only the `ATR` lookback/multiplier
   remain configurable tuning, same as `TargetOffsetTicks` in section 4 —
   the formula *shape* itself is fixed, not an open question.
4. **`PV51` durability across study reconstruction — resolved by
   operational invariant, not by new mechanism (section 10).** Study
   removal/re-add or other platform-level instance reconstruction still
   resets persistent storage in a way no ACSIL-level code can detect or
   prevent — that fact hasn't changed. What closes the resulting risk is a
   documented deployment requirement, not code: reconstructing the
   publisher's instance requires also recalculating/reloading the
   consuming Mode Baton instance before publishing resumes, which
   re-baselines Mode Baton's own tracker to match. Independent publisher
   reconstruction without that resynchronization is explicitly unsupported
   configuration, not a case this document's code defends against. This is
   not left open — it is closed by an operator/deployment discipline this
   document states directly, the same posture this framework already
   takes toward other manual-wiring invariants it doesn't runtime-enforce.
