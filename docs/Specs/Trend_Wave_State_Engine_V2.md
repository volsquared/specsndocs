# Trend Wave State Engine (V2)

## Purpose

The Trend Wave State Engine classifies the lifecycle of a directional
trend using moving-average geometry. It is a **market context state
machine**, not a trading strategy. It publishes context for downstream
Trigger components.

------------------------------------------------------------------------

# Core Principles

-   Uses **closed bars only**.
-   Separates **early continuation opportunity** from **confirmed
    continuation**.
-   Maintains persistent market context.
-   Never generates trade signals.
-   Is independent of strategy-specific rules.

------------------------------------------------------------------------

# Concepts

## Direction

Direction is an attribute, not a state.

-   NONE
-   UP
-   DOWN

## State

The engine has six behavioural states:

-   NEUTRAL
-   PENDING_CONFIRMATION
-   CONFIRMED
-   UNCERTAINTY
-   RESUME_ATTEMPT
-   IN_CHOP

A state together with Direction defines the complete market context.

Examples:

-   CONFIRMED + UP
-   UNCERTAINTY + DOWN
-   IN_CHOP + NONE

------------------------------------------------------------------------

# Inputs

-   Candle High
-   Candle Low
-   Candle Close
-   Fast MA
-   Mid MA
-   Slow MA
-   ATR
-   N (ATR-normalised confirmation threshold, N \> 0)
-   Z (Failed resume threshold before chop, Z \>= 1)

Derived each closed bar:

HighestMA = max(FastMA, MidMA, SlowMA)

LowestMA = min(FastMA, MidMA, SlowMA)

FastSlowDistanceATR = abs(FastMA - SlowMA) / ATR

------------------------------------------------------------------------

# Behaviour

## 1. Neutral

Wait for an observed Fast/Slow crossover.

Bullish crossover: PreviousFast \<= PreviousSlow CurrentFast \>
CurrentSlow

Direction = UP State = PENDING_CONFIRMATION

Bearish crossover:

PreviousFast \>= PreviousSlow CurrentFast \< CurrentSlow

Direction = DOWN State = PENDING_CONFIRMATION

------------------------------------------------------------------------

## 2. Pending Confirmation

Remain pending until:

FastSlowDistanceATR \>= N

State = CONFIRMED

On first confirmation:

WaveCount = 1 PullbackCount = 0 ResumeAttemptCount = 0 FailedResumeCount
= 0

------------------------------------------------------------------------

## 3. Confirmed

A pullback begins when:

UP: Low \<= SlowMA

DOWN: High \>= SlowMA

Transition:

CONFIRMED -\> UNCERTAINTY

PullbackCount++

------------------------------------------------------------------------

## 4. Uncertainty

The prior trend is unresolved.

Resume attempt:

UP: Close \> HighestMA

DOWN: Close \< LowestMA

Transition:

UNCERTAINTY -\> RESUME_ATTEMPT

ResumeAttemptCount++

------------------------------------------------------------------------

## 5. Resume Attempt

Confirmation:

FastSlowDistanceATR \>= N

Transition:

RESUME_ATTEMPT -\> CONFIRMED

WaveCount++ FailedResumeCount = 0

Failure:

UP: Close \< HighestMA

DOWN: Close \> LowestMA

NewFailedResumeCount = FailedResumeCount + 1

If:

NewFailedResumeCount \>= Z

Transition directly to:

IN_CHOP

Else:

Transition to:

UNCERTAINTY

FailedResumeCount = NewFailedResumeCount

Equality with HighestMA/LowestMA leaves the state unchanged.

------------------------------------------------------------------------

## 6. In Chop

Direction = NONE

The previous directional cycle is no longer authoritative.

Retain FailedResumeCount and diagnostic indices.

Wait for a completely fresh crossover.

Bullish crossover:

Direction = UP State = PENDING_CONFIRMATION

Bearish crossover:

Direction = DOWN State = PENDING_CONFIRMATION

Reset counters when the new directional cycle begins.

------------------------------------------------------------------------

# Counters

WaveCount
:   Number of confirmed legs in the current directional cycle.

PullbackCount
:   Number of distinct pullback episodes.

ResumeAttemptCount
:   Number of transitions from Uncertainty to Resume Attempt.

FailedResumeCount
:   Number of resume attempts that failed.

------------------------------------------------------------------------

# Invariants

-   Maximum one state transition per closed bar.
-   Opposite crossover always starts a new pending-confirmation cycle.
-   A crossover never confirms a trend by itself.
-   Confirmation always requires FastSlowDistanceATR \>= N.
-   The engine never emits trading signals.
-   Configuration changes affecting N or Z require full historical
    reconstruction.

------------------------------------------------------------------------

# Architectural Responsibility

## Sensor

Publishes raw measurements.

Examples:

-   Fast MA
-   Slow MA
-   HighestMA
-   LowestMA
-   FastSlowDistanceATR

No market state.

## Market Context (MCtx)

Owns this Trend Wave State Engine.

Publishes:

-   Direction
-   State
-   WaveCount
-   PullbackCount
-   ResumeAttemptCount
-   FailedResumeCount

## Trigger

Consumes MCtx and Sensor outputs to decide whether a market context is
tradable.

The Trigger owns all entry rules. The Trend Wave State Engine remains
purely descriptive.
