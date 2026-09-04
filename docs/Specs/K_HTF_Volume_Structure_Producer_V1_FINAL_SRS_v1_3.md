# K_HTF_Volume_Structure_Producer_V1
## Final Software Requirements Specification v1.4

---

# 0. Revision Note

v1.4 updates the finalized v1.3 document to match implemented study behavior:
- HVN candidate construction now expands stronger peaks first and prevents weaker peaks from claiming stronger territory.
- HVN1/HVN2 selection now ranks accepted candidates by the strongest contiguous window of `Min HVN Cluster Buckets` containing the peak bucket, rather than by whole-cluster total volume alone.
- LVN is now generated only when both `HVN1` and `HVN2` exist and the LVN lies between them.
- Debug drawing now uses two-line zone brackets at zone boundaries instead of a single representative dash.
- Additional peak metadata and targeted per-bar debug inputs are documented.

v1.3 applies three final editorial fixes to v1.2:
- `GetNumberOfBars()` guard now specified as explicit mandatory code block.
- Section numbering corrected throughout (duplicate `# 2`, mismatched subsections `## 1.1`/`## 1.2` under `# 2`).
- Section 9 subsection order corrected: 9.1–9.3 precede 9.4.

Prior history: v1.1 applied five implementation fixes (break vs continue, getter convention, `GetNumberOfBars` guard acknowledgement, `new/delete` clarification, plateau edge-case rule). v1.2 added bucket construction container (`std::map`) and attempted section/guard fixes.

---

# 1. Purpose

`K_HTF_Volume_Structure_Producer_V1` is an ACSIL study that runs on a Higher Timeframe (HTF) chart and computes volume structure for each completed HTF bar using Sierra Chart Volume-at-Price (VAP) data.

The study produces reusable structural data:

- Primary High Volume Node cluster (`HVN1`)
- Secondary High Volume Node cluster (`HVN2`)
- Low Volume Node cluster(s) (`LVN`)
- Bucketed VAP distribution
- Same-price delta metadata
- Optional debug dashes on the HTF chart

This study is a **producer**, not a trading signal engine.

A separate LTF consumer study may later read the producer’s store and draw projected zones, rectangles, interaction states, or multi-HTF confluence.

---

# 2. Design Principles

## 2.1 Core Principle

> Volume defines structure. Delta describes behaviour.

Therefore:

- HVNs are detected primarily from total volume.
- Delta is stored as metadata.
- Delta-weighted ranking may exist as an optional mode, but is not the default.

---

## 2.2 Architectural Principles

- HTF producer computes on HTF chart only.
- Computation occurs only after HTF bar completion.
- Completed profile records are immutable.
- Consumer reads producer state but does not mutate it.
- V1 avoids diagonal delta.
- V1 avoids shape classification.
- V1 avoids full Market Profile/TPO logic.
- V1 avoids strategy signals.

---

# 3. Scope

## 3.1 In Scope

- VAP bucket construction
- HVN1 detection
- HVN2 detection
- LVN detection
- Same-price bid/ask delta storage
- Persistent pointer store
- Latest-profile PV bridge
- Optional HTF debug drawing
- Replay/full-recalc/live determinism

---

## 3.2 Out of Scope for V1

- Diagonal imbalance
- POC / VAH / VAL
- P / b / D / B profile shape classification
- Composite multi-period profile
- Multi-HTF stacking logic
- LTF interaction logic
- Entry/exit signals

These are V2+ concerns.

---

# 4. Sierra Chart / ACSIL Execution Model

## 4.1 Required Study Settings

The producer shall set:

```cpp
sc.AutoLoop = 0;
sc.MaintainVolumeAtPriceData = 1;
sc.GraphRegion = 0;
```

Manual looping is required.

---

## 4.2 Chart Placement

The study runs on the HTF chart being profiled.

Examples:

- 4H NQ chart
- Daily NQ chart
- RTH session chart, if separately constructed

The study does not require a lower timeframe chart.

---

## 4.3 Closed-Bar Processing Gate

The producer shall process only closed HTF bars.

On each call:

```text
last_closed_index = sc.ArraySize - 2
```

The forming bar at `sc.ArraySize - 1` shall never be profiled.

If:

```text
last_closed_index <= store.last_processed_bar_index
```

then no computation is performed.

If:

```text
last_closed_index > store.last_processed_bar_index
```

then process all unprocessed closed bars:

```text
store.last_processed_bar_index + 1
through
last_closed_index
```

This supports:

- live processing
- replay
- chart reload
- full recalculation
- missed update catch-up

---

## 4.4 Full Recalculation Behaviour

On full recalculation:

- clear existing store contents
- reset `last_processed_bar_index = -1`
- rebuild closed historical bars only
- do not process the current forming bar

The resulting profile sequence must match replay/live processing over identical chart data and inputs.

---

## 4.5 Last Call Cleanup

On `sc.LastCallToFunction`:

- free allocated store memory
- clear persistent pointer slot
- do not leave stale consumer-readable pointers

Because `VolumeStructureStore` contains STL containers, the producer shall use ordinary C++ object lifetime management consistently:

```cpp
auto* store = new VolumeStructureStore();
delete store;
```

Do not allocate this STL-containing object with `sc.AllocateMemory()` unless placement-new and explicit destructor calls are also implemented correctly.

`sc.AllocateMemory()` may be used later only for a POD/slab allocator design. V1 accepts producer-owned CRT allocation as a pragmatic trade-off.

---

# 5. Inter-Study Communication / IPC

## 5.1 Primary IPC Mechanism

The producer exposes its store through a persistent pointer.

Producer:

```cpp
static constexpr int PERSISTENT_SLOT_VOLUME_STORE = 9001;

auto* store = static_cast<VolumeStructureStore*>(
    sc.GetPersistentPointer(PERSISTENT_SLOT_VOLUME_STORE)
);
```

Consumer:

```cpp
auto* store = static_cast<VolumeStructureStore*>(
    sc.GetPersistentPointerFromChartStudy(
        ProducerChartNumber,
        ProducerStudyID,
        PERSISTENT_SLOT_VOLUME_STORE
    )
);
```

`GetPersistentPointerFromChartStudy` exists in Sierra Chart headers and is the intended direct mechanism for this use case.

---

## 5.2 Do Not Use Pointer-Address Integer Bridge

The producer shall not pass pointer addresses through `SetPersistentInt`.

Reason:

- `SetPersistentInt` is 32-bit.
- Sierra Chart runs on 64-bit Windows.
- Pointer truncation would corrupt the address and likely crash.

If address bridging is ever required in another context, it must use the `Int64` persistent API, but V1 does not require it.

---

## 5.3 Ownership Contract

The producer owns the store memory.

The consumer:

- may read the store
- must not free it
- must not mutate it
- must not retain profile/bucket pointers across producer updates unless the final memory model guarantees stability

---

## 5.4 Store Validation

The store shall include:

```text
magic_number
schema_version
producer_chart_number
producer_study_id
symbol
tick_size
timeframe_seconds
```

Consumer must validate:

- pointer is non-null
- magic number matches
- schema version is supported
- tick size is valid
- symbol/timeframe match expected inputs, if configured

If validation fails, consumer disables plotting and logs an error.

---

# 6. Memory Model

## 6.1 Practical Decision

V1 shall use a producer-owned persistent heap store.

Dynamic internal containers are acceptable **inside the producer-owned store** provided:

- producer owns allocation and destruction
- consumer treats data as read-only
- completed records are not mutated
- consumer does not free anything
- store access is validated by magic/schema fields

This is a pragmatic ACSIL choice.

A full POD/fixed-array shared-memory model is safer in theory but not required for V1 and can become wasteful.

---

## 6.2 Required Safeguards

The store must enforce:

```text
Max Profiles To Store
Max Buckets Per Profile
```

If a profile exceeds `Max Buckets Per Profile`, V1 shall skip that profile and set/log:

```text
PROFILE_FLAG_BUCKET_LIMIT_EXCEEDED
```

Skipping is preferred to silent truncation.

---

## 6.3 Recommended Defaults

```text
Max Profiles To Store = 200
Max Buckets Per Profile = 1000
Bucket Factor = 4
```

Rationale:

For NQ/MNQ/ES/MES, a 4H profile at 1-point buckets usually needs far fewer than 1000 buckets. This keeps memory sane while leaving room for high-volatility periods.

---

## 6.4 Memory Size Guidance

Approximate bucket count:

```text
bucket_count ≈ HTF_range_ticks / BucketFactor
```

Example:

NQ 4H range = 400 ticks  
BucketFactor = 4

```text
~100 buckets
```

NQ 4H extreme range = 1600 ticks  
BucketFactor = 4

```text
~400 buckets
```

Therefore `MaxBucketsPerProfile = 1000` is a practical default.

---

# 7. Data Structures

## 7.1 Constants

```cpp
static constexpr uint32_t VOLUME_STORE_MAGIC = 0x48545653; // 'HTVS'
static constexpr int HTVS_SCHEMA_VERSION = 1;
static constexpr int PERSISTENT_SLOT_VOLUME_STORE = 9001;
```

---

## 7.2 Enums

```cpp
enum class ZoneType : int
{
    Unknown = 0,
    HVN = 1,
    LVN = 2
};

enum class ShapeClass : int
{
    Unknown = 0,
    DShape = 1,
    PShape = 2,
    bShape = 3,
    BShape = 4
};

enum class ZoneRankingMode : int
{
    TotalVolume = 0,
    TotalVolumeThenAbsDelta = 1,
    AbsDeltaWeightedVolume = 2
};
```

V1 always sets:

```text
ShapeClass = Unknown
```

---

## 7.3 Profile Flags

```text
PROFILE_FLAG_VALID                  = 1 << 0
PROFILE_FLAG_ZERO_VOLUME             = 1 << 1
PROFILE_FLAG_VAP_UNAVAILABLE         = 1 << 2
PROFILE_FLAG_DELTA_UNAVAILABLE       = 1 << 3
PROFILE_FLAG_BUCKET_LIMIT_EXCEEDED   = 1 << 4
PROFILE_FLAG_HVN1_FOUND              = 1 << 5
PROFILE_FLAG_HVN2_FOUND              = 1 << 6
PROFILE_FLAG_LVN_FOUND               = 1 << 7
PROFILE_FLAG_DEBUG_DRAWN             = 1 << 8
```

---

## 7.4 VolumeBucket

Volume fields shall use `double`, matching Sierra Chart VAP volume fields.

```cpp
struct VolumeBucket
{
    int bucket_index = 0;

    int price_low_ticks = 0;
    int price_high_ticks = 0;
    int price_mid_ticks = 0;
    int peak_price_ticks = 0;

    double total_volume = 0.0;
    double bid_volume = 0.0;
    double ask_volume = 0.0;
    double max_volume_at_price = 0.0;

    double delta = 0.0;
    double abs_delta = 0.0;
    double delta_pct = 0.0;

    double ranking_score = 0.0;

    bool is_peak = false;
    bool is_low_volume = false;
    bool assigned_to_zone = false;
};
```

---

## 7.5 VolumeZone

```cpp
struct VolumeZone
{
    int zone_id = 0;
    ZoneType zone_type = ZoneType::Unknown;
    int rank = 0;
    int peak_bucket_position = -1;

    int low_ticks = 0;
    int high_ticks = 0;
    int peak_ticks = 0;

    double peak_volume = 0.0;
    double total_volume = 0.0;

    int bucket_start = 0;
    int bucket_end = 0;
    int bucket_count = 0;

    double relative_to_max_pct = 0.0;
    int width_ticks = 0;

    double delta = 0.0;
    double abs_delta = 0.0;
    double delta_pct = 0.0;
};
```

---

## 7.6 HTFVolumeProfileRecord

```cpp
struct HTFVolumeProfileRecord
{
    int profile_id = 0;
    int schema_version = HTVS_SCHEMA_VERSION;

    int chart_number = 0;
    int study_id = 0;

    char symbol[64] = {};

    int timeframe_seconds = 0;

    int bar_index = -1;
    SCDateTime start_datetime;
    SCDateTime end_datetime;

    int open_ticks = 0;
    int high_ticks = 0;
    int low_ticks = 0;
    int close_ticks = 0;

    double total_volume = 0.0;

    double tick_size = 0.0;
    int bucket_factor = 1;
    int bucket_size_ticks = 1;
    int bucket_count = 0;

    std::vector<VolumeBucket> buckets;
    std::vector<VolumeZone> hvn_zones;
    std::vector<VolumeZone> lvn_zones;

    ShapeClass shape_class = ShapeClass::Unknown;
    double shape_confidence = 0.0;

    uint32_t flags = 0;
};
```

Notes:

- `char symbol[64]` is preferred over `SCString` inside shared records.
- Vectors are owned by the producer store and read-only to consumers.
- If later instability appears, V2 may replace vectors with fixed arrays or externally managed slabs.

---

## 7.7 VolumeStructureStore

```cpp
struct VolumeStructureStore
{
    uint32_t magic_number = VOLUME_STORE_MAGIC;
    int schema_version = HTVS_SCHEMA_VERSION;

    int producer_chart_number = 0;
    int producer_study_id = 0;

    char symbol[64] = {};
    double tick_size = 0.0;
    int timeframe_seconds = 0;

    int max_profiles = 200;
    int max_buckets_per_profile = 1000;

    int last_processed_bar_index = -1;
    int current_profile_id = 0;

    std::deque<HTFVolumeProfileRecord> profiles;
};
```

Do not claim pointer stability for individual profile records. Consumers should resolve profiles by `profile_id` or copy needed data during their own calculation pass.

---

# 8. Inputs

## 8.1 Core Inputs

| Input | Type | Default | Constraint |
|---|---:|---:|---|
| Bucket Factor | int | 4 | >= 1 |
| Min HVN Cluster Buckets | int | 1 | >= 1 |
| HVN Adjacency Threshold % | float | 70.0 | 0–100 |
| Secondary HVN Minimum % Of HVN1 Total | float | 50.0 | 0–100 |
| LVN Threshold % Of Max Bucket | float | 20.0 | 0–100 |
| Min LVN Cluster Buckets | int | 1 | >= 1 |
| Max Profiles To Store | int | 200 | 10–2000 |
| Max Buckets Per Profile | int | 1000 | 100–10000 |
| Enable Debug Logging | Yes/No | No | - |

---

## 8.2 Delta Inputs

| Input | Type | Default | Constraint |
|---|---:|---:|---|
| Enable Delta Metrics | Yes/No | Yes | Requires bid/ask VAP |
| Zone Ranking Mode | enum | TotalVolume | See below |
| Delta Weight | float | 0.0 | 0.0–1.0 |

Ranking modes:

```text
0 = TotalVolume
1 = TotalVolumeThenAbsDelta
2 = AbsDeltaWeightedVolume
```

V1 does not support diagonal delta.

---

## 8.3 Debug Drawing Inputs

| Input | Type | Default |
|---|---:|---:|
| Draw HTF Debug Dashes | Yes/No | Yes |
| Draw HVN1 | Yes/No | Yes |
| Draw HVN2 | Yes/No | Yes |
| Draw LVN | Yes/No | Yes |
| HVN1 Color | Color | Green |
| HVN2 Color | Color | Blue |
| LVN Color | Color | Red |
| Dash Line Width | int | 2 |
| Draw Extension Mode | enum | CompletedBarOnly |
| Extend Bars | int | 1 |
| Max Profiles To Draw | int | 100 |
| Enable Detailed Bar Debug | Yes/No | No |
| Debug Target Trade Date (YYYYMMDD, 0=Any) | int | 0 |
| Debug Target Time | Time | 15:00:00 |

Extension modes:

```text
0 = CompletedBarOnly
1 = ExtendNBars
2 = ExtendRight
```

Producer-side drawing is a debug/validation exception. LTF projection remains the consumer’s job.

---

# 9. VAP Access

## 9.1 Required Guards

The following two checks MUST be executed before any VAP access for a given bar.

```cpp
if (sc.VolumeAtPriceForBars == nullptr)
    return; // VAP container not available

if (static_cast<int>(sc.VolumeAtPriceForBars->GetNumberOfBars()) < sc.ArraySize)
    return; // VAP data not yet populated for all bars
```

Both checks are mandatory. The second guard prevents `GetSizeAtBarIndex` and `GetVAPElementAtIndex` from being called on an undersized container during chart load, replay initialisation, or study startup.

---

## 9.2 Required API Pattern

For each processed HTF bar:

```cpp
const s_VolumeAtPriceV2* vap = nullptr;
int num_vap_elements =
    sc.VolumeAtPriceForBars->GetSizeAtBarIndex(bar_index);

for (int i = 0; i < num_vap_elements; ++i)
{
    if (!sc.VolumeAtPriceForBars->GetVAPElementAtIndex(bar_index, i, &vap))
        break;

    // vap->PriceInTicks
    // vap->GetVolume()
    // vap->GetBidVolume()
    // vap->GetAskVolume()
}
```

Exact signatures may vary by Sierra Chart version, but the implementation must use per-bar VAP elements.

---

## 9.3 VAP Price Representation

Use:

```text
price_in_ticks = vap->PriceInTicks
```

Do not compute bucket keys from floating price.

Floating price conversion is only for display:

```text
price = ticks * sc.TickSize
```

---

## 9.4 Bid/Ask Availability

If bid/ask volumes are unavailable or structurally zero while total volume exists:

- set `PROFILE_FLAG_DELTA_UNAVAILABLE`
- set delta fields to zero
- fall back to total-volume ranking
- keep total volume profile valid

---

# 10. Bucket Construction

## 10.1 Bucket Size

```text
bucket_size_ticks = BucketFactor
```

Example:

MES / NQ tick = 0.25  
BucketFactor = 4  
Bucket size = 1.00 point

---

## 10.2 Bucket Index Formula

```text
bucket_index = price_in_ticks / bucket_size_ticks
```

Integer division is intentional.

---

## 10.3 Bucket Boundary Convention

Buckets are closed integer tick intervals:

```text
bucket_low_ticks  = bucket_index * bucket_size_ticks
bucket_high_ticks = bucket_low_ticks + bucket_size_ticks - 1
```

A price exactly on a bucket boundary belongs to the bucket beginning at that boundary.

Example with bucket size 4:

```text
ticks 1000,1001,1002,1003 -> bucket 250
ticks 1004,1005,1006,1007 -> bucket 251
```

No overlap. No double-counting.

---

## 10.4 Bucket Construction Container

During profile construction, the intermediate bucket container SHALL be:

```cpp
std::map<int, VolumeBucket> bucket_map;
```

Rationale:
- Maintains automatic sorting by bucket_index
- Simplifies downstream processing
- Avoids separate sort pass

Alternative containers (unordered_map, vector indexing) are not used in V1 to preserve determinism and simplicity.

---

# 11. Ranking Score

## 11.1 Default

```text
ranking_score = total_volume
```

---

## 11.2 TotalVolumeThenAbsDelta

Sort by:

1. total_volume descending
2. abs_delta descending

This is tie-break only.

---

## 11.3 AbsDeltaWeightedVolume

Only active if delta is available and user selects it.

```text
abs_delta_pct = abs(delta) / total_volume
ranking_score = total_volume * (1.0 + delta_weight * abs_delta_pct)
```

Use absolute delta percentage for scoring.

Signed delta is stored separately.

If delta is unavailable, fall back to `TotalVolume`.

---

# 12. HVN Detection Algorithm

## 12.1 Local Peak Definition

A local peak is a bucket whose ranking score is greater than or equal to its immediate neighbours.

Edge buckets may be peaks if they exceed their only neighbour.

---

## 12.2 Plateau Rule

Flat-top plateaus are common and must be deterministic.

A plateau is a consecutive run of buckets with equal ranking score that qualifies as a local high.

A plateau qualifies if:

- its score is greater than the bucket immediately before the run, if such bucket exists
- its score is greater than the bucket immediately after the run, if such bucket exists
- if the plateau touches one array edge, only the existing neighbour must be exceeded
- if the plateau is the only bucket/run in the profile, it qualifies

For each qualifying plateau:

- select the centre bucket as the peak
- if plateau length is even, select the lower-index centre bucket

Only one peak candidate is created per plateau.

---

## 12.3 Candidate Cluster Expansion

For each local peak candidate:

Expand left and right independently while:

```text
adjacent_bucket.ranking_score >= peak_score * adjacency_threshold
```

Expansion stops on a side when:

- no adjacent bucket exists
- threshold condition fails
- bucket belongs to already accepted higher-ranked cluster

Implementation note:

- all valid peak seeds are detected first
- peak seeds are sorted strongest-first
- stronger peaks claim their qualifying territory first
- weaker peaks may not expand into already-claimed stronger territory

---

## 12.4 Multi-Modal Behaviour

If two peaks are connected by an intervening valley that remains above the adjacency threshold, they are one broad HVN cluster.

If the valley falls below threshold, they remain separate HVN candidates.

---

## 12.5 Candidate Ranking

Accepted candidate clusters are ranked by:

1. strongest contiguous window of exactly `Min HVN Cluster Buckets` buckets that:
   - lies fully inside the candidate cluster
   - contains the candidate peak bucket
   - has the maximum summed total volume among such valid windows
2. cluster total volume descending
3. peak score descending
4. narrower width first
5. lower price first

Rationale:

- threshold expansion defines the candidate region
- `Min HVN Cluster Buckets` defines both minimum validity and the structural comparison width used for `HVN1` / `HVN2` priority
- this prevents a broad moderate-volume region from outranking a tighter higher-quality node simply because the broader region contains more total buckets

---

## 12.6 HVN1 Selection

HVN1 is the highest-ranked candidate satisfying:

```text
bucket_count >= Min HVN Cluster Buckets
```

Set:

```text
PROFILE_FLAG_HVN1_FOUND
```

---

## 12.7 HVN2 Selection

After HVN1 is accepted:

1. Mark HVN1 buckets as assigned.
2. Re-run peak/candidate generation on unassigned buckets.
3. Select highest-ranked candidate.
4. Accept as HVN2 only if:

```text
HVN2.total_volume >= HVN1.total_volume * SecondaryHVNThresholdPct
```

Set:

```text
PROFILE_FLAG_HVN2_FOUND
```

if accepted.

---

# 13. LVN Detection Algorithm

## 13.1 LVN Eligibility

A bucket is LVN-eligible if:

```text
bucket.total_volume <= max_bucket_volume * LVNThresholdPct
```

and:

```text
bucket.total_volume > 0
```

V1 ignores synthetic zero-volume gaps.

---

## 13.2 LVN Clustering

Adjacent eligible buckets are merged into LVN clusters.

Each cluster must satisfy:

```text
bucket_count >= Min LVN Cluster Buckets
```

---

## 13.3 Primary LVN Selection

Case A — HVN1 and HVN2 exist:

- select LVN clusters located between HVN1 and HVN2
- choose lowest total volume
- tie-break by widest cluster
- tie-break by closest to midpoint between HVN1 peak and HVN2 peak

If fewer than two HVNs exist, no LVN is recorded.

Set:

```text
PROFILE_FLAG_LVN_FOUND
```

if accepted.

---

# 14. Profile Creation Flow

For each eligible closed HTF bar:

1. Validate bar index.
2. Validate VAP container.
3. Read OHLC.
4. Convert OHLC to ticks.
5. Read VAP levels.
6. Build bucket map.
7. Validate bucket count.
8. Compute volume/delta fields.
9. Compute ranking scores.
10. Sort buckets by bucket index.
11. Detect HVN1.
12. Detect HVN2.
13. Detect LVN.
14. Assign profile ID:

```cpp
profile_id = ++store.current_profile_id;
```

15. Append immutable record.
16. Evict old records if needed.
17. Update latest PV bridge.
18. Draw optional HTF debug dashes.
19. Update `last_processed_bar_index`.

---

# 15. Latest Persistent Variable Bridge

The full store is authoritative.

The PV bridge exposes only latest profile summary.

Use slots separate from pointer slot to avoid confusion.

Persistent floats:

```text
Float 9101 = latest HVN1 low
Float 9102 = latest HVN1 high
Float 9103 = latest HVN2 low
Float 9104 = latest HVN2 high
Float 9105 = latest primary LVN low
Float 9106 = latest primary LVN high
Float 9107 = latest profile high
Float 9108 = latest profile low
```

Persistent ints:

```text
Int 9101 = latest profile id
Int 9102 = latest profile bar index
Int 9103 = latest profile flags
Int 9104 = latest delta availability flag
Int 9105 = latest shape class placeholder, always 0 in V1
```

---

# 16. Store Eviction

If:

```text
profiles.size() > max_profiles
```

evict oldest records.

Eviction shall not alter profile IDs.

Profile IDs are monotonic increasing and are never reused during the study instance lifetime.

Consumers must not assume `profile_id == array index`.

---

# 17. HTF Debug Drawing

## 17.1 Purpose

Debug drawing allows visual verification on the HTF chart.

It is not the primary rendering mechanism.

---

## 17.2 Drawing Objects

Use `sc.UseTool`.

Each drawing should use deterministic line numbers:

```text
line_number = base + profile_id * 10 + zone_offset
```

Suggested offsets:

```text
HVN1 low  = 1
HVN1 high = 2
HVN2 low  = 3
HVN2 high = 4
LVN1 low  = 5
LVN1 high = 6
```

---

## 17.3 Placement

For `CompletedBarOnly`:

```text
BeginDateTime = profile.start_datetime
EndDateTime   = profile.end_datetime
```

For `ExtendNBars`, extend through N future HTF bars if available.

For `ExtendRight`, use Sierra Chart's right-extension drawing behaviour.

---

## 17.4 Price Placement

Debug rendering uses two horizontal lines per zone:

- lower boundary at `zone.low_ticks * tick_size`
- upper boundary at `zone.high_ticks * tick_size`

For HVN zones wider than `Min HVN Cluster Buckets`, the drawn bracket is reduced to the strongest contiguous sub-window of exactly `Min HVN Cluster Buckets` buckets that:

- lies inside the accepted HVN cluster
- contains the HVN peak bucket
- has the maximum summed total volume among valid windows

Consumer interoperability requirement:

- any downstream consumer that wants visual parity with the producer must export `HVN1/HVN2` from this narrowed drawn bracket range, not from the raw accepted HVN cluster bounds
- exporting raw `zone.low_ticks` / `zone.high_ticks` can make the consumer appear slightly offset from the producer even when both are reading the same underlying profile

This means:

- the accepted HVN cluster may remain broader internally
- but the visual bracket highlights the strongest node-width segment inside that cluster

LVN uses the full accepted LVN zone width.

---

# 18. Error Handling

## 18.1 VAP Unavailable

If VAP container or VAP elements are unavailable:

- create profile with `PROFILE_FLAG_VAP_UNAVAILABLE`
- no zones
- update `last_processed_bar_index`
- log warning if debug enabled

---

## 18.2 Zero Volume

If total VAP volume is zero:

- set `PROFILE_FLAG_ZERO_VOLUME`
- no zones
- record remains invalid for structural use

---

## 18.3 Bucket Limit Exceeded

If bucket count exceeds `Max Buckets Per Profile`:

- set/log `PROFILE_FLAG_BUCKET_LIMIT_EXCEEDED`
- skip profile
- update `last_processed_bar_index`
- do not silently truncate

---

## 18.4 Delta Unavailable

If bid/ask unavailable:

- set `PROFILE_FLAG_DELTA_UNAVAILABLE`
- delta fields zero
- total volume logic still valid

---

# 19. Determinism Requirements

For identical chart data and inputs, the producer must generate identical profiles across:

- full recalculation
- replay
- live updating after bar close

Rules:

- process only closed bars
- use tick-based price calculations
- sort candidates deterministically
- avoid wall-clock time
- do not mutate completed records

---

# 20. Acceptance Tests

## Test 1 — Closed-Bar Gate

Given `sc.ArraySize = 100`:

```text
last_closed_index = 98
```

Expected:

- bar 99 is not processed
- profiles generated only through bar 98

---

## Test 2 — No Duplicate Processing

Repeated study calls without new closed bar.

Expected:

- profile count unchanged
- no duplicate drawing
- `last_processed_bar_index` unchanged

---

## Test 3 — Full Recalc Parity

Procedure:

1. Full recalculation.
2. Log profile summaries.
3. Replay same data.
4. Compare profile IDs, bar indexes, HVN/LVN prices.

Expected:

- identical sequence
- identical zones

---

## Test 4 — Bucket Boundary

Bucket size = 4 ticks.  
VAP ticks:

```text
1000, 1001, 1002, 1003, 1004
```

Expected:

```text
1000-1003 -> bucket 250
1004      -> bucket 251
```

---

## Test 5 — Bimodal HVN

Buckets by index and volume:

```text
0:10
1:100
2:90
3:10
4:85
5:95
6:10
```

Adjacency threshold = 70%.

Expected:

```text
Cluster A = indices 1-2
Cluster B = indices 4-5
Clusters are not merged
```

---

## Test 6 — Plateau Peak

Buckets:

```text
0:10
1:100
2:100
3:100
4:10
```

Expected:

```text
Plateau = 1-3
Selected peak = index 2
```

Even plateau:

```text
1:100
2:100
```

Expected:

```text
Selected peak = index 1
```

---

## Test 7 — LVN Requires Two HVNs

Given only `HVN1` exists and no valid `HVN2` is accepted.

Expected:

- no LVN is recorded
- `PROFILE_FLAG_LVN_FOUND` is not set

---

## Test 8 — Delta Unavailable

Given total volume exists but bid/ask volumes are unavailable/zero.

Expected:

- total volume zones still detected
- delta fields zero
- `PROFILE_FLAG_DELTA_UNAVAILABLE` set
- delta ranking mode falls back to `TotalVolume`

---

## Test 9 — Store Eviction

Given:

```text
max_profiles = 3
processed profiles = 1,2,3,4,5
```

Expected store contains:

```text
3,4,5
```

Latest profile ID:

```text
5
```

---

## Test 10 — Strongest Fixed-Width HVN Priority

Given `Min HVN Cluster Buckets = 3` and two accepted HVN candidates:

- Candidate A is broad and has higher whole-cluster total volume
- Candidate B contains the stronger contiguous 3-bucket window around its peak

Expected:

- Candidate B is selected as `HVN1`
- Candidate A may still be selected later as `HVN2` if it passes the secondary threshold

Rationale:

- HVN priority is based on the strongest contiguous `Min HVN Cluster Buckets` node window, not on whole-cluster total volume alone

---

# 21. Future V2 Hooks

V2 may add:

- POC
- VAH / VAL
- D / P / b / B shape classification
- profile skew
- modality count
- composite multi-period profile
- diagonal imbalance
- multi-HTF confluence consumer

Existing bucket and zone data is sufficient for these extensions.

---

# 22. Fixed vs Accepted vs Deferred

## 22.1 Fixed in Final v1.0

- Correct IPC API restored
- No pointer integer bridge
- Volume fields corrected to `double`
- Plateau rule added
- LVN overlap rule clarified
- Profile ID assignment pinned
- Bucket boundary rule pinned
- Flags restored
- Inputs restored
- Error handling restored
- Acceptance tests restored
- Memory sizing made realistic

---

## 22.2 Accepted Trade-Offs

- Producer-owned dynamic store is accepted for V1.
- Consumer read-only contract is mandatory.
- Vectors are accepted inside producer-owned records.
- Perfect cross-DLL allocator purity is deferred unless real instability appears.
- Latest PV bridge is summary-only, not authoritative.

---

## 22.3 Not Worth Fixing in V1

- Diagonal delta
- POC/VAH/VAL
- Shape classification
- Composite profile
- Strategy signals
- Perfect memory packing
- Full POD slab allocator

These add complexity without improving V1’s core deliverable.

---

# 23. Final Summary

This study is a deterministic HTF volume-structure producer.

It computes completed-bar VAP profiles once, stores immutable structural records, exposes latest-profile summary PVs, and optionally draws HTF debug dashes.

The clean rule remains:

```text
Volume defines structure.
Delta describes behaviour.
Consumer handles interpretation.
```

This is infrastructure for future strategy work, not a signal engine.

---

END OF SPEC
