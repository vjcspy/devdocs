# 📋 [TICKET-ID: 2025-09-07] - Whale Footprint Feature Engineering (v4)

## 🎯 Objective
> For each tick candle (grouped per trading day), compute shark/sheep buy/sell trade values across thresholds, and normalize by both:
> - trailing 5-day per-candle baseline (exclude current day), and
> - today's per-candle baseline.

**Critical Data Assumptions:**
- **All monetary values from upstream are ALREADY in millions** (daily_value, candle.value, trade_value)
- **Thresholds are specified directly in millions** (e.g., 450 instead of 450_000_000)
- **Strict error handling**: Any invalid/missing data immediately raises ValueError for data precision                    # 📋 [TICKET-ID: 2025-09-07] - Whale Footprint Feature Engineering (v4)

## 🎯 Objective
> For each tick candle (grouped per trading day), compute shark/sheep buy/sell trade values across thresholds, and normalize by both:
> - trailing 5-day per-candle baseline (exclude current day), and
> - today’s per-candle baseline.

## 🔄 Implementation Plan

### Phase 1: Analysis & Preparation
- [x] Analyze detailed requirements
  - **Outcome**:
    - Data sources:
      - Candles grouped by date: `StockDataCollector.tick_candles_by_date()` → dict[date] -> list[TickCandle]
      - Daily prices: `StockDataCollector.prices()` → list[Price] with `value` (already in millions)
    - For each TickAction: `trade_value_raw = price × volume` (raw units), then convert to millions by `/1_000_000`
    - Classification per threshold T (in millions):
      - sharkT: `trade_value_raw >= T * 1_000_000` (compare raw against scaled threshold)
      - sheepT: `trade_value_raw < T * 1_000_000`
    - Include only sides `B` (buy) and `S` (sell); ignore `Undefined`.
    - Baselines (per-candle, all in millions):
      - Today per-candle: `today_pc[date] = daily_value[date] / base_candle_count`
      - 5D trailing per-candle (exclude current date):
        - Uses exactly 5 prior trading days with valid `Price.value` (min_periods=5)
        - `pc_5d[date] = rolling_sum_prior_5_daily_values / (5 * base_candle_count)`
        - NaN if fewer than 5 prior days available
      - Ratios per metric (rounded to 4 decimals):
        - `{metric}_ratio_today_pc = round(metric_value / today_pc[date], 4)`
        - `{metric}_ratio_5d_pc = round(metric_value / pc_5d[date], 4)` (if 5D baseline available)
- [x] Define scope and edge cases
  - **Outcome**:
    - **Strict Error Handling for Data Precision**:
      - If `Price.value <= 0` for any date that has candles → raise `ValueError` with symbol/date
      - If computed today per-candle baseline <= 0 → raise `ValueError` with symbol/dates
      - If computed 5D per-candle baseline <= 0 → raise `ValueError` with symbol/date
      - If candle_count inconsistency across days (diff > 1) → raise `ValueError` with counts
      - If date has no candles but exists in grouped ticks → raise `ValueError`
      - If missing 5D baseline when expected → raise `ValueError`
    - **Data Validation**:
      - Base candle count determined from max(candle_counts) with validation of consistency
      - 5D baseline requires exactly 5 prior days (rolling with min_periods=5)
      - Unix `time` in candles preserved as-is; date comes from dict key
    - **Defaults**: Thresholds [450, 900] (in millions)

### Phase 2: Implementation (File/Code Structure)
> Implement the calculator and helpers. Keep it fast and deterministic.

```
packages/stock/metan/stock/trading/domain/feature/whale_footprint/
└── whale_footprint_feature_calculator.py   # ✅ IMPLEMENTED - feature calculator
```

- `WhaleFootprintFeatureCalculator` responsibilities:
  - **Data Collection & Validation**:
    - Fetch `by_date = tick_candles_by_date()` and `prices`
    - Build `daily_value[date]` (already in millions) and `candle_count[date]`
    - **Strict validation**: raise ValueError for any `daily_value <= 0` where candles exist
    - Validate consistent `base_candle_count` across days (max difference = 1)
  - **Baseline Computation**:
    - Today per-candle: `daily_value / base_candle_count` with error on non-positive results
    - 5D trailing per-candle: rolling 5-day sum with min_periods=5, error on non-positive results
  - **Feature Aggregation**:
    - Per-candle metrics aggregated across thresholds and sides
    - Trade value classification: `trade_value_raw >= threshold * 1_000_000` for shark classification
    - Convert aggregated trade values to millions: `int(trade_value_raw / 1_000_000)`
  - **Ratio Computation**:
    - All ratios rounded to 4 decimal places
    - Immediate error if 5D baseline missing when expected
  - Return a pandas DataFrame indexed by candle `time`

- **Column naming**:
  - Values: `{category}{thrM}_{side}_value` where `category ∈ {shark, sheep}`, `thrM ∈ {450, 900}`, `side ∈ {buy, sell}`
  - Ratios: `{category}{thrM}_{side}_ratio_today_pc`, `{category}{thrM}_{side}_ratio_5d_pc`

- **Error Handling Strategy**:
  - Fail-fast approach: any data inconsistency immediately raises ValueError with descriptive message
  - All errors include symbol context for debugging
  - No silent fallbacks or approximations that could compromise data precision

### Phase 3: Testing & Validation
- [ ] Manual Testing
  - **Scenarios**:
    - Multi-day data with varying candle counts (max diff ≤ 1) → per-candle baselines adjust correctly
    - Earliest dates with <5 priors → `_ratio_5d_pc` raises error (strict 5-day requirement)
    - Day with `Price.value <= 0` → raises `ValueError` with symbol/date immediately
    - Inconsistent candle counts → raises `ValueError` with actual counts
    - Missing expected 5D baseline → raises `ValueError` with symbol/date
    - Trade value thresholds → verify shark/sheep classification with million-unit thresholds
  - **Result**: Verify DataFrame columns, types, precise ratio calculations (4 decimals), and robust error handling
- [ ] Unit Testing (Optional)
  - **Scope**:
    - Classification and aggregation per threshold/side with million-unit conversions
    - 5D trailing requires exactly 5 prior days; denominator uses `5 * base_candle_count`
    - Error handling for all data validation scenarios
    - Ratio precision (4 decimal rounding) and baseline computation accuracy

## 📊 Summary of Results
- The calculator yields robust per-candle whale/sheep features with two normalizations that make cross-day comparison stable.
- **Data Precision**: Strict error handling ensures no invalid data passes through, maintaining calculation accuracy.
- **Monetary Units**: All values consistently handled in millions with proper threshold scaling and conversion.
- **Output**: DataFrame with time index, containing value and ratio columns rounded to 4 decimals for precision.

## 🚧 Outstanding Issues & Follow-up
- Parameterize thresholds via config if needed
- Optionally expose inclusion of `Undefined` side as separate features later

## 🔍 Review Checklist
- [x] Documentation updated (this doc and code docstrings) 
- [x] Code adheres to ruff/black rules
- [x] Strict error handling implemented for data precision
- [x] Million-unit monetary handling documented and implemented
- [ ] CHANGELOG updated when merged

---
**Last updated**: 2025-09-07
