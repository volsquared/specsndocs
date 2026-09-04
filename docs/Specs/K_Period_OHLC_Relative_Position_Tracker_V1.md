# K_Period_OHLC_Relative_Position_Tracker_V1

## Functional Specification Document

### Revision Note
- This revision reflects the implemented study behavior as of 2026-04-04.
- The study is now PV-first and banner-driven. Analytical subgraph plotting is no longer part of the operational design.

## 1. Purpose
- Track interaction of lower timeframe bars (for example, 1-minute bars) relative to prior higher timeframe Period OHLC values provided by a Sierra Chart Period OHLC study.
- Expose the resulting state through persistent variables for use by other studies.
- Display a compact status banner that updates only when its displayed values change.

## 2. Core Logic
- Evaluate only on closed bars.
- Use manual looping only: `sc.AutoLoop = 0`.
- Do not simulate Period OHLC behavior internally.
- Treat a change in referenced HTF OHLC values as a new higher timeframe period and reset per-period breach state.
- Study graph region is `0`.

## 3. Reference Values
- `PO = Period Open`
- `PH = Period High`
- `PL = Period Low`
- `PC = Period Close`
- `UpperInner = max(PO, PC)`
- `LowerInner = min(PO, PC)`
- `Mid50 = PL + 0.5 * (PH - PL)`

## 4. Zone Classification
- Zone is determined from the lower timeframe close relative to `PH`, `UpperInner`, `LowerInner`, and `PL`.
- Encoded values are:
- `10 = ABOVE_HIGH`
- `1 = L1`
- `2 = L2`
- `3 = L3`
- `-10 = BELOW_LOW`

### Zone Layout
- `L1 = High-side outer band = PH -> UpperInner`
- `L2 = Inner body band = UpperInner -> LowerInner`
- `L3 = Low-side outer band = LowerInner -> PL`

## 5. Midpoint State
- `-1 = BELOW_50`
- `0 = AT_50`
- `1 = ABOVE_50`

## 6. Width Calculations
- `L1 = PH - UpperInner`
- `L2 = UpperInner - LowerInner`
- `L3 = LowerInner - PL`

### Percent Width Display
- The banner shows the relative split of the HTF range as `L1:L2:L3`.
- This corresponds to `top:middle:bottom`.

## 7. Breach Logic
- High breach per bar: `BarHigh > PH`
- Low breach per bar: `BarLow < PL`

## 8. Per-Period Breach State
- `1 = INSIDE_ONLY`
- `2 = HIGH_BREACHED_ONLY`
- `3 = LOW_BREACHED_ONLY`
- `4 = BOTH_SIDES_BREACHED`

## 9. Engulfing Logic
- `1 = Bullish` if `close > PH`
- `-1 = Bearish` if `close < PL`
- `0 = Non-directional` otherwise

## 10. Reset Conditions
- Reset all per-period breach tracking state when referenced Period OHLC values change.

## 11. Inputs
- Banner Horizontal Position
- Banner Vertical Position
- Period Open Reference
- Period High Reference
- Period Low Reference
- Period Close Reference
- Enable Debug Logging

### Default Reference Subgraphs
- Open reference subgraph defaults to `0`
- High reference subgraph defaults to `1`
- Low reference subgraph defaults to `2`
- Close reference subgraph defaults to `3`

## 12. Banner
- Banner is drawn in chart region `0`.
- Banner updates only when any displayed value changes.
- Banner content format is:
- `L1%:L2%:L3%:HighBreach:LowBreach:Zone:ClosePosition`

### Banner Fields
- `L1%`, `L2%`, `L3%` are percent widths of the HTF range.
- `HighBreach` is cumulative per-period high-breach state shown as `1/0`.
- `LowBreach` is cumulative per-period low-breach state shown as `1/0`.
- `Zone` uses the implemented encoding `10 / 1 / 2 / 3 / -10`.
- `ClosePosition` is the normalized HTF close location metric described below.

## 13. HTF Close Position Metric
- Store where the current LTF close sits inside the HTF range assuming:
- `0 = PL`
- `100 = PH`
- If close is above `PH`, store `101`
- If close is below `PL`, store `-101`
- Saturation is applied regardless of how far outside the range the close is.

## 14. Nearest-Level Distance Metrics
- Store the actual distance in ticks from the current close to the two nearest structural levels.
- For closes above `PH`:
- both `d1` and `d2` equal `abs(close - PH)` in ticks
- For closes below `PL`:
- both `d1` and `d2` equal `abs(close - PL)` in ticks
- For closes inside the HTF range:
- `d1` and `d2` are the distances to the two boundaries of the containing zone

## 15. Width Metrics In Ticks
- Store each level width in ticks:
- `L1 width ticks`
- `L2 width ticks`
- `L3 width ticks`

## 16. Persistent Variable Outputs

### Integer PVs
- `Int 20 = Zone Classification`
- `Int 21 = Midpoint State`
- `Int 22 = Current Bar High Breach`
- `Int 23 = Current Bar Low Breach`
- `Int 24 = Per-Period Breach State`
- `Int 25 = Engulf State`
- `Int 26 = Valid Period Flag`
- `Int 27 = Period Reset Flag`

### Float PVs
- `Float 20 = L1 Width`
- `Float 21 = L2 Width`
- `Float 22 = L3 Width`
- `Float 23 = Period Open`
- `Float 24 = Period High`
- `Float 25 = Period Low`
- `Float 26 = Period Close`
- `Float 27 = UpperInner`
- `Float 28 = LowerInner`
- `Float 29 = Mid50`
- `Float 30 = HTF Close Position`
- `Float 31 = Distance To Nearest Level 1 In Ticks`
- `Float 32 = Distance To Nearest Level 2 In Ticks`
- `Float 33 = L1 Width In Ticks`
- `Float 34 = L2 Width In Ticks`
- `Float 35 = L3 Width In Ticks`

## 17. Logging
- When `Enable Debug Logging = Yes`, log the lower timeframe bar timestamp and the current PV values for the processed closed bar.

## 18. Edge Cases
- If `PH <= PL`, treat the period as invalid.
- Invalid periods zero the float metric PVs and set state PVs to neutral or invalid values as appropriate.
