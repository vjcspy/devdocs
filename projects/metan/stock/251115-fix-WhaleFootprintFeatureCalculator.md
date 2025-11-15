# 📋 [TICKET-ID: 2025-11-15] - Fix WhaleFootprintFeatureCalculator

## User Requirements
> 1. Read overview file in `devdocs/projects/metan/stock/OVERVIEW.md` to understand the project.
> 2. Read carefully `packages/stock/metan/stock/trading/domain/feature/calculator/whale_footprint/whale_footprint_feature_calculator.py`, `packages/stock/metan/stock/info/domain/stock_data_collector/stock_data_collector.py` and all relevant files (if needed) to understand the current implementation
> 3. You can see in `compute_per_candle_baselines` we need to have 5 previous days to calculate `pc_value_5d` value. But now in `tick_candles_by_date` we just return 1 day candles. Read `prices` method to know how we can retrieve 5 previous days.
> 4. I want to validate we have same total days in prices and ticks (compare every row in prices and ticks to ensure they are same)
> 5. Chỗ này tôi giải thích bằng Vietnamese cho rõ. Chúng ta chỉ cần tính toán feature từ start_date tới end_date, nhưng để tính toán các giá trị base line bắt buộc phải lấy thêm 5 ngày trước đó. Tuy nhiên, một số hàm validate hiện tại ví dụ như ở trong `compute_per_candle_baselines` đang check như sau:
>    ```
>    bad = daily_df[(daily_df["candle_count"].notna()) & (daily_df["pc_value_today"] <= 0)].index.tolist()
>    ```
>
>    Như vậy là đang bị sai do những ngày chúng ta lấy thêm sẽ không cần validate, nó chỉ nhằm mục đích tính toán các base values.
>
>    Chúng ta cần sửa chỉ validate các ngày trong khoảng thời gian từ start_date đến end_date mà thôi
>
>    Cũng do point này nên khi build rows ở method `iter_candle_dates` cũng chỉ lấy các ngày từ start_date đến end_date để tính toán cho các feature row. Nhưng cần kiểm tra nếu cần tính toán base value vẫn phải tính đủ. => chỗ này cần kiểm tra kỹ nhé

## 🎯 Objective
> Ensure 5-day baseline availability without over-validating extra days, and compute features strictly for `start_date…end_date` while verifying day-set consistency between prices and ticks.

- Provide `pc_value_5d` for in-range days by including five prior trading days from `prices()` and scoping validations to the requested range.
- Confirm that the set of trading days in ticks equals the set of days in prices for the requested window.

### ⚠️ Key Considerations
- Rolling baselines require five prior trading days; these extra days must be excluded from validations and row building.
- Units: tick candle `value` is in millions; prior-day daily `value` used for baselines must be normalized to the same units before averaging.
- `iter_candle_dates` must only yield dates with real candles in-range; extra baseline-only days should have `candle_count = NaN` so they’re naturally skipped.
- Clear logging and strict errors when day sets diverge ensure downstream persistence is consistent.

## 🔄 Implementation Plan
[Don't require running any test]

### Phase 1: Analysis & Preparation
- [ ] Analyze detailed requirements
  - **Outcome**: Five prior days must be included via `prices()` while avoiding validations and row generation outside `start_date…end_date`. Verify prices vs ticks day alignment.
- [ ] Define scope and edge cases
  - **Outcome**: Handle fewer than five prior days (fail fast), missing ticks for a price day (error), unit normalization for prior-day `value`.

### Phase 2: Implementation (File/Code Structure)
> Proposed changes across existing modules; status markers show intent.

```
packages/stock/metan/stock/info/domain/stock_data_collector/
├── stock_data_collector.py            # 🚧 TODO - Add prices vs ticks day-set validation helper
packages/stock/metan/stock/trading/domain/feature/calculator/common/
├── base.py                            # 🚧 TODO - Support building daily_df with extra 5 baseline-only days
packages/stock/metan/stock/trading/domain/feature/calculator/whale_footprint/
├── whale_footprint_feature_calculator.py # 🚧 TODO - Use combined daily_df; scope validations; build rows only in-range
```

- `stock_data_collector.py: prices()` already fetches five extra days before `start_date` (257–350).
- `stock_data_collector.py: tick_candles_by_date()` currently returns only in-range per-day candles (122–255).
- `common/base.py: compute_per_candle_baselines()` requires five prior days and validates non-NaN days (33–55).
- `common/base.py: iter_candle_dates()` yields only days with non-NaN `candle_count` (65–69).
- `whale_footprint_feature_calculator.py: _cal_candle_features()` builds `daily_info_df` from ticks and runs baselines (64–106).

### Phase 3: Detailed Implementation Steps
- Update daily_df construction to include baseline-only prior days
  - Extend `gather_daily_value_and_candle_counts_from_ticks(...)` usage by merging a prior-days frame sourced from `prices()` for dates `< start_date`.
  - For prior days, set `candle_count = NaN` and populate `daily_value` using price daily traded value normalized to millions (e.g., `price.value / 1_000_000` if provider value is raw).
  - Result: `daily_df` covers `[five prior days] + [start_date…end_date]`; validations in `compute_per_candle_baselines()` only apply to in-range days due to `notna()` checks.
- Scope validations to request window
  - Keep current checks in `compute_per_candle_baselines()` but ensure prior-days rows carry `candle_count = NaN` so `bad` and `missing` collect only in-range dates (`common/base.py:33–55`).
  - Confirm `iter_candle_dates()` continues to emit only in-range dates (`common/base.py:65–69`).
- Build rows strictly for in-range dates
  - In `_build_rows(...)`, iterate over `iter_candle_dates(daily_df)` so only in-range dates produce feature rows (`whale_footprint_feature_calculator.py:114–164`).
  - Use `pc_value_5d` computed from the merged `daily_df` so early in-range days have valid baselines.
- Add prices vs ticks day-set validation
  - Implement a helper in `StockDataCollector` to compare the set of days in `prices()` filtered to `start_date…end_date` with keys from `tick_candles_by_date()` and raise on mismatch.
  - Reference points: `stock_data_collector.py:122–255` for tick days and `stock_data_collector.py:257–350` for price days.
- Logging and errors
  - Add clear info logs for counts and sets used in the comparison; raise descriptive `ValueError` on divergence.
- Edge cases
  - If fewer than five prior trading days exist before `start_date`, fail with an explicit error indicating insufficient history for baselines.
  - If a price day has no ticks, abort with a descriptive error (existing behavior in `tick_candles_by_date`).

## 📊 Summary of Results

### ✅ Completed Achievements
- Planning defined to provide `pc_value_5d` without over-validating baseline-only days, generate features only for the requested window, and verify day-set consistency.

## 🚧 Outstanding Issues & Follow-up

### ⚠️ Known Issues (Optional)
- [ ] Provider `value` unit confirmation for daily prices; ensure normalization to millions to match tick candles.
- [ ] Handling symbols with holidays or gaps that reduce available prior-day count.

### 🔮 Future Improvements (Optional)
- [ ] Cache the prior-day merged `daily_df` per symbol+window to avoid recomputation across calculators.
- [ ] Add unit tests to assert day-set equality and baseline correctness under sparse histories.