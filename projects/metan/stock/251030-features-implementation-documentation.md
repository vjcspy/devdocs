# 📊 Whale Footprint Features - Implementation Documentation

[TOC]
> Document is written by mixing Vietnamese/English
**Tài liệu này hướng dẫn cho Dev/AI Agent nhanh chóng hiểu được context và key points của việc xây dựng Candle Feature cho hợp đồng tương lai VN30F1M

---

## Core Principles

- As this feature applies to the VN30F1M derivative, it must incorporate data from 30 distinct stocks.
- Although we require a feature for the VN30F1M futures contract, but essentially, it is the aggregation of features from the 30 stocks within that contract. Therefore, we have to build the calculator for individual symbol first, and then aggregate them.



------

## 🎯 TL;DR - Quick Context

### Whale Footprint là gì?
Phân tích dấu chân của "cá mập" (shark - nhà đầu tư lớn) và "cừu" (sheep - nhà đầu tư nhỏ lẻ) trong giao dịch chứng khoán để:
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

### JSON data
Read `packages/stock/metan/stock/trading/domain/feature/persistor/intraday/intraday_symbol_feature_persistor.py` to understand how to persist features to the database.
```json
{
  "id": 1408,
  "symbol": "CEO",
  "time": 1759716000,
  "interval": 300,
  "open": 23400,
  "high": 24000,
  "low": 23400,
  "close": 23700,
  "volume": 206100,
  "value": 4894,
  "features": {
    "whale_footprint": {
      "shark450_buy_value": 2024,
      "shark900_buy_value": 1272,
      "sheep450_buy_value": 655,
      "sheep900_buy_value": 1407,
      "shark450_sell_value": 855,
      "shark900_sell_value": 0,
      "sheep450_sell_value": 1280,
      "sheep900_sell_value": 2135,
      "shark450_buy_avg_price": 23811,
      "shark900_buy_avg_price": 24000,
      "sheep450_buy_avg_price": 23738,
      "sheep900_buy_avg_price": 23611,
      "shark450_sell_avg_price": 23700,
      "shark900_sell_avg_price": 23400,
      "sheep450_sell_avg_price": 23709,
      "sheep900_sell_avg_price": 23705,
      "shark450_buy_ratio_5d_pc": 0.2174,
      "shark900_buy_ratio_5d_pc": 0.1366,
      "sheep450_buy_ratio_5d_pc": 0.0703,
      "sheep900_buy_ratio_5d_pc": 0.1511,
      "shark450_sell_ratio_5d_pc": 0.0918,
      "shark900_sell_ratio_5d_pc": 0,
      "sheep450_sell_ratio_5d_pc": 0.1375,
      "sheep900_sell_ratio_5d_pc": 0.2293
    }
  },
  "date": "2025-10-06"
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

---

## 📚 Reference Links

### Code Locations
- Base calculator: `packages/stock/metan/stock/trading/domain/feature/calculator/base_feature_calculator.py`
- Whale footprint: `packages/stock/metan/stock/trading/domain/feature/calculator/whale_footprint/whale_footprint_feature_calculator.py`
- Data collector: `packages/stock/metan/stock/info/domain/stock_data_collector/stock_data_collector.py`
- Persistor: `packages/stock/metan/stock/trading/domain/feature/persistor/intraday/intraday_symbol_feature_persistor.py`

---
