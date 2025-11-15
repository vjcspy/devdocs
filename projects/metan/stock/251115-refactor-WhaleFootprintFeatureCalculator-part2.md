# 📋 [TICKET-ID: 2025-11-15] - Refactor WhaleFootprintFeatureCalculator Part 2 (Modular per-feature architecture)

## User Requirements
- Separate and group each feature into a single file to scale with many features.
- Keep ISO 8601 UTC `time` across calculation and persistence.
- Maintain existing outputs and `whale_footprint` namespace for backward compatibility.
- Apply naming conventions: point-in-time (plain), accumulative (`accum_`), moving-window (`mov_<N>_`).

## 🎯 Objective
Modularize whale footprint into a feature-per-file architecture backed by a shared engine. Preserve current outputs while enabling rapid addition of future features with clear boundaries and a lightweight registry.

### ⚠️ Key Considerations
- Units: all monetary values are in millions; thresholds provided in millions.
- Strict candle-count equality across trading days.
- Baselines: `pc_value_today`, `pc_value_5d` per trading day.
- ISO time strings end-to-end; Supabase `timestamptz` accepts ISO.
- Backward compatible feature keys and namespace.

## 🔄 Implementation Plan
[Don't require running any test]

### Phase 1: Analysis & Preparation
- [x] Analyze detailed requirements and taxonomy
  - **Outcome**: Identify distinct features: thresholded shark/sheep values, cumulative average prices, 5D ratios, urgency spread, future moving-window variants.
- [x] Define scope and edge cases
  - **Outcome**: Enforce 5 prior trading days for `pc_value_5d`; handle empty days; keep ISO time.
- [x] Define feature module interface
  - **Outcome**: `compute(row, ctx)` returning only additional columns, where `ctx` provides `agg`, `threshold_meta`, `pc_value_5d`, and daily trackers.

### Phase 2: Implementation (File/Code Structure)
> Single-folder approach and shared utilities under a common folder. Status markers (✅ Implemented, 🔄 In Progress, 🚧 To-Do)

```
packages/stock/metan/stock/trading/domain/feature/calculator/
├── common/                               # 🚧 Shared/general functions for all features
│   └── common.py                         # 🚧 Daily aggregation, baselines, ISO rows, trackers, ratios
└── whale_footprint/                      # 🔄 All whale footprint code in the same folder
    ├── whale_footprint_feature_calculator.py  # 🔄 Orchestrates baselines, trackers, and feature registry
    ├── shark_values.py                   # 🚧 Emits shark/sheep {T}_{buy|sell}_value
    ├── avg_prices.py                     # 🚧 Emits {cat}{T}_{side}_avg_price via cumulative trackers
    ├── ratios_5d_pc.py                   # 🚧 Emits {base}_ratio_5d_pc using pc_value_5d
    └── urgency_spread.py                 # 🚧 Emits shark_urgency_spread normalized by vwap
```

Alternative folder name for shared utilities: `core/` or `shared/` instead of `common/` if preferred.

### Phase 3: Detailed Implementation Steps
- Create a lightweight feature registry in the calculator
  - `FEATURES = [shark_values.compute, avg_prices.compute, ratios_5d_pc.compute, urgency_spread.compute]`
- For each trading day
  - Initialize daily trackers via `common.initialize_daily_trackers`
  - Read baselines from `common.compute_per_candle_baselines`
- For each candle
  - Build base row via `common.build_base_row_iso_time`
  - Aggregate thresholded actions via `common.aggregate_single_candle`
  - Construct a `ctx` object: `{agg, threshold_meta, pc_value_5d, cumulative_trackers, previous_avg_prices}`
  - Apply each `compute(row, ctx)` in registry to append columns
- Keep `common.py` as the single source for daily aggregation, baselines, ISO time, and shared helpers
- Maintain current columns and naming; add new features without breaking existing consumers

## 📊 Summary of Results
- Clear separation: shared engine vs per-feature modules.
- Scalable design: add features by creating a file and registering its `compute`.
- Consistent ISO time and baseline usage preserved.

### ✅ Completed Achievements
- Delegated 5D ratio logic to shared helper (to be moved into `common.py`).
- Verified calculator- and persistor-side ISO time alignment.

## 🚧 Outstanding Issues & Follow-up
- Implement `features/` package and migrate existing logic into dedicated modules.
- Add moving-window variants (`mov_15_`, etc.) for applicable features.
- Consider a `TimeKey` helper to standardize ISO UTC formatting across modules.

### ⚠️ Known Issues (Optional)
- `pc_value_5d` requires 5 prior trading days; missing history raises errors.

### 🔮 Future Improvements (Optional)
- Extend baselines (e.g., 20D) and configurable calendars.
- Cache tick-derived daily sums for large-range performance.
- Standardize naming suffixes across ratio fields for discoverability.