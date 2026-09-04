# Chakra TradePol HTF HVN Stop Candidate V1

Status: **authoritative implementation specification**  
Implementation: `ACS_Source/K_Chakra_TradePol_HTF_HVN_Stop_Candidate_V1.cpp`

## 1. Purpose

This study publishes one current LONG stop candidate and one current SHORT
stop candidate derived from configured HTF HVN zones. It answers only:

> Is there a sufficiently near, correctly positioned HVN zone against which
> a new trade could place its stop, and what is the resulting stop level?

The two sides are evaluated independently. The study does not enter trades,
choose between HVN and non-HVN stop sources, manage an open trade, score
setups, or maintain an interaction state machine. A later EntryGate or stop
decider may consume these candidates.

The study is latest-state and **persistent-variable only**. It publishes no
Subgraph output and reconstructs no historical candidate series.

## 2. Evaluation reference

Evaluation occurs from the latest completed bar of the chart containing this
study. `ReferencePrice` is that bar's close. ATR is read at the same bar from
the configured same-chart ATR study and Subgraph.

## 3. Configured HVN consumers

Four source periods are independently optional:

| Period | Inputs |
|---|---|
| Weekly | Enable Weekly HVNs; Weekly Consumer Study ID |
| Daily | Enable Daily HVNs; Daily Consumer Study ID |
| 4H | Enable 4H HVNs; 4H Consumer Study ID |
| 1H | Enable 1H HVNs; 1H Consumer Study ID |

All four enable switches default to `No`. An enable switch is a strict read
gate: when it is off, the study must not validate that period's Study ID,
request any of its arrays, read any of its SGs, or report an error about it.
This permits disabled inputs to be blank or arbitrary.

All enabled sources refer directly to a same-chart
`K_HTF_Volume_Structure_Consumer_V1`. Because every consumer has the same
layout, one shared SG mapping is configured, defaulting to:

- HVN1 Low = SG 0; HVN1 High = SG 1
- HVN2 Low = SG 2; HVN2 High = SG 3
- Active Profile ID = SG 8
- Active Profile Flags = SG 9

Consumer flag bits used are `PROFILE_VALID` bit 0, `HVN1_FOUND` bit 5, and
`HVN2_FOUND` bit 6. A raw zone exists only when its profile is valid, its
corresponding found bit is set, both boundaries are finite, and High > Low.

If an enabled source cannot be read or its current profile is invalid, the
whole snapshot is `NOT_READY`; the study must not silently bypass an unknown
higher-priority source. A valid profile simply lacking a particular HVN is
not an error and selection continues. If no period is enabled, the snapshot
is `NOT_READY` with both candidates invalid.

## 4. Candidate set and precedence

LONG and SHORT independently traverse exactly this fixed order. The first
raw candidate satisfying section 5 wins:

1. Weekly current HVN1
2. Weekly current HVN2
3. Daily current HVN1
4. Daily current HVN2
5. 4H current HVN1
6. 4H current HVN2
7. 4H immediately previous profile HVN1
8. 1H current HVN1

Thus precedence wins; this is not a global nearest-zone search. Weekly and
Daily previous-profile HVNs, 1H HVN2, and 1H previous-profile HVN1 are not
candidates.

For candidate 7, scan backward through the 4H Consumer's Active Profile ID
array from the evaluation bar and use the most recent positive profile ID
different from the current profile. Read its HVN1 bounds and flags at that
historical index. If no prior profile remains in the loaded arrays, candidate
7 is simply absent.

## 5. Raw selection criteria

Selection is performed before any tolerance is added.

For LONG, the entire zone must be below the close:

```text
ZoneHigh < Close
RawDistance = Close - ZoneLow
RawDistanceATR = RawDistance / ATR
```

For SHORT, the entire zone must be above the close:

```text
ZoneLow > Close
RawDistance = ZoneHigh - Close
RawDistanceATR = RawDistance / ATR
```

A candidate qualifies when `RawDistanceATR <= Maximum Raw SL Distance ATR`.
That maximum defaults to **2.0 ATR**. ATR, close, and all arithmetic must be
finite; ATR and tick size must be positive.

## 6. Tolerance and final stop

Tolerance is added only after a winning raw candidate has been selected. It
does not participate in selection and cannot make the already-selected
candidate fall through to a lower-priority zone.

The configured modes are:

- `Fixed Ticks`: `Tolerance = FixedTicks * TickSize`; default 10 ticks.
- `ATR Percentage`: `Tolerance = ATR * Percentage / 100`; default mode and
  default 25 percent (0.25 ATR).

Final ideal stop prices are:

```text
LONG  = selected ZoneLow  - Tolerance
SHORT = selected ZoneHigh + Tolerance
```

The stop price is then rounded away from entry to the instrument tick grid:
floor for LONG and ceiling for SHORT. Final distance is recomputed from the
rounded stop. The study does not reapply the 2 ATR raw-selection maximum
after tolerance.

## 7. Public persistent-variable contract

The common Chakra envelope is published as TradePol, latest-state pattern:

| Field | PV |
|---|---|
| SchemaVersion, Status, ComponentKind, ProducerTypeID | Int32 1-4 |
| EvalBarIndex, DecayClass, ReasonCode | Int32 5-7 |
| PayloadSchemaVersion | Int32 20 |
| LongCandidateValid, LongSelectedSource | Int32 21-22 |
| ShortCandidateValid, ShortSelectedSource | Int32 23-24 |
| PublicationSeq | Int64 1, written last |
| EvalDateTime, ValidUntilDateTime | DateTime 1-2 |
| LongStopLevel | Float 1 |
| LongRawDistanceATR | Float 2 |
| LongFinalDistanceATR | Float 3 |
| ShortStopLevel | Float 4 |
| ShortRawDistanceATR | Float 5 |
| ShortFinalDistanceATR | Float 6 |

`SelectedSource` is 0 for none or 1-8 matching section 4. ComponentKind is
`K_CHAKRA_KIND_TRADEPOL` (4). DecayClass is FAST. ValidUntil is zero.
Candidate payload is neutral whenever Status is not VALID.

## 8. Display and diagnostics

A boolean input guards one optional chart banner. It summarizes status and
the two current candidates, including source, stop, and final ATR distance.
The banner creates no data Subgraphs. It is deleted when display is disabled,
the study is disabled, or the study is removed.

Debug logging is change-gated. Normal absence of a qualifying candidate is
a valid result and is not an error.

## 9. Performance and acceptance

- Manual loop; evaluate only the latest completed chart bar.
- Never fetch arrays for a disabled period.
- Fetch each required enabled dependency array at most once per evaluation.
- Only the 4H prior-profile candidate performs a backward scan.
- No heap allocation in the evaluation path.
- No SG output or historical output population.
- LONG and SHORT can both be valid simultaneously and never overwrite each
  other's result.
