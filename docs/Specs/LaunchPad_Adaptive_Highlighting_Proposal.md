# LaunchPad Adaptive Highlighting Proposal

## Purpose

This note proposes a replacement for the current hardcoded `LPS Threshold to Highlight` logic in `K_LaunchPadLevels_V2`.

The goal is to highlight price levels where one side of Time and Sales is materially overpowering the other side without requiring a fixed absolute threshold that must be retuned for every instrument, session, and volatility regime.

## Problem With Fixed LPS Thresholds

A fixed LPS threshold is fragile because the meaning of a given score changes with:

- instrument volatility
- session type
- participation regime
- replay speed / update density
- current order-flow intensity
- bucket concentration across the active price range

As a result:

- a threshold that is useful in one regime can under-fire in quiet conditions
- the same threshold can over-fire in highly active conditions
- the user ends up tuning the threshold instead of reading the signal

## Design Principle

The study should separate:

1. level qualification
2. level highlighting

Qualification answers:
- is this level directionally meaningful at all?

Highlighting answers:
- among meaningful levels, which ones are true current outliers?

This avoids using one hardcoded number for both jobs.

## Proposed Model

### Stage 1. Qualification Gates

A bucket must first qualify before it can be highlighted.

Recommended gates:

- `abs(bucket.ewma_das) >= DominanceFloor`
- `bucket.ewma_ai / display_mean_ai >= ParticipationFloor`
- `maturity >= MaturityFloor`
- bucket is not expired

Where:

- `DominanceFloor` is a direct directional-imbalance floor
- `ParticipationFloor` ensures the level is meaningful relative to current background activity
- `MaturityFloor` prevents single-touch noise from being treated as extreme

Recommended starting defaults:

- `DominanceFloor = 0.45`
- `ParticipationFloor = 1.25`
- `MaturityFloor = 0.50`

Interpretation:

- the side imbalance must be strong
- the level must matter in size, not just direction
- the level must have at least partial touch maturity

### Stage 2. Adaptive Highlighting

After qualification, highlight only buckets that are outliers relative to the currently active qualified set.

Recommended primary method:

- compute `highlight_score = LPS` for all qualified active buckets
- compute a rolling percentile across those scores
- highlight only buckets at or above `HighlightPercentile`

Recommended default:

- `HighlightPercentile = 95%`

So the study still ranks by LPS, but the cyan / amber emphasis becomes relative to the current regime rather than tied to one fixed score.

## Optional Alternative Adaptive Methods

### Option A. Percentile Method

Highlight rule:

- highlight if `LPS >= percentile(qualified_levels, p)`

Recommended:

- `p = 95` for moderate selectivity
- `p = 98` for very selective highlighting

Pros:

- simple to reason about
- robust across instruments
- stable even when score scale changes

Cons:

- can still highlight mediocre levels if the whole regime is weak unless qualification gates are strong enough

### Option B. Mean Plus Standard Deviation

Highlight rule:

- compute `mu` and `sigma` of qualified active `LPS`
- highlight if `LPS >= mu + K * sigma`

Recommended:

- `K = 1.5` to `2.0`

Pros:

- responsive to score dispersion

Cons:

- less robust when score distribution is skewed or sample count is small

### Option C. Top-N Per Side

Highlight rule:

- choose strongest buy bucket and strongest sell bucket independently
- optionally choose top 2 per side if they pass qualification

Pros:

- guarantees balanced directional visibility
- useful when a one-sided regime would otherwise hide the weaker opposing side entirely

Cons:

- less statistically adaptive than percentile gating

## Recommended Direction For LaunchPad

Use a hybrid model:

1. hard qualification gates
2. percentile-based highlight selection
3. optional side separation for buy and sell buckets

This preserves interpretability while adapting automatically to current market conditions.

## Proposed Concrete Formula

For each active non-expired bucket:

```text
imbalance     = abs(bucket.ewma_das)
participation = min(bucket.ewma_ai / display_mean_ai, ParticipationCap)
maturity      = min(bucket.touches / effective_min_touches, 1.0)
LPS           = imbalance * participation * maturity
```

Qualification:

```text
qualified =
    imbalance     >= DominanceFloor
    and participation >= ParticipationFloor
    and maturity      >= MaturityFloor
```

Highlight score:

```text
highlight_score = LPS
```

Adaptive emphasis:

```text
highlighted if highlight_score >= percentile(all qualified active scores, HighlightPercentile)
```

Optional side-separated emphasis:

```text
buy_highlight_threshold  = percentile(qualified buy scores,  HighlightPercentile)
sell_highlight_threshold = percentile(qualified sell scores, HighlightPercentile)
```

Then:

- buy bucket gets extreme buy color only if `bucket.ewma_das > 0` and it exceeds the buy threshold
- sell bucket gets extreme sell color only if `bucket.ewma_das < 0` and it exceeds the sell threshold

## Why This Better Matches The User Intent

The user intent is not merely:

- show high LPS numbers

The actual intent is closer to:

- show places where one side is massively overpowering the other side
- but only when that overpowering is meaningful in size
- and only when it is unusually strong relative to the current market regime

That means the core trigger should not be a single absolute LPS number.

It should be:

- directional dominance
- meaningful participation
- sufficient maturity
- current-regime outlier status

## Suggested Input Changes

Current input:

- `LPS Threshold to Highlight`
- `LPS Extreme Threshold (Cyan/Amber)`

Suggested replacement set:

- `Dominance Floor`
- `Participation Floor`
- `Maturity Floor`
- `Highlight Percentile`
- `Participation Cap`
- optional: `Highlight Per Side`
- optional: `Min Qualified Levels For Adaptive Highlight`

Suggested defaults:

- `Dominance Floor = 0.45`
- `Participation Floor = 1.25`
- `Maturity Floor = 0.50`
- `Highlight Percentile = 95`
- `Participation Cap = 10.0` or `25.0`
- `Highlight Per Side = Yes`
- `Min Qualified Levels For Adaptive Highlight = 5`

## Small-Sample Fallback

Adaptive methods can become unstable when there are too few qualified levels.

Recommended fallback:

- if qualified level count is below `Min Qualified Levels For Adaptive Highlight`
- do not apply percentile highlighting
- instead use qualification-only coloring, or use top-1-per-side

This prevents noisy threshold jumps when very few levels are active.

## Visual Behavior Proposal

Three visual states are recommended:

1. Qualified
- standard green / red line
- normal width

2. Highlighted Adaptive Outlier
- cyan for buy / amber for sell
- thicker line

3. Faded / Expired
- existing fade / expire logic remains unchanged

This keeps the current visual language while making extreme highlighting adaptive.

## Recommended Implementation Order

1. Add qualification gates using existing per-bucket values.
2. Collect qualified active bucket scores each update.
3. Compute percentile threshold.
4. Apply extreme colors only to adaptive outliers.
5. Optionally split thresholds by buy and sell side.
6. Keep current top-K ranking and drawing model intact.

## Minimal-Change Implementation Strategy

To minimize disruption to `K_LaunchPadLevels_V2.cpp`:

- keep current `LPS` formula as the ranking score
- keep current top-K selection flow
- replace the fixed `LPS Extreme Threshold` color trigger only
- introduce adaptive highlight state after scoring but before drawing

This gives immediate benefit without rewriting the study's core structure.

## Recommendation

Recommended production approach:

- keep `LPS` for ranking
- remove fixed extreme highlighting threshold as the primary trigger
- add qualification gates plus percentile-based adaptive highlighting
- optionally split buy and sell highlight thresholds by side

This is the most direct way to make LaunchPad automatically surface levels where one side of Time and Sales is genuinely overpowering the other side under current market conditions.
