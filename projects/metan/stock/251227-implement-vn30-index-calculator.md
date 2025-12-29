# 📋 251227: Implement VN30 Index Calculator

## References

- Overview: `devdocs/projects/metan/stock/OVERVIEW.md`
- Stock Model: `packages/stock/metan/stock/info/domain/stock/models.py`
- Candle Models: `packages/stock/metan/stock/info/domain/candle/models.py`
- TCBS Candle Fetcher: `packages/stock/metan/stock/info/domain/stock_data_collector/external/tcbs/tcbs_symbol_candle_fetcher.py`
- Stock Data Collector: `packages/stock/metan/stock/info/domain/stock_data_collector/stock_data_collector.py`
- Config Data (Timepoints): `packages/stock/metan/stock/common/helper/config_data.py`
- Time Utils: `packages/stock/metan/stock/common/utils/time_utils.py`

## User Requirements

1. Tính VN30 Index từ 30 cổ phiếu top với interval 5 phút
2. Phương pháp: **Market Cap Weighted** dựa trên `total_shares`
3. Base value: Candle đầu tiên = **1000**
4. Free Float Ratio: Giả sử = **1** (sử dụng trực tiếp `total_shares`)
5. Missing Data: **Throw error**
6. Output: `list[VN30IndexCandle]`
7. Timepoints: Lấy từ config `get_intraday_timepoints()` để đảm bảo đồng bộ và performance
8. Trading dates: Chỉ lấy các ngày trong range `[start_date, end_date]`

**VN30 Symbols:**
```
ACB, BCM, BID, CTG, DGC, FPT, GAS, GVR, HDB, HPG,
LPB, MBB, MSN, MWG, PLX, SAB, SHB, SSB, SSI, STB,
TCB, TPB, VCB, VHM, VIB, VIC, VJC, VNM, VPB, VRE
```

## 🎯 Objective

Implement a VN30 Index calculator that computes the VN30 index value at 5-minute intervals based on market cap weighted methodology using intraday candle data from TCBS.

### ⚠️ Key Considerations

1. **Đồng bộ thời gian**: Tất cả 30 cổ phiếu phải có candle tại cùng một timepoint. Nếu thiếu bất kỳ symbol nào → throw error.

2. **Timepoints từ Config**: Sử dụng `get_intraday_timepoints("HSX", interval)` để lấy danh sách timepoints chuẩn, đảm bảo consistency với các module khác.

3. **ISO8601 Time Format**: Normalize tất cả ISO8601 time strings về format thống nhất `2025-12-26T02:55:00+00:00` (không có milliseconds, dùng `+00:00` suffix).

4. **Market Cap Weighting Formula**:
   ```
   MarketCap_i = Close_i × TotalShares_i
   TotalMarketCap = Σ MarketCap_i (for all 30 symbols)
   
   IndexValue = (TotalMarketCap / BaseTotalMarketCap) × BaseIndex
   ```
   Trong đó:
   - `BaseTotalMarketCap` = Total Market Cap của candle đầu tiên
   - `BaseIndex` = 1000

5. **OHLC cho Index Candle**:
   - **Open**: Tính từ giá open của tất cả symbols tại timepoint
   - **High**: Tính từ giá high của tất cả symbols tại timepoint
   - **Low**: Tính từ giá low của tất cả symbols tại timepoint
   - **Close**: Tính từ giá close của tất cả symbols tại timepoint
   - **Volume**: Tổng volume của tất cả 30 cổ phiếu
   - **Value**: Tổng value (triệu VND) của tất cả 30 cổ phiếu

6. **Performance**: Sử dụng caching từ `StockDataCollector`.

## 🔄 Implementation Plan

### Phase 1: Analysis & Preparation
- [x] Analyze data flow từ TCBS → PriceCandle → VN30IndexCandle
  - **Outcome**: Hiểu rõ cách lấy candles cho 30 symbols và merge chúng theo timepoint
- [x] Define edge cases
  - **Outcome**:
    - Missing candle cho bất kỳ symbol nào → raise `ValueError`
    - Symbol không tồn tại trong database → raise `ValueError`
    - Không có `total_shares` cho symbol → raise `ValueError`
    - Ngày không có đủ timepoints → raise `ValueError`

### Phase 2: Implementation (File/Code Structure)

```
packages/stock/metan/stock/
├── common/
│   └── utils/
│       └── time_utils.py            # ✅ IMPLEMENTED - normalize_iso8601()
└── info/
    └── domain/
        ├── index/
        │   ├── __init__.py          # ✅ IMPLEMENTED - Export public APIs
        │   ├── constants.py         # ✅ IMPLEMENTED - VN30_SYMBOLS list
        │   ├── models.py            # ✅ IMPLEMENTED - VN30IndexCandle, IndexComponent
        │   └── vn30_index_calculator.py  # ✅ IMPLEMENTED - VN30IndexCalculator class
        └── stock_data_collector/
            └── external/tcbs/
                └── tcbs_symbol_candle_fetcher.py  # 🔄 UPDATED - Normalize time format
```

### Phase 3: Detailed Implementation Steps

#### Step 1: Create `time_utils.py` ✅
**File**: `packages/stock/metan/stock/common/utils/time_utils.py`

```python
import pendulum


def normalize_iso8601(iso_string: str) -> str:
    """
    Normalize various ISO8601 time formats to a consistent format.

    Handles different formats:
        - "2025-12-26T02:55:00.000Z" (with milliseconds, Z suffix)
        - "2025-12-25T02:15:00+00:00" (with timezone offset)
        - "2025-12-25T02:15:00Z" (with Z suffix)

    Returns:
        Normalized format: "2025-12-26T02:55:00+00:00"
    """
    dt = pendulum.parse(iso_string)
    return dt.set(microsecond=0).in_tz("UTC").to_iso8601_string()
```

---

#### Step 2: Update `tcbs_symbol_candle_fetcher.py` ✅
**File**: `packages/stock/metan/stock/info/domain/stock_data_collector/external/tcbs/tcbs_symbol_candle_fetcher.py`

**Changes:**
- Import `normalize_iso8601` from `metan.stock.common.utils.time_utils`
- Normalize `tradingDate` when creating `PriceCandle`:

```python
PriceCandle(
    time=normalize_iso8601(candle["tradingDate"]),  # Normalized
    open=int(candle["open"]),
    close=int(candle["close"]),
    high=int(candle["high"]),
    low=int(candle["low"]),
    volume=int(candle["volume"]),
)
```

---

#### Step 3: Create `constants.py` ✅
**File**: `packages/stock/metan/stock/info/domain/index/constants.py`

```python
VN30_SYMBOLS: list[str] = [
    "ACB", "BCM", "BID", "CTG", "DGC", "FPT", "GAS", "GVR", "HDB", "HPG",
    "LPB", "MBB", "MSN", "MWG", "PLX", "SAB", "SHB", "SSB", "SSI", "STB",
    "TCB", "TPB", "VCB", "VHM", "VIB", "VIC", "VJC", "VNM", "VPB", "VRE",
]

DEFAULT_BASE_INDEX: float = 1000.0
DEFAULT_FREE_FLOAT_RATIO: float = 1.0
```

---

#### Step 4: Create `models.py` ✅
**File**: `packages/stock/metan/stock/info/domain/index/models.py`

```python
from pydantic import BaseModel


class IndexComponent(BaseModel):
    """Represents a single component in the index calculation."""
    symbol: str
    total_shares: int
    close_price: int
    market_cap: int  # close_price × total_shares


class VN30IndexCandle(BaseModel):
    """
    Represents a single VN30 index candle at a specific timepoint.
    
    Attributes:
        time: ISO 8601 UTC timestamp
        open: Index value calculated from open prices
        high: Index value calculated from high prices
        low: Index value calculated from low prices  
        close: Index value calculated from close prices
        volume: Total volume of all 30 symbols
        value: Total traded value in millions VND
    """
    time: str  # ISO 8601 UTC
    open: float
    high: float
    low: float
    close: float
    volume: int
    value: int  # millions VND
```

---

#### Step 5: Create `vn30_index_calculator.py` ✅
**File**: `packages/stock/metan/stock/info/domain/index/vn30_index_calculator.py`

**Key Methods:**

| Method | Description |
|--------|-------------|
| `calculate()` | Main entry point - returns `list[VN30IndexCandle]` |
| `_fetch_all_stocks_info()` | Fetch stock info (total_shares) for all symbols |
| `_fetch_all_symbols_candles()` | Fetch price candles from TCBS |
| `_get_expected_timepoints_by_date()` | Get timepoints from config for each trading date |
| `_validate_all_symbols_have_complete_data()` | Strict validation - raise error if missing any candle |
| `_build_iso_timepoint()` | Convert date + HH:MM (UTC) to ISO8601 UTC string |
| `_calculate_index_at_timepoint()` | Calculate index OHLCV at a specific timepoint |
| `_find_candle_at_timepoint()` | Find candle by timepoint |

**Core Algorithm:**

```python
def calculate(self) -> list[VN30IndexCandle]:
    # 1. Fetch stock info for all symbols (get total_shares)
    stocks_info = self._fetch_all_stocks_info()
    
    # 2. Validate all symbols have total_shares
    for symbol, stock in stocks_info.items():
        if stock.total_shares is None or stock.total_shares <= 0:
            raise ValueError(f"Symbol {symbol} missing total_shares")
    
    # 3. Fetch price candles for all symbols
    candles_by_symbol = self._fetch_all_symbols_candles()
    
    # 4. Get all trading dates within [start_date, end_date] range
    first_symbol = self._symbols[0]
    all_dates = candles_by_symbol[first_symbol].keys()
    trading_dates = sorted(
        date for date in all_dates if self._start_date <= date <= self._end_date
    )
    
    # 5. Get expected timepoints from config
    expected_timepoints_by_date = self._get_expected_timepoints_by_date(trading_dates)
    
    # 6. STRICT VALIDATION: Check all symbols have complete data
    self._validate_all_symbols_have_complete_data(
        candles_by_symbol, expected_timepoints_by_date
    )
    
    # 7. Flatten to sorted list of all timepoints
    all_timepoints = [...]
    
    # 8. Calculate index for each timepoint
    for timepoint in all_timepoints:
        index_candle, total_mc = self._calculate_index_at_timepoint(...)
        result.append(index_candle)
    
    return result
```

**`_build_iso_timepoint` method:**

```python
def _build_iso_timepoint(self, date: str, hhmm: str) -> str:
    """
    Convert date + HH:MM (UTC) to ISO8601 UTC string.

    Args:
        date: "2025-01-01"
        hhmm: "02:20"

    Returns:
        ISO8601 UTC string (e.g., "2025-01-01T02:20:00+00:00")
    """
    hh, mm = hhmm.split(":")
    utc_dt = pendulum.parse(f"{date}T{hh}:{mm}:00", tz="UTC")
    return utc_dt.set(microsecond=0).to_iso8601_string()
```

---

#### Step 6: Create `__init__.py` ✅
**File**: `packages/stock/metan/stock/info/domain/index/__init__.py`

```python
from metan.stock.info.domain.index.constants import (
    DEFAULT_BASE_INDEX,
    DEFAULT_FREE_FLOAT_RATIO,
    VN30_SYMBOLS,
)
from metan.stock.info.domain.index.models import IndexComponent, VN30IndexCandle
from metan.stock.info.domain.index.vn30_index_calculator import VN30IndexCalculator

__all__ = [
    "VN30_SYMBOLS",
    "DEFAULT_BASE_INDEX",
    "DEFAULT_FREE_FLOAT_RATIO",
    "IndexComponent",
    "VN30IndexCandle",
    "VN30IndexCalculator",
]
```

---

#### Step 7: Usage Example

```python
from metan.stock.info.domain.index import VN30IndexCalculator

calculator = VN30IndexCalculator(
    start_date="2025-12-01",
    end_date="2025-12-05",
    base_index=1000,
)

index_candles = calculator.calculate()

for candle in index_candles[:5]:
    print(f"{candle.time}: O={candle.open:.2f} H={candle.high:.2f} L={candle.low:.2f} C={candle.close:.2f}")
```

## 📊 Summary of Results

### ✅ Completed Achievements

- [x] Created `normalize_iso8601()` utility function for consistent time format
- [x] Updated `tcbs_symbol_candle_fetcher.py` to normalize time in PriceCandle
- [x] Implemented `VN30IndexCalculator` with market cap weighted methodology
- [x] Timepoints synced from config via `get_intraday_timepoints()`
- [x] Trading dates filtered to `[start_date, end_date]` range
- [x] Strict validation: raise `ValueError` if any symbol missing any timepoint
- [x] Full OHLCV calculation for index candles
- [x] Logging for debugging and monitoring
- [x] All linter checks passed

### 🔧 Bug Fixes Applied

1. **ISO8601 Format Mismatch**: 
   - TCBS trả về: `2025-12-26T02:55:00.000Z`
   - Expected: `2025-12-26T02:55:00+00:00`
   - **Fix**: Created `normalize_iso8601()` utility

2. **Trading Dates Out of Range**:
   - `price_candles_by_date()` trả về thêm các ngày trước `start_date`
   - **Fix**: Filter trading dates to only include `[start_date, end_date]`

## 🚧 Outstanding Issues & Follow-up

### ⚠️ Potential Future Improvements

1. **Performance Optimization**: 
   - Consider parallel fetching với `asyncio` hoặc `concurrent.futures` cho 30 symbols

2. **Future Extensions**:
   - Support cho các index khác (VN100, VNINDEX)
   - Persist index candles to Supabase
   - Historical divisor adjustment khi có corporate actions
