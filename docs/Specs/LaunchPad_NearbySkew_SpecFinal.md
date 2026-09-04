# LaunchPad Nearby Skew — V1 Specification

Version: 1.1 (revised after critique cycle)  
Baseline: `K_LaunchPadLevels_V3.cpp`  
Output: enhancement to existing study, no new file

---

## Intention

The current LaunchPad V3 study plots the strongest nearby levels visually.
It does not provide a compact machine-readable summary of local directional structure.

This enhancement exposes a small current-state summary so the trader or a downstream
study can answer:

- "Within X ticks, are buy levels or sell levels dominating?"
- "What is the overhead resistance structure vs the downside support structure?"
- "What are the nearest actionable buy and sell levels right now?"

---

## Design Principles

- Do not store a historical matrix. Write only on the current bar.
- Do not expose every active level. Expose a distilled local summary only.
- Compute skew from the already-scored level pool. Do not rescan raw buckets.
- Do not use `draw_levels` as the source pool — use the full scored pool before top-K truncation.
- Do not change bucket accumulation, LPS math, adaptive threshold, or drawing behaviour.

---

## The Core Conceptual Requirement

**A single blended `NearbySkew` is insufficient.**

Buy levels above price and buy levels below price represent fundamentally different
structural information:

| Location | Meaning |
|---|---|
| Buy level **below** price | Support — buyers defending structure you may fall into |
| Sell level **above** price | Resistance — sellers defending structure you must move through |
| Buy level **above** price | Overhead aggression memory |
| Sell level **below** price | Downside aggression memory |

A single blended skew collapses these into one number and destroys the distinction.
Example: strong buy support below + strong sell resistance above = blended skew ≈ 0,
which reads as "balanced" when the correct read is "compressed between two walls."

**V1 must expose above-price and below-price skew separately.**

---

## Definitions

### Nearby Radius
Configurable input in ticks. Defines the local field used for all skew and
nearest-level computations.

### Nearby Level
A scored, non-expired LaunchPad level where:
- `lps >= i_LPSThreshold` (absolute floor, reuse existing input)
- `abs(level.price - current_price) / tick_size <= NearbyRadiusTicks`
- `das != 0` (zero-DAS levels are excluded — see classification below)

### Side Classification

```
das > 0  → buy side
das < 0  → sell side
das == 0 → exclude from all pools
```

**Do not use `das >= 0` as buy-side.** A zero-DAS bucket has no directional
information and must not inflate either pool.

### Location Classification

```
level.price > current_price → above price
level.price < current_price → below price
level.price == current_price → assign to above (implementation convention)
```

### Four Internal Accumulators

These are internal only — not all exposed as SGs:

```
buy_above_lps_sum   // buy-side levels above current price
buy_below_lps_sum   // buy-side levels below current price
sell_above_lps_sum  // sell-side levels above current price
sell_below_lps_sum  // sell-side levels below current price
```

---

## Skew Formulas

### Above-Price Skew

```
above_total = buy_above_lps_sum + sell_above_lps_sum

AbovePriceSkew =
    (buy_above_lps_sum - sell_above_lps_sum) / max(0.001, above_total)
```

Interpretation:
- `+1.0` = all overhead structure is buy-aggression
- `-1.0` = all overhead structure is sell-resistance
- `0.0`  = balanced overhead, or no overhead levels

The `max(0.001, ...)` guard prevents divide-by-zero when no above-price levels
exist. When `above_total == 0`, the output is `0 / 0.001 = 0.0` — correct
behaviour, not a magic number. Document this explicitly in code comments.

### Below-Price Skew

```
below_total = buy_below_lps_sum + sell_below_lps_sum

BelowPriceSkew =
    (buy_below_lps_sum - sell_below_lps_sum) / max(0.001, below_total)
```

Interpretation:
- `+1.0` = all downside structure is buy-support
- `-1.0` = all downside structure is sell-aggression beneath
- `0.0`  = balanced, or no below-price levels

### Blended NearbySkew (retained as secondary)

```
NearbySkew =
    (buy_above_lps_sum + buy_below_lps_sum - sell_above_lps_sum - sell_below_lps_sum) /
    max(0.001, buy_above_lps_sum + buy_below_lps_sum + sell_above_lps_sum + sell_below_lps_sum)
```

Retained because it is useful as a single-number directional summary for alerts
and downstream studies that don't need the above/below distinction. But it should
not be the primary output.

---

## Minimum Count Consideration

When only one level exists within the radius:

```
1 buy level, 0 sell levels → AbovePriceSkew or BelowPriceSkew = ±1.0
```

This is mathematically correct but can be misleading — it implies strong structural
conviction from a single data point.

**Do not force skew to zero when count is low.** That destroys information.

Instead: expose `SG_NearbyQualifiedCount` as a mandatory output so downstream
consumers can apply their own confidence threshold.

Recommended downstream rule (not enforced in study): require
`NearbyQualifiedCount >= 2` before treating skew as actionable.

---

## SG Outputs

### Mandatory SGs (implement all of these)

| Priority | SG Name | Type | Content |
|---|---|---|---|
| 1 | `SG_AbovePriceSkew` | Float [-1, +1] | Directional skew of levels above current price |
| 1 | `SG_BelowPriceSkew` | Float [-1, +1] | Directional skew of levels below current price |
| 2 | `SG_NearbyQualifiedCount` | Float (int-valued) | Total nearby levels in skew pool (confidence proxy) |
| 3 | `SG_NearestBuyNearby` | Price | Nearest buy-side level price within radius, 0 if none |
| 3 | `SG_NearestSellNearby` | Price | Nearest sell-side level price within radius, 0 if none |

### Optional SGs (add if SG budget allows)

| SG Name | Type | Content |
|---|---|---|
| `SG_NearbySkew` | Float [-1, +1] | Blended directional skew (secondary) |
| `SG_NearbyBuyCount` | Float | Count of nearby buy-side levels |
| `SG_NearbySellCount` | Float | Count of nearby sell-side levels |
| `SG_NearbyBuyLPSSum` | Float | Aggregate LPS of nearby buy levels |
| `SG_NearbySellLPSSum` | Float | Aggregate LPS of nearby sell levels |

---

## Computation Spec

For each study update:

```cpp
// Guard: skip if no valid current price
if (p_Data->last_trade_price < 0.001f) return;

float current_price = p_Data->last_trade_price;

// Accumulators
float buy_above_lps  = 0.0f, buy_below_lps  = 0.0f;
float sell_above_lps = 0.0f, sell_below_lps = 0.0f;
int   buy_count      = 0,    sell_count      = 0;
float nearest_buy_price  = 0.0f, nearest_buy_lps   = 0.0f;
float nearest_sell_price = 0.0f, nearest_sell_lps  = 0.0f;
float nearest_buy_dist   = FLT_MAX, nearest_sell_dist = FLT_MAX;

// Iterate full scored active pool — BEFORE resize(top_k)
for each level in scored_levels_pre_topk:

    // Distance gate
    float dist_ticks = fabsf(level.price - current_price) / effective_tick;
    if (dist_ticks > NearbyRadiusTicks) continue;

    // LPS floor
    if (level.lps < i_LPSThreshold) continue;

    // Side classification — exclude das == 0
    bool is_buy  = (level.das > 0.0f);
    bool is_sell = (level.das < 0.0f);
    if (!is_buy && !is_sell) continue;

    // Location classification
    // Levels exactly at current_price are assigned to "above".
    // This convention is arbitrary — chosen only for deterministic partitioning.
    // There is no behavioural meaning to this assignment.
    bool is_above = (level.price >= current_price);

    // Accumulate
    if (is_buy) {
        buy_count++;
        if (is_above) buy_above_lps  += level.lps;
        else          buy_below_lps  += level.lps;

        // Nearest buy: distance first, LPS as tie-break (mandatory, not optional)
        if (dist_ticks < nearest_buy_dist ||
            (dist_ticks == nearest_buy_dist && level.lps > nearest_buy_lps)) {
            nearest_buy_price = level.price;
            nearest_buy_dist  = dist_ticks;
            nearest_buy_lps   = level.lps;
        }
    }

    if (is_sell) {
        sell_count++;
        if (is_above) sell_above_lps += level.lps;
        else          sell_below_lps += level.lps;

        if (dist_ticks < nearest_sell_dist ||
            (dist_ticks == nearest_sell_dist && level.lps > nearest_sell_lps)) {
            nearest_sell_price = level.price;
            nearest_sell_dist  = dist_ticks;
            nearest_sell_lps   = level.lps;
        }
    }

// Compute skew outputs
float above_total = buy_above_lps + sell_above_lps;
float below_total = buy_below_lps + sell_below_lps;
float total       = above_total + below_total;

float above_skew = (buy_above_lps - sell_above_lps) / max(0.001f, above_total);
float below_skew = (buy_below_lps - sell_below_lps) / max(0.001f, below_total);
float blended    = (buy_above_lps + buy_below_lps - sell_above_lps - sell_below_lps)
                   / max(0.001f, total);

int qualified_count = buy_count + sell_count;

// Write to current bar only — clear then write
int bar = sc.ArraySize - 1;
SG_AbovePriceSkew[bar]      = above_skew;
SG_BelowPriceSkew[bar]      = below_skew;
SG_NearestBuyNearby[bar]    = nearest_buy_price;
SG_NearestSellNearby[bar]   = nearest_sell_price;
SG_NearbyQualifiedCount[bar] = (float)qualified_count;
// optional:
SG_NearbySkew[bar]          = blended;
SG_NearbyBuyCount[bar]      = (float)buy_count;
SG_NearbySellCount[bar]     = (float)sell_count;
```

---

## Behavioural Examples

### Example 1 — Buy-supported, sell-resisted (compressed)

Within 24 ticks:
- 3 buy levels below price, LPS sum = 4.2
- 2 sell levels above price, LPS sum = 3.8

```
BelowPriceSkew = (4.2 - 0) / 4.2 = +1.0  (pure buy support below)
AbovePriceSkew = (0 - 3.8) / 3.8 = -1.0  (pure sell resistance above)
NearbySkew     = (4.2 - 3.8) / 8.0 = +0.05
```

Single blended skew reads +0.05 (near neutral). Above/below split correctly
identifies the compression. This is the failure case the blended number misses.

### Example 2 — Strong directional field

Within 24 ticks:
- 4 buy levels, 3 below price (LPS sum 5.1), 1 above price (LPS sum 0.9)
- 1 sell level above price (LPS sum 0.7)

```
BelowPriceSkew = (5.1 - 0) / 5.1 = +1.0
AbovePriceSkew = (0.9 - 0.7) / 1.6 = +0.125
NearbySkew     = (6.0 - 0.7) / 6.7 = +0.79
```

Strong buy support below. Modest buy edge above. Overall buy-skewed.

### Example 3 — Empty radius

All outputs = 0. `NearbyQualifiedCount = 0`. Skew formulas return 0 via
`max(0.001, 0) = 0.001` denominator → `0 / 0.001 = 0.0`.

---

## Input Additions

| Input | Type | Default | Notes |
|---|---|---|---|
| `Nearby Radius Ticks` | Int | 24 | Local field radius. 24 is calibrated for MES/ES with 4-tick bucket size. Adjust proportionally for NQ/MNQ — NQ tick volatility may warrant 40–60. No instrument-aware default is possible in ACSIL inputs. |

**Do not add `SkewMinLPS` in V1.** The weighted LPS formula already partially
mitigates weak-level pollution. An additional threshold creates a calibration
rabbit hole. If noise is observed, first try squaring LPS before summing
(`level.lps * level.lps`) as an internal weight — that requires no new input.

---

## Source Pool Requirement

Skew **must** be computed from the full scored active level pool **before**
`resize(top_k)` / `draw_levels` truncation.

Reason: top-K is a display constraint, not a structural truth constraint. Computing
skew from `draw_levels` makes it dependent on the `TopK` display setting, which
is wrong. A `TopK = 3` setting with 8 meaningful nearby levels should not produce
a different skew than `TopK = 8`.

This is the most important implementation constraint in the spec.

---

## What Is Explicitly Not In V1

- Distance weighting (closer levels weighted more heavily) — valid concept, V2
- `SkewMinLPS` input — not needed, see above
- Session-reset-aware skew — V2
- Strongest (not nearest) level outputs — V2
- Skew filtered by maturity or age — V2
- Automatic tradability classification — never in this study
- Level suppression based on skew — never in this study

---

## Known Limitations

| Limitation | Impact | Notes |
|---|---|---|
| Symmetric radius treats above/below equally | Partially mitigated by AboveSkew/BelowSkew split | Distance weighting deferred to V2 |
| `NearbyQualifiedCount = 1` produces ±1.0 skew | Mathematically correct but low confidence | Expose count; let consumer apply minimum |
| Radius default not instrument-aware | User must tune for NQ/MNQ | Document in input description |
| `das == 0` buckets excluded | Correct behaviour but reduces pool in early-session conditions | Acceptable — zero-DAS levels carry no directional information |
| Radius-edge instability | A level oscillating near the NearbyRadiusTicks boundary causes skew to flicker | Mitigations (hysteresis, soft distance weighting) deferred to V2 |

---

## Implementation Notes for Codex

- Reuse the existing `LevelScore` pool built by the study. Do not rescan buckets.
- Access pool before `resize(top_k)` — this is the single most important constraint.
- Do not read from `draw_levels`.
- Clear SG values at the start of each call before recomputing (prevents stale
  values persisting if the study skips an update).
- All SG writes go to `sc.ArraySize - 1` only.
- Tie-break on nearest level is mandatory: equal distance → higher LPS wins.
  This prevents non-deterministic oscillation between updates.
