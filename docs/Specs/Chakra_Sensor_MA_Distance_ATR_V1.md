# Chakra — Sensor: MA Distance in ATR (Revision 1)

Status: **implemented.**

Builds directly on:

- `Chakra_Common_Interface_Envelope_Contract.md` (Revision 8).
- `Chakra_Sensor_Interface_Contract.md` (Revision 4).

------------------------------------------------------------------------

# Purpose

A reusable, local-chart Sensor that reads two configured moving-average
subgraphs and one ATR subgraph and publishes their unsigned separation in ATR
units:

```text
MADistanceATR = abs(MA1 - MA2) / ATRUsed
```

The Sensor measures geometry only. It does not classify proximity, trend,
direction, crossover, candle patterns, or trading signals.

------------------------------------------------------------------------

# 1. Identity and cadence

- Name: `K_Chakra_Sensor_MA_Distance_ATR_V1`.
- `ComponentKind = K_CHAKRA_KIND_SENSOR`.
- `ProducerTypeID = K_CHAKRA_SENSOR_MA_DISTANCE_ATR_TYPE_V1`.
- `PayloadSchemaVersion = 1`.
- Local-chart dependencies only.
- Closed-bar cadence only; the forming bar is never published as valid.
- `DecayClass = FAST`.

------------------------------------------------------------------------

# 2. Inputs

| Input | Default | Purpose |
|---|---:|---|
| `Enabled` | Yes | Standard enable/disable. |
| `MA 1 Study` / `MA 1 SG` | 0 / 0 | First local-chart MA source. |
| `MA 2 Study` / `MA 2 SG` | 0 / 0 | Second local-chart MA source. |
| `ATR Study` / `ATR SG` | 0 / 0 | Local-chart ATR source. |
| `Debug Logging` | No | Logs status/reason transitions only. |

MA order has no semantic meaning. A consumer needing ordering compares `MA1`
and `MA2` directly.

------------------------------------------------------------------------

# 3. Validation and calculation

For evaluation bar `i`:

1. All three study IDs must be configured.
2. All three arrays must cover `i` and `i` must be at or beyond each
   dependency's `DataStartIndex`.
3. `MA1`, `MA2`, and `ATRUsed` must be finite.
4. `ATRUsed > 0`.
5. Calculate `abs(MA1 - MA2) / ATRUsed` at full float precision without
   rounding; the result must be finite.

`MADistanceATR` is an unsigned, non-negative ratio. Zero is valid and means
the two MA values are equal.

------------------------------------------------------------------------

# 4. Public snapshot

## Common envelope

| Field | Namespace/PV |
|---|---|
| `SchemaVersion` | Int32 PV 1 |
| `Status` | Int32 PV 2 |
| `ComponentKind` | Int32 PV 3 |
| `ProducerTypeID` | Int32 PV 4 |
| `EvalBarIndex` | Int32 PV 5 |
| `DecayClass` | Int32 PV 6 |
| `ReasonCode` | Int32 PV 7 |
| `PublicationSeq` | Int64 PV 1, written last |
| `EvalDateTime` | DateTime PV 1 |
| `ValidUntilDateTime` | DateTime PV 2, always `0` |

## Sensor payload

| Field | Namespace/PV | Units |
|---|---|---|
| `PayloadSchemaVersion` | Int32 PV 20 | schema identity |
| `MA1` | Float PV 1 | price |
| `MA2` | Float PV 2 | price |
| `ATRUsed` | Float PV 3 | price |
| `MADistanceATR` | Float PV 4 | non-negative ATR ratio |

`Status != VALID` makes the complete payload neutral and untrusted.

------------------------------------------------------------------------

# 5. Historical arrays

Hidden machine-readable Subgraphs rebuild during full recalculation:

| SG | Field |
|---:|---|
| 0 | Historical `Status` |
| 1 | `MA1` |
| 2 | `MA2` |
| 3 | `ATRUsed` |
| 4 | `MADistanceATR` |

A historical consumer requires array coverage, SG 0 equal to `VALID`, and a
finite/in-range measurement. The value Subgraphs are zeroed whenever the bar
is not valid.

------------------------------------------------------------------------

# 6. Lifecycle

- Same closed bar after successful evaluation: pure no-op in live operation.
- Full recalculation: publish `NOT_READY`, clear/rebuild all historical
  Subgraphs, then publish one final snapshot for the latest closed bar.
- Dependency/input changes use Sierra Chart's full-recalculation lifecycle.
- Disable-entry: clear historical outputs once, publish `DISABLED`, clear the
  live evaluation guard, and do not fetch dependencies.
- No external actions, event sequence, alerts, drawings, or trading output.

------------------------------------------------------------------------

# 7. Diagnostics

```cpp
enum K_CHAKRA_SENSOR_MA_DISTANCE_ATR_REASON_CODE
{
    K_CHAKRA_SENSOR_MA_DISTANCE_ATR_REASON_NONE = 0,
    K_CHAKRA_SENSOR_MA_DISTANCE_ATR_REASON_DEPENDENCY_UNWIRED = 1,
    K_CHAKRA_SENSOR_MA_DISTANCE_ATR_REASON_ARRAYS_NOT_READY = 2,
    K_CHAKRA_SENSOR_MA_DISTANCE_ATR_REASON_DEPENDENCY_UNAVAILABLE = 3,
    K_CHAKRA_SENSOR_MA_DISTANCE_ATR_REASON_MA_VALUE_INVALID = 4,
    K_CHAKRA_SENSOR_MA_DISTANCE_ATR_REASON_ATR_INVALID = 5,
    K_CHAKRA_SENSOR_MA_DISTANCE_ATR_REASON_MEASUREMENT_PUBLISHED = 6
};
```

------------------------------------------------------------------------

# 8. Acceptance cases

1. `MA1=101`, `MA2=100`, `ATR=2` publishes `MADistanceATR=0.5`.
2. Reversing MA inputs publishes the same distance while swapping `MA1/MA2`.
3. Equal MAs publish a valid zero distance.
4. ATR `<= 0` or a non-finite input publishes `ERROR` with neutral payload.
5. Warmup bars publish historical `NOT_READY` and zero value Subgraphs.
6. Live and full-recalculation evaluation produce the same latest value.
