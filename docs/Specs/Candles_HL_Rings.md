Yes. Here’s the MD spec.

# MA Breach Swing Ring Marker

## Purpose

Create a candle-chart swing marker utility that retrospectively pins the last confirmed structural high or low when price breaches a selected Moving Average by a configurable percentage of the candle’s total range.

This is not an entry signal.
It is a visual structure marker for confirmed highs and lows.

---

## Core Concept

The study tracks which side of the selected Moving Average price is operating on.

When price is above the MA, the study continuously tracks the highest high.

When a candle breaches below the MA by at least the configured percentage of its full range, the prior tracked high is confirmed and marked with a ring.

When price is below the MA, the study continuously tracks the lowest low.

When a candle breaches above the MA by at least the configured percentage of its full range, the prior tracked low is confirmed and marked with a ring.

---

## Inputs

### 1. Moving Average Study/Input

User-selectable Moving Average used as the structural reference.

Examples:

* 21 EMA
* 34 EMA
* 55 EMA
* 89 EMA
* Any Sierra-supported MA source

The MA should be configurable as an input parameter.

---

### 2. Breach Percentage

Percentage of the candle’s full range that must be on the opposite side of the MA before a transition is confirmed.

Example:

```text
Breach Percentage = 50
```

Means at least 50% of the candle’s total High-Low range must be beyond the MA.

---

## Candle Range Calculations

```text
Candle Range = High - Low
```

If `Candle Range <= 0`, ignore the bar.

---

## Percentage Below MA

Used when current state is ABOVE_MA.

```text
Range Below MA = MA - Low
```

Clamp result:

```text
Range Below MA = max(0, min(MA - Low, Candle Range))
```

Then:

```text
Percentage Below MA = Range Below MA / Candle Range * 100
```

If:

```text
Percentage Below MA >= Breach Percentage
```

Then the candle confirms transition from ABOVE_MA to BELOW_MA.

---

## Percentage Above MA

Used when current state is BELOW_MA.

```text
Range Above MA = High - MA
```

Clamp result:

```text
Range Above MA = max(0, min(High - MA, Candle Range))
```

Then:

```text
Percentage Above MA = Range Above MA / Candle Range * 100
```

If:

```text
Percentage Above MA >= Breach Percentage
```

Then the candle confirms transition from BELOW_MA to ABOVE_MA.

---

## State Machine

### State: ABOVE_MA

While in ABOVE_MA state:

* Track the highest high since the last confirmed low ring.
* Store:

  * highest high value
  * bar index where highest high occurred

Transition condition:

```text
Percentage Below MA >= Breach Percentage
```

On transition:

* Plot high ring at tracked highest high bar.
* Clear/reset high tracking.
* Switch state to BELOW_MA.
* Begin tracking lows from current bar.

---

### State: BELOW_MA

While in BELOW_MA state:

* Track the lowest low since the last confirmed high ring.
* Store:

  * lowest low value
  * bar index where lowest low occurred

Transition condition:

```text
Percentage Above MA >= Breach Percentage
```

On transition:

* Plot low ring at tracked lowest low bar.
* Clear/reset low tracking.
* Switch state to ABOVE_MA.
* Begin tracking highs from current bar.

---

## Subgraphs

### Subgraph 1: High Rings

Used to plot confirmed structural highs.

```text
Name: High Rings
Draw Style: Ring / Point / Circle
Value: High price at confirmed high bar
```

Only populated at confirmed high-ring bars.
All other bars should be zero or ignored depending on Sierra Chart draw requirements.

---

### Subgraph 2: Low Rings

Used to plot confirmed structural lows.

```text
Name: Low Rings
Draw Style: Ring / Point / Circle
Value: Low price at confirmed low bar
```

Only populated at confirmed low-ring bars.
All other bars should be zero or ignored depending on Sierra Chart draw requirements.

---

## Initialization Logic

On first valid bar where MA is available:

```text
If Close >= MA:
    State = ABOVE_MA
    Start tracking highs

Else:
    State = BELOW_MA
    Start tracking lows
```

No rings should be plotted until the first valid state transition occurs.

---

## Output Behaviour

The study should back-plot rings.

Example:

Price is above MA and makes a high at bar 120.

Price then pulls back.

At bar 126, at least 50% of the candle range is below the MA.

The study plots a high ring back at bar 120, not bar 126.

Same logic applies inversely for low rings.

---

## Important Notes

This marker is retrospective by design.

The ring does not mean:

```text
Enter trade here
```

It means:

```text
This was the last confirmed structural extreme before price transitioned across the MA.
```

Recommended naming:

```text
MA-Confirmed High Ring
MA-Confirmed Low Ring
```

Avoid calling it a strict fractal swing high/low because it is not based on left/right candle fractal logic.

---

## Minimal Required Inputs

```text
Input 1: Moving Average Source / Type / Length
Input 2: Breach Percentage
```

Optional later inputs:

```text
Minimum Bars Between Rings
Minimum Candle Range as ATR Fraction
Enable/Disable High Rings
Enable/Disable Low Rings
Ring Offset In Ticks
```

---

## Default Suggested Values

```text
Moving Average: 34 EMA or 55 EMA
Breach Percentage: 50
```

For faster marking:

```text
MA: 21 EMA
Breach Percentage: 40-50
```

For cleaner but later marking:

```text
MA: 55 EMA or 89 EMA
Breach Percentage: 50-60
```
