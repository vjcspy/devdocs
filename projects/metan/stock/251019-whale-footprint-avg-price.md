# Updated Implementation Plan: Average Buy/Sell Price Feature

## 📋 **Overview**

Add per-candle cumulative average buy and sell prices for shark and sheep categories (at thresholds 450 and 900), with daily reset and fallback mechanisms.

------

## 🎯 **Feature Specification**

**New columns to add per candle:** For each threshold (450, 900) and each side (buy, sell):

- `shark{threshold}_avg_buy_price` - Cumulative average buy price for shark trades
- `shark{threshold}_avg_sell_price` - Cumulative average sell price for shark trades
- `sheep{threshold}_avg_buy_price` - Cumulative average buy price for sheep trades
- `sheep{threshold}_avg_sell_price` - Cumulative average sell price for sheep trades

**Example output columns (total: 8):**

- `shark450_avg_buy_price`, `shark450_avg_sell_price`
- `sheep450_avg_buy_price`, `sheep450_avg_sell_price`
- `shark900_avg_buy_price`, `shark900_avg_sell_price`
- `sheep900_avg_buy_price`, `sheep900_avg_sell_price`

------

## 📐 **Calculation Logic**

### **Formula:**

```
avg_price = total_weighted_value / total_volume
```

Where:

- **total_weighted_value** = sum of (price × volume) for all trades of that side/category from **start of the trading day**
- **total_volume** = sum of volume for all trades of that side/category from **start of the trading day**

### **Fallback Rules (when volume = 0):**

1. **First candle of the day:** Use `candle.open` price
2. **Middle of trading session:** Use previous candle's average price for same category/side
3. **Store as:** `int` (matching OHLC precision)

### **Classification Rules:**

1. Buy trades: `side == "B"`
2. Sell trades: `side == "S"`
3. Shark/Sheep: `trade_value_raw >= threshold * 1_000_000`
4. Reset cumulative counters at the **start of each trading day**

------

## 🏗️ **Implementation Steps**

### **Step 1: Modify `_build_rows` method signature**

**Location:** `whale_footprint_feature_calculator.py`

**Changes:**

- Add daily reset logic for cumulative trackers
- Track previous candle's average prices for fallback
- Iterate by date, then by candles within each date

------

### **Step 2: Update `_aggregate_single_candle` method**

**Location:** `whale_footprint_feature_calculator.py`

**Current signature:**

```
def _aggregate_single_candle(self, candle, meta: list[dict]) -> dict:
```

**New signature:**

```
def _aggregate_single_candle(self, candle, meta: list[dict]) -> tuple[dict, dict]:
```

**Return structure:**

```
(
    agg: dict[str, float],  # existing shark/sheep buy/sell values
    accumulations: dict[str, float]  # new volume and weighted value accumulators
)
```

**Implementation:**

```
def _aggregate_single_candle(self, candle, meta: list[dict]) -> tuple[dict, dict]:
    agg: dict[str, float] = {}
    accumulations: dict[str, float] = {}
    actions: list[TickAction] = candle.tick_actions or []
    
    for m in meta:
        label = m["label"]
        threshold_val = m["threshold"]
        
        # Initialize accumulators for buy and sell
        for side in ("buy", "sell"):
            for cat in ("shark", "sheep"):
                agg[f"{cat}{label}_{side}_value"] = 0.0
                accumulations[f"{cat}{label}_{side}_volume"] = 0.0
                accumulations[f"{cat}{label}_{side}_weighted_value"] = 0.0
        
        # Process each action
        for a in actions:
            if a.side not in ("B", "S"):
                continue
            
            trade_value_raw = float(a.price) * float(a.volume)
            side_key = "buy" if a.side == "B" else "sell"
            cat = "shark" if trade_value_raw >= threshold_val * 1_000_000 else "sheep"
            
            # Existing value aggregation (in millions)
            agg[f"{cat}{label}_{side_key}_value"] += int(trade_value_raw / 1_000_000)
            
            # New: Accumulate for average price calculation (in raw units)
            accumulations[f"{cat}{label}_{side_key}_volume"] += float(a.volume)
            accumulations[f"{cat}{label}_{side_key}_weighted_value"] += trade_value_raw
    
    return agg, accumulations
```

------

### **Step 3: Refactor `_build_rows` for daily reset**

**Location:** `whale_footprint_feature_calculator.py`

**Implementation:**

```
def _build_rows(self, daily_df: pd.DataFrame, tick_by_date, meta: list[dict]) -> list[dict]:
    rows: list[dict] = []
    
    for d in self._iter_candle_dates(daily_df):
        tick_candles = tick_by_date.get(d) or []
        if not tick_candles:
            raise ValueError(
                f"Date {d} has no tick_candles but exists in grouped ticks (symbol={self.data_collector.symbol})"
            )
        
        # Daily reset: Initialize cumulative trackers for this trading day
        cumulative_trackers: dict[str, dict[str, float]] = {}
        previous_avg_prices: dict[str, int] = {}
        
        for m in meta:
            label = m["label"]
            for side in ("buy", "sell"):
                for cat in ("shark", "sheep"):
                    key = f"{cat}{label}_{side}"
                    cumulative_trackers[key] = {
                        "volume": 0.0,
                        "weighted_value": 0.0
                    }
                    # Initialize with day's open price
                    previous_avg_prices[key] = int(tick_candles[0].open)
        
        # Process candles for this day
        today_pc = float(daily_df.at[d, "today_pc"])
        pc_5d = float(daily_df.at[d, "pc_5d"]) if pd.notna(daily_df.at[d, "pc_5d"]) else None
        if pc_5d is not None and pc_5d <= 0:
            raise ValueError(
                f"Computed non-positive 5D per-candle baseline for {d} (symbol={self.data_collector.symbol})"
            )
        
        for candle_idx, candle in enumerate(tick_candles):
            is_first_candle = (candle_idx == 0)
            
            agg, accumulations = self._aggregate_single_candle(candle, meta)
            
            # Build row with existing values
            row: dict[str, float | int | str] = {
                "time": int(candle.time),
                "date": d,
                "candle_volume": int(candle.volume),
                "candle_value": float(candle.value),
            }
            
            # Add existing shark/sheep value features
            for col in agg:
                row[col] = float(agg[col])
            
            # Update cumulative trackers and calculate average prices
            for m in meta:
                label = m["label"]
                for side in ("buy", "sell"):
                    for cat in ("shark", "sheep"):
                        key = f"{cat}{label}_{side}"
                        
                        # Update cumulative counters
                        cumulative_trackers[key]["volume"] += accumulations[f"{key}_volume"]
                        cumulative_trackers[key]["weighted_value"] += accumulations[f"{key}_weighted_value"]
                        
                        vol = cumulative_trackers[key]["volume"]
                        weighted_val = cumulative_trackers[key]["weighted_value"]
                        
                        # Calculate average price with fallback
                        if vol > 0:
                            avg_price = int(weighted_val / vol)
                            previous_avg_prices[key] = avg_price
                        elif is_first_candle:
                            # First candle: use open price
                            avg_price = int(candle.open)
                            previous_avg_prices[key] = avg_price
                        else:
                            # Middle of session: use previous candle's value
                            avg_price = previous_avg_prices[key]
                        
                        row[f"{key}_avg_price"] = avg_price
            
            # Add existing 5D ratio features
            has_5d = (
                bool(daily_df.at[d, "has_5d_baseline"])
                if "has_5d_baseline" in daily_df.columns
                else pc_5d is not None
            )
            if has_5d and pc_5d is not None:
                for col in agg:
                    base = col[:-6]
                    row[f"{base}_ratio_5d_pc"] = round(float(row[col]) / pc_5d, 4) if row[col] else 0.0
            else:
                raise ValueError(f"Missing 5D per-candle baseline for {d} (symbol={self.data_collector.symbol})")
            
            rows.append(row)
    
    return rows
```

------

## ✅ **Validation & Testing Checklist**

### **Unit Tests:**

1. **`_aggregate_single_candle`:**
   - Candle with only buy actions
   - Candle with only sell actions
   - Candle with mixed buy/sell actions
   - Candle with no actions
   - Trade exactly at threshold boundary
2. **Daily reset logic:**
   - Verify trackers reset between trading days
   - Check first candle uses open price as fallback
   - Verify cumulative values don't leak across dates
3. **Fallback scenarios:**
   - First candle of day with no trades
   - Middle candle with no trades (uses previous)
   - Multiple consecutive candles with no trades

### **Integration Tests:**

1. Multi-day calculation with various trading patterns
2. Verify average prices stay within reasonable bounds (close to OHLC)
3. Check all 8 new columns exist and have valid int values

------

## 🔍 **Edge Cases to Handle**

1. **No trades in category:** Use fallback (open for first, previous for others)
2. **First candle of day:** Always use `candle.open` as fallback
3. **All trades are "Undefined":** All avg prices use fallback
4. **Very first candle processed:** `previous_avg_prices` initialized with `candle.open`
5. **Daily boundaries:** Ensure clean reset, no data leakage

------

## 📊 **Expected Output Example**

For a candle at time `1757058305` on date `2025-09-05`:

```
{
    "time": 1757058305,
    "date": "2025-09-05",
    # ... existing OHLCV fields ...
    "shark450_buy_value": 1250,
    "shark450_sell_value": 800,
    "shark450_avg_buy_price": 26150,   # NEW
    "shark450_avg_sell_price": 26050,  # NEW
    "sheep450_avg_buy_price": 25980,   # NEW
    "sheep450_avg_sell_price": 25920,  # NEW
    # ... same pattern for 900 threshold ...
    # ... existing ratio fields ...
}
```

------

## 🎨 **Design Decisions Confirmed**

| Decision                 | Choice                                          |
| ------------------------ | ----------------------------------------------- |
| **Buy and Sell**         | ✅ Both sides tracked separately                 |
| **Zero volume handling** | Use fallback: open (first) or previous (middle) |
| **Price precision**      | `int` (matching OHLC)                           |
| **Cumulative reset**     | ✅ Reset at start of each trading day            |
| **Column naming**        | `{cat}{threshold}_avg_{side}_price`             |
| **Fallback strategy**    | First candle: `open`, Middle: previous value    |

------

## 📝 **Summary**

**Files to modify:**

- `whale_footprint_feature_calculator.py` (isolated changes)

**Methods to update:**

1. `_aggregate_single_candle` - Return tuple with accumulations
2. `_build_rows` - Add daily reset + cumulative tracking + fallback logic

**New columns per threshold:** 4 (shark_avg_buy_price, shark_avg_sell_price, sheep_avg_buy_price, sheep_avg_sell_price) **Total new columns:** 8 (for thresholds 450, 900)

