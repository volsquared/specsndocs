# Candles Pattern Setups V1 — Setup A

Status: **authoritative implementation specification**
Implementation: `ACS_Source/K_Candles_Inner_Breach_Displacement_V1.cpp`

## 1. Scope and purpose

This study is the home for multiple objective candle-pattern setups. Revision 1
implements only **Setup A**: a confirmed inner-candle range whose latest
body-qualified directional breach remains active until frozen-ATR displacement
or a valid boundary replacement.

Setup A does not permit entries, inspect market context, place trades, or
manage positions. Future setups must receive separate authoritative rules and
Subgraphs before being added.

## 2. Evaluation cadence

- Use a manual loop and completed bars only.
- Put `BHCS_BAR_HAS_NOT_CLOSED` at the top of the loop.
- Process each completed bar once and reconstruct from bar 1 on full
  recalculation.
- Use chart OHLC and a configured same-chart ATR Study/Subgraph.

## 3. Inputs

| Input | Default | Rule |
|---|---:|---|
| Enabled | Yes | Disable processing and clear private state when No. |
| ATR Study ID | 0 | Same-chart ATR dependency. |
| ATR Subgraph | 0 | ATR output SG. |
| Setup A Minimum Breach Body Percent | 40.0 | Inclusive; valid range `(0, 100]`. |
| Setup A Displacement ATR Multiple | 2.0 | Must be positive. |
| Marker Offset In Ticks | 16 | Non-negative display offset. |

## 4. Definitions

For Mother `M`, Inner `I`, and following confirmation candle `C`:

```text
I.High <= M.High
I.Low  >= M.Low
```

At least one inequality must be strict. Mother and Inner OHLC must be finite
with `High > Low`.

`C` confirms when its completed close is inclusively inside the Mother range:

```text
M.Low <= C.Close <= M.High
```

`C` need not itself be inner. If it closes outside the Mother range, the
candidate fails. The body-percent, ATR, inner-boundary breach, and displacement
rules do not apply to `C`; breach evaluation starts on later completed bars.

## 5. Primary states

- `IDLE`: no confirmed Setup A boundaries.
- `PENDING_INITIAL_CONFIRMATION`: Mother and Inner captured; the immediately
  following candle must confirm inside the Mother.
- `ARMED`: confirmed boundaries exist and no qualified breach is active.
- `ACTIVE_LONG`: latest qualified breach is above the boundaries.
- `ACTIVE_SHORT`: latest qualified breach is below the boundaries.

While ACTIVE, one orthogonal pending-replacement candidate may retain a new
Mother/Inner pair without changing the current active direction, boundaries,
frozen ATR, or target.

## 6. Initial arming

In `IDLE`, an Inner starts `PENDING_INITIAL_CONFIRMATION`. Store the Mother
High/Low and Inner High/Low and mark the Inner candle.

On the immediately following completed candle:

- confirming close inside the stored Mother installs the stored Inner
  boundaries, enters `ARMED`, and marks the confirmation candle;
- close outside the Mother, or invalid current OHLC, discards the candidate and
  returns to `IDLE`.

The confirmation candle cannot also create a breach.

## 7. Boundary replacement

### 7.1 Before a qualified breach (`ARMED`)

Any newer Inner relative to its immediately preceding Mother replaces the
breach boundaries immediately. State remains `ARMED`; mark that Inner as
`Setup A Re-Armed`. No confirmation candle is required because Setup A is
already armed and no displacement attempt exists.

### 7.2 After a qualified breach (`ACTIVE_LONG` or `ACTIVE_SHORT`)

A newer Inner starts a pending replacement but does not change the existing
boundaries, direction, frozen ATR, or target. Mark the candidate Inner.

On the immediately following candle:

- if its close is inside the candidate Mother range, install the candidate
  Inner boundaries, clear the old direction/ATR/target, enter `ARMED`, and mark
  the confirmation candle as `Setup A Re-Armed`;
- otherwise discard only the replacement candidate; the old active attempt
  remains authoritative and the same candle may still be evaluated against the
  old boundaries.

An ACTIVE bar may both create a qualified directional breach against the old
boundaries and start a pending replacement candidate. The replacement is not
effective unless its following candle confirms.

## 8. Qualified breach

For a later breach candle `B`:

```text
BodyPercent = abs(B.Close - B.Open) / (B.High - B.Low) * 100
```

The current breach bar must have finite OHLC and positive range.

```text
LONG:  B.Close > BoundaryHigh and BodyPercent >= configured minimum
SHORT: B.Close < BoundaryLow  and BodyPercent >= configured minimum
```

A wick-only excursion does not qualify. ATR at `B` must be finite and positive.
Freeze it and set:

```text
LONG target  = BoundaryHigh + configured multiple * FrozenATR
SHORT target = BoundaryLow  - configured multiple * FrozenATR
```

Repeated same-direction closes are continuation and do not republish or move
the target. A qualified opposite breach wins, freezes ATR anew, replaces the
target, and marks the new breach. Returning inside the boundaries does not
invalidate an active attempt.

## 9. Displacement and precedence

Evaluate displacement from current High/Low independently of previous-bar
validity:

```text
ACTIVE_LONG:  High >= LongTarget
ACTIVE_SHORT: Low  <= ShortTarget
```

On completion, mark displacement, clear all Setup A and pending-replacement
state, and enter `IDLE`. Do not reuse that bar for another transition.

Per-bar precedence:

1. active-direction displacement;
2. pending initial/replacement confirmation;
3. immediate ARMED-state inner replacement;
4. qualified breach against current authoritative boundaries;
5. ACTIVE-state capture of a new pending-replacement Inner;
6. no transition.

Previous-bar validity is required only for detecting an Inner. A degenerate
previous bar must not suppress current-bar displacement or breach evaluation.

## 10. Subgraphs

Populate event markers only on transition/candidate bars and zero otherwise:

| SG | Name | Candle marked | Placement |
|---:|---|---|---|
| 0 | Setup A Candidate Inner | Initial Inner awaiting confirmation | Below |
| 1 | Setup A Replacement Inner | ACTIVE-state Inner awaiting confirmation | Below |
| 2 | Setup A Armed | Initial confirmation candle | Below |
| 3 | Setup A Re-Armed | Immediate ARMED replacement Inner or confirmed ACTIVE replacement candle | Below |
| 4 | Setup A Long Breach | Qualified LONG breach | Below |
| 5 | Setup A Short Breach | Qualified SHORT breach | Above |
| 6 | Setup A Long Displaced | LONG displacement | Below |
| 7 | Setup A Short Displaced | SHORT displacement | Above |

Below is `Low - MarkerOffsetTicks * TickSize`; above is
`High + MarkerOffsetTicks * TickSize`.

## 11. ATR and lifecycle

- Never fetch ATR on a forming-bar-only call.
- Fetch ATR at most once per call and only for a breach candidate.
- If the ATR array is unavailable or lacks the candidate index, do not consume
  that bar; retry it later so chronological state is preserved.
- A present nonfinite/non-positive ATR makes that breach ineligible; consume
  the bar and continue.
- Full recalculation resets every private field and rebuilds completed history.
- Disabled state clears all private Setup A state.

## 12. Acceptance cases

1. Mother + Inner alone does not arm; confirming close inside Mother arms on C.
2. C outside Mother discards the initial candidate without breach/ATR logic.
3. While ARMED, a new Inner immediately replaces boundaries.
4. While ACTIVE, a new Inner alone does not replace boundaries.
5. ACTIVE replacement confirms only through the following close inside its
   Mother; failure preserves the old active attempt.
6. Body-qualified breach freezes ATR; same-side continuation does not move the
   target; qualified opposite breach wins.
7. Pullback inside boundaries leaves the active attempt alive.
8. Displacement wins before replacement/inner/breach processing.
9. A degenerate previous bar cannot suppress a valid current displacement or
   breach.
