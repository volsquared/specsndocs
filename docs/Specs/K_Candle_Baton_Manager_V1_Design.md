# K Candle Baton Manager V1 Design

## Purpose

This document defines the design contract for a new candle-chart post-entry trade management study.

Working name:

- `K_Util_Candle_Baton_Manager_V1`

Primary intent:

- manage already-open trades on a candle chart
- read the live trade/order handoff from the existing master baton
- support configurable break-even, trailing stop, close, and target-move behaviors
- support different trailing stop strategies per contract/order

Non-goals for V1:

- no order entry logic
- no armed setup logic
- no direct signal generation
- no Renko ring-dependent management assumptions unless explicitly selected by a strategy later

This study is a slave/follower manager only.

## Relationship To Existing Baton

The existing `K_Util_ModeBaton_V1.cpp` remains the entry authority and source of truth for:

- trade creation
- initial attached orders
- master trade sequencing
- the persistent-variable handoff contract

The new candle manager must not replace this ownership in V1.

The candle manager will:

- detect a new master-owned trade
- sync the master order IDs and trade metadata
- manage stops/targets/BE/flattening from the candle chart after sync

## Core Lifecycle

### Stage 1: Idle

No live synced trade is being managed locally.

### Stage 2: Sync

When the master baton indicates a new trade, the candle manager reads the handoff block and initializes local management state.

### Stage 3: Active Management

While the trade is live, the candle manager:

- evaluates BE strategy
- evaluates close/flatten strategy
- evaluates trailing stop strategy per stop order
- evaluates target strategy per target order

### Stage 4: Reset

When the position is flat or the master/live order state is no longer valid, the candle manager clears local management state and returns to idle.

## Master To Slave Handoff Contract

The current baton already contains the master/follower sequencing pattern.

The handoff trigger is:

```cpp
if (currentTradeFollowerNumberAsId < currentTradeLeaderNumberAsIdFromMaster)
```

Meaning:

- master increments its trade leader ID when a new trade is actually taken
- follower compares its local follower ID against the master leader ID
- follower syncs only once per newly opened trade

This pattern must be preserved.

## Existing Persistent Variables Already In Use

These indexes are already established by the existing baton contract and must be treated as reserved/owned.

### Existing Local Baton Runtime / Order PVs

- `18` `float lastHighForRunner`
- `19` `float lastLowForRunner`
- `20` `int StopAllOrderID`
- `21` `int Target1OrderID`
- `22` `int Target2OrderID`
- `23` `int Target3OrderID`
- `24` `int Target4OrderID`
- `25` `int Target5OrderID`
- `26` `double currentTargetPrice`
- `27` `int currentPositionSizingLevel`
- `28` `int rampDown1DonePV`
- `29` `int rampDown2DonePV`
- `30` `int effectPositionSizePV`
- `31` `int paintedInitialPositionSize`
- `32` `int incomingTradeIndex`
- `33` `int lookbackScanDone`
- `34` `int Stop1OrderID`
- `35` `int Stop2OrderID`
- `36` `int Stop3OrderID`
- `37` `int Stop4OrderID`
- `38` `int Stop5OrderID`
- `39` `int isCurrentTradeCustomTrailing`
- `40` `int behindDotTrailLastLowRingIndex`
- `41` `int behindDotTrailLastHighRingIndex`
- `42` `float Stop1Price`
- `43` `int currentTradeLeaderNumberAsId`
- `44` `int currentTradeFollowerNumberAsId`
- `45` `int moveToBEDone`
- `46` `int lastFibSLHighIndex`
- `47` `int lastFibSLLowIndex`

### Existing Main/Master Trade Metadata PVs

- `52` `int mainChartTradeDirection`
- `53` `int mainChartTradeQuantity`
- `54` `double mainChartTradeStopLevel`
- `55` `float mainChartT1Distance`
- `56` `float mainChartT2Distance`
- `57` `float mainChartT3Distance`
- `58` `int mainChartMoveToBEDistance`
- `59` `int mainChartMoveToX`
- `60` `int mainChartMoveToY`
- `61` `float batonTradeInvalidationPrice`
- `62` `float batonTradeRingAdaptiveSLTriggerPrice`
- `63` `float batonTradeInvalidationPriceInDirection`
- `64` `float batonTradeRingPrice`
- `65` `int justTakeTheTrade`
- `66` `int tradeSignalCandleIndex`
- `67` `int trailBehindIndicatorPV`
- `68` `double batonTradeTrailBehindIndicatorStopLevelPV`
- `69` `int batonTradeGoBigUpTruePV`
- `70` `int batonTradeGoBigDownTruePV`
- `71` `float slPriceLevelFromMainChart`
- `72` `int tradeSignalCandleMaxIndex`
- `73` `float timeCandleUpBreachLevelForEntry`
- `74` `float timeCandleDownBreachLevelForEntry`
- `75` `int currentBatonTradeId2`
- `76` `int currentBatonTradeId3`

## PV Rules For New Candle Manager

### Rule 1

Do not repurpose or reinterpret any existing master baton PV index listed above.

### Rule 2

The candle manager may read master baton PVs through `GetPersistent*FromChartStudy(...)`, but should not write back into the master baton’s PV namespace in V1.

### Rule 3

The candle manager should maintain its own local state in a new dedicated PV range.

### Proposed New Local PV Namespace

Use a new local range starting at `120`.

Suggested reservation:

- `120` `int syncedTradeLeaderId`
- `121` `int activeTradeDirection`
- `122` `int activeTradeQuantity`
- `123` `int activeInitialQuantity`
- `124` `int beDone`
- `125` `int activeTradeState`
- `126` `int lastManagementBarIndex`
- `127` `int lastCloseActionBarIndex`
- `128` `int lastTargetMoveBarIndex1`
- `129` `int lastTargetMoveBarIndex2`
- `130` `int lastTargetMoveBarIndex3`
- `131` `int lastTargetMoveBarIndex4`
- `132` `int lastTargetMoveBarIndex5`
- `133` `float trailStopApplied1`
- `134` `float trailStopApplied2`
- `135` `float trailStopApplied3`
- `136` `float trailStopApplied4`
- `137` `float trailStopApplied5`
- `138` `float targetPriceApplied1`
- `139` `float targetPriceApplied2`
- `140` `float targetPriceApplied3`
- `141` `float targetPriceApplied4`
- `142` `float targetPriceApplied5`
- `143` `int patternCloseCooldownBarIndex`
- `144` `float lastReferencePrice1`
- `145` `float lastReferencePrice2`
- `146` `float lastReferencePrice3`
- `147` `int localResetCounter`

This range can be extended later, but V1 should stay within a clearly documented local namespace.

## Candle Manager Responsibilities

### In Scope

- read master order IDs and trade metadata
- manage stop orders after entry
- manage target orders after entry
- move stops to break-even according to selected strategy
- flatten trade according to selected close strategy
- support per-contract trailing stop strategies
- support per-contract target strategies

### Out Of Scope For V1

- trade entry
- armed setup state machine
- setup invalidation logic before entry
- signal generation
- master baton refactoring

## Input Model

The study should expose strategy selection through dropdown-style custom inputs.

### Core Control Inputs

- `Manager Enabled`
- `Master Trade Chart/Study`
- `Fetch Order IDs From Master`
- `Manage Long Trades`
- `Manage Short Trades`
- `Evaluate On Closed Bars Only`
- `Debug Logging`

### BE Strategy Inputs

- `Break Even Strategy`
- `BE Profit Trigger Ticks`
- `BE Offset Ticks`
- `BE Reference Study`
- `BE Reference Subgraph`

### Close Strategy Inputs

- `Close Strategy`
- `Close Reference Study 1`
- `Close Reference Subgraph 1`
- `Close Reference Study 2`
- `Close Reference Subgraph 2`
- `Close Cooldown Bars`

### Per-Stop Trailing Strategy Inputs

- `Trail Strategy Stop 1`
- `Trail Strategy Stop 2`
- `Trail Strategy Stop 3`
- `Trail Strategy Stop 4`
- `Trail Strategy Stop 5`

### Shared Trailing Config Inputs

- `Trail Reference Study 1`
- `Trail Reference Subgraph 1`
- `Trail Reference Study 2`
- `Trail Reference Subgraph 2`
- `Trail Offset Ticks`
- `Trail Minimum Improvement Ticks`
- `Trail Activation Profit Ticks`

### Per-Target Strategy Inputs

- `Target Strategy 1`
- `Target Strategy 2`
- `Target Strategy 3`
- `Target Strategy 4`
- `Target Strategy 5`

### Shared Target Config Inputs

- `Target Near Distance Ticks`
- `Target Move By Ticks`
- `Target Minimum Improvement Ticks`

## Strategy Injection Contract

Strategies must not directly modify or flatten orders.

Strategies only evaluate and return decisions.

The management engine remains the only code that:

- calls `modifyOrder(...)`
- calls flatten/exit functions
- applies guardrails

This is a hard architectural rule to keep the code clean.

## Decision Contracts

### Break Even Strategy Result

Break-even strategy returns:

- `NoAction`
- `MoveStopsToPrice`

Result fields:

- `bool shouldMove`
- `float newStopPrice`
- `SCString reason`

### Close Strategy Result

Close strategy returns:

- `Hold`
- `CloseAll`

Result fields:

- `bool shouldClose`
- `int closeReasonCode`
- `SCString reason`

### Trail Strategy Result

Per-stop trailing strategy returns:

- `NoAction`
- `MoveStopToPrice`

Result fields:

- `bool shouldMove`
- `float newStopPrice`
- `SCString reason`

### Target Strategy Result

Per-target strategy returns:

- `NoAction`
- `MoveTargetToPrice`

Result fields:

- `bool shouldMove`
- `float newTargetPrice`
- `SCString reason`

## Initial V1 Strategy Enum Set

### Break Even Strategies

- `0 = Off`
- `1 = FixedTicksInProfit`
- `2 = ReferenceStudyCross`

### Close Strategies

- `0 = Off`
- `1 = CloseOnStudyBooleanTrue`
- `2 = CloseOnStudyCross`

### Trail Strategies

- `0 = Off`
- `1 = FixedPriceFromMasterStop`
- `2 = TrailBehindStudyValue`
- `3 = TrailBehindCandleLowHigh`

### Target Strategies

- `0 = Fixed`
- `1 = ChaseWhenNear`

This is intentionally small for V1.

## Long/Short Symmetry Rules

All management logic must be direction-aware.

For long trades:

- stop movement may only raise stops, never lower them
- target movement may only move targets higher, never lower them

For short trades:

- stop movement may only lower stops, never raise them
- target movement may only move targets lower, never raise them

## Order Application Guardrails

Before any stop or target modification:

- confirm order ID is non-zero
- confirm the order is still active
- confirm the proposed price improves the existing order
- confirm improvement exceeds minimum tick threshold
- do not re-send the same effective price repeatedly

Before any close action:

- confirm there is still an open position
- confirm the close action has not already fired for the same bar/state

## Sync Contract From Master

On new trade detection, the candle manager should pull at minimum:

- `20` `StopAllOrderID`
- `21-25` target order IDs
- `34-38` stop order IDs
- `43` master leader trade ID
- `52` direction
- `53` quantity
- `54` master stop level
- `55-57` target distances
- `58-60` BE and target-move parameters if needed for compatibility

Optional compatibility reads may include:

- `67` trail-behind-indicator selector
- `68` master-provided trail level

The new candle manager should not depend on unrelated entry/setup PVs for V1.

## Local Active Trade State Machine

Suggested values for `activeTradeState`:

- `0 = Idle`
- `1 = Synced`
- `2 = Active`
- `3 = Closing`
- `4 = ResetPending`

## Reset Conditions

The local management state should reset when any of the following occurs:

- live position quantity becomes zero
- all relevant order IDs are invalid/inactive
- master leader trade ID advances and a new trade sync occurs
- study is disabled

Reset must clear only the candle manager’s own local PV namespace.

Do not clear master baton state from this study.

## Target Move Behavior

V1 target movement modes:

### Fixed

Do not move the target after entry.

### ChaseWhenNear

If price comes within `X` ticks of current target, push the target further by `Y` ticks in the favorable direction.

Rules:

- apply per target order
- never move target backward
- do not churn the same target repeatedly within the same bar/state

## Trailing Stop Behavior

Trailing stop selection is independent per contract/order.

Examples:

- `Stop1OrderID` can use a fast-protection strategy
- `Stop2OrderID` can use a medium runner strategy
- `Stop3OrderID` can use a structure strategy
- `Stop4OrderID` can use an MA strategy
- `Stop5OrderID` can use a deep runner strategy

This is a foundational requirement, not an optional enhancement.

## Close Strategy Intent

The preferred happy path is that positions close because managed trailing stops are hit.

Explicit close/flatten strategies are exception handlers for:

- candle-pattern based invalidation
- external study-driven flatten conditions
- safety exits

Therefore:

- flatten logic should be configurable
- flatten logic should not dominate the management engine

## Clean Code Requirements

The new study should be structured into explicit layers:

1. `syncMasterTradeState(...)`
2. `readLivePositionState(...)`
3. `evaluateBreakEvenStrategy(...)`
4. `evaluateCloseStrategy(...)`
5. `evaluateTrailStrategyForStopN(...)`
6. `evaluateTargetStrategyForTargetN(...)`
7. `applyManagementActions(...)`
8. `resetLocalManagerState(...)`

Avoid a monolithic baton-style file where signal logic, order entry, trade management, and chart references all mix together.

## V1 Build Order

Recommended implementation order:

1. Master sync only
2. Flat/reset handling
3. Fixed tick BE strategy
4. Per-stop trailing strategy skeleton
5. Fixed target and ChaseWhenNear target strategy
6. Simple close strategy based on an external study boolean/cross
7. Additional strategy modules later

## Future Expansion

Planned later, but not part of initial V1:

- armed setup / delayed entry framework
- candle-pattern-based close modules
- richer MA/VWAP entry reference logic
- partial close/reduction strategies
- more advanced multi-reference strategy composition

