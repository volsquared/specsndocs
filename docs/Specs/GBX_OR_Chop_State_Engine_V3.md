# GBX OR Chop State Engine (V3)

## Goal

Build a reusable Sierra Chart ACSIL **Market State Engine**.

This study does **not** decide when to trade.

Its only responsibility is to determine the current chop state around the GBX Opening Range (OR).

---

# Philosophy

The engine owns **market state** only.

It does **not** know about:

- Tatva breakout logic
- Delta confirmation
- Candle body %
- Entries
- Exits
- Position management

Those studies consume the state produced by this engine.

---

# Inputs

## Opening Range
- OR Study ID
- OR High Subgraph
- OR Low Subgraph

## EMA
- EMA Study ID
- EMA Subgraph

## ATR
- ATR Study ID
- ATR Subgraph

---

# Parameters

## Initial Chop Detection

- EMAInsideBarsRequired = X
- DisplacementLookback = X
- DisplacementATRMultiplier = Y

## Chop Exit Confirmation

- EMAOutsideBarsRequired = Z

## Failed Chop Exit

- EMAInsideBarsAfterFailure = M
- FailureDisplacementLookback = M
- FailureDisplacementATRMultiplier = Y

Typically M < X.

X establishes a brand-new chop.

M detects a return to an already-established chop after a failed exit.

---

# State Machine

0 = UNKNOWN

1 = CHOP_INSIDE_CONFIRMED

2 = CHOP_EXIT_PROBE

3 = CHOP_EXIT_CONFIRMED

---

# State Definitions

## UNKNOWN

Default state.

Transition:

- EMA inside OR for X consecutive bars
- Displacement over X bars <= ATR × Y

→ CHOP_INSIDE_CONFIRMED

---

## CHOP_INSIDE_CONFIRMED

Meaning:

The market is accepted inside the Opening Range.

Transition:

EMA closes outside the OR

→ CHOP_EXIT_PROBE

No other condition is required.

---

## CHOP_EXIT_PROBE

Meaning:

The market may be leaving the confirmed OR chop.

### Success

EMA remains outside the OR for Z consecutive bars

→ CHOP_EXIT_CONFIRMED

### Failure

EMA returns inside the OR for M consecutive bars

AND

Displacement over M bars <= ATR × Y

→ CHOP_INSIDE_CONFIRMED

---

## CHOP_EXIT_CONFIRMED

Meaning:

The previously confirmed OR chop has been exited.

This study deliberately makes no assumption about what happens next.

It does not imply:

- Trend
- Breakout
- Continuation
- Reversal

Only that the original inside-OR chop is no longer valid.

A fresh chop is established only when:

- EMA inside OR for X bars
- Displacement <= ATR × Y

→ CHOP_INSIDE_CONFIRMED

---

# Outputs

## Visible Markers

SG1 - CHOP_INSIDE_CONFIRMED

SG2 - CHOP_EXIT_PROBE

SG3 - CHOP_EXIT_CONFIRMED

Plot markers only on transition bars.

---

## Diagnostic Outputs

- StateCode
- EMAInsideCounter
- EMAOutsideCounter
- CurrentDisplacement

---

# Design Principle

This study publishes **market state only**.

Trade studies (Tatva, Delta, Baton or future strategies) consume the state externally.

The state engine must remain independent of trade logic.
