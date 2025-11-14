# 📋 [TICKET-ID: 2025-11-13] - Refactor WhaleFootprintFeatureCalculator + Align Persistor (ISO time)

## User Requirements
- Align `WhaleFootprintFeatureCalculator` with `StockDataCollector` refactor (see `devdocs/projects/metan/stock/251109-refactor-StockDataCollector.md`).
- Use ISO 8601 UTC for all `time` properties. Feature rows must return `time` as ISO string.
- Compute `daily_value` from `TickCandle` by summing `TickCandle.value` per date (stop using `Price.value`).
- In `_validate_and_get_base_candle_count`, ensure all `candle_count` are identical across trading days.
- Extract and group calculation functions into `packages/stock/metan/stock/trading/domain/feature/calculator/whale_footprint/_whale_footprint_feature_calculator` for readability and future scaling.
- Rename baselines: `today_pc` → `pc_value_today`, `pc_5d` → `pc_value_5d`.
- Align `IntradaySymbolFeaturePersistor` so `time` is persisted as ISO string compatible with Supabase `timestamptz`.

## 🎯 Objective
Standardize `time` handling to ISO UTC across calculation and persistence, base `daily_value` on tick-derived sums, enforce strict candle-count consistency, rename baseline columns for clarity, and consolidate helpers to support future features. Ensure Supabase receives ISO strings in `timestamptz` `time` column.

### ⚠️ Key Considerations
- `TickAction.time` and `TickCandle.time` are ISO UTC; never cast to `int` in calculator or persistor.
- Supabase `time` column is `timestamp with time zone`; it accepts ISO 8601 strings (e.g., `...Z`).
- 5D baseline requires at least 5 prior trading days; raise clear errors when insufficient.
- All “value” units (daily, candle, trade) are in millions; keep unit consistency.
- Strict `candle_count` equality prevents subtle data drift across days.
- Sorting by ISO strings is chronological when all times are UTC and consistent.

## 🔄 Implementation Plan
[Don’t require running any test]

### Phase 1: Analysis & Preparation
- [x] Analyze detailed requirements and prior refactor
  - **Outcome**: Confirm ISO time throughout; daily_value from ticks; strict candle_count equality; baseline renames; helper consolidation; persistor alignment.
- [x] Define scope and edge cases
  - **Outcome**: Ensure 5 prior trading days exist for `pc_value_5d`; handle days without candles; note downstream impacts of switching `time` to ISO string.

### Phase 2: Implementation (File/Code Structure)
- `packages/stock/metan/stock/trading/domain/feature/calculator/whale_footprint/whale_footprint_feature_calculator.py` – 🚧 TODO
  - Return row `time` as ISO string; index DataFrame by `time` (string).
  - `_gather_daily_price_and_candle_counts`: compute `daily_value = Σ TickCandle.value` per date; derive `candle_count` from ticks.
  - `_validate_and_get_base_candle_count`: enforce strict equality across days; no min/max tolerance.
  - `_compute_per_candle_baselines`: rename columns `today_pc` → `pc_value_today`, `pc_5d` → `pc_value_5d`.
  - Update all call sites and logs to new names and types.
- `packages/stock/metan/stock/trading/domain/feature/calculator/whale_footprint/_whale_footprint_feature_calculator/engine.py` – 🚧 TODO
  - Consolidate helpers into a single module with clear docstrings:
    - `gather_daily_value_and_candle_counts_from_ticks(...)`
    - `validate_and_get_base_candle_count_strict(...)`
    - `compute_per_candle_baselines(...)`
    - `iter_candle_dates(...)`
    - `build_base_row_iso_time(...)`
    - `aggregate_single_candle(...)`
    - `add_shark_sheep_values(...)`
    - `calculate_and_add_avg_prices(...)`
    - `add_5d_ratios(...)`
  - Avoid hidden class state; pass explicit parameters (logger, thresholds) as needed.
- `packages/stock/metan/stock/trading/domain/feature/persistor/intraday/intraday_symbol_feature_persistor.py` – 🚧 TODO
  - `_build_base_candles`: set `time=str(c.time)`; keep OHLCV/value unchanged; sort by ISO `time` string.
  - `_infer_feature_frame`: accept `time` index as string; if `time` column exists, set as index without coercion to `int`.
  - `_merge_features_into_rows`: use per-namespace lookup keyed by ISO `time` strings; remove integer coercion; match `c.time` against string keys.
  - `_persist_rows`: upsert rows with ISO `time`; keep `on_conflict="symbol,time"`.

### Phase 3: Detailed Implementation Steps
- ISO `time` in calculator rows
  - Replace `int(candle.time)` with `str(candle.time)` in row builder; keep DataFrame sort on `date`, `time` and set index to `time` (string).
- Tick-based `daily_value`
  - Iterate `tick_candles_by_date` to compute `daily_value = sum(float(tc.value) for tc in candles)` and `candle_count = len(candles)`.
  - Stop reading `self.data_collector.prices()` for daily value purposes.
- Strict `candle_count` equality
  - Derive unique non-null counts; if `len(unique_counts) != 1` → raise `ValueError("Inconsistent candle_count across days: ...")`.
- Baseline renames and usage
  - `pc_value_today = daily_value / base_candle_count` for trading days.
  - `pc_value_5d = (Σ prior 5 daily_value) / (5 * base_candle_count)`; ensure NaN for non-trading days.
  - Use these columns in row building and logging.
- Helper extraction
  - Move existing helper methods into `engine.py` and update calculator to delegate.
- Persistor alignment
  - Base candles use ISO `time` strings.
  - Feature frame inference accepts string `time` index; exclude non-feature columns.
  - Feature merging keyed on string `time`; raise if missing per namespace.
  - Upsert with ISO `time` to Supabase `timestamptz`.

## 📊 Summary of Results
- Feature calculation and persistence both operate with ISO string `time` aligned to Supabase `timestamptz`.
- `daily_value` is computed from tick-derived `TickCandle.value` sums per date.
- Strict `candle_count` equality enforced across trading days.
- Baseline columns renamed to `pc_value_today` and `pc_value_5d` for clarity.
- Calculation helpers consolidated in a single module for readability and scalability.

### ✅ Completed Achievements
- Prepared a detailed refactor plan aligned with StockDataCollector time changes and persistor requirements.

## 🚧 Outstanding Issues & Follow-up
- Confirm `FeatureBaseCandleRow.time` type supports `str`; update model if it enforces `int`.
- Validate Supabase ingestion of ISO strings via client library; ensure consistent `Z` suffix.
- Audit downstream consumers expecting integer `time` and migrate them to ISO strings.
- Decide if ratio column suffixes should be updated for consistency (currently `_ratio_5d_pc`).

### ⚠️ Known Issues (Optional)
- 5D baseline requires 5 prior trading days; ranges without sufficient history will error.

### 🔮 Future Improvements (Optional)
- Add a common `TimeKey` utility to normalize ISO formatting and enforce UTC.
- Introduce fixed baseline candle counts for common intervals (`5, 15, 30, 60`) if business rules allow.
- Add caching for daily tick sums to improve performance over large ranges.
- Extend baselines (e.g., 20D) and adopt trading-day calendars.

---
**Last updated**: 2025-11-13
---