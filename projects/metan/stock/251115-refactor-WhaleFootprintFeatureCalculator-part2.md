# 📋 [TICKET-ID: 2025-11-15] - Refactor WhaleFootprintFeatureCalculator Part 2 (Modular per-feature architecture)

## User Requirements
- Separate and group each feature into a single file to scale with many features, using the same `whale_footprint` folder.
- Keep ISO 8601 UTC `time` across calculation and persistence.
- Maintain existing outputs and `whale_footprint` namespace for backward compatibility.
- Apply naming conventions: point-in-time (plain), accumulative (`accum_`), moving-window (`mov_<N>_`).
- Use `common/base.py` for shared/general functions; keep it feature-agnostic.

## 🎯 Objective
Modularize whale footprint into a feature-per-file architecture backed by shared base utilities. Preserve current outputs while enabling rapid addition of future features with clear boundaries and a lightweight registry.

### ⚠️ Key Considerations
- Units: all monetary values are in millions; thresholds provided in millions.
- Strict candle-count equality across trading days.
- Baselines: `value_today_pc`, `value_5d_pc` per trading day.
- ISO time strings end-to-end; Supabase `timestamptz` accepts ISO.
- Backward compatible feature keys and namespace.

## 🔄 Implementation Plan
[Don't require running any test]

### Phase 1: Analysis & Preparation
- [x] Analyze detailed requirements and taxonomy
  - **Outcome**: Identify distinct features: thresholded shark/sheep values, cumulative average prices, 5D ratios, urgency spread, future moving-window variants.
- [x] Define scope and edge cases
  - **Outcome**: Enforce 5 prior trading days for `value_5d_pc`; handle empty days; keep ISO time.
- [x] Define feature module interface
  - **Outcome**: `compute(row, ctx)` returning only additional columns, where `ctx` provides `agg`, `threshold_meta`, `value_5d_pc`, and daily trackers.

### Phase 2: Implementation (File/Code Structure)
> Single-folder approach and shared utilities under a common folder. Status markers (✅ Implemented, 🔄 In Progress, 🚧 To-Do)

```
packages/stock/metan/stock/trading/domain/feature/calculator/
├── common/                               # ✅ Shared/general functions for all features
│   └── base.py                           # ✅ Daily aggregation, baselines, ISO rows, trackers (no feature-specific logic)
└── whale_footprint/                      # 🔄 All whale footprint code in the same folder
    ├── whale_footprint_feature_calculator.py  # 🔄 Orchestrates baselines, trackers, and feature registry
    ├── shark_values.py                   # ✅ Aggregates ticks → merged `agg` (values, volumes, weighted_values) and writes value fields
    ├── avg_prices.py                     # ✅ Uses `agg` to update trackers and emit `{cat}{T}_{side}_avg_price`
    ├── ratios_5d_pc.py                   # ✅ Emits `{base}_ratio_5d_pc` using `value_5d_pc` and only `_value` keys
    └── urgency_spread.py                 # ✅ Computes VWAP from `agg` and emits `shark_urgency_spread`
```

### Phase 3: Detailed Implementation Steps
- Create a lightweight feature registry in the calculator
  - `FEATURES = [shark_values.compute, avg_prices.compute, ratios_5d_pc.compute, urgency_spread.compute]`
- For each trading day
  - Initialize daily trackers via `common.base.initialize_daily_trackers`
  - Read baselines via `common.base.compute_per_candle_baselines`
    - Baseline field names: `value_today_pc`, `value_5d_pc`
- For each candle
  - Build base row via `common.base.build_base_row_iso_time`
  - Aggregate thresholded actions via `whale_footprint.shark_values.aggregate_single_candle` to produce merged `agg`
    - `agg` contains `{cat}{T}_{side}_value`, `{cat}{T}_{side}_volume`, `{cat}{T}_{side}_weighted_value`
  - Construct `ctx`: `{agg, threshold_meta, value_5d_pc, cumulative_trackers, previous_avg_prices}`
  - Apply each `compute(row, ctx)` in registry to append columns
- Keep `common/base.py` as the source for base utilities only (no feature-specific row additions)
- Maintain current columns and naming; add new features without breaking existing consumers

## 📊 Summary of Results
- Clear separation: base utilities vs. per-feature modules.
- Scalable design: add features by creating a file and registering its `compute`.
- Consistent ISO time and baseline usage preserved (`value_today_pc`, `value_5d_pc`).

### ✅ Completed Achievements
- Renamed baselines to `value_today_pc` and `value_5d_pc` across computation and logging.
- Merged accumulations into a unified `agg` dict returned by `shark_values.aggregate_single_candle`.
- Updated feature modules to consume `agg` only and emit their columns accordingly.
- Verified calculator wiring with ISO time and persistor expectations.

## 🚧 Outstanding Issues & Follow-up
- Add moving-window variants (`mov_15_`, etc.) for applicable features.
- Consider a `TimeKey` helper to standardize ISO UTC formatting across modules.

### ⚠️ Known Issues (Optional)
- `value_5d_pc` requires 5 prior trading days; missing history raises errors.

### 🔮 Future Improvements (Optional)
- Extend baselines (e.g., 20D) and configurable calendars.
- Cache tick-derived daily sums for large-range performance.
- Standardize naming suffixes across ratio fields for discoverability.