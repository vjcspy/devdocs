# 📊 Features - Implementation Documentation

[TOC]
> Document is written by mixing Vietnamese/English
**Tài liệu này hướng dẫn cho Dev/AI Agent nhanh chóng hiểu được context và key points của việc xây dựng Candle Feature cho hợp đồng tương lai VN30F1M

---

## Core Principles

- As this feature applies to the VN30F1M derivative, it must incorporate data from 30 distinct stocks.
- Although we require a feature for the VN30F1M futures contract, but essentially, it is the aggregation of features from the 30 stocks within that contract. Therefore, we have to build the calculator for individual symbol first, and then aggregate them.



------

## 🎯 TL;DR - Quick Context

### Features

#### Whale Footprint là gì?
Phân tích dấu chân của "cá mập" (shark - nhà đầu tư lớn) và "cừu" (sheep - nhà đầu tư nhỏ lẻ) trong giao dịch chứng khoán để:

- Phát hiện hoạt động của tổ chức lớn (institutional trading)
- Đo lường sức mạnh mua/bán thực sự của các thế lực lớn
- Chuẩn hóa các giá trị để so sánh cross-day và cross-symbol

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
│   ├── common/
│   │   └── base.py                          # Shared aggregation/validation utilities
│   ├── base_feature_calculator.py           # Abstract base
│   └── whale_footprint/
│       ├── shark_values.py                  # Per-candle buy/sell values (millions)
│       ├── avg_prices.py                    # Cumulative avg price per side/category
│       ├── ratios_5d_pc.py                  # Per-candle value vs 5D baseline
│       ├── urgency_spread.py                # Shark buy-sell spread vs VWAP (%)
│       └── whale_footprint_feature_calculator.py  # Main implementation
├── persistor/
│   └── intraday/
│       └── intraday_symbol_feature_persistor.py    # Merge + upsert to DB
```

---

## ⚠️ Strict Error Handling (Data Precision Focus)

**Fail-Fast Strategy**: Không có silent fallback, mọi inconsistency đều raise ValueError

**Example Error Message**:

```python
raise ValueError(
    "Insufficient prior trading days for 5D baseline calculation. "
    f"Missing/invalid pc_value_5d for dates={missing}. Ensure at least 5 prior trading days exist."
)

# Ngoài ra, khi bộ ngày của prices và tick_candles không khớp trong phạm vi yêu cầu
raise ValueError(
    f"Mismatch between price days and tick days in range: price_days={price_days} tick_days={tick_days}"
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
  "time": "2025-10-06T09:05:00Z",
  "interval": 300,
  "open": 23400,
  "high": 24000,
  "low": 23400,
  "close": 23700,
  "volume": 206100,
  "value": 4894,
  "features": {
    "whale_footprint": {
      "date" : "2025-11-03",
      "candle_volume" : 172000,
      "candle_value" : 4311.0,
      "shark450_buy_value" : 480.0,
      "sheep450_buy_value" : 2517.0,
      "shark450_sell_value" : 0.0,
      "sheep450_sell_value" : 1282.0,
      "shark900_buy_value" : 0.0,
      "sheep900_buy_value" : 2997.0,
      "shark900_sell_value" : 0.0,
      "sheep900_sell_value" : 1282.0,
      "shark450_buy_avg_price" : 25000,
      "sheep450_buy_avg_price" : 25073,
      "shark450_sell_avg_price" : 26000,
      "sheep450_sell_avg_price" : 25085,
      "shark900_buy_avg_price" : 26000,
      "sheep900_buy_avg_price" : 25061,
      "shark900_sell_avg_price" : 26000,
      "sheep900_sell_avg_price" : 25085,
      "shark450_buy_ratio_5d_pc" : 0.0713,
      "sheep450_buy_ratio_5d_pc" : 0.3741,
      "shark450_sell_ratio_5d_pc" : 0.0,
      "sheep450_sell_ratio_5d_pc" : 0.1905,
      "shark900_buy_ratio_5d_pc" : 0.0,
      "sheep900_buy_ratio_5d_pc" : 0.4455,
      "shark900_sell_ratio_5d_pc" : 0.0,
      "sheep900_sell_ratio_5d_pc" : 0.1905,
      "shark450_urgency_spread" : -3.989,
      "shark900_urgency_spread" : 0.0
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
metric ∈ {value, avg_price, ratio_5d_pc, urgency_spread}
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
   - Hiện tại sử dụng các metric: `value`, `avg_price`, `ratio_5d_pc`, `urgency_spread`
   - Nếu mở rộng thêm moving-window: dùng `mov_{N}_` theo chuẩn

3. **Maintain units consistency**:
   - Values: always in **millions**
   - Prices: always in **VNĐ** (raw)
   - Volumes: always in **shares** (raw)

   Lưu ý: Phân loại shark/sheep so sánh theo `trade_value_raw = price × volume`,
   trong khi các trường đầu ra `*_value` và `candle.value` đã được chuẩn hóa về đơn vị triệu.

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
