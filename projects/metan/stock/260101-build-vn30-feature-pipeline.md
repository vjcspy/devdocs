# 📋 260101: Build VN30 Complete Feature Pipeline

> **Status:** ✅ IMPLEMENTED  
> **Date:** 2026-01-01

## References

- Overview: `devdocs/projects/metan/stock/OVERVIEW.md`
- Tick VN30 Index Calculator: `devdocs/projects/metan/stock/260101-implement-tick-vn30-index-calculator.md`
- VN30 Aggregate Features Plan: `devdocs/projects/metan/stock/251229-vn30-aggregate-features-plan.md`
- Existing VN30 Aggregate Script: `packages/stock/metan/stock/testbed/calculate_vn30_aggregate.py`
- Feature Persistor: `packages/stock/metan/stock/trading/domain/feature/persistor/intraday/intraday_symbol_feature_persistor.py`
- VN30 Aggregator: `packages/stock/metan/stock/trading/domain/feature/aggregator/vn30/vn30_whale_footprint_aggregator.py`

## User Requirements

1. Tạo script build đầy đủ thông tin cho VN30 để lưu vào `stock_trading_feature_candles`
2. Gọi tính toán index candle (sử dụng `TickVN30IndexCalculator`)
3. Gọi tính toán features cho từng symbol trong rổ VN30 - **tối ưu: skip nếu đã tồn tại trong DB**
4. Aggregate features cho VN30
5. Merge index OHLCV + aggregated features và persist

**Tối ưu quan trọng:**
- Fetch dữ liệu đã có từ DB 1 lần duy nhất với time range
- Check các ngày đã tính → không tính lại
- Áp dụng tương tự cho cả 30 symbols và VN30 symbol

## 🎯 Objective

Implement `VN30FeaturePipeline` class that:
1. Calculates VN30 index candles from tick data
2. Calculates whale footprint features for all VN30 component symbols (with smart skip)
3. Aggregates features into VN30-level metrics
4. Persists complete VN30 data (OHLCV + features) to `stock_trading_feature_candles`

### ⚠️ Key Considerations

1. **VN30 as a Symbol**: Treat VN30 as `symbol="VN30"` in `stock_trading_feature_candles` table, với đầy đủ OHLCV từ index calculator và features từ aggregator.

2. **Optimization Strategy - Batch Date Check**:
   ```python
   # 1. Query existing dates for ALL symbols at once (including VN30)
   existing_dates_by_symbol = fetch_existing_dates(symbols, start_date, end_date)
   
   # 2. Determine which symbols need calculation for which dates
   for symbol in symbols:
       missing_dates = requested_dates - existing_dates_by_symbol[symbol]
       if missing_dates:
           calculate_features_for_dates(symbol, missing_dates)
   ```

3. **Data Flow**:
   ```
   ┌─────────────────────────────────────────────────────────────┐
   │ Step 1: Check existing data in DB (batch query)            │
   │ → Get existing dates for all 30 symbols + VN30             │
   └─────────────────────────────────────────────────────────────┘
                              ↓
   ┌─────────────────────────────────────────────────────────────┐
   │ Step 2: Calculate features for component symbols           │
   │ → Skip symbols/dates already in DB                         │
   │ → Persist new features immediately                         │
   └─────────────────────────────────────────────────────────────┘
                              ↓
   ┌─────────────────────────────────────────────────────────────┐
   │ Step 3: Calculate VN30 Index candles                       │
   │ → TickVN30IndexCalculator                                  │
   └─────────────────────────────────────────────────────────────┘
                              ↓
   ┌─────────────────────────────────────────────────────────────┐
   │ Step 4: Aggregate features from component symbols          │
   │ → VN30WhaleFootprintAggregator (fetches from DB)           │
   └─────────────────────────────────────────────────────────────┘
                              ↓
   ┌─────────────────────────────────────────────────────────────┐
   │ Step 5: Merge & Persist VN30                               │
   │ → Combine index OHLCV + aggregated features                │
   │ → Upsert to stock_trading_feature_candles (symbol="VN30")  │
   └─────────────────────────────────────────────────────────────┘
   ```

4. **Output Schema for VN30**:
   ```python
   {
       "symbol": "VN30",
       "time": "2025-01-02T02:20:00+00:00",
       "interval": 300,
       "open": 1000,          # From index calculator (rounded to int)
       "high": 1003,          # From index calculator (rounded to int)
       "low": 998,            # From index calculator (rounded to int)
       "close": 1001,         # From index calculator (rounded to int)
       "volume": 12345678,    # Total volume of 30 stocks
       "value": 456789,       # Total value in millions VND
       "features": {
           "whale_footprint": {  # Same namespace as component symbols
               "vn30_shark450_buy_value": 48942.0,
               "vn30_shark450_sell_value": 16111.0,
               "vn30_percent_shark450_buy_sell": 75.23,
               "vn30_shark450_urgency_spread": 0.17,
               # ... other aggregated features
           }
       }
   }
   ```

## 🔄 Implementation Plan

### Phase 1: Analysis & Preparation
- [x] Analyze current `IntradaySymbolFeaturePersistor` để hiểu cách persist
- [x] Xác định query để batch fetch existing dates
- [x] Define output schema cho VN30 symbol

### Phase 2: Implementation (File/Code Structure)

```
packages/stock/metan/stock/trading/domain/feature/
├── persistor/
│   ├── intraday/
│   │   └── intraday_symbol_feature_persistor.py  # ✅ EXISTS
│   └── vn30/                                      # ✅ IMPLEMENTED
│       ├── __init__.py
│       └── vn30_feature_pipeline.py               # ✅ IMPLEMENTED - Main pipeline class
│
└── aggregator/
    └── vn30/
        └── vn30_whale_footprint_aggregator.py     # ✅ EXISTS

packages/stock/metan/stock/testbed/
├── calculate_vn30_aggregate.py                    # ✅ EXISTS (legacy)
└── build_vn30_features.py                         # ✅ IMPLEMENTED - Entry point script
```

### Phase 3: Detailed Implementation Steps

#### Step 1: Create Helper for Batch Date Check

```python
def fetch_existing_dates_by_symbol(
    symbols: list[str],
    start_date: str,
    end_date: str,
    interval: int,
) -> dict[str, set[str]]:
    """
    Fetch existing dates for multiple symbols in a single query.
    
    Returns:
        Dict[symbol, Set[date_string]]
    """
    response = (
        supabase.table("stock_trading_feature_candles")
        .select("symbol, time")
        .in_("symbol", symbols)
        .eq("interval", interval)
        .gte("time", f"{start_date}T00:00:00")
        .lte("time", f"{end_date}T23:59:59")
        .execute()
    )
    
    existing_dates: dict[str, set[str]] = {s: set() for s in symbols}
    for row in response.data:
        symbol = row["symbol"]
        date = row["time"][:10]  # Extract YYYY-MM-DD
        existing_dates[symbol].add(date)
    
    return existing_dates
```

#### Step 2: Create `VN30FeaturePipeline` Class

```python
class VN30FeaturePipeline:
    """
    Complete pipeline for building VN30 features:
    1. Calculate features for component symbols (with smart skip)
    2. Calculate VN30 index candles
    3. Aggregate features
    4. Persist VN30 data
    """
    
    def __init__(
        self,
        start_date: str,
        end_date: str,
        interval: IntradayInterval = IntradayInterval.FIVE_MINUTES,
        force_recalculate: bool = False,  # If True, ignore existing data
    ):
        ...
    
    def run(self) -> dict[str, Any]:
        """
        Run the complete pipeline.
        
        Returns:
            Summary dict with statistics
        """
        # 1. Check existing data (batch query)
        existing_dates = self._fetch_existing_dates()
        
        # 2. Calculate features for component symbols
        self._calculate_component_features(existing_dates)
        
        # 3. Calculate VN30 index candles
        index_candles = self._calculate_index_candles()
        
        # 4. Aggregate features
        aggregated_df = self._aggregate_features()
        
        # 5. Merge and persist VN30
        self._persist_vn30(index_candles, aggregated_df)
        
        return {"status": "success", ...}
```

#### Step 3: Implement Smart Skip for Component Symbols

```python
def _calculate_component_features(
    self,
    existing_dates: dict[str, set[str]],
) -> None:
    """Calculate features for component symbols, skipping existing dates."""
    
    # Get all requested dates
    requested_dates = self._get_trading_dates()
    
    for symbol in VN30_SYMBOLS:
        existing = existing_dates.get(symbol, set())
        missing_dates = requested_dates - existing
        
        if not missing_dates:
            self._logger.info(f"[VN30Pipeline] {symbol}: All dates exist, skipping")
            continue
        
        self._logger.info(
            f"[VN30Pipeline] {symbol}: Calculating {len(missing_dates)} missing dates "
            f"(existing: {len(existing)})"
        )
        
        # Calculate only for missing dates
        # Option A: Calculate all and let DB upsert handle it
        # Option B: Filter tick_candles to only missing dates before calculating
        
        persistor = IntradaySymbolFeaturePersistor(
            symbol=symbol,
            start_date=min(missing_dates),
            end_date=max(missing_dates),
            interval=self.interval,
        )
        persistor.persist()
```

#### Step 4: Merge Index Candles with Aggregated Features

```python
def _persist_vn30(
    self,
    index_candles: list[VN30IndexCandle],
    aggregated_df: pd.DataFrame,
) -> int:
    """
    Merge index candles with aggregated features and persist.
    
    Args:
        index_candles: List of VN30IndexCandle from TickVN30IndexCalculator
        aggregated_df: DataFrame with VN30 aggregated features (indexed by time)
        
    Raises:
        ValueError: If any index candle is missing aggregated features
    """
    rows = []
    
    for candle in index_candles:
        # Get aggregated features for this timepoint - raise error if missing
        if candle.time not in aggregated_df.index:
            raise ValueError(
                f"Missing aggregated features for timepoint {candle.time}. "
                f"Check if all component symbols have features calculated."
            )
        
        features = aggregated_df.loc[candle.time].to_dict()
        
        row = {
            "symbol": "VN30",
            "time": candle.time,
            "interval": int(self.interval),
            "open": round(candle.open),      # Round to int
            "high": round(candle.high),      # Round to int
            "low": round(candle.low),        # Round to int
            "close": round(candle.close),    # Round to int
            "volume": candle.volume,
            "value": candle.value,
            "features": {
                "whale_footprint": features  # Same namespace as component symbols
            },
        }
        rows.append(row)
    
    # Upsert to DB
    return self._persist_rows(rows)
```

#### Step 5: Create Entry Point Script

**File:** `packages/stock/metan/stock/testbed/build_vn30_features.py`

```python
"""
Build complete VN30 features pipeline.

Usage:
    python -m metan.stock.testbed.build_vn30_features
"""

from metan.stock.trading.domain.feature.persistor.vn30.vn30_feature_pipeline import (
    VN30FeaturePipeline,
)


def main(
    start_date: str = "2025-01-02",
    end_date: str = "2025-01-03",
    force_recalculate: bool = False,
):
    pipeline = VN30FeaturePipeline(
        start_date=start_date,
        end_date=end_date,
        force_recalculate=force_recalculate,
    )
    
    result = pipeline.run()
    print(f"Pipeline complete: {result}")


if __name__ == "__main__":
    main()
```

---

## 📊 Summary of Results

### ✅ Completed Achievements

- [x] Created `VN30FeaturePipeline` class at `packages/stock/.../persistor/vn30/vn30_feature_pipeline.py`
- [x] Implemented batch date check optimization (single query for all 31 symbols)
- [x] Integrated `TickVN30IndexCalculator` for index candle calculation
- [x] Smart skip: Only calculate features for symbols/dates not in DB
- [x] Merged index OHLCV with aggregated features
- [x] Persisted VN30 as `symbol="VN30"` to `stock_trading_feature_candles`
- [x] Created entry point script at `packages/stock/.../testbed/build_vn30_features.py`
- [x] All linter checks passed

### 🔧 Usage

```python
from metan.stock.trading.domain.feature.persistor.vn30 import VN30FeaturePipeline

pipeline = VN30FeaturePipeline(
    start_date="2025-01-02",
    end_date="2025-01-03",
    force_recalculate=False,  # Skip existing dates
)
result = pipeline.run()
```

**Run via command line:**

```bash
python -m metan.stock.testbed.build_vn30_features --start-date 2025-01-02 --end-date 2025-01-03

# Force recalculate all:
python -m metan.stock.testbed.build_vn30_features --start-date 2025-01-02 --end-date 2025-01-03 --force
```

## 🚧 Outstanding Issues & Follow-up

### ✅ Clarifications Resolved

| Question | Decision |
|----------|----------|
| VN30 Symbol Name | `"VN30"` |
| Feature Namespace | `"whale_footprint"` (same as component symbols) |
| Missing Data Handling | Raise error |
| Index Candle OHLC Type | Round to `int` |

### 💡 Future Improvements

1. **Parallel Processing**: Calculate features cho 30 symbols song song
2. **Incremental Update**: Chỉ calculate cho ngày mới nhất (daily job)
3. **Retry Logic**: Retry failed symbols

