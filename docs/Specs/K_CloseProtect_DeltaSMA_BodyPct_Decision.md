Create a new ACSIL close/protect decision study:

  File:
  K_CloseProtect_DeltaSMA_BodyPct_Decision.cpp

  Base:
  Use K_Template_CloseProtect_DecisionStudy.cpp as the contract template, but replace the placeholder trigger inputs/
  logic.

  Contract:
  - Include K_BatonDecisionContract.h.
  - Never place, modify, flatten, cancel, BuyExit, or SellExit.
  - Publish only:
    PV 1 = SourceEventSeq
    PV 2 = SourceID
  - Increment PV 1 only once per new event.
  - Set PV 2 to K_CLOSE_SOURCE_DELTA_FLIP, or add/use a more specific source enum if one exists.
  - On full recalculation: reset SourceEventSeq and SourceID, baseline edge state.
  - When disabled: set PV 2 to K_CLOSE_SOURCE_NONE, baseline edge state, do not clear PV 1.
  - Aggregator must not need any code changes.

  Inputs:
  0. Enable Decision Study, Yes/No, default Yes
  1. Source ID, default Delta Flip
  2. Evaluate On Closed Bars Only, Yes/No, default Yes
  3. Adverse Side Mode
     - Long Position Risk: negative delta + close below SMA
     - Short Position Risk: positive delta + close above SMA
  4. Delta Games Study
  5. Positive Delta Breach Count Subgraph
     - default subgraph 3 from K_Alert_Candle_Delta_Games_V1, "DeltaPosCount"
  6. Negative Delta Breach Count Subgraph
     - default subgraph 4 from K_Alert_Candle_Delta_Games_V1, "DeltaNegCount"
  7. Moving Average Study
  8. Moving Average Subgraph
  9. Minimum Body Percent Across SMA
     - float, default e.g. 50.0
     - limits 0.0 to 100.0
  10. Lookback Bars For Delta Breach
     - int, default 2
     - meaning current bar plus previous N bars
     - default 2 means check current, previous, and candle before that
  11. Debug Logging, Yes/No, default No

  Delta source:
  Use K_Alert_Candle_Delta_Games_V1 outputs:
  - DeltaPosCount is subgraph 3.
  - DeltaNegCount is subgraph 4.
  A value greater than 0 means at least one delta breach was detected on that candle.
  Do not try to read the internal Arrays[0] magnitude buffers from that study.

  Logic:
  For each eligible eval bar:

  1. Determine evalIndex:
     - If Evaluate On Closed Bars Only = Yes, use the most recent closed candle.
     - Otherwise use the current last bar.

  2. Read delta breach arrays once:
     - Positive delta breach count array from configured positive SG.
     - Negative delta breach count array from configured negative SG.

  3. Read SMA array once from configured MA study/subgraph.

  4. Delta condition:
     - Long Position Risk:
       true if Negative Delta Breach Count > 0 on evalIndex, evalIndex - 1, or evalIndex - 2 by default.
     - Short Position Risk:
       true if Positive Delta Breach Count > 0 on evalIndex, evalIndex - 1, or evalIndex - 2 by default.
     - Clamp lookback so indexes below 0 are ignored.

  5. SMA close condition:
     - Long Position Risk:
       current candle close must be below SMA.
     - Short Position Risk:
       current candle close must be above SMA.

  6. Body-percent-across-SMA condition:
     Compute candle body only:
     - bodyHigh = max(Open, Close)
     - bodyLow = min(Open, Close)
     - bodySize = bodyHigh - bodyLow
     - If bodySize <= 0, condition fails.

     Long Position Risk:
     - SMA must lie inside or above the candle body such that some body is below it.
     - bodyBelowSMA = clamp(SMA - bodyLow, 0, bodySize)
     - percentBelow = 100 * bodyBelowSMA / bodySize
     - require percentBelow >= Minimum Body Percent Across SMA

     Short Position Risk:
     - bodyAboveSMA = clamp(bodyHigh - SMA, 0, bodySize)
     - percentAbove = 100 * bodyAboveSMA / bodySize
     - require percentAbove >= Minimum Body Percent Across SMA

  7. Fire event when:
     delta condition AND SMA close condition AND body-percent condition are all true.

  8. Edge gating:
     Fire only on a false-to-true transition of the combined condition, or once per evalIndex.
     Do not increment SourceEventSeq repeatedly while the condition remains true on the same bar or across reload/
     recalc.

  9. When event fires:
     ++SourceEventSeq;
     publishedSourceId = SourceID.GetIndex();
     InternalState[evalIndex] = SourceEventSeq;
     optionally log:
     "Delta/SMA body close-protect event. Seq=%i SideMode=%i EvalIndex=%i DeltaLookback=%i SMA=%.5f BodyPct=%.2f"

  Important caveat:
  Because this decision study is position-agnostic, it must be configured for the intended side. A Long Position Risk
  instance should only be used when managing long-position risk; a Short Position Risk instance should only be used when
  managing short-position risk. Otherwise it can publish an adverse event for the wrong trade direction.

  The only thing I’d tighten before coding: rename “% body above/under” to Minimum Body Percent Across SMA or Minimum
  Body Percent Beyond SMA. It makes the calculation unambiguous.