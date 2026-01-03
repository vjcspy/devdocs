# 📋 260101: Implement Tick-based VN30 Index Calculator

> **Status:** ✅ IMPLEMENTED  
> **Date:** 2026-01-01

## References

- Overview: `devdocs/projects/metan/stock/OVERVIEW.md`
- VN30 Features Plan: `devdocs/projects/metan/stock/251229-vn30-aggregate-features-plan.md`
- TCBS VN30 Index Calculator: `devdocs/projects/metan/stock/251227-implement-vn30-index-calculator.md`
- Stock Data Collector: `packages/stock/metan/stock/info/domain/stock_data_collector/stock_data_collector.py`
- Existing Index Module: `packages/stock/metan/stock/info/domain/index/`
- Candle Models: `packages/stock/metan/stock/info/domain/candle/models.py`

## User Requirements

1. Build a VN30 Index calculator based on **tick candle data** (from Supabase) instead of TCBS REST API
2. Ensure data consistency between VN30 Index and VN30 whale footprint features (both use same tick candle source)
3. Remove dependency on TCBS for index calculation

## 🎯 Objective

Implement `TickVN30IndexCalculator` that calculates VN30 Index from tick candle data, ensuring consistency with the `VN30WhaleFootprintAggregator` which also uses tick candles as its data source.

### ⚠️ Key Considerations

1. **Data Consistency**: Both VN30 Index and whale footprint features must use the same underlying data source (`tick_candles_by_date()`) to ensure consistency for AI model training.

2. **Existing Infrastructure**: Reuse `StockDataCollector.tick_candles_by_date()` which returns `TickCandle` objects with OHLCV data.

3. **Market Cap Weighting Formula**:
   ```
   MarketCap_i = Price_i × TotalShares_i × FreeFloatRatio_i
   TotalMarketCap = Σ MarketCap_i (for all 30 symbols)
   
   IndexValue = (TotalMarketCap / BaseTotalMarketCap) × BaseIndex
   ```

4. **Difference from TCBS Calculator**:

   | Aspect | TcbsVN30IndexCalculator | TickVN30IndexCalculator |
   |--------|------------------------|------------------------|
   | Data Source | `price_candles_by_date()` → TCBS REST | `tick_candles_by_date()` → Supabase |
   | Candle Type | `PriceCandle` | `TickCandle` |
   | Consistency | May differ from features | Same source as whale footprint |

5. **Base Index**: Use `1000` as the base index value (first candle = 1000).

6. **Edge Cases** (Strict Mode - Raise Errors):
   - Missing tick candle for a symbol at timepoint → Raise `ValueError`
   - Symbol missing `total_shares` → Raise `ValueError`
   - Tick candle with no trades (volume=0) → Use OHLC from candle (may be 0)

## 🔄 Implementation Plan

### Phase 1: Analysis & Preparation
- [x] Analyze `TickCandle` structure and compare with `PriceCandle`
  - **Outcome**: TickCandle has same OHLCV fields as PriceCandle, plus `tick_actions` and `value`
- [x] Review existing `TcbsVN30IndexCalculator` implementation
  - **Outcome**: Reused pattern for market cap weighted calculation
- [x] Define edge cases
  - **Outcome**:
    - Missing candle for any symbol → raise `ValueError`
    - Symbol missing `total_shares` → raise `ValueError`
    - Days with incomplete data → raise `ValueError` (strict mode)

### Phase 2: Implementation (File/Code Structure)

```
packages/stock/metan/stock/info/domain/index/
├── __init__.py                        # ✅ UPDATED - Added TickVN30IndexCalculator export
├── constants.py                       # ✅ EXISTS - VN30_SYMBOLS, VN30_FREE_FLOAT_RATIOS
├── models.py                          # ✅ EXISTS - VN30IndexCandle, IndexComponent
├── tcbs_vn30_index_calculator.py      # ✅ EXISTS - TCBS-based calculator
└── tick_vn30_index_calculator.py      # ✅ IMPLEMENTED - Tick-based calculator

packages/stock/metan/stock/testbed/
├── calculate_vn30_aggregate.py        # ✅ EXISTS - VN30 aggregation pipeline
└── calculate_vn30_index_from_ticks.py # ✅ IMPLEMENTED - Test script for tick-based index
```

### Phase 3: Detailed Implementation Steps

#### Step 1: Create `tick_vn30_index_calculator.py`

**File**: `packages/stock/metan/stock/info/domain/index/tick_vn30_index_calculator.py`

**Class signature:**

```python
from metan.stock.info.domain.candle.models import IntradayInterval, TickCandle
from metan.stock.info.domain.index.constants import (
    DEFAULT_BASE_INDEX,
    VN30_FREE_FLOAT_RATIOS,
    VN30_SYMBOLS,
)
from metan.stock.info.domain.index.models import VN30IndexCandle
from metan.stock.info.domain.stock_data_collector.stock_data_collector import StockDataCollector


class TickVN30IndexCalculator:
    """
    Calculate VN30 Index from tick candle data (Supabase).
    
    Uses the same data source as VN30WhaleFootprintAggregator to ensure
    consistency between index and feature calculations.
    
    Formula:
        IndexValue = (TotalMarketCap / BaseTotalMarketCap) × BaseIndex
        
    Where:
        TotalMarketCap = Σ(Price_i × TotalShares_i × FreeFloatRatio_i)
    """
    
    def __init__(
        self,
        start_date: str,
        end_date: str,
        interval: IntradayInterval = IntradayInterval.FIVE_MINUTES,
        base_index: float = DEFAULT_BASE_INDEX,
        use_free_float: bool = True,
    ):
        self._start_date = start_date
        self._end_date = end_date
        self._interval = interval
        self._base_index = base_index
        self._use_free_float = use_free_float
        self._symbols = VN30_SYMBOLS
    
    def calculate(self) -> list[VN30IndexCandle]:
        """
        Calculate VN30 index candles from tick data.
        
        Returns:
            List of VN30IndexCandle sorted by time ascending.
            
        Raises:
            ValueError: If any symbol is missing data at any timepoint.
        """
        ...
```

**Core Algorithm:**

```python
def calculate(self) -> list[VN30IndexCandle]:
    # 1. Fetch stock info for all 30 symbols (total_shares)
    stocks_info = self._fetch_all_stocks_info()
    
    # 2. Validate all symbols have total_shares
    for symbol, stock in stocks_info.items():
        if stock.total_shares is None or stock.total_shares <= 0:
            raise ValueError(f"Symbol {symbol} missing total_shares")
    
    # 3. Fetch tick candles for all 30 symbols
    tick_candles_by_symbol: dict[str, dict[str, list[TickCandle]]] = {}
    for symbol in self._symbols:
        collector = StockDataCollector(
            symbol=symbol,
            start_date=self._start_date,
            end_date=self._end_date,
            interval=self._interval,
        )
        tick_candles_by_symbol[symbol] = collector.tick_candles_by_date()
    
    # 4. Get trading dates (intersection of all symbols)
    trading_dates = self._get_common_trading_dates(tick_candles_by_symbol)
    
    # 5. Get expected timepoints from config (for HSX)
    expected_timepoints = get_intraday_timepoints("HSX", self._interval)
    
    # 6. Calculate index for each date × timepoint
    result: list[VN30IndexCandle] = []
    base_total_market_cap: float | None = None
    
    for date in sorted(trading_dates):
        for hhmm in expected_timepoints:
            timepoint_iso = self._build_iso_timepoint(date, hhmm)
            
            # Calculate market cap at this timepoint
            index_candle, total_mc = self._calculate_index_at_timepoint(
                timepoint_iso=timepoint_iso,
                date=date,
                tick_candles_by_symbol=tick_candles_by_symbol,
                stocks_info=stocks_info,
                base_total_market_cap=base_total_market_cap,
            )
            
            # Set base on first candle
            if base_total_market_cap is None:
                base_total_market_cap = total_mc
                # Recalculate first candle with proper base
                index_candle = VN30IndexCandle(
                    time=timepoint_iso,
                    open=self._base_index,
                    high=self._base_index,
                    low=self._base_index,
                    close=self._base_index,
                    volume=index_candle.volume,
                    value=index_candle.value,
                )
            
            result.append(index_candle)
    
    return result
```

**Helper Methods:**

```python
def _fetch_all_stocks_info(self) -> dict[str, Stock]:
    """Fetch stock metadata for all VN30 symbols."""
    result = {}
    for symbol in self._symbols:
        collector = StockDataCollector(
            symbol=symbol,
            start_date=self._start_date,
            end_date=self._end_date,
            interval=self._interval,
        )
        result[symbol] = collector.stock()
    return result

def _get_common_trading_dates(
    self, 
    tick_candles_by_symbol: dict[str, dict[str, list[TickCandle]]]
) -> set[str]:
    """Get dates where all symbols have data."""
    if not tick_candles_by_symbol:
        return set()
    
    # Start with first symbol's dates
    first_symbol = self._symbols[0]
    common_dates = set(tick_candles_by_symbol[first_symbol].keys())
    
    # Intersect with other symbols
    for symbol in self._symbols[1:]:
        symbol_dates = set(tick_candles_by_symbol[symbol].keys())
        common_dates &= symbol_dates
    
    # Filter to requested date range
    return {d for d in common_dates if self._start_date <= d <= self._end_date}

def _build_iso_timepoint(self, date: str, hhmm: str) -> str:
    """Convert date + HH:MM (UTC) to ISO8601 UTC string."""
    hh, mm = hhmm.split(":")
    utc_dt = pendulum.parse(f"{date}T{hh}:{mm}:00", tz="UTC")
    return utc_dt.set(microsecond=0).to_iso8601_string()

def _find_tick_candle_at_timepoint(
    self,
    candles_by_date: dict[str, list[TickCandle]],
    date: str,
    timepoint_iso: str,
) -> TickCandle:
    """Find tick candle at specific timepoint. Raises ValueError if not found."""
    candles = candles_by_date.get(date, [])
    for candle in candles:
        if candle.time == timepoint_iso:
            return candle
    raise ValueError(f"No candle found at {timepoint_iso}")

def _calculate_index_at_timepoint(
    self,
    timepoint_iso: str,
    date: str,
    tick_candles_by_symbol: dict[str, dict[str, list[TickCandle]]],
    stocks_info: dict[str, Stock],
    base_total_market_cap: float | None,
) -> tuple[VN30IndexCandle, float]:
    """Calculate index OHLCV at a specific timepoint."""
    open_mc, high_mc, low_mc, close_mc = 0.0, 0.0, 0.0, 0.0
    total_volume, total_value = 0, 0
    
    for symbol in self._symbols:
        candle = self._find_tick_candle_at_timepoint(
            tick_candles_by_symbol[symbol], date, timepoint_iso
        )
        stock = stocks_info[symbol]
        
        free_float = VN30_FREE_FLOAT_RATIOS.get(symbol, 1.0) if self._use_free_float else 1.0
        effective_shares = stock.total_shares * free_float
        
        open_mc += candle.open * effective_shares
        high_mc += candle.high * effective_shares
        low_mc += candle.low * effective_shares
        close_mc += candle.close * effective_shares
        total_volume += candle.volume
        total_value += candle.value
    
    # Calculate index values
    if base_total_market_cap is None:
        # First candle - use placeholder, will be recalculated
        index_open = index_high = index_low = index_close = self._base_index
    else:
        index_open = (open_mc / base_total_market_cap) * self._base_index
        index_high = (high_mc / base_total_market_cap) * self._base_index
        index_low = (low_mc / base_total_market_cap) * self._base_index
        index_close = (close_mc / base_total_market_cap) * self._base_index
    
    return VN30IndexCandle(
        time=timepoint_iso,
        open=index_open,
        high=index_high,
        low=index_low,
        close=index_close,
        volume=total_volume,
        value=total_value,
    ), close_mc
```

---

#### Step 2: Update `__init__.py`

**File**: `packages/stock/metan/stock/info/domain/index/__init__.py`

Add the new calculator to exports:

```python
from metan.stock.info.domain.index.constants import (
    DEFAULT_BASE_INDEX,
    DEFAULT_FREE_FLOAT_RATIO,
    VN30_FREE_FLOAT_RATIOS,
    VN30_SYMBOLS,
)
from metan.stock.info.domain.index.models import IndexComponent, VN30IndexCandle
from metan.stock.info.domain.index.tcbs_vn30_index_calculator import TcbsVN30IndexCalculator
from metan.stock.info.domain.index.tick_vn30_index_calculator import TickVN30IndexCalculator

__all__ = [
    "VN30_SYMBOLS",
    "VN30_FREE_FLOAT_RATIOS",
    "DEFAULT_BASE_INDEX",
    "DEFAULT_FREE_FLOAT_RATIO",
    "IndexComponent",
    "VN30IndexCandle",
    "TcbsVN30IndexCalculator",
    "TickVN30IndexCalculator",
]
```

---

#### Step 3: Create Testbed Script

**File**: `packages/stock/metan/stock/testbed/calculate_vn30_index_from_ticks.py`

```python
"""
Testbed script for TickVN30IndexCalculator.

Usage:
    python -m metan.stock.testbed.calculate_vn30_index_from_ticks
"""

from metan.core.logger import Logger
from metan.stock.info.domain.index import TickVN30IndexCalculator

logger = Logger(__name__)


def main():
    calculator = TickVN30IndexCalculator(
        start_date="2025-01-02",
        end_date="2025-01-03",
        use_free_float=True,
    )
    
    logger.info("Calculating VN30 Index from tick candles...")
    candles = calculator.calculate()
    
    logger.info(f"Generated {len(candles)} index candles")
    
    for candle in candles[:10]:
        logger.info(
            f"{candle.time}: O={candle.open:.2f} H={candle.high:.2f} "
            f"L={candle.low:.2f} C={candle.close:.2f} V={candle.volume}"
        )


if __name__ == "__main__":
    main()
```

---

## 📊 Summary of Results

### ✅ Completed Achievements

- [x] Created `TickVN30IndexCalculator` class at `packages/stock/metan/stock/info/domain/index/tick_vn30_index_calculator.py`
- [x] Reused existing infrastructure (`StockDataCollector`, `VN30IndexCandle`, `VN30_FREE_FLOAT_RATIOS`)
- [x] Implemented market cap weighted methodology with base index = 1000
- [x] Added strict validation - raises `ValueError` on missing data
- [x] Created testbed script at `packages/stock/metan/stock/testbed/calculate_vn30_index_from_ticks.py`
- [x] Updated module exports in `__init__.py`
- [x] All linter checks passed
- [x] Tested successfully ✓

### 🔧 Usage

```python
from metan.stock.info.domain.index import TickVN30IndexCalculator

calculator = TickVN30IndexCalculator(
    start_date="2025-01-02",
    end_date="2025-01-03",
    use_free_float=True,  # Apply free-float ratios
)

index_candles = calculator.calculate()  # Returns list[VN30IndexCandle]

for candle in index_candles[:5]:
    print(f"{candle.time}: O={candle.open:.2f} H={candle.high:.2f} L={candle.low:.2f} C={candle.close:.2f}")
```

**Run testbed script:**

```bash
python -m metan.stock.testbed.calculate_vn30_index_from_ticks
```

## 🚧 Outstanding Issues & Follow-up

### ✅ All Issues Resolved

| Issue | Resolution |
|-------|------------|
| Base Index Value | Use `1000` |
| Validation Strategy | Strict mode - raise `ValueError` on missing data |
| Implementation | ✅ Completed and tested successfully |

### 💡 Future Improvements (Optional)

1. **Performance Optimization**: Consider parallel fetching for 30 symbols using `asyncio` or `concurrent.futures`.
2. **Caching**: Implement caching for stock info to reduce DB queries.
