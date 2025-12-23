# 📋 [TICKET-ID: 251221-Refactor-StockDataCollector-TickCandles] - Refactor Tick Candles By Date Logic

## References

> `packages/stock/metan/stock/info/domain/stock_data_collector/stock_data_collector.py`
> `devdocs/projects/metan/stock/OVERVIEW.md`

## User Requirements

> 1. Khi chúng ta iterate: `for bucket_time in schedule_bucket_times:` đang check nếu không có pairs trong bucket_time đó thì đưa ra lỗi ngay. Tuy nhiên trong thực tế có 1 số cổ phiểu không có giao dịch trong một số bucket.
> 2. Chúng ta chỉ cần verify `interval_tick_buckets` > 20 chẳng hạn để đảm bảo là phải có ít nhất 20 bucket có giao dịch.
> 3. Còn trường hợp không có giao dịch trong bucket đó thì hãy lấy giá mở, đóng, cao, thấp theo bucket trước đó.
> 4. Còn nếu là bucket đầu tiên thì lấy theo giá mở của từ price.

## 🎯 Objective

Refactor `tick_candles_by_date` method in `StockDataCollector` to handle missing trading activity in specific time buckets gracefully. Instead of raising an error for any missing bucket, the system should allow gaps, filling them with previous close prices (or daily open price for the first bucket), provided that a minimum threshold of active buckets (e.g., > 20) is met for the day.

### ⚠️ Key Considerations

- **Data Integrity**: Ensure that days with very little activity (<= 20 active buckets) are still flagged as invalid to avoid bad data.
- **Data Continuity**: Missing buckets should be filled as flat candles (OHLC = previous close, Volume = 0) to maintain time continuity for downstream consumers.
- **Fallback Source**: Need to access the daily `Price` object to get the open price for the case where the first bucket of the day is empty.

## 🔄 Implementation Plan

[Don't require running any test]

### Phase 1: Analysis & Preparation

- [ ] **Analyze Data Availability**:
    -   The `StockDataCollector` already has a `prices()` method that returns daily prices. We need to index these by date to easily access the daily Open price.
- [ ] **Define Logic for Filling Gaps**:
    -   **Validation**: `count(active_buckets) > 20`. If not, raise `ValueError`.
    -   **Iteration**: Loop through `schedule_bucket_times`.
    -   **Case 1 (Active Bucket)**: Calculate OHLCV from ticks as currently implemented. Update `last_close_price`.
    -   **Case 2 (Empty Bucket)**:
        -   **If First Bucket**: Use `daily_price.open` as OHLC. Update `last_close_price`.
        -   **If Subsequent Bucket**: Use `last_close_price` as OHLC. Volume/Value = 0.

### Phase 2: Implementation (File/Code Structure)

**File**: `packages/stock/metan/stock/info/domain/stock_data_collector/stock_data_collector.py`

- [ ] **Refactor `tick_candles_by_date`**:
    -   Fetch `prices` list and convert to a dict `prices_by_date` for O(1) access.
    -   Remove the strict check inside the loop that raises `ValueError` if `pairs` is empty.
    -   Add pre-check: `if len(interval_tick_buckets) <= 20: raise ValueError(...)`.
    -   Implement the logic to maintain `previous_close` state.
    -   Handle the first bucket empty case using `prices_by_date[date].open`.

### Phase 3: Detailed Implementation Steps

1.  **Prepare Daily Prices**:
    -   Call `self.prices()` at the beginning of `tick_candles_by_date`.
    -   Create `prices_map = {p.date: p for p in self.prices()}`.

2.  **Loop Modification**:
    -   Inside the date loop (`for date in sorted(ticks_by_date.keys()):`):
        -   ... (existing bucket calculation) ...
        -   **Add Check**: `if len(interval_tick_buckets) <= 20: raise ValueError(...)`.
        -   Get `daily_price = prices_map.get(date)`. Ensure it exists (it should, based on `prices()` logic, but good to handle safety).
        -   Initialize `previous_close = daily_price.open`.
        -   Iterate `schedule_bucket_times`:
            -   Check if `bucket_time` in `interval_tick_buckets`.
            -   **If yes**:
                -   Compute Candle (existing logic).
                -   `previous_close = candle.close`.
            -   **If no**:
                -   Create filler Candle:
                    -   `open = close = high = low = previous_close`.
                    -   `volume = 0`, `value = 0`.
                    -   `tick_actions = []`.

## 📊 Summary of Results

### ✅ Completed Achievements
- [ ] Refactored `tick_candles_by_date` to support sparse trading days.
- [ ] Implemented gap filling strategy using previous close or daily open.
- [ ] Enforced minimum activity threshold (> 20 active buckets).

## 🚧 Outstanding Issues & Follow-up

### ⚠️ Issues/Clarifications (Optional)
- [ ] Confirm if 20 buckets is the hard fixed number or should be configurable (will stick to 20 hardcoded for now as per prompt).
