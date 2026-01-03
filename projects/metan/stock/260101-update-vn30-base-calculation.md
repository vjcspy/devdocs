# 📋 260101: Update VN30 Index Base Calculation

> **Status:** ✅ IMPLEMENTED  
> **Date:** 2026-01-01

## References

- Tick VN30 Index Calculator: `packages/stock/metan/stock/info/domain/index/tick_vn30_index_calculator.py`
- TCBS VN30 Index Calculator: `packages/stock/metan/stock/info/domain/index/tcbs_vn30_index_calculator.py`
- Constants: `packages/stock/metan/stock/info/domain/index/constants.py`
- Config Data Helper: `packages/stock/metan/stock/common/helper/config_data.py`

## User Requirements

Sửa toàn bộ các hàm liên quan đến tính VN30 index:
- **Base date**: 2025-12-31
- **Base time**: 03:10 UTC (10:10 giờ Việt Nam - thời điểm mở cửa)
- **Base index**: 2012 điểm
- **Storage**: Lưu base market cap vào Supabase table `stock_common_configuration`
- **Missing candle**: Raise error

## 🎯 Objective

Update VN30 index calculators to use a **fixed base point**:
- Date: **2025-12-31**
- Time: **03:10 UTC** (10:10 Vietnam)
- Index value: **2012**

Base market cap được tính 1 lần và lưu vào `stock_common_configuration`, các lần tính sau sẽ load từ DB.

### ⚠️ Key Considerations

1. **Current Behavior**:
   ```python
   # First candle becomes base, index = 1000
   if base_market_cap is None:
       base_market_cap = total_mc  # First candle's market cap
   ```

2. **New Behavior**:
   ```python
   # Load base market cap from DB (calculated at 2025-12-31 03:10 UTC)
   base_market_cap = self._get_or_calculate_base_market_cap()
   # All candles calculated relative to this fixed base, index = 2012
   ```

3. **Storage in `stock_common_configuration`**:
   ```python
   {
       "key": "VN30_INDEX_BASE_MARKET_CAP",
       "value": {
           "base_date": "2025-12-31",
           "base_time": "03:10",
           "base_timepoint": "2025-12-31T03:10:00+00:00",
           "base_market_cap": 1234567890123,  # Total market cap at base time
           "base_index": 2012,
           "use_free_float": true,
           "calculated_at": "2026-01-01T10:00:00+00:00"
       }
   }
   ```

4. **Files to Update/Create**:
   - `constants.py` - Add base constants
   - `tick_vn30_index_calculator.py` - Update to use stored base
   - `tcbs_vn30_index_calculator.py` - Update to use stored base
   - `vn30_base_calculator.py` (NEW) - Calculate and store base market cap

## 🔄 Implementation Plan

### Phase 1: Update Constants

**File:** `packages/stock/metan/stock/info/domain/index/constants.py`

```python
# Base configuration for VN30 Index
VN30_BASE_DATE: str = "2025-12-31"
VN30_BASE_TIME: str = "03:10"  # UTC (10:10 Vietnam time - market opening)
VN30_BASE_INDEX: float = 2012.0

# Configuration key in stock_common_configuration table
VN30_BASE_MARKET_CAP_CONFIG_KEY: str = "VN30_INDEX_BASE_MARKET_CAP"
```

### Phase 2: Create VN30BaseMarketCapCalculator

**File:** `packages/stock/metan/stock/info/domain/index/vn30_base_calculator.py`

```python
class VN30BaseMarketCapCalculator:
    """
    Calculate and store VN30 base market cap for index calculation.
    
    The base market cap is calculated at 2025-12-31 03:10 UTC and stored
    in stock_common_configuration table. All index calculations use this
    stored value as the base.
    """
    
    def get_or_calculate_base(self, force_recalculate: bool = False) -> dict:
        """
        Get base market cap from DB, or calculate and store if not exists.
        
        Returns:
            Dict with base_market_cap, base_timepoint, base_index, etc.
        """
        # Try to load from DB first
        if not force_recalculate:
            stored = self._load_from_db()
            if stored:
                return stored
        
        # Calculate base market cap
        base_data = self._calculate_base_market_cap()
        
        # Store to DB
        self._save_to_db(base_data)
        
        return base_data
    
    def _calculate_base_market_cap(self) -> dict:
        """Calculate total market cap at 2025-12-31 03:10 UTC."""
        # Fetch tick candles for base date
        # Calculate market cap for all 30 symbols
        # Return base data dict
        ...
    
    def _load_from_db(self) -> dict | None:
        """Load base market cap from stock_common_configuration."""
        response = (
            supabase.table("stock_common_configuration")
            .select("value")
            .eq("key", VN30_BASE_MARKET_CAP_CONFIG_KEY)
            .execute()
        )
        if response.data:
            return response.data[0]["value"]
        return None
    
    def _save_to_db(self, base_data: dict) -> None:
        """Save base market cap to stock_common_configuration."""
        supabase.table("stock_common_configuration").upsert({
            "key": VN30_BASE_MARKET_CAP_CONFIG_KEY,
            "value": base_data,
        }, on_conflict="key").execute()
```

### Phase 3: Update TickVN30IndexCalculator

**Changes to `calculate()` method:**

```python
def calculate(self) -> list[VN30IndexCandle]:
    # ... existing steps 1-6 ...
    
    # 7. Get base market cap (from DB or calculate)
    base_calculator = VN30BaseMarketCapCalculator()
    base_data = base_calculator.get_or_calculate_base()
    base_market_cap = base_data["base_market_cap"]
    
    self._logger.info(
        f"[TickVN30IndexCalculator] Using base: {base_data['base_timepoint']}, "
        f"market_cap={base_market_cap:,}, index={self._base_index}"
    )
    
    # 8. Calculate index for each timepoint using this fixed base
    result: list[VN30IndexCandle] = []
    for timepoint in all_timepoints:
        index_candle, _ = self._calculate_index_at_timepoint(
            timepoint, tick_candles_by_symbol, stocks_info, base_market_cap
        )
        result.append(index_candle)
    
    return result
```

### Phase 4: Update TcbsVN30IndexCalculator

Apply the same changes to use `VN30BaseMarketCapCalculator`.

### Phase 5: Update Default Base Index

**File:** `tick_vn30_index_calculator.py`

```python
# Change default from 1000 to 2012
TICK_DEFAULT_BASE_INDEX: float = 2012.0  # Was 1000.0
```

**File:** `constants.py`

```python
# Update default base index
DEFAULT_BASE_INDEX: float = 2012.0  # Was 2020.01
```

### Phase 6: Create Testbed Script for Base Calculation

**File:** `packages/stock/metan/stock/testbed/calculate_vn30_base.py`

```python
"""
Calculate and store VN30 base market cap.

This should be run ONCE to establish the base at 2025-12-31 03:10 UTC.

Usage:
    python -m metan.stock.testbed.calculate_vn30_base
    python -m metan.stock.testbed.calculate_vn30_base --force  # Recalculate
"""

from metan.stock.info.domain.index.vn30_base_calculator import VN30BaseMarketCapCalculator

def main(force: bool = False):
    calculator = VN30BaseMarketCapCalculator()
    base_data = calculator.get_or_calculate_base(force_recalculate=force)
    
    print(f"Base Date: {base_data['base_date']}")
    print(f"Base Time: {base_data['base_time']}")
    print(f"Base Timepoint: {base_data['base_timepoint']}")
    print(f"Base Market Cap: {base_data['base_market_cap']:,}")
    print(f"Base Index: {base_data['base_index']}")

if __name__ == "__main__":
    main()
```

---

## File Structure

```
packages/stock/metan/stock/info/domain/index/
├── __init__.py                        # ✅ UPDATED - Added exports
├── constants.py                       # ✅ UPDATED - Added base constants
├── models.py                          # ✅ EXISTS
├── tcbs_vn30_index_calculator.py      # ✅ UPDATED - Uses stored base
├── tick_vn30_index_calculator.py      # ✅ UPDATED - Uses stored base
└── vn30_base_calculator.py            # ✅ IMPLEMENTED - Base calculation & storage

packages/stock/metan/stock/testbed/
└── calculate_vn30_base.py             # ✅ IMPLEMENTED - Entry point for base calculation
```

---

## 📊 Summary of Results

### ✅ Completed Achievements

- [x] Added base constants (`VN30_BASE_DATE`, `VN30_BASE_TIME`, `VN30_BASE_INDEX`, `VN30_BASE_MARKET_CAP_CONFIG_KEY`)
- [x] Removed legacy constants (`DEFAULT_BASE_INDEX`, `TICK_DEFAULT_BASE_INDEX`)
- [x] Created `VN30BaseMarketCapCalculator` class at `packages/stock/.../index/vn30_base_calculator.py`
- [x] Updated `TickVN30IndexCalculator` to use stored base from DB
- [x] Updated `TcbsVN30IndexCalculator` to use stored base from DB
- [x] Created testbed script at `packages/stock/.../testbed/calculate_vn30_base.py`
- [x] Updated `__init__.py` exports
- [x] All linter checks passed

### 🔧 Usage

**Step 1: Calculate and store base (run ONCE)**

```bash
python -m metan.stock.testbed.calculate_vn30_base

# Or force recalculate:
python -m metan.stock.testbed.calculate_vn30_base --force
```

**Step 2: Use index calculators (base is loaded from DB automatically)**

```python
from metan.stock.info.domain.index import TickVN30IndexCalculator

calculator = TickVN30IndexCalculator(
    start_date="2025-01-02",
    end_date="2025-01-03",
)
candles = calculator.calculate()  # Uses base from DB
```

## 🚧 Outstanding Issues & Follow-up

### ✅ Clarifications Resolved

| Question | Decision |
|----------|----------|
| Base per Day or Single Base? | **Single Base** - Use 2025-12-31 03:10 UTC for ALL dates |
| Missing 03:10 UTC candle? | **Raise error** |
| Storage | Store base market cap in `stock_common_configuration` |

### 💡 Future Improvements

1. Add CLI command to recalculate base if VN30 composition changes
2. Add validation to ensure base date data exists before calculation

