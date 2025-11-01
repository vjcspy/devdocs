# 📊 Whale Footprint Features - Implementation Documentation

[TOC]

**Tài liệu này hướng dẫn cho Dev/AI Agent nhanh chóng hiểu được context và key points của Whale Footprint Features**

> **Ngày tạo**: 2025-10-30  
> **Trạng thái**: Phase 1 - Basic Shark/Sheep Classification & Normalization ✅

---

## 🎯 TL;DR - Quick Context

### Whale Footprint là gì?
Phân tích dấu chân của "cá mập" (shark - nhà đầu tư lớn) và "cá cơm" (sheep - nhà đầu tư nhỏ lẻ) trong giao dịch chứng khoán để:
- Phát hiện hoạt động của tổ chức lớn (institutional trading)
- Đo lường sức mạnh mua/bán thực sự của các thế lực lớn
- Chuẩn hóa các giá trị để so sánh cross-day và cross-symbol

### Use Case trong AI Model
- **Feature Engineering**: Input cho LSTM, XGBoost, RL models
- **Pattern Recognition**: Phát hiện accumulation/distribution patterns
- **Signal Generation**: Kết hợp với RSI, MACD để tăng độ chính xác

---

## 📐 Architecture Overview

### Class Hierarchy
```
BaseFeatureCalculator (Abstract)
    ↓
WhaleFootprintFeatureCalculator
    ↓ uses
StockDataCollector
```

### Data Flow
```
Supabase (Raw Data)
    ↓
StockDataCollector
    ├─→ ticks() → list[Tick]
    ├─→ tick_candles_by_date() → dict[date, list[TickCandle]]
    └─→ prices() → list[Price]
    ↓
WhaleFootprintFeatureCalculator
    ├─→ Classification (shark vs sheep)
    ├─→ Aggregation (buy/sell values)
    ├─→ Average Price Tracking
    └─→ Normalization (ratios)
    ↓
pd.DataFrame (Features indexed by time)
```

---

## 🔑 Key Concepts

### 1. Phân loại Shark/Sheep theo Threshold

**Input**: Trade value (tính bằng raw units)
```python
trade_value_raw = price × volume  # đơn vị: đồng (VNĐ)
```

**Classification Logic** (per threshold T):
- T được định nghĩa trong **millions** (e.g., 450 = 450 triệu VNĐ)
- So sánh: `trade_value_raw >= T * 1_000_000`
  - ✅ → **shark**: Giao dịch lớn (nhà đầu tư tổ chức)
  - ❌ → **sheep**: Giao dịch nhỏ (nhà đầu tư cá nhân)

**Default Thresholds**: `[450, 900]` (450M và 900M VNĐ)

### 2. Sides (Hướng Giao Dịch)

Từ `TickAction.side`:
- `'B'` (Buy): Lệnh MUA
- `'S'` (Sell): Lệnh BÁN
- `'Undefined'`: Phiên ATO/ATC (KHÔNG tính trong whale footprint)

### 3. Point-in-Time vs Accumulative vs Moving-Window

**Naming Convention trong Code**:

| Loại | Prefix | Ví dụ | Mô tả |
|------|--------|-------|-------|
| **Point-in-time** | _(none)_ | `high`, `low`, `close` | Giá trị tại thời điểm trong candle |
| **Accumulative** | `accum_` | `accum_shark450_buy_value` | Cộng dồn trong khoảng thời gian (e.g., intraday) |
| **Moving-window** | `mov_{N}_` | `mov_15_shark_ratio` | Trung bình trượt N periods |

**Trong WhaleFootprintFeatureCalculator Phase 1**:
- Các features hiện tại là **point-in-time** (per candle)
- Average prices được track **cumulatively** trong ngày

### 4. Monetary Units - QUAN TRỌNG ⚠️

**Tất cả giá trị tiền tệ (value) trong application đều có đơn vị TRIỆU (millions)**

```python
# ✅ ĐÚNG - Flow trong code
trade_value_raw = price × volume          # raw units (VNĐ)
threshold_scaled = 450 * 1_000_000        # scale threshold to raw
is_shark = trade_value_raw >= threshold_scaled
value_in_millions = trade_value_raw / 1_000_000  # convert to millions

# 📊 Output
"shark450_buy_value": 1250  # = 1,250 triệu VNĐ = 1.25 tỷ VNĐ
```

**Lý do**:
- Tránh overflow khi làm việc với số lớn
- Dễ đọc, dễ hiểu trong báo cáo
- Consistency across entire application

---

## 🏗️ Implementation Details

### File Structure
```
packages/stock/metan/stock/trading/domain/feature/
├── calculator/
│   ├── base_feature_calculator.py          # Abstract base
│   └── whale_footprint/
│       └── whale_footprint_feature_calculator.py  # ✅ Main implementation
```

### Core Methods Workflow

#### 1. `_cal_candle_features()` - Main Entry Point
```python
def _cal_candle_features(self) -> pd.DataFrame:
    """
    Returns DataFrame với các columns:
    - time, date, candle_volume, candle_value
    - shark{T}_{side}_value: Point-in-time value trong candle
    - shark{T}_{side}_avg_price: Cumulative average price trong ngày
    - sheep{T}_{side}_value, sheep{T}_{side}_avg_price
    - {metric}_ratio_5d_pc: Normalized ratio với 5D baseline
    
    Indexed by: time (Unix timestamp)
    """
```

**Flow**:
1. Gather daily data → `_gather_daily_price_and_candle_counts()`
2. Validate base candle count → `_validate_and_get_base_candle_count()`
3. Compute baselines → `_compute_per_candle_baselines()`
4. Build feature rows → `_build_rows()`

#### 2. Baseline Computation

**Per-Candle Baselines** (đơn vị: millions):

```python
# Today's per-candle baseline
today_pc[date] = daily_value[date] / base_candle_count

# 5-Day trailing per-candle baseline (EXCLUDE current day)
prior_vals = daily_value.shift(1)
rolling_prior_sum = prior_vals.rolling(window=5, min_periods=5).sum()
pc_5d[date] = rolling_prior_sum / (5 * base_candle_count)
```

**Strict Requirements**:
- `today_pc` phải > 0 cho mọi ngày có giao dịch
- `pc_5d` yêu cầu đủ 5 ngày prior → raise ValueError nếu thiếu
- NaN cho non-trading days

#### 3. Average Price Tracking

**Cumulative Average Price** được tính trong ngày:

```python
# Initialize daily trackers
cumulative_trackers[key] = {
    "volume": 0.0,
    "weighted_value": 0.0  # price × volume (raw units)
}
previous_avg_prices[key] = first_candle_open  # Fallback

# Per candle update
cumulative_trackers[key]["volume"] += trade_volume
cumulative_trackers[key]["weighted_value"] += price × volume

# Calculate average
if cumulative_trackers[key]["volume"] > 0:
    avg_price = weighted_value / volume
else:
    avg_price = previous_avg_prices[key]  # Use fallback
```

**Key Design**:
- **Volume-weighted**: `avg_price = Σ(price × volume) / Σ(volume)`
- **Fallback mechanism**: Sử dụng first candle open hoặc previous average
- **Reset daily**: Mỗi ngày giao dịch reset lại trackers
- **Stateful**: Accumulate trong suốt phiên giao dịch

#### 4. Feature Aggregation

**Per Candle** (`_aggregate_single_candle()`):
```python
for threshold in thresholds:  # e.g., [450, 900]
    for side in ['buy', 'sell']:
        for category in ['shark', 'sheep']:
            # Aggregated value (millions)
            agg[f"{category}{threshold}_{side}_value"] = 0.0
            
            # For average price calculation (raw units)
            accumulations[f"{category}{threshold}_{side}_volume"] = 0.0
            accumulations[f"{category}{threshold}_{side}_weighted_value"] = 0.0

# Process each TickAction
for action in candle.tick_actions:
    trade_value_raw = action.price × action.volume
    category = 'shark' if trade_value_raw >= threshold × 1M else 'sheep'
    
    # Aggregate value (convert to millions)
    agg[f"{category}{threshold}_{side}_value"] += trade_value_raw / 1M
    
    # Accumulate for average (keep raw)
    accumulations[f"{category}{threshold}_{side}_volume"] += action.volume
    accumulations[f"{category}{threshold}_{side}_weighted_value"] += trade_value_raw
```

#### 5. Ratio Computation

**5D Per-Candle Ratios**:
```python
{metric}_ratio_5d_pc = round(metric_value / pc_5d[date], 4)
```

**Example**:
```python
# Candle có shark450_buy_value = 600 (triệu)
# pc_5d[date] = 150 (triệu per candle)
shark450_buy_ratio_5d_pc = round(600 / 150, 4) = 4.0000

# Interpretation: Shark buy trong candle này gấp 4 lần baseline 5 ngày
```

---

## ⚠️ Strict Error Handling (Data Precision Focus)

**Fail-Fast Strategy**: Không có silent fallback, mọi inconsistency đều raise ValueError

### Validation Checklist

| Condition | Error Raised | Reason |
|-----------|--------------|--------|
| `Price.value <= 0` cho ngày có candles | ✅ ValueError | Không thể tính ratio với baseline = 0 |
| `today_pc <= 0` | ✅ ValueError | Baseline không hợp lệ |
| `pc_5d <= 0` hoặc NaN khi expected | ✅ ValueError | Thiếu data cho normalization |
| `candle_count` diff > 1 across days | ✅ ValueError | Data inconsistency |
| Date có ticks nhưng không có candles | ✅ ValueError | Data processing error |

**Example Error Message**:
```python
raise ValueError(
    f"Insufficient prior trading days for 5D baseline calculation. "
    f"Missing/invalid pc_5d for dates={trading_days_without_5d} "
    f"(symbol={self.data_collector.symbol}). "
    f"Ensure at least 5 prior trading days exist before the first date in your range."
)
```

---

## 📊 Output Schema

### DataFrame Structure
```python
# Index: time (Unix timestamp)
# Columns:
{
    "time": 1757058305,
    "date": "2025-09-05",
    "candle_volume": 50000,
    "candle_value": 1300,  # millions
    
    # Shark features (threshold 450)
    "shark450_buy_value": 1250,        # millions
    "shark450_sell_value": 800,        # millions
    "shark450_buy_avg_price": 26150,   # VNĐ (cumulative trong ngày)
    "shark450_sell_avg_price": 26050,  # VNĐ
    "shark450_buy_ratio_5d_pc": 4.0000,
    "shark450_sell_ratio_5d_pc": 2.5000,
    
    # Sheep features (threshold 450)
    "sheep450_buy_value": 50,
    "sheep450_sell_value": 80,
    "sheep450_buy_avg_price": 25980,
    "sheep450_sell_avg_price": 25920,
    "sheep450_buy_ratio_5d_pc": 0.1500,
    "sheep450_sell_ratio_5d_pc": 0.2000,
    
    # ... same pattern for threshold 900
}
```

### Column Naming Pattern
```
{category}{threshold}_{side}_{metric}

category ∈ {shark, sheep}
threshold ∈ {450, 900}  # configurable
side ∈ {buy, sell}
metric ∈ {value, avg_price, ratio_5d_pc}
```

---

## ✅ Completed Features

### 1. Basic Classification ✅
- [x] Shark/Sheep classification theo thresholds
- [x] Buy/Sell side separation
- [x] Trade value aggregation trong millions

### 2. Average Price Tracking ✅
- [x] Cumulative volume-weighted average prices
- [x] Per-category, per-threshold, per-side
- [x] Fallback mechanism (first candle open)
- [x] Daily reset

### 3. Normalization ✅
- [x] 5-day trailing per-candle baseline (exclude current)
- [x] Ratio computation (rounded to 4 decimals)
- [x] Strict validation (min_periods=5)

### 4. Data Validation ✅
- [x] Price.value > 0 validation
- [x] Base candle count consistency check
- [x] Baseline computation validation
- [x] Comprehensive error messages with context

---

## 🚧 Remaining Features (From FEATURE_ENGINEERING.md)

### Extended Shark/Sheep Metrics (Planned)
- [ ] `shark_net_flow_volume` = shark_buy_volume - shark_sell_volume
- [ ] `shark_net_flow_value` = shark_buy_value - shark_sell_value
- [ ] `shark_volume_participation` = total_shark_volume / candle_volume
- [ ] `diff_shark` = shark_buy_value - shark_sell_value (per candle)
- [ ] `accum_diff_shark_7` = 7-day cumulative diff_shark
- [ ] `accum_diff_shark_15` = 15-day cumulative diff_shark

### Advanced Ratios & Comparisons (Planned)
- [ ] `shark_buy_volume_to_average_5_days`
- [ ] `shark_buy_value_to_average_5_days`
- [ ] `shark_urgency_spread = (avg_price_large_buys - avg_price_large_sells) / vwap`
- [ ] `price_change_per_shark_flow = (close - open) / shark_net_flow_value`

### Absorption & Pressure (Planned)
- [ ] `absorption_score`: Đếm events "bán nhiều nhưng giá không giảm"
- [ ] `brittle_market_score`: Hiệu quả của lực bán (price_change / sell_pressure)
- [ ] Window-based analysis (5-min, 15-min windows)

### Session Dynamics (Planned)
- [ ] `morning_buy_sell_ratio`: Phiên sáng
- [ ] `afternoon_buy_sell_ratio`: Phiên chiều
- [ ] `closing_power_ratio`: 30 phút cuối phiên
- [ ] `close_vs_vwap`: Giá đóng cửa vs VWAP
- [ ] `afternoon_reversal_flag`: Detect reversal patterns

### Volatility Features (Planned)
- [ ] `intraday_volatility`: Std dev của price returns
- [ ] `volatility_concentration`: Phân bố biến động trong phiên
- [ ] `volatility_momentum`: Volatility chiều/sáng ratio
- [ ] ATR-normalized volatility

---

## 🔧 Usage Example

### Basic Usage
```python
from metan.stock.trading.domain.feature.calculator.whale_footprint import (
    WhaleFootprintFeatureCalculator
)
from metan.stock.info.domain.candle.models import IntradayInterval

# Initialize
calculator = WhaleFootprintFeatureCalculator(
    symbol='VN30',
    start_date='2025-01-15',  # Cần ít nhất 5 ngày prior data
    end_date='2025-01-31',
    interval=IntradayInterval.FIVE_MINUTES,
    thresholds=[450, 900]  # Optional, defaults to [450, 900]
)

# Calculate features
features_df = calculator.cal()

# Access data
print(features_df.head())
print(features_df.columns.tolist())

# Example: Filter shark activity
shark_heavy_candles = features_df[
    features_df['shark450_buy_value'] > features_df['shark450_sell_value']
]
```

### Integration với AI Models
```python
# Prepare features for LSTM
import numpy as np

# Select relevant columns
feature_cols = [
    'shark450_buy_ratio_5d_pc',
    'shark450_sell_ratio_5d_pc',
    'shark450_buy_avg_price',
    'shark450_sell_avg_price',
    'sheep450_buy_ratio_5d_pc',
    'sheep450_sell_ratio_5d_pc',
]

X = features_df[feature_cols].values

# Reshape for LSTM (samples, timesteps, features)
X_lstm = X.reshape((X.shape[0], 1, X.shape[1]))
```

---

## 🎓 AI Agent Guidelines

### Khi cần extend features:

1. **Check data availability** từ StockDataCollector:
   - `ticks()`: Có TickAction với price, volume, side
   - `tick_candles_by_date()`: Có OHLCV per candle
   - `prices()`: Có daily price data

2. **Follow naming convention**:
   - Point-in-time: no prefix
   - Accumulative: `accum_` prefix
   - Moving-window: `mov_{N}_` prefix

3. **Maintain units consistency**:
   - Values: always in **millions**
   - Prices: always in **VNĐ** (raw)
   - Volumes: always in **shares** (raw)

4. **Error handling strategy**:
   - Validate inputs strictly
   - Raise ValueError with descriptive message + symbol context
   - No silent fallbacks for data precision features

5. **Testing checklist**:
   - Multi-day scenarios
   - Edge cases (insufficient data, missing values)
   - Ratio precision (4 decimals)
   - Performance (large datasets)

### Khi debug issues:

1. Check logs với symbol context
2. Validate baseline computation
3. Verify threshold scaling (millions)
4. Inspect cumulative trackers state
5. Compare output schema với expected

---

## 📚 Reference Links

### Internal Docs
- [FEATURE_ENGINEERING.md](./FEATURE_ENGINEERING.md): Full feature roadmap
- [250907-whale-footprint-feature-engineering.md](./250907-whale-footprint-feature-engineering.md): Original ticket v4

### Code Locations
- Base calculator: `packages/stock/metan/stock/trading/domain/feature/calculator/base_feature_calculator.py`
- Whale footprint: `packages/stock/metan/stock/trading/domain/feature/calculator/whale_footprint/whale_footprint_feature_calculator.py`
- Data collector: `packages/stock/metan/stock/info/domain/stock_data_collector/stock_data_collector.py`

---

## 🔄 Changelog

| Date | Phase | Changes |
|------|-------|---------|
| 2025-10-30 | Initial | Created documentation for Phase 1 implementation |
| 2025-09-07 | Phase 1 | Basic shark/sheep classification + average price tracking + 5D normalization |

---

**Maintained by**: Stock AI Team  
**Last updated**: 2025-10-30  
**Status**: ✅ Phase 1 Complete | 🚧 Phase 2-6 Planned
