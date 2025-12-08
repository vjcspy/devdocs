# 📋 [251130] - Add Whale Footprint Ratios

## References

- `/Users/kai/work/k/python/metan/devdocs/projects/metan/stock/business/251115-whale-footsprint-feature.md`

## User Requirements

- Add ratio features for whale footprint:
    - `ratio_sharkXXX_buy_sell`: Ratio between Shark Buy and Shark Sell.
    - `ratio_buy_sharkXXX_sheep`: Ratio between Shark Buy and Sheep Buy.
    - `ratio_sell_sharkXXX_sheep`: Ratio between Shark Sell and Sheep Sell.
- Calculate using accumulated values from the start of the session.
- Use correct naming (fix "shell" typo to "sell").

## 🎯 Objective

Implement new accumulated ratio features in the Whale Footprint Feature Calculator to provide deeper insights into the relative strength of Shark and Sheep activities throughout the trading session.

### ⚠️ Key Considerations

- **Accumulation**: Must use values accumulated from the start of the day, not just the current candle.
- **Division by Zero**: Handle cases where the denominator is zero (likely default to 0.0).
- **Naming**: Ensure keys match the convention `ratio_{numerator}_{denominator}`.
- **Typo Correction**: User's "shell" -> "sell".

## 🔄 Implementation Plan

### Phase 1: Analysis & Preparation
- [x] Analyze existing calculator structure (`whale_footprint_feature_calculator.py`, `avg_prices.py`).
  - **Outcome**: Confirmed `cumulative_trackers` in `ctx` contains the necessary accumulated weighted values.
- [x] Define ratios and logic.
  - **Outcome**:
    - `accum_percent_shark{label}_buy_sell` = (Accum Shark Buy / Accum Shark Sell) * 100
    - `accum_percent_sheep{label}_buy_sell` = (Accum Sheep Buy / Accum Sheep Sell) * 100
    - `accum_percent_buy_shark{label}_sheep` = (Accum Shark Buy / Accum Sheep Buy) * 100
    - `accum_percent_sell_shark{label}_sheep` = (Accum Shark Sell / Accum Sheep Sell) * 100
    - `percent_shark{label}_buy_sell` = (Current Candle Shark Buy / Current Candle Shark Sell) * 100
    - `percent_sheep{label}_buy_sell` = (Current Candle Sheep Buy / Current Candle Sheep Sell) * 100
    - `percent_buy_shark{label}_sheep` = (Current Candle Shark Buy / Current Candle Sheep Buy) * 100
    - `percent_sell_shark{label}_sheep` = (Current Candle Shark Sell / Current Candle Sheep Sell) * 100

### Phase 2: Implementation (File/Code Structure)

```
packages/stock/metan/stock/trading/domain/feature/calculator/whale_footprint/
├── shark_sheep_ratios.py                 # ✅ Implemented - New module for both accumulated and current candle ratios
├── whale_footprint_feature_calculator.py # ✅ Updated - Registered new feature module
```

### Phase 3: Detailed Implementation Steps

1.  **Create `accum_ratios.py`**:
    -   Implement `compute(row, ctx)` function.
    -   Iterate through `threshold_meta`.
    -   Retrieve accumulated weighted values from `cumulative_trackers`.
    -   Compute ratios with zero-division protection.
    -   Update `row` with new features.

2.  **Update `whale_footprint_feature_calculator.py`**:
    -   Import `accum_ratios`.
    -   Add `accum_ratios.compute` to `FEATURES` list (after `avg_prices.compute` to ensure accumulators are up-to-date).

## 📊 Summary of Results
> Do not summarize the results until the implementation is done and I request it
