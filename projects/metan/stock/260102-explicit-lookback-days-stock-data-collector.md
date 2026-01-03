# 📋 [STOCK-260102: 2026-01-02] - Explicit Lookback Days in StockDataCollector

## References

- `packages/stock/metan/stock/info/domain/stock_data_collector/stock_data_collector.py`
- `packages/stock/metan/stock/trading/domain/feature/calculator/common/base.py`
- `packages/stock/metan/stock/trading/domain/feature/persistor/intraday/intraday_symbol_feature_persistor.py`
- `packages/stock/metan/stock/trading/domain/feature/calculator/whale_footprint/whale_footprint_feature_calculator.py`

## User Requirements

Currently, `StockDataCollector` implicitly fetches 5 extra days before `start_date` in `_effective_start_date()`. This is hidden behavior needed for feature calculations (5-day rolling window). However:
- It's confusing and implicit
- Other use cases that don't need lookback still get redundant data
- The "5" is a magic number with no documentation

**Goal**: Make `lookback_days` an explicit constructor parameter with a default of 5 days.

## 🎯 Objective

Refactor `StockDataCollector` to accept an explicit `lookback_days` parameter, making the data fetching behavior transparent and configurable while maintaining backward compatibility.

### ⚠️ Key Considerations

1. **Backward Compatibility**: Default value of 5 ensures existing code continues to work without changes
2. **Documentation**: The parameter should be well-documented explaining its purpose
3. **Constant Definition**: Define `FEATURE_ROLLING_WINDOW_DAYS = 5` as a named constant for clarity
4. **Usage Sites**: All places using `StockDataCollector` should be reviewed to ensure they use the appropriate lookback

## 🔄 Implementation Plan

### Phase 1: Analysis & Preparation

- [x] Analyze current implementation
  - **Outcome**: `_effective_start_date()` adds 5 extra days implicitly via `desired_limit = count_val + 5`
- [x] Identify all usage sites
  - **Outcome**: 
    - `IntradaySymbolFeaturePersistor` - needs lookback for feature calculation
    - `WhaleFootprintFeatureCalculator` - receives collector, uses lookback data
    - `VN30BaseCalculator` - needs lookback
    - `TickVN30IndexCalculator` - needs lookback

### Phase 2: Implementation (File/Code Structure)

```
packages/stock/metan/stock/
├── info/domain/stock_data_collector/
│   └── stock_data_collector.py      # ✅ IMPLEMENTED - lookback_days parameter
├── trading/domain/feature/
│   ├── calculator/
│   │   └── common/
│   │       └── base.py              # ✅ IMPLEMENTED - FEATURE_ROLLING_WINDOW_DAYS constant
│   └── persistor/
│       └── intraday/
│           └── intraday_symbol_feature_persistor.py  # ✅ Uses default (backward compatible)
```

### Phase 3: Detailed Implementation Steps

#### Step 1: Define constant in `base.py`

Add a named constant to document the rolling window requirement:

```python
# packages/stock/metan/stock/trading/domain/feature/calculator/common/base.py

# At top of file, after imports
FEATURE_ROLLING_WINDOW_DAYS = 5  # Required for pc_value_5d 5-day rolling calculation
```

#### Step 2: Update `StockDataCollector.__init__()` 

Modify constructor to accept explicit `lookback_days` parameter:

```python
# packages/stock/metan/stock/info/domain/stock_data_collector/stock_data_collector.py

def __init__(
    self, 
    symbol: str, 
    start_date: str, 
    end_date: str, 
    interval: IntradayInterval,
    lookback_days: int = 5,  # Default 5 for backward compatibility
):
    self.symbol = symbol
    self.start_date = start_date
    self.end_date = end_date
    self.interval = interval
    self.lookback_days = lookback_days
    self._start_date_to_fetch: str | None = None
    self._logger.info(f"[{self._ctx()}] init StockDataCollector (lookback_days={lookback_days})")
```

#### Step 3: Update `_effective_start_date()` method

Replace hardcoded `5` with `self.lookback_days`:

```python
def _effective_start_date(self) -> str:
    if self._start_date_to_fetch is not None:
        return self._start_date_to_fetch

    # If no lookback needed, return start_date directly
    if self.lookback_days == 0:
        self._start_date_to_fetch = self.start_date
        return self._start_date_to_fetch

    # Step 1: count how many rows exist in the requested range [start_date, end_date]
    count_resp = (
        supabase.table("stock_info_prices")
        .select("id", count=cast(Any, "exact"))
        .eq("symbol", self.symbol)
        .gte("date", self.start_date)
        .lte("date", self.end_date)
        .execute()
    )
    count_val = count_resp.count
    if count_val == 0:
        raise ValueError(f"No price data found for {self.symbol} between {self.start_date} and {self.end_date}")

    # Use lookback_days instead of hardcoded 5
    desired_limit = int(count_val or 0) + self.lookback_days
    self._logger.info(f"[{self._ctx()}] price count={count_val}, desired_limit={desired_limit} (lookback={self.lookback_days})")

    # Step 2: fetch up to end_date only, limiting to the window size + lookback
    fetch_resp = (
        supabase.table("stock_info_prices")
        .select("*")
        .eq("symbol", self.symbol)
        .lte("date", self.end_date)
        .order("date", desc=True)
        .limit(desired_limit)
        .execute()
    )

    if not fetch_resp.data and len(fetch_resp.data) < 1:
        raise ValueError(f"No price data found for {self.symbol} between {self.start_date} and {self.end_date}")

    self._start_date_to_fetch = fetch_resp.data[-1]["date"]

    return self._start_date_to_fetch
```

#### Step 4: Update `_ctx()` method (optional, for better logging)

```python
def _ctx(self) -> str:
    return f"{self.symbol}|{self.start_date}->{self.end_date}|{int(self.interval)}s|lb={self.lookback_days}"
```

### Phase 4: Verification

After implementation:
1. Existing code should work unchanged (default `lookback_days=5`)
2. New callers can explicitly pass `lookback_days=0` when no lookback is needed
3. Feature calculations continue to work correctly

## 📊 Summary of Results

### ✅ Completed Achievements
- [x] Added `FEATURE_ROLLING_WINDOW_DAYS = 5` constant in `base.py`
- [x] Added `lookback_days: int = 5` parameter to `StockDataCollector.__init__()`
- [x] Updated `_effective_start_date()` to use `self.lookback_days` instead of hardcoded `5`
- [x] Updated `_ctx()` to include lookback info in logs (`lb={self.lookback_days}`)
- [x] Added docstring explaining the `lookback_days` parameter

## 🚧 Outstanding Issues & Follow-up

### Future Considerations
- [ ] Consider adding a `FeatureDataProvider` wrapper class if more feature-specific data requirements emerge
- [ ] Consider allowing each feature calculator to specify its own required lookback period

