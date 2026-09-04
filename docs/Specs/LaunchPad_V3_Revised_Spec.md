# LaunchPad Levels V2 — Revised Implementation Spec

## Status

This document supersedes the earlier adaptive highlighting proposal.
It incorporates two corrections identified during code review:

1. The bucket engine must be rebuilt to accumulate **per-trade, per-level** buy/sell volume.
2. The adaptive extreme threshold must be computed from the corrected per-level metric.

These are treated as two sequential deliverables.

---

## Part 1 — Bucket Engine Correction (v2.1)

### Problem With Current Engine

The current study updates buckets here:

```cpp
// Map last trade price to bucket and update with blended contribution
bucket->ewma_das = lambda * bucket->ewma_das + (1.0f - lambda) * weighted_contribution;
bucket->ewma_ai  = lambda * bucket->ewma_ai  + (1.0f - lambda) * blended_ai;
```

`weighted_contribution` and `blended_ai` are computed from the global rolling bar window —
not from trades that actually occurred at that price level.

This means a bucket accumulates "what was the global regime when price visited here",
not "what did aggressive buyers and sellers actually do at this level."

Those two things correlate in trending conditions and diverge in rotation/range conditions.

### Required Correction

Each classified trade must update the bucket that corresponds to its own trade price.

#### Step 1 — Update LevelBucket struct

Add two fields. Remove `ewma_das` and `ewma_ai` as primary fields (they become derived).

```cpp
struct LevelBucket {
    int bin_ticks;
    float buy_vol;            // time-decayed aggressive buy volume at this level
    float sell_vol;           // time-decayed aggressive sell volume at this level
    int touches;              // number of distinct visits (debounced, not raw trade count)
    SCDateTime last_seen;     // last trade timestamp — used for decay anchor and expiry
    SCDateTime last_touch_time; // last visit timestamp — used for debounce only
    float vol_sum;            // diagnostic only — not used in LPS or highlighting logic
    bool active;
};
```

**`last_seen` vs `last_touch_time` — two separate roles:**

| Field             | Role                          | Updated when              |
|-------------------|-------------------------------|---------------------------|
| `last_seen`       | Decay anchor, expiry check    | Every classified trade    |
| `last_touch_time` | Visit debounce anchor         | Only when touch increments |

These must not share the same timestamp. Using `last_seen` for debounce causes
the debounce window to reset on every trade, undercounting visits in active zones
where volume arrives in rapid bursts at the same level.

**On `touches` semantics:** `touches` counts **distinct visits**, not raw T&S records.
A visit is the first classified trade at a bucket after `VISIT_DEBOUNCE_MS` has elapsed
since the last visit. This prevents a single aggressive sweep from instantly inflating
maturity to 1.0 via 50–200 rapid-fire records at the same level.
`min_touches` as a maturity gate only has meaning if each touch represents a
genuinely separate engagement with the level.

#### Step 2 — Update in T&S loop

Inside the existing T&S classification loop, after `is_buy` / `is_sell` is determined,
add a per-trade bucket update.

**Critical rules:**
- **Skip entirely if neither `is_buy` nor `is_sell`** — unclassified trades carry no
  directional information and must not update bucket state.
- **Do not speculate side from price movement or bid/ask comparison.** The feed is
  high-quality and side is reliably set via `SC_TS_ASK` / `SC_TS_BID`. If the feed
  does not classify a trade, skip it entirely — do not attempt to infer direction.
  The fallback classification blocks from V1 (bid/ask price comparison, uptick/downtick)
  are retained for structural robustness but their behaviour changes: they must skip
  the trade rather than assign a side guess.
- **Use T&S record timestamp, not `sc.CurrentSystemDateTime`**, for decay and debounce.
  `sc.CurrentSystemDateTime` is fine live but distorts age/decay calculations during
  replay or historical recalculation. Use `TimeSales[ts_idx].DateTime` instead.

```cpp
// Only update buckets for classified trades.
// Do NOT speculate side — if feed did not classify, skip.
if (!is_buy && !is_sell) {
    p_Data->last_sequence = sequence;
    continue;
}

// Use T&S record time (not system time) — correct for replay and live
SCDateTime trade_time = TimeSales[ts_idx].DateTime;

// Map trade to its price bucket
int price_ticks = static_cast<int>(roundf(price / effective_tick));
int bin_ticks   = (price_ticks / bucket_size_ticks) * bucket_size_ticks;

// Find or create bucket
int bucket_idx = find_or_create_bucket(p_Data, bin_ticks);

if (bucket_idx >= 0) {
    LevelBucket* bucket = &p_Data->buckets[bucket_idx];

    // [OPTIONAL] Per-bucket duplicate sequence guard.
    // The global last_sequence check prevents reprocessing at the study level,
    // but some feeds replay duplicate sequence IDs under reconnects. A per-bucket
    // guard is cheap insurance if this study becomes production-critical.
    // Requires adding int64_t last_sequence_seen to LevelBucket struct.
    // if (sequence == bucket->last_sequence_seen) { continue; }

    // Skip out-of-order records entirely.
    // dt_hours < 0 means this record arrived with a timestamp earlier than the last
    // update to this bucket (replay glitch or feed gap). Clamping to 0 neutralises
    // decay (lambda=1) but still adds volume, creating a subtle bias. Better to skip.
    if (bucket->touches > 0 && trade_time < bucket->last_seen) {
        p_Data->last_sequence = sequence;
        continue;
    }

    // Apply time decay since last update to this bucket
    // Uses trade_time for replay correctness
    float lambda = 1.0f;
    if (bucket->touches > 0) {
        float dt_hours = (trade_time - bucket->last_seen).GetTimeInSeconds() / 3600.0f;
        lambda = expf(-logf(2.0f) * dt_hours / half_life_hours);
    }

    bucket->buy_vol  = lambda * bucket->buy_vol  + (is_buy  ? (float)volume : 0.0f);
    bucket->sell_vol = lambda * bucket->sell_vol + (is_sell ? (float)volume : 0.0f);

    // Increment touches once per visit, not per trade.
    // Uses last_touch_time (NOT last_seen) for debounce — these serve different roles.
    // last_seen is updated every trade (decay anchor).
    // last_touch_time is updated only when a visit is counted (debounce anchor).
    const int VISIT_DEBOUNCE_MS = 150;
    int ms_since_visit = (bucket->touches == 0) ? 99999 :
        (int)((trade_time - bucket->last_touch_time).GetTimeInSeconds() * 1000.0f);
    if (ms_since_visit >= VISIT_DEBOUNCE_MS) {
        bucket->touches++;
        bucket->last_touch_time = trade_time;
    }

    bucket->last_seen = trade_time;  // always update decay anchor

    // vol_sum retained for diagnostics only — not used in LPS or highlighting logic
    bucket->vol_sum += (float)volume;
}
```

#### Step 3 — Remove the old bucket update block

The existing block starting with:

```cpp
// Map last trade price to bucket and update with blended contribution
```

is removed entirely. Bucket state is now maintained exclusively from the T&S loop.

#### Step 4 — Derived quantities at scoring time

At scoring time, compute DAS and AI from the bucket's own accumulated volume:

```cpp
float ai  = bucket->buy_vol + bucket->sell_vol;
float das = (ai > 0.001f) ? (bucket->buy_vol - bucket->sell_vol) / ai : 0.0f;

// Winsorise DAS
if (das >  winsor_cap) das =  winsor_cap;
if (das < -winsor_cap) das = -winsor_cap;
```

### Participation Normalisation — display_mean_ai Fix

The current code sets:

```cpp
float display_mean_ai = tf[0].enabled ? tf[0].mean_ai : 0.001f;
```

When Primary is disabled, this collapses to `0.001f`, making participation values meaningless.

**Replacement:**

Compute `display_mean_ai` as the mean `ai` across all non-expired, distance-gated active buckets:

```cpp
float display_mean_ai = 0.0f;
int   mean_ai_count   = 0;

for (int i = 0; i < p_Data->num_buckets; i++) {
    LevelBucket* b = &p_Data->buckets[i];
    if (!b->active) continue;

    int age_sec   = (current_time - b->last_seen).GetTimeInSeconds();
    float dist_t  = fabsf(b->bin_ticks * effective_tick - p_Data->last_trade_price) / effective_tick;

    if (age_sec > expire_seconds)    continue;
    if (dist_t  > max_dist_ticks)    continue;

    display_mean_ai += b->buy_vol + b->sell_vol;
    mean_ai_count++;
}

if (mean_ai_count > 0) display_mean_ai /= mean_ai_count;
if (display_mean_ai < 0.001f) display_mean_ai = 0.001f;
```

This is regime baseline derived from actual per-level activity — which is what
participation normalisation was always supposed to measure.

**Early-session baseline guard:**

When fewer than `MIN_BUCKETS_FOR_BASELINE` buckets exist in the active pool,
`display_mean_ai` is unstable and participation values will spike. Guard against this:

```cpp
const int MIN_BUCKETS_FOR_BASELINE = 5;

// In LPS scoring loop, after computing participation:
float participation;
if (mean_ai_count < MIN_BUCKETS_FOR_BASELINE) {
    participation = 1.0f;  // neutral — do not penalise or amplify early-session levels
} else {
    participation = min(ai / display_mean_ai, ParticipationCap);
}
```

This prevents LPS from producing misleading extreme values at session open before
enough levels have accumulated to form a stable baseline.

### Updated LPS Formula

```
ai            = bucket.buy_vol + bucket.sell_vol
das           = (bucket.buy_vol - bucket.sell_vol) / ai    [winsorised]
imbalance     = abs(das)
participation = min(ai / display_mean_ai, ParticipationCap)   // cap applied HERE
maturity      = min(1.0, touches / effective_min_touches)

LPS = imbalance * participation * maturity
```

**Participation cap must be applied before LPS multiplication, not after.**
Capping after the multiply allows extreme uncapped participation values into the
percentile pool, which inflates the adaptive threshold and suppresses legitimate highlights.

This is now a clean per-level order flow metric.

### Distance Gate — Intentional Scope Difference

The distance gate (`max_dist_ticks`) is applied differently across three computations:

| Computation          | Distance-gated? | Rationale |
|----------------------|-----------------|-----------|
| `display_mean_ai`    | Yes             | Baseline should reflect activity in the vicinity of current price, not distant stale levels |
| Percentile pool      | Yes             | Adaptive threshold should reflect the current local regime, not price levels far from action |
| LPS scoring / top-K  | No (drawn levels gated at draw time) | All non-expired buckets are scored; distance gate applies at draw step only |

**Do not "fix" the `display_mean_ai` computation to be unfiltered.** The distance gate
on the baseline is intentional — it ensures participation normalisation reflects local
regime intensity, not a session-wide average diluted by distant inactive levels.

---

### Multi-Timeframe Blending — Explicit Removal Notice

**The v2.1 bucket engine change removes multi-timeframe blending from the level metric.**

Prior to v2.1, each bucket accumulated a weighted blend of Primary, Confirmation, and
Structural timeframe contributions. That was the mechanism by which TF window settings
affected LPS values.

After v2.1, each bucket accumulates only **raw trade volume from T&S at that price**,
time-decayed by `HalfLifeHours`. The TF window settings no longer affect what is stored
in buckets or how LPS is computed.

**What the TF inputs still do after v2.1:**

| Input group                  | Still active? | Effect                                         |
|------------------------------|---------------|------------------------------------------------|
| Primary/Confirmation/Structural Bars, Weight, Baseline | No — removed from bucket engine | No effect on LPS |
| Primary/Confirmation/Structural MinTouches | Yes | `effective_min_touches` = min across enabled TFs — still gates maturity |
| Half-Life Hours              | Yes           | Controls time decay on `buy_vol` / `sell_vol`  |
| Bucket Size, TopK, Fade, Expire, MaxDist | Yes | Unchanged                              |

The MinTouches inputs retain value as visit-count gates for maturity.
All other TF tuning inputs (`Bars`, `Weight`, `Baseline`) are now vestigial.

**User-facing recommendation:** In a follow-on UI cleanup (v3), the `Bars`, `Weight`,
and `Baseline` inputs per timeframe should be hidden or removed to avoid confusing users
into believing they are still shaping the level signal. For v2.1, they can be left in
place — they are simply ignored by the new engine.

---

## Part 2 — Adaptive Extreme Threshold (v2.2)

### What "Auto Threshold" Means

The original intent of `i_LPSExtremeThreshold` was to highlight levels where one side
is genuinely overpowering the other. A fixed value fails because the LPS scale changes
with instrument, session, and volatility regime.

The auto threshold answers: **among the levels that qualify right now,
which ones are true outliers relative to the current regime?**

### Qualification Gates

Before a bucket can be considered for extreme highlighting it must pass all gates:

```
non-expired:    age_seconds <= expire_seconds
distance-gated: dist_ticks  <= max_dist_ticks
lps_floor:      LPS >= i_LPSThreshold             (existing input, repurposed as absolute floor)
imbalance_gate: imbalance >= DominanceFloor
participation:  participation >= ParticipationFloor
maturity:       maturity >= MaturityFloor
```

Recommended defaults:

| Parameter          | Default | Meaning                                              |
|--------------------|---------|------------------------------------------------------|
| `DominanceFloor`   | 0.45    | Directional imbalance must be strong                 |
| `ParticipationFloor` | 1.25  | Level must be above-average in size                  |
| `MaturityFloor`    | 0.50    | At least half the minimum touches reached            |
| `ParticipationCap` | 10.0    | Cap participation to prevent outlier distortion      |

Note: `i_LPSThreshold` (currently 0.40) becomes the `AbsoluteMinLPS` floor.
No new input is needed for this role — it is already present.

### Adaptive Threshold Math

#### Active Qualified Pool

Collect LPS scores for all buckets that pass the qualification gates
**before `resize(top_k)`**.

This is critical. Post-truncation the pool has at most `top_k` values (default 3),
which makes any percentile meaningless.

```cpp
std::vector<float> qualified_lps;

for (int i = 0; i < p_Data->num_buckets; i++) {
    // ... compute LPS for bucket i ...
    if (passes_all_gates(bucket, lps)) {
        qualified_lps.push_back(lps);
        scored_levels.push_back(...);  // existing logic
    }
}

// Guard: empty pool — skip adaptive threshold entirely
// An empty qualified_lps would cause undefined access in percentile computation.
// This is separate from MinQualifiedForAdaptive (which guards against fake precision
// at small-but-nonzero N). This guard prevents a crash or garbage threshold at N=0.
if (qualified_lps.empty()) {
    // no extreme highlighting this update — all levels draw as standard green/red
    // skip to top-K truncation and drawing
}
```

#### Primary Method — Percentile

Sort the qualified pool and find the threshold at `HighlightPercentile`:

```cpp
float adaptive_threshold = compute_percentile(qualified_lps, HighlightPercentile);
```

Where:

```
percentile(scores, p):
    sort scores ascending
    idx = floor((p / 100.0) * (N - 1))
    return scores[idx]
```

Recommended default: `HighlightPercentile = 95`

A bucket is extreme if:

```
is_extreme =
    LPS >= i_LPSThreshold          // absolute floor (existing gate)
    AND LPS >= adaptive_threshold  // regime-relative outlier
```

#### Small-Sample Fallback

Percentile over small N is fake precision:

| N        | Behaviour of 95th percentile       |
|----------|-------------------------------------|
| 1–4      | Always returns the single max value |
| 5–10     | Effectively top-1 or top-2          |
| 11–19    | Marginally meaningful               |
| 20+      | Reliable                            |

Rule:

```
if len(qualified_lps) < MinQualifiedForAdaptive:
    is_extreme = False          // suppress extreme highlighting entirely
    // standard green/red highlighting still applies
```

Recommended default: `MinQualifiedForAdaptive = 10`

Do **not** fall back to global percentile when per-side N is thin.
Suppress extreme highlighting entirely. A suppressed highlight is less harmful
than a misleading one.

#### Optional — Side-Separated Thresholds

Split the qualified pool into buy-side and sell-side:

```cpp
std::vector<float> buy_lps, sell_lps;

for each qualified bucket:
    if das > 0: buy_lps.push_back(lps)   // das derived from buy_vol/sell_vol; ewma_das no longer exists
    else:            sell_lps.push_back(lps)

buy_threshold  = (buy_lps.size()  >= MinQualifiedForAdaptive)
                 ? percentile(buy_lps,  HighlightPercentile)
                 : NO_EXTREME;

sell_threshold = (sell_lps.size() >= MinQualifiedForAdaptive)
                 ? percentile(sell_lps, HighlightPercentile)
                 : NO_EXTREME;
```

Per-side min count uses the same `MinQualifiedForAdaptive`.
If one side is thin, suppress extreme highlighting for that side only.
Do not fall back to the global pool — a one-sided regime with 2 buy buckets
does not have a meaningful buy-side percentile.

Default: `HighlightPerSide = true`

### Complete Highlight Decision

```
qualified =
    non-expired
    AND distance-gated
    AND LPS         >= i_LPSThreshold
    AND imbalance   >= DominanceFloor
    AND participation >= ParticipationFloor
    AND maturity    >= MaturityFloor

if HighlightPerSide:
    threshold = buy_threshold  if bucket is buy side
                sell_threshold if bucket is sell side
else:
    threshold = global_adaptive_threshold

is_extreme = qualified
             AND threshold != NO_EXTREME
             AND LPS >= threshold
```

### Hysteresis (v1 scope)

No hysteresis is implemented in v2.2.

If a bucket's LPS oscillates near the percentile boundary it will flicker between
green/red and cyan/amber. This is a known v1 limitation.

Document it. Do not add hold-bars logic until flickering is confirmed as a real
problem in live use. The fix is a per-bucket `highlight_hold_bars` counter in
`LevelBucket`, decremented each update, only clearing the extreme state when
the counter reaches zero.

---

## Part 3 — New Input Parameters

### Remove

- `i_LPSExtremeThreshold` (fixed threshold replaced by adaptive)

### Add

| Input                        | Type  | Default | Purpose                                      |
|------------------------------|-------|---------|----------------------------------------------|
| `Dominance Floor`            | Float | 0.45    | Minimum abs(DAS) to qualify                  |
| `Participation Floor`        | Float | 1.25    | Minimum participation ratio to qualify       |
| `Maturity Floor`             | Float | 0.50    | Minimum maturity fraction to qualify         |
| `Participation Cap`          | Float | 10.0    | Winsorise participation before LPS multiply  |
| `Highlight Percentile`       | Float | 95.0    | Percentile threshold for extreme colour      |
| `Min Qualified For Adaptive` | Int   | 10      | Suppress extreme highlight below this count  |
| `Highlight Per Side`         | Bool  | Yes     | Separate buy/sell percentile pools           |

### Retain (repurposed)

- `i_LPSThreshold` — becomes the absolute minimum LPS floor (no rename required, existing default 0.40 is reasonable)

---

## Visual States

| State              | Colour             | Width | Condition                          |
|--------------------|--------------------|-------|------------------------------------|
| Qualified          | Green (buy) / Red (sell) | 2 | passes LPS floor, not extreme  |
| Extreme            | Cyan (buy) / Amber (sell) | 3 | passes adaptive threshold      |
| Faded              | Green/Red dashed   | 1     | age > FadeMinutes                  |
| Expired            | not drawn          | —     | age > ExpireMinutes                |

---

## Implementation Order

1. **v2.1 — Bucket engine fix**
   - Add `buy_vol`, `sell_vol` to `LevelBucket`; update `touches` semantics to visits (debounced)
   - Accumulate per-trade in T&S loop with time decay and 150ms visit debounce
   - Remove old global-regime bucket update block
   - Replace `display_mean_ai` with active-bucket mean
   - Derive `das` / `ai` at scoring time from bucket's own volume
   - Document that TF Bars/Weight/Baseline inputs are now vestigial
   - Increment `SCHEMA_VERSION` to force reset

2. **v2.2 — Adaptive extreme threshold**
   - Add qualification gates
   - Collect qualified LPS pool before `resize(top_k)`
   - Compute percentile threshold
   - Apply side separation with per-side min-count guard
   - Replace fixed `is_extreme` line with adaptive decision
   - Remove `i_LPSExtremeThreshold`, add new inputs

3. **v2.3 — Active pool refinement (later)**
   - Review distance gate and expiry interaction
   - Consider recency-weighting in percentile computation
   - Evaluate hysteresis hold-bars if flicker confirmed in live use

---

## Known Limitations at v2.2

| Limitation                  | Impact                          | Fix Version |
|-----------------------------|---------------------------------|-------------|
| No hysteresis               | Possible highlight flicker near percentile boundary | v2.3 |
| New input defaults unvalidated against live data | Thresholds provisional — require calibration against MES replay | Post-v2.2 |
| Multi-TF Bars/Weight/Baseline inputs vestigial | Users may tune inputs that have no effect | v3 UI cleanup |
| No absorption detection | Aggressive volume without price movement not captured | v3 |
| Bucket lookup is O(n) per trade | Acceptable at MAX_BUCKETS=2048; revisit if hot in profiling | v3 |

---

## Notes for GPT / Codex

### Baseline File

**Use `K_LaunchPadLevels_V1.cpp` as the implementation baseline.**

Do not use V2. V2 added multi-timeframe blending which is architecturally removed by
this spec. V1 is the correct and cleaner starting point.

The output file should be named `K_LaunchPadLevels_V3.cpp` with:
- `SCDLLName("Launch Pad Levels - Per-Level Order Flow V3")`
- `SCHEMA_VERSION = 5` (forces clean reset on load)

### V1 Source Map

| What to change | V1 line(s) | Action |
|---|---|---|
| T&S classification loop | 289–371 | Add per-trade bucket accumulation after `is_buy`/`is_sell` is resolved |
| Old bucket update block | 422–463 | **DELETE entirely** — the `// Map last trade price to bucket` block |
| `mean_ai` computation | 398–416 | **REPLACE** with active-bucket mean (see Part 1 of spec) |
| LPS scoring loop | 480–511 | Derive `das`/`ai` from `buy_vol`/`sell_vol` instead of `ewma_das`/`ewma_ai` |
| Adaptive threshold | After line 519 (`resize(top_k)`) | **ADD** percentile computation over pre-truncation qualified pool |
| Drawing loop | 565–611 | Add cyan/amber extreme colour branch |
| Input block | 87–106, 122–186 | Add new adaptive inputs (see Part 3 of spec) |

### What Does Not Change in V1

- Persistent memory allocation pattern (lines 204–215) — unchanged
- Reset logic (lines 217–249) — unchanged except SCHEMA_VERSION bump to 5
- Bar accumulator (`bars[]` buy_vol/sell_vol) — retained for HUD classification stats only
- Drawing infrastructure (probe line, best DAS line, HUD text) — unchanged
- `std::vector`, `std::sort` — already present, no new dependencies
- Distance gate, fade, expire logic — unchanged

### V1-Specific Note on Vestigial Inputs

V1 has two inputs that become vestigial after the bucket engine fix:
- `i_RollingBars` — was the rolling bar window for global DAS; no longer used in bucket scoring
- `i_AIWindowSeconds` — was the baseline window for `mean_ai`; replaced by active-bucket mean

These can remain in the input panel for v3 without causing harm (they will simply be unused).
Remove them in a later UI cleanup pass.

### Bucket Lookup Performance

The existing bucket lookup is an O(n) linear scan (V1 lines 428–434):

```cpp
for (int i = 0; i < p_Data->num_buckets; i++) {
    if (p_Data->buckets[i].active && p_Data->buckets[i].bin_ticks == bin_ticks) {
        bucket_idx = i;
        break;
    }
}
```

This scan now runs **per classified trade** inside the T&S loop rather than once per
bar update. On a high-activity MES session this will be called hundreds of times per second.

At `MAX_BUCKETS = 2048` with typical active bucket counts of 50–200, the worst-case scan
cost per trade is ~2048 integer comparisons. On modern hardware this is not a bottleneck,
but it is O(n) where it was previously effectively O(1 per bar).

**v3 implementation: keep the linear scan.** It is correct and the constant factor is small.
If profiling shows it is a real cost, the fix is a hash map keyed on `bin_ticks`. Do not
optimise prematurely — the linear scan has zero bug risk and MAX_BUCKETS provides a hard
upper bound.

### Percentile Index — Worked Example

With `N = 10` qualified buckets and `HighlightPercentile = 95`:

```
idx = floor(0.95 * (10 - 1)) = floor(8.55) = 8
```

Scores sorted ascending: `[s0, s1, s2, s3, s4, s5, s6, s7, s8, s9]`

Threshold = `s8` (second-highest value). Correct behaviour at minimum qualifying
count — the top value is only selected if it is unambiguously the single outlier.

At `N = 20` and `p = 95`:

```
idx = floor(0.95 * 19) = floor(18.05) = 18
```

Threshold = `s18`. Stable and sensible across N.
