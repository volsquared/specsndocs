# MA Trend Wave State Engine -- Design Note (Draft)

## Objective

Model trend continuation using moving-average geometry rather than price
swings.

The engine distinguishes between:

-   Confirmed trend
-   Pullback / uncertainty
-   Resume attempt
-   Confirmed continuation

A crossover alone **does not** confirm a new trend. Confirmation
requires sufficient separation between the Fast and Slow moving
averages.

------------------------------------------------------------------------

# Inputs

-   Fast MA
-   Mid MA
-   Slow MA
-   ATR
-   Separation Threshold `N` (ATR multiple)

------------------------------------------------------------------------

# Definitions

## Highest MA

For every closed bar:

``` text
HighestMA = max(FastMA, MidMA, SlowMA)
```

This value is recalculated every bar.

------------------------------------------------------------------------

## Trend Confirmation

A trend is confirmed only when:

``` text
|FastMA - SlowMA| / ATR >= N
```

Direction is determined by Fast MA relative to Slow MA.

------------------------------------------------------------------------

# States

## 1. UP_CONFIRMED

An established upward trend.

Metadata:

-   Wave Count
-   Resume Attempt Count

------------------------------------------------------------------------

## 2. UP_UNCERTAINTY

Entered when price touches or pierces the Slow MA during an established
uptrend.

While in this state, no attempt is made to predict direction.

------------------------------------------------------------------------

## 3. UP_RESUME_ATTEMPT

Entered when a completed candle closes above the current Highest MA.

``` text
Close > HighestMA
```

This represents the first indication that the pullback may have ended.

It is **not** confirmation of a new leg.

------------------------------------------------------------------------

# State Transitions

## UP_CONFIRMED → UP_UNCERTAINTY

Trigger:

-   Candle touches or pierces Slow MA.

------------------------------------------------------------------------

## UP_UNCERTAINTY → UP_RESUME_ATTEMPT

Trigger:

-   Candle closes above HighestMA.

Actions:

-   ResumeAttemptCount++

------------------------------------------------------------------------

## UP_RESUME_ATTEMPT → UP_UNCERTAINTY

Trigger:

-   Candle closes back below HighestMA.

Meaning:

-   Resume attempt failed.
-   Trend remains unresolved.

------------------------------------------------------------------------

## UP_RESUME_ATTEMPT → UP_CONFIRMED

Trigger:

``` text
|FastMA - SlowMA| / ATR >= N
```

Actions:

-   WaveCount++
-   Reset uncertainty state.

------------------------------------------------------------------------

# Downside

Mirror the entire model.

A bearish crossover alone does **not** confirm a new downtrend.

A bearish trend is confirmed only after:

``` text
|FastMA - SlowMA| / ATR >= N
```

in the bearish direction.

------------------------------------------------------------------------

# Design Principles

-   Separate **entry opportunity** from **trend confirmation**.
-   Use only closed bars.
-   Keep the state machine intentionally small.
-   Treat resume attempts as transient states.
-   Count resume attempts for future statistical analysis.
-   Use ATR-normalised MA separation for confirmation.
