# VN30 Aggregate Whale Footprint Features

> **Status:** ✅ IMPLEMENTED  
> **Branch:** `feature/vn30-aggregate-whale-footprint`  
> **Last Updated:** 2025-12-31

## Overview

Tổng hợp các whale footprint features từ 30 cổ phiếu VN30 thành các features cấp Index để sử dụng cho AI model dự đoán VN30F1M.

## Nguyên tắc chung

### Tại sao không thể đơn giản cộng/chia trung bình các features đã build?

Có 2 loại features:
1. **Additive Features**: Có thể cộng trực tiếp (như value)
2. **Ratio/Rate Features**: KHÔNG thể cộng rồi chia trung bình vì mẫu số khác nhau

**Ví dụ minh họa `shark450_buy_ratio_5d_pc`:**
```
Stock A: ratio = 100 / 1000 = 0.10 (shark_buy=100, pc_5d=1000)
Stock B: ratio = 200 / 400  = 0.50 (shark_buy=200, pc_5d=400)

SAI: (0.10 + 0.50) / 2 = 0.30
ĐÚNG: (100 + 200) / (1000 + 400) = 300 / 1400 = 0.214
```

### Về việc sử dụng `total_shares` và `free_float`

Hiện tại đã có:
- `Stock.total_shares`: Tổng số cổ phiếu phát hành
- `VN30_FREE_FLOAT_RATIOS`: Tỷ lệ free-float cho từng mã

**Market Cap Weight:**
```python
weight_i = price_i × total_shares_i × free_float_ratio_i
```

**✅ Quyết định sử dụng weight:**
| Feature | Weight? | Method |
|---------|---------|--------|
| `*_value` | ❌ | Simple Sum |
| `*_ratio_5d_pc` | ❌ | Sum(value) / Sum(pc_5d) |
| `percent_*` | ❌ | Simple Sum → tính ratio từ aggregated values |
| `*_avg_price` | ❌ | **KHÔNG aggregate** |
| `urgency_spread` | ✅ | Market Cap Weighted Average |

---

## Chi tiết từng Feature

### 1. `sharkXXX_buy_value`, `sharkXXX_sell_value`, `sheepXXX_buy_value`, `sheepXXX_sell_value`

| Item | Value |
|------|-------|
| **Type** | Additive |
| **Có thể tận dụng built features?** | ✅ **CÓ** |
| **Aggregate Method** | Simple Sum |

**Formula:**
```python
vn30_shark450_buy_value = Σ(stock_i.shark450_buy_value) for i in VN30
vn30_shark450_sell_value = Σ(stock_i.shark450_sell_value) for i in VN30
# Tương tự cho sheep và threshold 900
```

**Implementation:**
```python
# Query từ stock_trading_feature_candles
features_all_stocks = fetch_features_for_all_vn30_at_time(time)
vn30_shark450_buy_value = sum(f.shark450_buy_value for f in features_all_stocks)
```

---

### 2. `sharkXXX_buy_ratio_5d_pc`, `sharkXXX_sell_ratio_5d_pc`, `sheepXXX_*_ratio_5d_pc`

| Item | Value |
|------|-------|
| **Type** | Ratio (tử số và mẫu số đều là additive) |
| **Có thể tận dụng built features?** | ⚠️ **MỘT PHẦN** |
| **Aggregate Method** | Tính lại từ aggregated components |

**Vấn đề:**
- `ratio_5d_pc = shark_buy_value / pc_value_5d`
- `pc_value_5d` của từng stock khác nhau
- Không thể average các ratio!

**Formula ĐÚNG:**
```python
# Tử số: có thể lấy từ built features
vn30_shark450_buy_value = Σ(stock_i.shark450_buy_value)

# Mẫu số: CẦN TÍNH LẠI
# Option A: Tính từ raw data (chính xác nhất)
vn30_pc_value_5d = Σ(stock_i.daily_value_5days_prior) / (5 * candle_count)

# Option B: Query pc_value_5d từ intermediate data nếu đã lưu
vn30_pc_value_5d = Σ(stock_i.pc_value_5d)  # Nếu đã lưu pc_value_5d cho từng stock

# Final ratio
vn30_shark450_buy_ratio_5d_pc = vn30_shark450_buy_value / vn30_pc_value_5d
```

**Có 2 options để implement:**

**Option A - Tính từ raw tick data (Recommended for accuracy):**
```python
# Cần gather daily_value cho tất cả 30 stocks
vn30_daily_value = Σ(stock_i.daily_value) for each day
vn30_pc_5d = rolling_5d_sum(vn30_daily_value) / (5 * candle_count)
```

**Option B - Tận dụng features đã build (Faster, cần lưu thêm data):**
- Cần lưu thêm `pc_value_5d` cho từng stock trong feature table
- Sau đó: `vn30_pc_5d = Σ(stock_i.pc_value_5d)`

---

### 3. `sharkXXX_buy_avg_price`, `sharkXXX_sell_avg_price` - ❌ KHÔNG AGGREGATE

| Item | Value |
|------|-------|
| **VN30 Aggregate** | ❌ **KHÔNG CẦN** |
| **Lý do** | Giá mỗi cổ phiếu khác nhau hoàn toàn, aggregate không có ý nghĩa |
| **Thay thế** | Sử dụng `urgency_spread` (đã normalize) |

**Vấn đề khi aggregate:**
```
VNM avg_buy = 70,000 VND
BID avg_buy = 50,000 VND  
VIC avg_buy = 40,000 VND

→ VN30 avg = ??? Không có ý nghĩa kinh tế!
```

**Quyết định:**
- ✅ Giữ `avg_price` cho **từng symbol riêng** (phục vụ pick top stocks to trade)
- ❌ Không aggregate cho VN30 level
- ✅ Dùng `urgency_spread` để đo hành vi shark trên VN30 (đã normalize bằng VWAP)

---

### 4. Percent Ratios - Plan Chi Tiết

**Features (8 × 2 thresholds = 16 features):**
- `percent_sharkXXX_buy_sell` = Shark Buy / (Shark Buy + Shark Sell)
- `percent_sheepXXX_buy_sell` = Sheep Buy / (Sheep Buy + Sheep Sell)
- `percent_buy_sharkXXX_sheep` = Shark Buy / (Shark Buy + Sheep Buy)
- `percent_sell_sharkXXX_sheep` = Shark Sell / (Shark Sell + Sheep Sell)
- + `accum_percent_*` (accumulated versions từ đầu phiên)

| Item | Value |
|------|-------|
| **Type** | Ratio |
| **Aggregate Method** | ✅ **Simple Sum** (không dùng weight) |
| **Có thể tận dụng built features?** | ✅ Yes (dùng `*_value` đã có) |

#### ✅ Formula cho VN30 (Simple Sum)

```python
# Step 1: Aggregate raw values (đã có từ Feature #1)
vn30_shark_buy = Σ(stock_i.shark450_buy_value)
vn30_shark_sell = Σ(stock_i.shark450_sell_value)
vn30_sheep_buy = Σ(stock_i.sheep450_buy_value)
vn30_sheep_sell = Σ(stock_i.sheep450_sell_value)

# Step 2: Tính percent từ aggregated
vn30_percent_shark_buy_sell = vn30_shark_buy / (vn30_shark_buy + vn30_shark_sell) * 100
vn30_percent_sheep_buy_sell = vn30_sheep_buy / (vn30_sheep_buy + vn30_sheep_sell) * 100
vn30_percent_buy_shark_sheep = vn30_shark_buy / (vn30_shark_buy + vn30_sheep_buy) * 100
vn30_percent_sell_shark_sheep = vn30_shark_sell / (vn30_shark_sell + vn30_sheep_sell) * 100
```

#### Accumulated Percent - Cần lưu thêm data

**Vấn đề:** Current features table chỉ lưu `accum_percent_*`, không lưu `accum_*_value`

**Solution:** Lưu thêm accumulated values

```python
# Cần thêm vào feature table
NEW_COLUMNS = [
    "accum_shark450_buy_value",    # Accumulated từ đầu phiên
    "accum_shark450_sell_value",
    "accum_sheep450_buy_value", 
    "accum_sheep450_sell_value",
    # Same for 900 threshold
]

# Sau đó VN30 aggregate:
vn30_accum_shark_buy = Σ(stock_i.accum_shark450_buy_value)
vn30_accum_percent_shark_buy_sell = (
    vn30_accum_shark_buy / (vn30_accum_shark_buy + vn30_accum_shark_sell)
) * 100
```

#### Ý nghĩa VN30 Percent Features

| Feature | Ý nghĩa |
|---------|---------|
| `vn30_percent_shark_buy_sell = 70%` | 70% dòng tiền lớn trên VN30 đang MUA |
| `vn30_percent_buy_shark_sheep = 40%` | 40% lực MUA là từ shark, 60% từ retail |
| `vn30_accum_percent_shark_buy_sell = 65%` | Tích lũy từ đầu phiên, 65% shark activity là MUA |

---

### 5. `sharkXXX_urgency_spread` - Giải thích Chi Tiết

| Item | Value |
|------|-------|
| **Type** | Normalized ratio (dimensionless %) |
| **Có thể tận dụng built features?** | ✅ **CÓ** (đã normalize) |
| **Aggregate Method** | Weighted Average by Market Cap |

#### Ý nghĩa của Feature (Per-Stock)

```python
urgency_spread = (shark_buy_avg_price - shark_sell_avg_price) * 100 / VWAP
```

| Giá trị | Ý nghĩa | Signal |
|---------|---------|--------|
| **+2% đến +5%** | Shark mua cao hơn bán 2-5% | 🟢 Bullish - urgent to buy |
| **-2% đến -5%** | Shark bán thấp hơn mua 2-5% | 🔴 Bearish - urgent to sell |
| **Gần 0** | Shark mua/bán cân bằng | ⚪ Neutral |

**Ví dụ thực tế:**
```
VNM: shark_buy_avg = 71,000, shark_sell_avg = 70,000, VWAP = 70,500
→ urgency_spread = (71000 - 70000) * 100 / 70500 = +1.42%
→ Shark sẵn sàng mua cao hơn bán ~1.4%, có urgency mua
```

#### Ý nghĩa khi Aggregate cho VN30

**`vn30_urgency_spread`** đại diện cho:
> "Mức độ aggressive của dòng tiền lớn trên TOÀN BỘ VN30"

| Giá trị VN30 | Market Signal |
|--------------|---------------|
| **> +1.5%** | Shark đang aggressive mua trên diện rộng, bullish pressure |
| **< -1.5%** | Shark đang urgent bán, bearish pressure |
| **-0.5% đến +0.5%** | Thị trường cân bằng, chờ direction |

#### Cách tính cho VN30 - Weighted Average (✅ Recommended)

Vì `urgency_spread` đã được **normalize bằng VWAP** (dimensionless %), có thể weighted average:

```python
# Weight = market_cap = price × total_shares × free_float_ratio
from metan.stock.info.domain.index.constants import VN30_FREE_FLOAT_RATIOS

def compute_vn30_urgency_spread(features_by_symbol: dict, stocks_info: dict) -> float:
    """
    Compute VN30 weighted urgency spread.
    
    Args:
        features_by_symbol: {symbol: {urgency_spread, close_price, ...}}
        stocks_info: {symbol: Stock}
    """
    total_weighted_spread = 0.0
    total_market_cap = 0.0
    
    for symbol in VN30_SYMBOLS:
        stock = stocks_info[symbol]
        feature = features_by_symbol[symbol]
        
        # Market cap weight
        free_float = VN30_FREE_FLOAT_RATIOS.get(symbol, 1.0)
        market_cap = feature["close_price"] * stock.total_shares * free_float
        
        # Weighted spread
        spread = feature["shark450_urgency_spread"]  # hoặc threshold 900
        total_weighted_spread += market_cap * spread
        total_market_cap += market_cap
    
    return total_weighted_spread / total_market_cap if total_market_cap > 0 else 0.0
```

#### ✅ Decision: Market Cap Weighted

```python
weights = compute_market_cap_weights(total_shares, free_float, prices)
vn30_urgency_spread = Σ(weight_i × stock_i.urgency_spread)
```

Lý do: Target là dự đoán VN30F1M → cần weight theo ảnh hưởng lên index.

---

## Summary Table

| Feature | VN30 Aggregate? | Method | Tận dụng built features? | Cần thêm data? |
|---------|-----------------|--------|-------------------------|----------------|
| `sharkXXX_*_value` | ✅ | Simple Sum | ✅ Yes | ❌ |
| `sharkXXX_*_ratio_5d_pc` | ✅ | Sum(value) / Sum(pc_5d) | ⚠️ Partial | ✅ `pc_value_5d` |
| `sharkXXX_*_avg_price` | ❌ **KHÔNG** | N/A | N/A | N/A |
| `percent_*` (current) | ✅ | Tính từ aggregated values | ✅ Yes | ❌ |
| `accum_percent_*` | ✅ | Tính từ accum values | ⚠️ Partial | ✅ `accum_*_value` |
| `urgency_spread` | ✅ | Market Cap Weighted Avg | ✅ Yes | ✅ `total_shares`, `close_price` |

---

## Recommended Implementation Strategy

### Phase 1: Extend Current Feature Table Schema

Thêm các columns intermediate vào `stock_trading_feature_candles`:

```python
# Columns mới cần thêm
NEW_COLUMNS = [
    # For ratio_5d_pc calculation
    "pc_value_5d",  # baseline per-candle value 5 days
    
    # For accumulated percent calculation  
    "accum_shark450_buy_value",
    "accum_shark450_sell_value",
    "accum_sheep450_buy_value",
    "accum_sheep450_sell_value",
    # Same for threshold 900...
    "accum_shark900_buy_value",
    "accum_shark900_sell_value",
    "accum_sheep900_buy_value",
    "accum_sheep900_sell_value",
    
    # close_price (nếu chưa có) cho urgency_spread weight
    "close_price",
]
```

**Lưu ý:** KHÔNG cần lưu `volume`, `weighted_value` cho avg_price vì feature này không aggregate cho VN30.

### Phase 2: Update WhaleFootprintFeatureCalculator

Thêm việc lưu các intermediate values vào output row:

```python
# Trong _build_rows():
row["pc_value_5d"] = pc_value_5d
row["close_price"] = candle.close

# Accumulated values (đã có trong cumulative_trackers)
for threshold in thresholds:
    for cat in ["shark", "sheep"]:
        for side in ["buy", "sell"]:
            key = f"{cat}{threshold}_{side}"
            row[f"accum_{key}_value"] = cumulative_trackers[key]["weighted_value"] / 1_000_000
```

### Phase 3: Create VN30AggregateFeatureCalculator

```python
class VN30WhaleFootprintAggregator:
    """Aggregate whale footprint features from all VN30 stocks."""
    
    def __init__(self, start_date: str, end_date: str, interval: IntradayInterval):
        self.symbols = VN30_SYMBOLS
        self.start_date = start_date
        self.end_date = end_date
        self.interval = interval
        
    def calculate(self) -> pd.DataFrame:
        """
        Returns DataFrame with VN30-level features:
        - vn30_shark450_buy_value, vn30_shark450_sell_value, ...
        - vn30_shark450_buy_ratio_5d_pc, ...
        - vn30_percent_shark450_buy_sell, ...
        - vn30_accum_percent_shark450_buy_sell, ...
        - vn30_shark450_urgency_spread
        """
        # 1. Fetch features for all 30 stocks at each timepoint
        features_by_symbol = self._fetch_all_symbol_features()
        
        # 2. Fetch stock info (total_shares) 
        stocks_info = self._fetch_stocks_info()
        
        # 3. Group by timepoint and aggregate
        rows = []
        for timepoint in self._get_all_timepoints():
            row = self._aggregate_at_timepoint(
                timepoint, features_by_symbol, stocks_info
            )
            rows.append(row)
        
        return pd.DataFrame(rows)
    
    def _aggregate_at_timepoint(self, timepoint, features_by_symbol, stocks_info) -> dict:
        row = {"time": timepoint}
        
        # 1. Sum values
        row["vn30_shark450_buy_value"] = sum(
            f.shark450_buy_value for f in features_at_time
        )
        # ... other value sums
        
        # 2. Ratio_5d_pc
        sum_shark_buy = row["vn30_shark450_buy_value"]
        sum_pc_5d = sum(f.pc_value_5d for f in features_at_time)
        row["vn30_shark450_buy_ratio_5d_pc"] = sum_shark_buy / sum_pc_5d
        
        # 3. Percent (from aggregated values)
        row["vn30_percent_shark450_buy_sell"] = ...
        
        # 4. Urgency spread (market cap weighted)
        row["vn30_shark450_urgency_spread"] = self._compute_weighted_urgency(
            features_at_time, stocks_info
        )
        
        return row
```

### Phase 4 (Optional): Add Cross-Stock Statistics

Thêm các features thống kê để capture "breadth" và "concentration":

```python
# Additional VN30 features cho AI model
CROSS_STOCK_FEATURES = [
    "vn30_shark_activity_breadth",    # số cổ phiếu có shark activity > 0
    "vn30_shark_buy_concentration",   # Herfindahl index: shark tập trung hay phân tán?
    "vn30_shark_buy_std",             # standard deviation across stocks
    "vn30_top5_shark_contribution",   # % contribution từ top 5 stocks có shark nhiều nhất
]
```

**Ý nghĩa:**
- `breadth = 25/30` → 25 cổ phiếu có shark activity, signal mạnh hơn `breadth = 5/30`
- `concentration` cao → Shark tập trung vào vài cổ phiếu (có thể manipulation)
- `concentration` thấp → Shark phân tán, consensus signal

---

## ✅ Final Decisions

| Question | Decision |
|----------|----------|
| Lưu intermediate data? | ✅ **Lưu** - reuse nhiều lần, storage rẻ |
| Aggregate `avg_price`? | ❌ **KHÔNG** - giá khác nhau, không có ý nghĩa |
| Weight cho `percent_*`? | ❌ **Simple Sum** - không dùng weight |
| Weight cho `urgency_spread`? | ✅ **Market Cap Weighted** - phản ánh ảnh hưởng VN30 |

---

## VN30 Features Final List

Sau khi aggregate, các features cho VN30:

```python
VN30_FEATURES = [
    # Value (Sum)
    "vn30_shark450_buy_value",
    "vn30_shark450_sell_value",
    "vn30_sheep450_buy_value",
    "vn30_sheep450_sell_value",
    # Same for 900
    
    # Ratio 5d (Sum/Sum)
    "vn30_shark450_buy_ratio_5d_pc",
    "vn30_shark450_sell_ratio_5d_pc",
    # ... same pattern
    
    # Percent (from aggregated)
    "vn30_percent_shark450_buy_sell",
    "vn30_percent_sheep450_buy_sell",
    "vn30_percent_buy_shark450_sheep",
    "vn30_percent_sell_shark450_sheep",
    # Same for 900 and accumulated versions
    
    # Urgency (Weighted Avg)
    "vn30_shark450_urgency_spread",
    "vn30_shark900_urgency_spread",
    
    # Cross-stock statistics (Optional Phase 4)
    "vn30_shark_activity_breadth",
    "vn30_shark_buy_concentration",
]
```

---

## ✅ Implementation Status

| Phase | Status | Description |
|-------|--------|-------------|
| Phase 1 | ✅ Done | Update `WhaleFootprintFeatureCalculator` - thêm intermediate values |
| Phase 2 | ✅ Done | Create `VN30WhaleFootprintAggregator` |
| Phase 3 | ✅ Done | Test aggregate calculations |
| Phase 4 | ⏳ Pending | Add cross-stock statistics (optional) |

---

## Implementation Details

### Files Created/Modified

```
packages/stock/metan/stock/trading/domain/feature/
├── calculator/
│   └── whale_footprint/
│       ├── constants.py                    # NEW - Shared thresholds
│       ├── whale_footprint_feature_calculator.py  # MODIFIED
│       └── features/
│           └── intermediate_values.py      # NEW - Intermediate values for VN30
└── aggregator/                             # NEW - Directory
    └── vn30/
        ├── __init__.py
        └── vn30_whale_footprint_aggregator.py
```

### Constants (Shared)

```python
# packages/stock/.../whale_footprint/constants.py
WHALE_FOOTPRINT_THRESHOLDS: list[int] = [450, 900]
WHALE_FOOTPRINT_CATEGORIES: list[str] = ["shark", "sheep"]
WHALE_FOOTPRINT_SIDES: list[str] = ["buy", "sell"]
```

### Intermediate Values Added

Per-symbol features table now includes:

```python
# Added to each candle row
{
    "pc_value_5d": 169.85,           # Baseline for ratio calculation
    "close_price": 25000,            # For market cap weight
    "accum_shark450_buy_value": 123.45,   # Accumulated values
    "accum_shark450_sell_value": 67.89,
    "accum_sheep450_buy_value": 234.56,
    "accum_sheep450_sell_value": 189.01,
    # Same for threshold 900...
}
```

### Error Handling

```python
# Raises ValueError if:
# - sum_pc_5d <= 0 (invalid baseline data)
# - close_price <= 0 (invalid price data)
# - missing total_shares for any symbol
```

---

## Sample Output

Tested with date range `2023-09-20` to `2023-12-20`:

```python
# Sample row at first candle
{
    # Value Features (Simple Sum across 30 stocks) - Unit: millions VND
    "vn30_shark450_buy_value": 48942.0,     # 48.9 billion VND shark buy
    "vn30_shark450_sell_value": 16111.0,    # 16.1 billion VND shark sell
    "vn30_sheep450_buy_value": 36914.0,     # 36.9 billion VND retail buy
    "vn30_sheep450_sell_value": 30481.0,    # 30.5 billion VND retail sell
    "vn30_shark900_buy_value": 30357.0,
    "vn30_shark900_sell_value": 7971.0,
    "vn30_sheep900_buy_value": 55499.0,
    "vn30_sheep900_sell_value": 38621.0,
    
    # Ratio vs 5-day Baseline (dimensionless)
    "vn30_shark450_buy_ratio_5d_pc": 0.2881,  # 28.8% of avg 5-day per-candle value
    "vn30_shark450_sell_ratio_5d_pc": 0.0949,
    "vn30_sheep450_buy_ratio_5d_pc": 0.2173,
    "vn30_sheep450_sell_ratio_5d_pc": 0.1795,
    "vn30_shark900_buy_ratio_5d_pc": 0.1787,
    "vn30_shark900_sell_ratio_5d_pc": 0.0469,
    "vn30_sheep900_buy_ratio_5d_pc": 0.3267,
    "vn30_sheep900_sell_ratio_5d_pc": 0.2274,
    
    # Percent Features (current candle)
    "vn30_percent_shark450_buy_sell": 75.23,      # 75% of shark activity is BUY
    "vn30_percent_sheep450_buy_sell": 54.77,      # 55% of retail activity is BUY
    "vn30_percent_buy_shark450_sheep": 57.0,      # 57% of buy pressure from shark
    "vn30_percent_sell_shark450_sheep": 34.58,    # 35% of sell pressure from shark
    "vn30_percent_shark900_buy_sell": 79.2,
    "vn30_percent_sheep900_buy_sell": 58.97,
    "vn30_percent_buy_shark900_sheep": 35.36,
    "vn30_percent_sell_shark900_sheep": 17.11,
    
    # Accumulated Percent (from session start)
    "vn30_accum_percent_shark450_buy_sell": 75.23,
    "vn30_accum_percent_sheep450_buy_sell": 54.79,
    "vn30_accum_percent_buy_shark450_sheep": 56.85,
    "vn30_accum_percent_sell_shark450_sheep": 34.45,
    "vn30_accum_percent_shark900_buy_sell": 79.2,
    "vn30_accum_percent_sheep900_buy_sell": 58.96,
    "vn30_accum_percent_buy_shark900_sheep": 35.26,
    "vn30_accum_percent_sell_shark900_sheep": 17.04,
    
    # Urgency Spread (Market Cap Weighted, %)
    "vn30_shark450_urgency_spread": 0.17,   # Shark buying 0.17% higher than selling
    "vn30_shark900_urgency_spread": 0.1177,
}
```

### Feature Interpretation for AI Model

| Feature | Value | Interpretation |
|---------|-------|----------------|
| `vn30_percent_shark450_buy_sell = 75%` | Bullish | 75% of institutional money is buying |
| `vn30_percent_buy_shark450_sheep = 57%` | Strong | Institutions dominate buy side |
| `vn30_shark450_urgency_spread = +0.17%` | Mild Bullish | Sharks willing to pay slightly higher |
| `vn30_shark450_buy_ratio_5d_pc = 0.29` | Above Average | Shark buying 29% of avg 5-day volume |

---

## Usage

### Running the Pipeline

```python
from metan.stock.testbed.calculate_vn30_aggregate import run_full_pipeline

# Full pipeline: calculate for each symbol + aggregate
df = run_full_pipeline(
    start_date="2023-09-20",
    end_date="2023-12-20",
    skip_symbol_calculation=False,  # Set True if features already calculated
)
```

### Using in AI Model

```python
from metan.stock.trading.domain.feature.aggregator.vn30 import VN30WhaleFootprintAggregator

# Get VN30 features for model input
aggregator = VN30WhaleFootprintAggregator(
    start_date="2025-01-02",
    end_date="2025-01-03",
)
features_df = aggregator.calculate()

# Features ready for ML model
X = features_df[[
    "vn30_shark450_buy_ratio_5d_pc",
    "vn30_percent_shark450_buy_sell",
    "vn30_shark450_urgency_spread",
    # ... other features
]]
```

---

## Next Steps for AI Model

1. [ ] Combine with VN30F1M price data as target variable
2. [ ] Add time-based features (time of day, day of week)
3. [ ] Add technical indicators on VN30 index
4. [ ] Feature selection / importance analysis
5. [ ] Train prediction model (classification or regression)
6. [ ] Backtest trading strategy

