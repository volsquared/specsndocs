# LaunchPad Resting Liquidity Interaction Engine
## Merged Specification — V1

Version: 1.0  
Status: Implementation-ready draft  
Baseline study: `K_LaunchPadLevels_V3.cpp` (per-level aggressive flow engine)  
Output file: `K_RestingLiquidityInteraction_V1.cpp`

---

## Design Philosophy

This study answers one question:

> **How is resting liquidity responding under stress?**

Not: "How big is the wall?"

Static DOM size is low-confidence signal. Liquidity behaviour under aggression is the
measurable phenomenon. Everything in this spec follows from that distinction.

The study is the passive-liquidity counterpart to the Launchpad aggressive flow engine.
The two studies share bucket geometry but maintain independent state. Neither owns the
other's data.

---

## Honest Inference Boundaries

The study operates on sampled Level 2 DOM state via ACSIL APIs. It cannot:

- distinguish order cancellations from fills
- detect hidden/iceberg liquidity
- reconstruct true queue position or priority
- identify participant intent
- detect spoofing with certainty
- replicate matching-engine event stream accuracy

All outputs are **probabilistic behavioural inferences** derived from observable DOM state
transitions. State labels are lightweight interpretations of raw scores. The raw scores
are the ground truth — always expose them.

---

## ACSIL Update Architecture

### The Fundamental Ceiling

`sc.UpdateAlwaysInRealTime = 1` gives sampled DOM snapshots, not a true event stream.
Rapid pull-and-refill within the sample window (e.g. a 200ms pull-and-return on NQ)
may be invisible. This cannot be engineered around in standard ACSIL. The study is
**behaviourally approximate**, not matching-engine accurate. Document this in the HUD.

### Snapshot Model

```cpp
// In SetDefaults:
sc.UpdateAlwaysInRealTime = 1;
sc.SetUseMarketDepthPullingStackingData(true);  // REQUIRED — returns zero without this

// Throttle via persistent timestamp:
SCDateTime& last_snapshot_time = sc.GetPersistentSCDateTime(1);
SCDateTime   now               = sc.CurrentSystemDateTime;

float elapsed_ms = (now - last_snapshot_time).GetTimeInSeconds() * 1000.0f;
if (elapsed_ms < SnapshotIntervalMS) return;  // skip this update

last_snapshot_time = now;
// ... proceed with DOM snapshot acquisition
```

Recommended defaults:

| Mode        | `SnapshotIntervalMS` |
|-------------|----------------------|
| Aggressive  | 100ms                |
| Default     | 250ms                |
| Lightweight | 500ms                |

### CPU Discipline — Hard Limits

Explicitly constrain all DOM scanning to avoid CPU furnace conditions on NQ during
volatility, replay acceleration, or multi-chart setups:

| Parameter                  | Default | Purpose                                      |
|----------------------------|---------|----------------------------------------------|
| `MaxDOMDepthLevels`        | 10      | Max DOM levels scanned per side per update   |
| `MaxActiveBuckets`         | 64      | Max buckets in interaction engine             |
| `SnapshotIntervalMS`       | 250     | Minimum ms between full DOM snapshots        |
| `MaxDistanceTicks`         | 40      | Only process buckets within N ticks of price |

Do not scan the full book. Do not process every DOM update. Do not allocate on
the hot path.

---

## Bucket Architecture

### Geometry

Bucket size and alignment must match Launchpad V3 by default but remain
independently configurable:

```cpp
int bucket_size_ticks = i_BucketSizeTicks;  // default: match Launchpad input
int bin_ticks = (price_ticks / bucket_size_ticks) * bucket_size_ticks;
```

### LiquidityBucket Struct

```cpp
struct LiquidityBucket {
    int   bin_ticks;
    bool  active;

    // --- Layer 1: Raw Measurements (primary outputs) ---

    // Current DOM state
    float bid_depth;               // current resting bid size in bucket
    float ask_depth;               // current resting ask size in bucket
    int   bid_num_orders;          // number of resting bid orders
    int   ask_num_orders;          // number of resting ask orders

    // Depth history for rolling median
    // Buffer size controlled by DepthHistorySize input (default 64).
    // At 250ms snapshot interval, 64 entries = ~16 seconds of history.
    // 16 entries (4 seconds) is too shallow — captures microstructure turbulence
    // rather than meaningful liquidity persistence.
    // Use heap allocation via sc.AllocateMemory at init; MAX = 128 entries.
    float* bid_depth_history;      // circular buffer of recent bid snapshots (heap)
    float* ask_depth_history;      // circular buffer of recent ask snapshots (heap)
    int   depth_history_idx;       // current write position
    int   depth_history_count;     // number of valid entries (max = DepthHistorySize)
    int   depth_history_size;      // actual allocated size (set at init from input)

    // Derived depth baselines
    float bid_depth_median;        // rolling median of bid_depth_history
    float ask_depth_median;        // rolling median of ask_depth_history

    // Stack/pull (bucket-aggregated across all ticks in bucket)
    float bid_sp_bucket;           // sum of GetBidMarketDepthStackPullValueAtPrice
                                   // across all ticks in this bucket
    float ask_sp_bucket;           // sum of GetAskMarketDepthStackPullValueAtPrice
    SCDateTime sp_window_start;    // when current SP window began

    // Replenishment tracking
    float replenishment_ratio;                    // depth_after_attack / depth_before_attack
    float replenishment_strength_trend;           // slope of successive replenishment_ratio values
                                                  // positive = each reload stronger than last
                                                  // negative = replenishment fatigue — each reload weaker
                                                  // the decay sequence (300→250→180→120→collapse)
                                                  // is captured here as increasingly negative trend
    float replenishment_ratio_history[8];         // last 8 replenishment ratios for trend computation
    int   replenishment_ratio_history_idx;
    float replenishment_velocity;                 // depth recovered per second
    float replenishment_velocity_post_effective;  // velocity after price-moving aggression
    float replenishment_velocity_post_ineffective;// velocity after absorbed aggression
    float replenishment_half_life_ms;             // ms to recover 50% after depletion
    float replenishment_decay_slope;              // rate of change of depth during recovery
                                                  // positive = reloading, negative = bleeding out
    int   replenishment_count;                    // times depth recovered meaningfully this window
    int   failed_recovery_count;                  // raw count: times recovery threshold NOT achieved
    float failed_recovery_score;                  // decayed score: count with time decay applied
                                                  // use this in collapse_score, not raw count
                                                  // raw count alone is poisoned by stale failures
                                                  // e.g. one failure 20 minutes ago should not
                                                  // contribute equally to a failure 5 seconds ago
    SCDateTime last_failed_recovery_time;
    SCDateTime last_replenishment_time;
    float pre_attack_depth;                       // snapshot before aggression arrived
    float post_attack_depth;                      // snapshot when aggression peaked
    float shock_ratio;                            // depth_removed / rolling_median_depth
                                                  // normalised shock — use instead of absolute thresholds
    // [OPTIONAL V1] Effective vs ineffective aggression tagging
    // This concept is valid but increases calibration complexity.
    // Mark as optional — may be cut if signal-to-complexity ratio is poor.
    // The core absorption_score + net_displacement_ticks may be sufficient.
    bool  last_aggression_effective;              // true if net_displacement_ticks > EffectiveThreshold
    float replenishment_velocity_post_effective;  // velocity after price-moving aggression
    float replenishment_velocity_post_ineffective;// velocity after absorbed aggression
    float estimated_fill_rate;                    // traded_vol_at_level / dt
    float estimated_cancellation_rate;            // (depth_drop - traded_vol_at_level) / dt
    float fill_to_cancel_ratio;                   // fill_rate / cancel_rate (LOW-CONFIDENCE)
                                                  // research metric only — not a state driver

    // Absorption tracking
    float aggression_at_level;     // from Launchpad V3 bucket buy_vol or sell_vol
    float net_displacement_ticks;  // price movement since first aggression at level
    SCDateTime absorption_start;   // when sustained aggression first began
    float absorption_duration_ms;  // how long tested-without-displacement has held

    // Vacuum tracking
    float vacuum_score;            // thinness of book ahead of price (0–1)
    float depth_decay_rate;        // rate of depth loss per second

    // Lifecycle
    SCDateTime first_seen;
    SCDateTime last_seen;
    int   touches;                 // distinct visits (debounced)

    // --- Layer 2: Derived scores (computed from Layer 1) ---
    float replenishment_score;     // 0–1, primary signal
    float persistence_score;       // 0–1
    float collapse_score;          // 0–1
    float absorption_score;        // 0–1

    // Lifecycle state (derived from scores — see Section 7)
    int   interaction_state;       // IDLE / BUILDING / TESTED / ABSORBING /
                                   // WEAKENING / COLLAPSING / VACUUM / RELOADING

    bool  active_interaction;
};
```

---

## DOM Snapshot Update (per throttled interval)

For each active bucket within `MaxDistanceTicks` of current price:

### Step 1 — Acquire current DOM depth

Iterate DOM levels on both sides, accumulate into bucket:

```cpp
float new_bid_depth = 0.0f;
float new_ask_depth = 0.0f;
int   new_bid_orders = 0;
int   new_ask_orders = 0;

// Scan ask side up to MaxDOMDepthLevels
for (int level = 0; level < MaxDOMDepthLevels; level++) {
    float level_price;
    int   level_qty;
    int   level_orders;
    if (!sc.GetAskMarketDepthEntryAtLevel(level, level_price, level_qty)) break;

    int level_ticks = static_cast<int>(roundf(level_price / effective_tick));
    int level_bin   = (level_ticks / bucket_size_ticks) * bucket_size_ticks;
    if (level_bin == bucket->bin_ticks) {
        new_ask_depth  += (float)level_qty;
        new_ask_orders += level_orders;
    }
}
// Mirror for bid side using sc.GetBidMarketDepthEntryAtLevel()
```

### Step 2 — Update rolling depth history and median

```cpp
// depth_history_size set from DepthHistorySize input at bucket creation (default 64)
bucket->ask_depth_history[bucket->depth_history_idx] = new_ask_depth;
bucket->depth_history_idx = (bucket->depth_history_idx + 1) % bucket->depth_history_size;
bucket->depth_history_count = min(bucket->depth_history_count + 1, bucket->depth_history_size);

// Compute median (sort copy of valid entries)
bucket->ask_depth_median = compute_median(
    bucket->ask_depth_history, bucket->depth_history_count);

// Persistence score: how stable is depth relative to its own median
float depth_ratio = (bucket->ask_depth_median > 0.001f)
    ? new_ask_depth / bucket->ask_depth_median : 1.0f;
bucket->persistence_score = clamp(depth_ratio, 0.0f, 1.0f);
// Note: use median, NOT historical max. Max is unstable — one spoof spike
// poisons the reference for the entire session.
```

### Step 3 — Bucket-aggregated SP

Do NOT query a single tick price. Aggregate across all ticks in the bucket:

```cpp
bucket->ask_sp_bucket = 0.0f;
for (int tick = bucket->bin_ticks;
     tick < bucket->bin_ticks + bucket_size_ticks;
     tick++) {
    float tick_price = tick * effective_tick;
    bucket->ask_sp_bucket += sc.GetAskMarketDepthStackPullValueAtPrice(tick_price);
}
// Mirror for bid side
```

### Step 4 — SP time-window reset

Do NOT reset per bar. Use a time-window:

```cpp
float sp_elapsed_sec = (now - bucket->sp_window_start).GetTimeInSeconds();
if (sp_elapsed_sec >= SPDecayWindowSeconds) {
    sc.ClearMarketDepthPullingStackingData();
    bucket->sp_window_start = now;
    bucket->ask_sp_bucket = 0.0f;
    bucket->bid_sp_bucket = 0.0f;
}
```

Default `SPDecayWindowSeconds = 30.0f`. This is market-behaviour-semantic,
not chart-structure-semantic. On Renko especially, per-bar reset is wrong
because bar formation is price-driven, not time-driven.

### Step 4b — Attack Segmentation State Machine

**This is the most critical lifecycle definition in the spec.** Without explicit
attack start/end detection, the replenishment metrics (half-life, decay slope,
failed recovery) become unstable — they may accumulate across multiple separate
interactions or reset prematurely.

An "attack" is defined as a sustained period of aggressive flow at a bucket level.

```cpp
// Attack state per bucket (add to LevelBucket struct):
// enum AttackPhase { ATTACK_IDLE, ATTACK_ACTIVE, ATTACK_RECOVERY, ATTACK_FAILED };
// int attack_phase;                   // current phase
// SCDateTime attack_start_time;       // when attack began
// SCDateTime attack_end_time;         // when aggression subsided
// float attack_peak_aggression;       // max aggression seen during attack
// float attack_entry_depth;           // depth at attack_start_time
// int   attack_snapshot_count;        // snapshots elapsed since attack start

// --- Attack Segmentation Rules ---

// ATTACK_IDLE → ATTACK_ACTIVE:
//   aggression_at_level >= AggressionThreshold
//   AND depth dropping (new_depth < prev_depth * (1.0f - ShockRatioThreshold))

// ATTACK_ACTIVE → ATTACK_RECOVERY:
//   aggression_at_level < AggressionThreshold
//   for at least AttackCooldownSnapshots consecutive snapshots
//   (default: 4 snapshots = 1 second at 250ms)
//   This prevents a brief lull mid-attack from resetting state.

// ATTACK_RECOVERY → ATTACK_IDLE (success):
//   depth recovered to >= attack_entry_depth * RecoveryFractionTarget
//   → triggers replenishment_count++, half_life recording, ratio tracking

// ATTACK_RECOVERY → ATTACK_FAILED:
//   recovery window elapsed (RecoveryWindowMS) without reaching target
//   → triggers failed_recovery_count++, failed_recovery_score update
//   → then resets to ATTACK_IDLE

// ATTACK_ACTIVE → ATTACK_FAILED (direct collapse):
//   depth drops below CollapseDepthFraction * attack_entry_depth
//   while still under active aggression
//   → immediate collapse signal without waiting for recovery window
```

**Calibration sensitivity — highest-impact knob in the study:**
`AggressionThreshold` now controls attack segmentation entry, which in turn
controls replenishment tracking, half-life, failed recovery, and collapse timing.
If this threshold is miscalibrated:
- Too low → constant false attack activations, metrics become noise
- Too high → real attacks not detected, study is blind

Recommended calibration approach: start with the same `AggressionThreshold` as
Launchpad V3's `i_LPSThreshold`. Verify on 3 replay sessions that attack
activations visually align with obvious aggressive sweeps before tuning further.

**Why this matters:** Without this state machine:
- A slow grind attack split across 10 snapshots triggers 10 replenishment events
- A quiet period mid-attack resets `pre_attack_depth` to a wrong baseline
- `replenishment_half_life_ms` is measured from the wrong start point

The attack state machine is the foundation. Replenishment metrics are meaningless
without it being correct.

### Step 5 — Replenishment detection

```cpp
float prev_depth = bucket->ask_depth;  // last snapshot value
float depth_drop = prev_depth - new_ask_depth;

// Normalised shock — use rolling median, not absolute threshold.
// Absolute thresholds break across volatility regimes.
if (bucket->ask_depth_median > 0.001f) {
    bucket->shock_ratio = depth_drop / bucket->ask_depth_median;
}

// Trigger pre-attack recording on normalised shock
if (bucket->shock_ratio > ShockRatioThreshold) {
    bucket->pre_attack_depth = prev_depth;
}

float recovery = new_ask_depth - bucket->post_attack_depth;
// RecoveryFractionTarget is a configurable input (default 0.5)
    // Do not hardcode — MES and NQ may have different natural recovery profiles

if (bucket->pre_attack_depth > 0) {

    if (recovery >= bucket->pre_attack_depth * ReplenishmentRecoveryRatio) {
        // Full recovery achieved
        float dt = (now - bucket->last_replenishment_time).GetTimeInSeconds();
        if (dt > 0.001f) {
            bucket->replenishment_velocity = recovery / dt;
            // Tag velocity by aggression type
            if (bucket->last_aggression_effective)
                bucket->replenishment_velocity_post_effective   = bucket->replenishment_velocity;
            else
                bucket->replenishment_velocity_post_ineffective = bucket->replenishment_velocity;
        }
        bucket->replenishment_count++;
        bucket->last_replenishment_time = now;
        bucket->replenishment_ratio     = new_ask_depth / bucket->pre_attack_depth;

        // Track replenishment fatigue via successive ratio trend
        // Captures: 300→250→180→120 (negative trend = passive exhaustion)
        bucket->replenishment_ratio_history[bucket->replenishment_ratio_history_idx] =
            bucket->replenishment_ratio;
        bucket->replenishment_ratio_history_idx =
            (bucket->replenishment_ratio_history_idx + 1) % 8;
        int n_ratios = min(bucket->replenishment_count, 8);
        if (n_ratios >= 3) {
            // Simple slope over last N ratio values
            float sum_x=0,sum_y=0,sum_xy=0,sum_xx=0;
            for (int i = 0; i < n_ratios; i++) {
                int idx = (bucket->replenishment_ratio_history_idx - 1 - i + 8) % 8;
                float x = (float)i, y = bucket->replenishment_ratio_history[idx];
                sum_x+=x; sum_y+=y; sum_xy+=x*y; sum_xx+=x*x;
            }
            float d = n_ratios*sum_xx - sum_x*sum_x;
            bucket->replenishment_strength_trend = (fabsf(d) > 0.001f)
                ? (n_ratios*sum_xy - sum_x*sum_y) / d : 0.0f;
            // Negative trend = replenishment fatigue = passive exhaustion
            // More actionable than replenishment existence alone
        }

    } else if (new_ask_depth >= bucket->pre_attack_depth * RecoveryFractionTarget &&
               bucket->replenishment_half_life_ms < 0.001f) {
        // First time crossing RecoveryFractionTarget — record half-life
        float elapsed_ms = (now - bucket->last_replenishment_time).GetTimeInSeconds() * 1000.0f;
        bucket->replenishment_half_life_ms = elapsed_ms;

    } else if (/* timeout — recovery window expired without recovery */ false) {
        // Recovery failed — increment failure count
        // Trigger when elapsed_ms > RecoveryWindowMS and recovery < threshold
        bucket->failed_recovery_count++;
        bucket->last_failed_recovery_time = now;
        // failed_recovery_count is the primary vacuum precursor signal.
        // BUT use failed_recovery_score (decayed) in collapse_score,
        // not the raw count. A failure 20 minutes ago should not equal
        // a failure 5 seconds ago.
        bucket->replenishment_half_life_ms = 0.0f;  // reset for next attack
    }
}

// Replenishment decay slope — computed over depth_history circular buffer.
// Positive = recovering. Negative = bleeding out.
// Simple linear regression over last N valid history entries:
if (bucket->depth_history_count >= 4) {
    float sum_x = 0, sum_y = 0, sum_xy = 0, sum_xx = 0;
    int n = min(bucket->depth_history_count, 8);
    for (int i = 0; i < n; i++) {
        int idx = (bucket->depth_history_idx - 1 - i + 16) % 16;
        float x = (float)i;
        float y = bucket->ask_depth_history[idx];
        sum_x += x; sum_y += y; sum_xy += x * y; sum_xx += x * x;
    }
    float denom = n * sum_xx - sum_x * sum_x;
    bucket->replenishment_decay_slope = (fabsf(denom) > 0.001f)
        ? (n * sum_xy - sum_x * sum_y) / denom
        : 0.0f;
    // Positive slope = depth recovering over recent snapshots.
    // Negative slope = depth still falling = wall bleeding out.
}

// Update failed_recovery_score with time decay
// Recent failures weight heavily; old failures decay away
if (bucket->failed_recovery_count > 0 && bucket->last_failed_recovery_time != 0) {
    float failure_age_sec = (now - bucket->last_failed_recovery_time).GetTimeInSeconds();
    float decay_factor    = expf(-logf(2.0f) * failure_age_sec / FailureDecayHalfLifeSec);
    bucket->failed_recovery_score = clamp(
        (float)bucket->failed_recovery_count * decay_factor /
        (float)CollapseFailureThreshold,
        0.0f, 1.0f);
}

// Update collapse_score — driven by decayed failed_recovery_score, not raw count
bucket->collapse_score = bucket->failed_recovery_score;

// Replenishment score: weighted combination of count, ratio, velocity, half-life
float half_life_factor = (bucket->replenishment_half_life_ms > 0.001f)
    ? clamp(1.0f - bucket->replenishment_half_life_ms / ReplenishmentHalfLifeRef, 0.0f, 1.0f)
    : 0.0f;

bucket->replenishment_score = clamp(
    0.30f * min((float)bucket->replenishment_count / 3.0f, 1.0f) +
    0.30f * clamp(bucket->replenishment_ratio, 0.0f, 1.0f) +
    0.20f * clamp(bucket->replenishment_velocity / ReplenishmentVelocityRef, 0.0f, 1.0f) +
    0.20f * half_life_factor,
    0.0f, 1.0f);
```

`replenishment_score` is the primary signal. `failed_recovery_count` is the primary
vacuum precursor. Both are Tier 1.

**Recovery shape interpretation:**
- `replenishment_decay_slope > 0` = depth recovering → reloading wall
- `replenishment_decay_slope < 0` = depth still falling → wall bleeding out
- Instant large positive slope after depletion = strong passive interest / iceberg-like behaviour
- Gradual negative slope despite no further aggression = passive exhaustion

### Step 6 — Absorption measurement (time-dimension included)

Requires Launchpad V3 `buy_vol` per bucket. Pass via shared persistent pointer
or inter-study subgraph reference:

```cpp
float aggression = launchpad_bucket->buy_vol;  // or sell_vol for sell-side

if (aggression >= AggressionThreshold && bucket->absorption_start == 0) {
    bucket->absorption_start = now;
}

if (bucket->absorption_start != 0) {
    bucket->absorption_duration_ms =
        (now - bucket->absorption_start).GetTimeInSeconds() * 1000.0f;

    bucket->net_displacement_ticks =
        fabsf(current_price - bucket->first_aggression_price) / effective_tick;

    // Absorption score: aggression present, price not moving, sustained
    float disp_factor = clamp(1.0f - bucket->net_displacement_ticks /
                              MaxAbsorptionDisplacementTicks, 0.0f, 1.0f);
    float time_factor = clamp(bucket->absorption_duration_ms / 5000.0f, 0.0f, 1.0f);

    bucket->absorption_score = clamp(
        0.5f * disp_factor + 0.5f * time_factor, 0.0f, 1.0f);
}
```

Note: `absorption_duration_ms` is first-class. 200ms stall vs 15 second stall
are materially different events. Without time, absorption is incomplete.

### Step 7 — Vacuum scoring

```cpp
// Vacuum: thin book ahead of price in the direction of pressure
float book_depth_ahead = 0.0f;
int   levels_checked   = 0;

for (int level = 0; level < VacuumScanLevels && level < MaxDOMDepthLevels; level++) {
    float lp; int lq;
    if (!sc.GetAskMarketDepthEntryAtLevel(level, lp, lq)) break;
    book_depth_ahead += (float)lq;
    levels_checked++;
}

float expected_depth  = bucket->ask_depth_median * VacuumScanLevels;
bucket->vacuum_score  = (expected_depth > 0.001f)
    ? clamp(1.0f - book_depth_ahead / expected_depth, 0.0f, 1.0f)
    : 0.0f;

// Decay rate
float dt = (now - bucket->last_seen).GetTimeInSeconds();
bucket->depth_decay_rate = (dt > 0.001f)
    ? (prev_depth - new_ask_depth) / dt : 0.0f;

bucket->ask_depth = new_ask_depth;
bucket->last_seen = now;
```

---

## Layer 1 — Raw Score Outputs

These are always exposed. They are the ground truth. Never bury these under labels.

| Score | Range | Meaning |
|---|---|---|
| `replenishment_score` | 0–1 | Primary signal — repeated depth recovery under attack |
| `replenishment_velocity` | float | Depth recovered per second |
| `replenishment_count` | int | Times depth recovered in current window |
| `absorption_score` | 0–1 | Aggression present, price not moving, sustained |
| `absorption_duration_ms` | float | Duration of tested-without-displacement |
| `net_displacement_ticks` | float | Price movement since aggression began |
| `persistence_score` | 0–1 | Depth stability vs rolling median |
| `vacuum_score` | 0–1 | Book thinness ahead of price |
| `depth_decay_rate` | float | Rate of depth loss per second |
| `ask_sp_bucket` | float | Bucket-aggregated stacking (+) or pulling (-) |
| `bid_sp_bucket` | float | Same for bid side |
| `replenishment_half_life_ms` | float | ms to recover 50% after depletion — fast = strong defence |
| `replenishment_decay_slope` | float | Positive = recovering, negative = bleeding out |
| `failed_recovery_count` | int | Primary vacuum precursor — feeds collapse_score |
| `shock_ratio` | float | Normalised depth removal vs rolling median |
| `fill_to_cancel_ratio` | float | High = wall consumed, low = wall running |
| `fragility_score` | float | Weighted blend of vacuum and failed replenishment — breakout vs fade |
| `replenishment_strength_trend` | float | Slope of successive reload ratios — negative = passive exhaustion |
| `failed_recovery_score` | float | Time-decayed failure count — recent failures weight more than old |

---

## Layer 2 — Interaction Lifecycle State

States are derived lightly from Layer 1 scores. They do NOT replace the scores.
Do not add logic that is only expressible via state labels — keep state derivation
simple and falsifiable.

**`fragility_score` — from Fed paper:**
```cpp
// Weighted blend — not purely multiplicative.
// Multiplication suppresses intermediate states too aggressively.
// Example: vacuum=0.9, replenishment=0.6 → product=0.36, blend=0.70 (more accurate)
fragility_score =
    0.6f * vacuum_score +
    0.4f * (1.0f - replenishment_score);
```
High fragility = thin book ahead AND slow/failed replenishment = breakout candidate.
Low fragility despite thin book = providers absorbing and reloading = fade candidate.
This single derived metric is the most direct answer to the breakout vs fade question.

### Lifecycle Sequence

```text
IDLE → BUILDING → TESTED → ABSORBING → WEAKENING → COLLAPSING → VACUUM → RELOADING
                                  ↑                                         ↓
                                  └─────────────────────────────────────────┘
```

A level can absorb, then weaken, then collapse. The evolution itself is signal —
not just the current state.

### State Derivation

```cpp
int derive_state(LiquidityBucket* b) {
    if (b->vacuum_score > VacuumThreshold)
        return STATE_VACUUM;

    if (b->collapse_score > CollapseThreshold)
        return STATE_COLLAPSING;

    if (b->ask_sp_bucket < -PullingThreshold && b->aggression_at_level > 0)
        return STATE_WEAKENING;

    if (b->absorption_score > AbsorptionThreshold)
        return STATE_ABSORBING;

    if (b->aggression_at_level >= AggressionThreshold &&
        b->replenishment_score < 0.2f)
        return STATE_TESTED;

    if (b->replenishment_count > 0 && b->absorption_score < 0.3f)
        return STATE_RELOADING;

    if (b->ask_sp_bucket > StackingThreshold)
        return STATE_BUILDING;

    return STATE_IDLE;
}
```

### Naming Convention

Use behavioural names. Not narrative names.

| State | Preferred label | Avoid |
|---|---|---|
| ABSORBING | `HIGH_REPLENISHMENT_RESISTANCE` | `IRON_DOOR` |
| COLLAPSING | `DEPTH_COLLAPSING` | `WALL_BROKEN` |
| VACUUM | `LIQUIDITY_VACUUM` | `CLEAR_PATH` |
| BUILDING | `PASSIVE_BUILDING` | `WALL_FORMING` |
| WEAKENING | `PULLING_UNDER_PRESSURE` | `FAKE_WALL` |

`IRON_DOOR` is psychologically dangerous. Traders will treat it as conviction.
The study cannot convict. Use names that describe observable behaviour only.

---

## Four-Pillar Architecture

The signal set organises into four coherent pillars. Do not add signals that
do not clearly belong to one of these four. That is the complexity cliff guard.

| Pillar | Signals | Core question |
|---|---|---|
| **Resiliency** | `replenishment_half_life_ms`, `replenishment_velocity`, `replenishment_count` | How fast does liquidity recover? |
| **Exhaustion** | `replenishment_decay_slope`, `replenishment_strength_trend`, `failed_recovery_score` | Is defence weakening over time? |
| **Fragility** | `vacuum_score`, `fragility_score`, `shock_ratio` | How thin is the book and how dependent on replenishment? |
| **Displacement** | `absorption_score`, `absorption_duration_ms`, `net_displacement_ticks`, `replenishment_velocity_post_effective` | Did aggression move price or get absorbed? |

---

## Tier Priority (Research Value)

| Tier | Signal | Why |
|---|---|---|
| 1 | `replenishment_score` + `failed_recovery_count` under sustained aggression | Primary signal — persistent replenishment vs collapse |
| 1 | `replenishment_half_life_ms` | Objective, comparable across sessions and instruments |
| 1 | `replenishment_decay_slope` | Recovery shape — reloading vs bleeding out |
| 1 | `fragility_score` | Thin book + slow replenishment = breakout candidate |
| 2 | `vacuum_score` transitions | Absence of liquidity often more actionable than presence |
| 2 | `fill_to_cancel_ratio` | Distinguishes wall being consumed from wall running |
| 3 | `absorption_score` with `absorption_duration_ms` | Failed displacement under timed aggression |
| 3 | `replenishment_velocity_post_effective` | Elevated = strongest absorption signal |
| 4 | Static depth / wall size alone | Low confidence without interaction context |

Do not invert this priority. Static wall size is Tier 4.

---

## Launchpad V3 Integration

The two studies share bucket geometry. Integration is optional for V1.

Interaction pattern examples:

| Launchpad V3 signal | Resting liquidity signal | Combined read |
|---|---|---|
| High buy LPS | `replenishment_score` high on ask | Buyers aggressive, sellers replenishing — absorption |
| High buy LPS | `vacuum_score` high on ask | Ask book thin — potential expansion |
| High buy LPS | `ask_sp_bucket` strongly negative | Ask pulling — resistance clearing |
| High sell LPS | `replenishment_score` high on bid | Sellers aggressive, buyers defending |

---

## Replay Degradation Map

| Signal | Live | Replay with depth data | Replay without depth |
|---|---|---|---|
| `replenishment_score` | ✅ | ✅ (if depth recorded) | ❌ |
| `absorption_score` | ✅ | ✅ | ❌ |
| `absorption_duration_ms` | ✅ | ✅ | ❌ |
| `vacuum_score` | ✅ | ✅ | ❌ |
| `ask_sp_bucket` / `bid_sp_bucket` | ✅ | ⚠️ degraded | ❌ |
| `persistence_score` | ✅ | ✅ | ❌ |
| Launchpad V3 `buy_vol` / `sell_vol` | ✅ | ✅ | ✅ |

Stack/pull APIs are degraded in replay because the accumulator is live-state-dependent.
All other signals require `sc.GetMarketDepthBars()` with recorded historical depth.

---

## Input Parameters

| Input | Type | Default | Purpose |
|---|---|---|---|
| `Snapshot Interval MS` | Int | 250 | DOM update throttle |
| `Depth History Size` | Int | 64 | Rolling depth history entries (64 × 250ms = ~16 seconds) |
| `Max DOM Depth Levels` | Int | 10 | Levels scanned per side per snapshot |
| `Max Distance Ticks` | Int | 40 | Only process buckets within N ticks of price |
| `Bucket Size Ticks` | Int | match Launchpad | Aggregation granularity |
| `SP Decay Window Seconds` | Float | 30.0 | Stack/pull accumulator reset window |
| `Aggression Threshold` | Float | match Launchpad AbsoluteMinLPS | Minimum LPS to begin absorption tracking |
| `Max Absorption Displacement Ticks` | Int | 4 | Max price move to still count as absorbed |
| `Replenishment Drop Threshold` | Float | 0.30 | Fraction of median depth lost to trigger tracking |
| `Replenishment Recovery Ratio` | Float | 0.70 | Fraction recovered to count as replenishment |
| `Replenishment Velocity Ref` | Float | 50.0 | Reference speed (lots/sec) for score normalisation |
| `Vacuum Scan Levels` | Int | 5 | DOM levels ahead of price for vacuum scoring |
| `Vacuum Threshold` | Float | 0.70 | vacuum_score above this → VACUUM state |
| `Absorption Threshold` | Float | 0.60 | absorption_score above this → ABSORBING state |
| `Stacking Threshold` | Float | 50.0 | SP value above this → BUILDING state |
| `Pulling Threshold` | Float | 50.0 | SP value below this → WEAKENING state |
| `Shock Ratio Threshold` | Float | 0.30 | Normalised depth drop to trigger replenishment tracking |
| `Replenishment Half Life Ref` | Float | 2000.0 | Reference half-life ms for score normalisation (2 seconds) |
| `Recovery Window MS` | Float | 5000.0 | Max ms to wait for recovery before logging failure |
| `Collapse Failure Threshold` | Int | 3 | failed_recovery_count above this → collapse_score = 1.0 |
| `Effective Aggression Threshold` | Float | 2.0 | Net displacement ticks above this = effective aggression |
| `Recovery Fraction Target` | Float | 0.50 | Fraction of pre-attack depth to count as half-life crossing |
| `Failure Decay Half Life Sec` | Float | 120.0 | Half-life for failed_recovery_score decay (2 minutes) |

---

## Visual Output

### Philosophy

Expose raw scores. Derive visual states lightly. Prioritise proximity to price.
Do not replicate a DOM ladder. Do not add state labels the user cannot validate.

### Rendering

| Output | Type | Content |
|---|---|---|
| Bucket overlay | Horizontal line at bucket price | Colour = lifecycle state |
| Score HUD | Text panel | Raw Layer 1 scores for nearest 3 buckets |
| State label | Small text tag on line | Behavioural label (optional, off by default) |

### Colour scheme

| State | Colour |
|---|---|
| BUILDING | Blue |
| TESTED | Yellow |
| ABSORBING | Cyan (ask) / Amber (bid) |
| WEAKENING | Grey |
| COLLAPSING | Red |
| VACUUM | White |
| RELOADING | Green |
| IDLE | Not rendered |

HUD must always show raw scores, not just state labels. Without raw metrics,
debugging and calibration are impossible and the study becomes a black box.

---

## Known Limitations

| Limitation | Impact | Status |
|---|---|---|
| ACSIL is sampled, not event-stream | Rapid transient events may be missed | Fundamental ceiling — cannot be fixed |
| Stack/pull APIs degrade in replay | SP signals unavailable without live DOM | Document in HUD when in replay |
| Rolling median uses heap-allocated circular buffer (default 64 entries) | Requires sc.AllocateMemory at init; ~256 bytes per bucket at 64 entries | Acceptable at MAX_BUCKETS = 64 |
| Iceberg refill indistinguishable from genuine replenishment | replenishment_score may overstate on thin products | Known — Tier 4 validation |
| Heap-allocated depth history requires explicit cleanup | Memory leak on study reload if `sc.LastCallToFunction` cleanup omitted | See Memory Lifecycle section |
| Stale bucket state after price absence | Old attack state contaminates fresh visit if not reset | See Stale Bucket Reset Policy |
| Multi-bar persistence not tracked (c_ACSILDepthBars is current-bar) | historical_max not used — replaced by rolling median | Rolling median is more robust |
| `SetUseMarketDepthPullingStackingData` omission | All SP functions return zero silently | Hard requirement in SetDefaults |

---

## Complexity Cliff Warning

This spec currently defines:

- 15 Layer 1 raw scores
- 8 lifecycle states
- 16-entry depth history circular buffer
- 10 replenishment metrics (including half-life, decay slope, fatigue trend, decayed failure score, shock ratio)
- 6 attack segmentation fields (phase, start/end time, peak aggression, entry depth, snapshot count)
- 2 absorption metrics
- 1 derived fragility score
- Fill-to-cancel approximation
- Effective/ineffective aggression split

Adding further states, filters, or decays risks making the study unfalsifiable.
Before adding any new metric ask:

> **What measurable behaviour does this represent, and how do I validate it?**

If the answer requires reading the state labels to explain the metric, the metric
is likely redundant. Keep Layer 1 sparse and measurable. Keep Layer 2 thin.

---

## Primary Validation Hypothesis

Before expanding the study further, validate this hypothesis first in replay:

> **Does `fragility_score` rising + `replenishment_strength_trend` negative +
> `failed_recovery_score` increasing together precede impulsive price expansion
> or liquidity vacuum moves?**

This is the most promising signal combination in the spec. If it does not show
measurable predictive value in replay, the exhaustion/fragility framing needs
revision before adding further complexity.

Test protocol:
1. Identify 10 impulsive expansion moves in MES replay (> 8 ticks in < 30 seconds)
2. Check whether the above three signals were elevated in the 30–60 seconds prior
3. Compare against 10 range-bound periods where no expansion occurred
4. If hit rate < 60% on that comparison, the signal is noise at current calibration

Do this before building any execution logic on top of the study.

---

## Memory Lifecycle

Depth history buffers are heap-allocated via `sc.AllocateMemory`. Codex must
implement explicit cleanup or memory will leak on study reload, recalculation,
chart close, or settings change.

```cpp
// At init (sc.UpdateStartIndex == 0 or sc.IsFullRecalculation):
for (int i = 0; i < MAX_BUCKETS; i++) {
    if (p_Data->buckets[i].ask_depth_history == NULL) {
        p_Data->buckets[i].ask_depth_history =
            (float*)sc.AllocateMemory(DepthHistorySize * sizeof(float));
        p_Data->buckets[i].bid_depth_history =
            (float*)sc.AllocateMemory(DepthHistorySize * sizeof(float));
        p_Data->buckets[i].depth_history_size = DepthHistorySize;
        memset(p_Data->buckets[i].ask_depth_history, 0, DepthHistorySize * sizeof(float));
        memset(p_Data->buckets[i].bid_depth_history, 0, DepthHistorySize * sizeof(float));
    }
}

// At cleanup (sc.LastCallToFunction == true):
if (sc.LastCallToFunction) {
    for (int i = 0; i < MAX_BUCKETS; i++) {
        if (p_Data->buckets[i].ask_depth_history != NULL) {
            sc.FreeMemory(p_Data->buckets[i].ask_depth_history);
            p_Data->buckets[i].ask_depth_history = NULL;
        }
        if (p_Data->buckets[i].bid_depth_history != NULL) {
            sc.FreeMemory(p_Data->buckets[i].bid_depth_history);
            p_Data->buckets[i].bid_depth_history = NULL;
        }
    }
    return;
}
```

Also free and reallocate if `DepthHistorySize` input changes between calls
(check `p_Data->buckets[i].depth_history_size != DepthHistorySize`).

---

## Stale Bucket Reset Policy

When price moves away from a level for an extended period and returns, old attack
state must not contaminate the fresh visit.

**Policy: time-decay scores, full reset on expiry.**

```cpp
// On each snapshot, for each bucket:
float inactive_sec = (now - bucket->last_seen).GetTimeInSeconds();

if (inactive_sec > BucketResetTimeoutSec) {
    // Full reset — price was away long enough that old state is irrelevant
    bucket->attack_phase              = ATTACK_IDLE;
    bucket->replenishment_count       = 0;
    bucket->failed_recovery_count     = 0;
    bucket->failed_recovery_score     = 0.0f;
    bucket->replenishment_strength_trend = 0.0f;
    bucket->replenishment_half_life_ms = 0.0f;
    bucket->absorption_start          = SCDateTime();
    bucket->absorption_duration_ms    = 0.0f;
    // Keep depth history — it may still reflect valid persistence baseline
    // Reset attack-specific state only, not the depth baseline
} else if (inactive_sec > ScoreDecayStartSec) {
    // Partial decay — score interpolation toward zero
    float decay = 1.0f - (inactive_sec - ScoreDecayStartSec) /
                  (BucketResetTimeoutSec - ScoreDecayStartSec);
    bucket->fragility_score      *= decay;
    bucket->replenishment_score  *= decay;
    bucket->failed_recovery_score *= decay;
    bucket->vacuum_score         *= decay;
}
```

Recommended defaults:

| Parameter | Default | Meaning |
|---|---|---|
| `Score Decay Start Sec` | 30.0 | Begin score decay after 30s inactivity |
| `Bucket Reset Timeout Sec` | 120.0 | Full attack state reset after 2 min inactivity |

Depth history is retained across inactivity — it reflects the level's baseline
behaviour and should not be discarded just because price moved away temporarily.

---

## Implementation Order

1. Scaffold `LiquidityBucket` struct and persistent allocator
3. Implement DOM depth acquisition and bucket aggregation
4. Implement rolling median computation (circular buffer sort)
5. Implement bucket-aggregated SP with time-window reset
6. Implement replenishment detection and scoring
7. Implement absorption scoring with time dimension
8. Implement vacuum scoring
9. Derive lifecycle states from Layer 1 scores
10. Implement HUD with raw score display
11. Implement bucket line drawing with state colours
12. Wire optional Launchpad V3 integration

Do **not** implement persistence (CSV/binary) in V1. Validate signal utility first.
Building telemetry for unvalidated signals is a classic quant development trap.

---

## Validation Protocol (Post-Build)

Before trusting any output, run these three sessions in replay with depth data:

### Trend day
Expected: high `replenishment_score` on opposing side, state cycling
ABSORBING → WEAKENING → COLLAPSING in direction of trend

### Range day
Expected: symmetric ABSORBING on both sides, low `vacuum_score`,
`replenishment_count` elevated on both sides

### Session open (first 10 minutes)
Expected: no score spikes from thin book, `persistence_score` stable,
`vacuum_score` transitions visible around key levels

If behaviour does not match expectations, the metric is wrong before the signal
question even becomes relevant.

### First Live-Test Checklist

Use this checklist during the first live market session after build validation.

1. Opening sanity
   - Confirm lines appear only near current price.
   - Confirm HUD updates without unstable or obviously nonsensical values.
   - Confirm SP-driven states are not all zero if stacking/pulling data is available.

2. Obvious defense test
   - Find a level repeatedly hit without immediate displacement.
   - Expect:
     - `replenishment_score` rises
     - `absorption_score` rises if price does not move through
     - state leans `ABSORBING` or `RELOADING`
   - Reject if:
     - score stays dead while the level is clearly reloading
     - state flips randomly with no visible reason

3. Obvious failure test
   - Find a level that gets hit and stops reloading.
   - Expect:
     - `failed_recovery_score` increases
     - `fragility_score` rises
     - state transitions toward `WEAKENING` or `COLLAPSING`
   - Reject if:
     - collapse never appears even after clear depletion
     - replenishment stays high while the level is visibly failing

4. Expansion / vacuum test
   - Before a quick move, watch whether the ahead-side book thins.
   - Expect:
     - `vacuum_score` rises
     - `fragility_score` rises if replenishment is weak
   - Reject if:
     - vacuum stays flat during obvious thin-book expansion setups

5. Range test
   - In a rotational patch, watch both sides.
   - Expect:
     - both sides can show replenishment at different nearby levels
     - fewer collapse signals
     - more `ABSORBING` / `RELOADING`
   - Reject if:
     - everything constantly reads collapse or vacuum in a balanced range

6. Persistence / stale-state test
   - Let price leave a level and return later.
   - Expect:
     - old attack state should not contaminate the revisit after enough time
   - Reject if:
     - an old level resumes with stale high scores immediately on return

7. Replay/live caveat
   - If testing in replay, remember stack/pull is degraded.
   - Focus more on:
     - `replenishment_score`
     - `failed_recovery_score`
     - `fragility_score`
     - `absorption_score`
   - Be less strict about SP-based state behaviour.

Best things to record during the first test:
- timestamp
- visible level price
- displayed state
- HUD raw values
- what price did next

Even 3-5 clean examples are enough to start threshold calibration.
