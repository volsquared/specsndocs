# Volatility Calibrator V2.2 — Final SRS (Build Version)

## 1. Purpose
Volatility Calibrator V2.2 provides context-aware calibration of intraday VWAP/BB width conditioned on daily ATR regime.

This is a calibration engine only. No runtime logic, no signals.

---

## 2. Scope

Included:
- ATR-conditioned VWAP width calibration
- Fixed ATR buckets
- Per-bucket percentiles
- Logging output

Excluded:
- Runtime classification
- HUD/UI
- Trade signals
- Renko logic

---

## 3. Inputs

Required:
- VWAP Width Study ID
- VWAP Width Subgraph Index
- Daily ATR Study ID
- Daily ATR Subgraph Index

Config:
- Lookback Days (default 360)
- Session (RTH or Globex only)
- ATR Bucket Size
- Recency Weighting (default ON)
- Weight Tiers (default 1;0.7;0.4)
- Min Days per Bucket (default 20)

---

## 4. ATR Bucketing

bucket_index = floor(ATR / bucket_size)

Final bucket is open-ended (>= max).

Overflow is capped and logged.

---

## 5. ATR Retrieval

- Must use Daily timeframe study
- Use previous completed day
- Map by trading date
- Never use current developing ATR

---

## 6. Data Collection

For each day:
- Get prior ATR
- Assign bucket

For each intraday bar:
- Extract VWAP width
- Append to bucket

---

## 7. Recency Weighting

Default ON:
- Recent 120d: 1.0
- Mid 120d: 0.7
- Old 120d: 0.4

Weight applied per day.

---

## 8. Bucket Integrity

If bucket < min days:
- DO NOT merge
- Log warning

---

## 9. Percentiles

Compute per bucket:
P10, P20, P30, P40, P50, P60, P70, P80, P90

---

## 10. Session Rules

- RTH and Globex must be run separately
- No combined session mode

---

## 11. Outputs (Logs)

Log per bucket:
- Range
- Days
- Samples
- Percentiles

Also log:
- Overflow events
- Sparse warnings

---

## Appendix A — Bheeshm Output Contract

JSON example:

{
  "instrument": "CL",
  "session": "RTH",
  "bucket_size": 1.0,
  "buckets": [
    {
      "range": "3-4",
      "days": 42,
      "samples": 18240,
      "percentiles": {
        "p10": 0.8,
        "p20": 1.0,
        "p30": 1.2,
        "p40": 1.4,
        "p50": 1.6,
        "p60": 1.9,
        "p70": 2.2,
        "p80": 2.6,
        "p90": 3.1
      }
    }
  ]
}

---

## 12. Defaults

CL: 1.0  
ES: 10  
NQ: 50–100  

---

## 13. Summary

V2.2 provides a simple, robust, regime-aware calibration tool for discretionary trading.
