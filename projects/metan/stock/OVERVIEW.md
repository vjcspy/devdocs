# Overview
>
> **Branch:** master  
> **Last Commit:** 6fc4cba  
> **Last Updated:** Tue Dec 30 11:29:07 2025 +0700

## Stock Package Overview (TL;DR)

- **Heart of the package**: `StockDataCollector` (`packages/stock/metan/stock/info/domain/stock_data_collector/stock_data_collector.py`) - central data loader providing all foundational data for feature computation.
- Primary data source for features: `tick_candles_by_date()` → candles built from actual ticks, containing `tick_actions` for shark/sheep trade classification.
- Data sources: Supabase (ticks, prices, stock metadata) and TCBS REST (intraday candles - limited).
- Core flows: `info` (prepare market data) → `trading` (compute features) → Supabase persistence.
- **NEW (Dec 2025)**: VN30 Index Calculator & VN30 Whale Footprint Aggregator for VN30F1M AI prediction.

## Recent Changes Log

Since `15a8728` → `6fc4cba`:

- **Added `info/domain/index/` module**: New `TcbsVN30IndexCalculator` calculates VN30 index using market cap weighted methodology with free-float ratios.
- **Added `trading/domain/feature/aggregator/vn30/` module**: New `VN30WhaleFootprintAggregator` aggregates whale footprint features from 30 VN30 stocks into index-level features.
- **Added `common/utils/time_utils.py`**: Time normalization utility (`normalize_iso8601`).
- **Refactored `IntradayTimepointsService`**: Moved to `info/domain/candle/` for better organization.
- **Added intermediate values feature**: New `intermediate_values.py` computes `pc_value_5d`, `close_price`, and accumulated values required for VN30 aggregation.
- **Enhanced `StockDataCollector`**: Added `stock()` method to fetch stock metadata including `total_shares`.

## Repo Purpose & Bounded Context

- Part of `metan-workspace` (see `pyproject.toml` workspace members), this package (`metan-stock`) delivers stock-specific data collection and feature generation used by trading analytics and derivative products (e.g., VN30F1M aggregation described in feature doc).
- Bounded to intraday equities data (Vietnam exchanges HSX/HNX) and downstream feature persistence; relies on other workspace packages for logging (`metan-core`) and database connectivity (`metan-supabase`).

## Project Structure

```text
packages/stock/
├─ pyproject.toml                # package metadata (deps: metan-supabase, pendulum)
├─ metan/stock/
│  ├─ main.py                    # CLI stub
│  ├─ common/
│  │  ├─ helper/config_data.py   # fetch intraday timepoint configs from Supabase
│  │  └─ utils/time_utils.py     # ISO8601 normalization helpers
│  ├─ info/
│  │  ├─ configuration.py        # env settings (tcbs_token)
│  │  ├─ domain/
│  │  │  ├─ candle/
│  │  │  │  ├─ models.py         # IntradayInterval enum, TickCandle, PriceCandle
│  │  │  │  └─ intraday_timepoints_service.py  # builds & persists intraday schedule configs
│  │  │  ├─ index/               # 🆕 VN30 Index calculation
│  │  │  │  ├─ constants.py      # VN30_SYMBOLS, VN30_FREE_FLOAT_RATIOS, DEFAULT_BASE_INDEX
│  │  │  │  ├─ models.py         # VN30IndexCandle, IndexComponent
│  │  │  │  └─ tcbs_vn30_index_calculator.py  # market cap weighted index calculator
│  │  │  ├─ price/models.py      # daily Price schema
│  │  │  ├─ tick/models.py       # Tick & TickAction
│  │  │  ├─ stock/models.py      # Stock metadata model (incl. total_shares)
│  │  │  └─ stock_data_collector/
│  │  │     ├─ abstract.py       # CandleFetcher base with interval/time helpers
│  │  │     ├─ constants.py      # expected candle counts per exchange
│  │  │     ├─ stock_data_collector.py   # central data loader/cacher
│  │  │     └─ external/tcbs/
│  │  │        ├─ tcbs_symbol_candle_fetcher.py    # REST client for TCBS bars endpoint
│  │  │        └─ tcbs_contract_candle_fetcher.py  # placeholder
│  ├─ trading/
│  │  ├─ domain/feature/
│  │  │  ├─ aggregator/          # 🆕 VN30 feature aggregation
│  │  │  │  └─ vn30/
│  │  │  │     └─ vn30_whale_footprint_aggregator.py  # aggregate 30 stocks → VN30-level features
│  │  │  ├─ calculator/
│  │  │  │  ├─ base_feature_calculator.py   # abstract calculator contract
│  │  │  │  ├─ common/base.py               # shared validation & aggregation helpers
│  │  │  │  └─ whale_footprint/
│  │  │  │     ├─ constants.py              # thresholds, categories, sides
│  │  │  │     ├─ features/
│  │  │  │     │  ├─ avg_prices.py          # cumulative avg price tracking
│  │  │  │     │  ├─ intermediate_values.py # 🆕 pc_value_5d, close_price for VN30 agg
│  │  │  │     │  ├─ ratios_5d_pc.py        # value ratios vs 5D baseline
│  │  │  │     │  ├─ shark_sheep_ratios.py  # shark vs sheep percent features
│  │  │  │     │  ├─ shark_values.py        # shark/sheep buy/sell values
│  │  │  │     │  └─ urgency_spread.py      # VWAP urgency spread
│  │  │  │     └─ whale_footprint_feature_calculator.py
│  │  │  ├─ models.py                       # FeatureBaseCandleRow dataclass
│  │  │  └─ persistor/intraday/
│  │  │     └─ intraday_symbol_feature_persistor.py  # upserts to Supabase
│  └─ testbed/
│     ├─ calculate_feature.py               # quick-run feature calc
│     └─ calculate_vn30_aggregate.py        # 🆕 VN30 aggregation pipeline
└─ tests/metan/stock/info/domain/stock_data_collector/test_candles_by_date.py
```

## Core Services & Logic

### 🔴 StockDataCollector - Heart of the Stock Package

> **File:** `packages/stock/metan/stock/info/domain/stock_data_collector/stock_data_collector.py`

`StockDataCollector` is the **central data loader** - providing all foundational data for building features and technical indicators. Every feature calculator depends on the output of this class.

#### Initialization

```python
from metan.stock.info.domain.stock_data_collector.stock_data_collector import StockDataCollector
from metan.stock.info.domain.candle.models import IntradayInterval

collector = StockDataCollector(
    symbol="VNM",
    start_date="2025-01-01",
    end_date="2025-01-10",
    interval=IntradayInterval.FIVE_MINUTES  # 300s or 3600s
)
```

---

### 📊 Method Reference & Return Types

#### 1. `stock()` → `Stock` 🆕

**Purpose:** Retrieve stock metadata from Supabase.

**Return Type:**

```python
class Stock(BaseModel):
    code: str                # Stock symbol (e.g., "VNM")
    exchange: str            # Exchange (e.g., "HSX", "HNX")
    total_shares: int | None # Total outstanding shares
    # ... other metadata fields
```

**Usage:** Essential for VN30 Index calculation (requires `total_shares` for market cap weighting).

---

#### 2. `ticks()` → `list[Tick]`

**Purpose:** Retrieve raw tick data (individual trades) from Supabase.

**Return Type:**

```python
class TickAction(BaseModel):
    time: str                              # ISO 8601 UTC timestamp
    volume: int                            # Trade volume
    price: int                             # Trade price (VND)
    side: Literal["B", "S", "Undefined"]   # Buy/Sell/ATO-ATC

class Tick(BaseModel):
    symbol: str                            # Stock symbol (e.g., "VNM")
    date: str                              # Trading date (YYYY-MM-DD)
    meta: list[TickAction]                 # List of trades for the day
```

**Characteristics:**

- Data stored in Supabase table `stock_info_ticks`
- Only includes trades with `side` as `"B"` (Buy) or `"S"` (Sell)
- Excludes `"Undefined"` trades (ATO/ATC sessions)
- Timestamps normalized to ISO 8601 UTC

**Usage:** Foundational data for building tick candles and money-flow based features.

---

#### 3. `tick_candles_by_date()` → `dict[str, list[TickCandle]]` ⭐ **RECOMMENDED**

**Purpose:** Build candles from actual tick data. This is the primary data source for feature computation.

**Return Type:**

```python
class TickCandle(BaseModel):
    time: str                    # ISO 8601 UTC - candle start time
    tick_actions: list[TickAction]  # Individual trades within this candle
    open: int                    # Opening price
    close: int                   # Closing price
    high: int                    # Highest price
    low: int                     # Lowest price
    volume: int                  # Total volume
    value: int                   # Trade value (unit: MILLION VND)

# Return: dict[date_string, list[TickCandle]]
# Example: {"2025-01-01": [TickCandle(...), ...], "2025-01-02": [...]}
```

**Why use tick candles instead of price candles from external sources?**

1. **Contains `tick_actions`**: Each candle includes individual trades, enabling:
   - Shark/sheep trade classification based on value (`price × volume`)
   - Actual buy/sell money flow calculation
   - Trade direction identification (Buy/Sell)

2. **Data consistency**: All features are built from the same source, ensuring consistency.

3. **Schedule-normalized**: Candle count is normalized per exchange via `get_intraday_timepoints()`.

**Usage Example:**

```python
tick_candles = collector.tick_candles_by_date()
# {"2025-01-01": [TickCandle(...), ...], ...}

for date, candles in tick_candles.items():
    for candle in candles:
        # Analyze individual trades within the candle
        for action in candle.tick_actions:
            trade_value = action.price * action.volume
            is_shark = trade_value >= 450_000_000  # 450 million VND
            is_buy = action.side == "B"
```

---

#### 4. `prices()` → `list[Price]`

**Purpose:** Retrieve daily OHLCV data from Supabase.

**Return Type:**

```python
class Price(BaseModel):
    # Identity
    symbol: str                              # Stock symbol
    date: str                                # Date (YYYY-MM-DD)

    # OHLCV
    open: int                                # Opening price
    close: int                               # Closing price
    high: int                                # Highest price
    low: int                                 # Lowest price
    volume: int                              # Volume
    value: int                               # Trade value

    # Foreign flow (optional)
    average: int | None                      # Average price
    basic: int | None                        # Reference price
    deal_volume: int | None                  # Matched volume
    foreign_buy_qty: int | None              # Foreign buy quantity
    foreign_buy_value: int | None            # Foreign buy value
    foreign_sell_qty: int | None             # Foreign sell quantity
    foreign_sell_value: int | None           # Foreign sell value
    current_foreign_room: int | None         # Remaining foreign room
```

**Characteristics:**

- Data stored in Supabase table `stock_info_prices`
- Includes foreign investor flow information
- Sorted by `date` ascending

**Usage:**

- Fallback for opening price when first bucket has no ticks
- Computing indicators based on daily data (MA, baseline values...)
- Retrieving foreign flow information

---

#### 5. `price_candles_by_date()` → `dict[str, list[PriceCandle]]` ⚠️ **LIMITED USE**

**Purpose:** Retrieve intraday candles from TCBS REST API.

**Return Type:**

```python
class PriceCandle(BaseModel):
    time: str      # ISO 8601 format
    open: int      # Opening price
    close: int     # Closing price
    high: int      # Highest price
    low: int       # Lowest price
    volume: int    # Volume
```

**⚠️ WARNING - Not recommended for general use:**

1. **Limited data**: TCBS API only provides recent data (~30 days)
2. **No tick_actions**: Cannot classify shark/sheep trades due to missing individual trade details
3. **External API dependency**: May fail due to network/rate limiting

**When to use:**

- Quick sanity checks on candle data
- Validation by comparing with tick candles
- VN30 Index calculation (uses close prices for market cap)

---

### 📋 Summary: Which Method to Use?

| Use Case | Method | Reason |
|----------|--------|--------|
| **Build features (Whale, Shark...)** | `tick_candles_by_date()` | Contains `tick_actions` for trade classification |
| **Compute baseline/MA** | `prices()` | Complete daily OHLCV with historical data |
| **Analyze foreign flow** | `prices()` | Contains foreign buy/sell information |
| **Get stock metadata (total_shares)** | `stock()` | Required for market cap calculations |
| **VN30 Index calculation** | `price_candles_by_date()` + `stock()` | Market cap weighted from TCBS prices |
| **Debug/Compare** | `price_candles_by_date()` | Compare with TCBS (limited) |

---

### Caching

`StockDataCollector` caches results by key:

- `stock`: `{symbol}`
- `ticks`: `{symbol}|{start_date}|{end_date}`
- `tick_candles_by_date`: `{symbol}|{start_date}|{end_date}|{interval}`
- `prices`: `{symbol}|{start_date}|{end_date}`
- `price_candles_by_date`: `{symbol}|{start_date}|{end_date}|{interval}`

This reduces database/API calls when requesting the same data multiple times.

---

### 🆕 TcbsVN30IndexCalculator (info.domain.index)

> **File:** `packages/stock/metan/stock/info/domain/index/tcbs_vn30_index_calculator.py`

Calculates VN30 Index using market cap weighted methodology.

**Formula:**

```text
IndexValue = (TotalMarketCap / BaseTotalMarketCap) × BaseIndex
```

Where:

- `TotalMarketCap = Σ(Price_i × TotalShares_i × FreeFloatRatio_i)` for all 30 symbols
- `BaseTotalMarketCap` = TotalMarketCap of the first candle
- `BaseIndex` = 2020.01 (default)

**Usage:**

```python
from metan.stock.info.domain.index.tcbs_vn30_index_calculator import TcbsVN30IndexCalculator

calculator = TcbsVN30IndexCalculator(
    start_date="2025-01-01",
    end_date="2025-01-05",
    use_free_float=True,  # Apply free-float ratios
)
index_candles = calculator.calculate()  # Returns list[VN30IndexCandle]
```

**Return Type:**

```python
class VN30IndexCandle(BaseModel):
    time: str      # ISO 8601 UTC
    open: float    # Index value from open prices
    high: float    # Index value from high prices
    low: float     # Index value from low prices
    close: float   # Index value from close prices
    volume: int    # Total volume of all 30 symbols
    value: int     # Total traded value in millions VND
```

**Constants:**

- `VN30_SYMBOLS`: List of 30 VN30 component symbols
- `VN30_FREE_FLOAT_RATIOS`: Free-float ratio for each symbol (default 1.0)
- `DEFAULT_BASE_INDEX`: 2020.01

---

### 🆕 VN30WhaleFootprintAggregator (trading.domain.feature.aggregator.vn30)

> **File:** `packages/stock/metan/stock/trading/domain/feature/aggregator/vn30/vn30_whale_footprint_aggregator.py`

Aggregates whale footprint features from all 30 VN30 stocks into index-level features for AI model prediction of VN30F1M.

**Aggregation Methods:**

- **Value features**: Simple Sum across all stocks
- **Ratio_5d_pc**: `Sum(value) / Sum(pc_5d)`
- **Percent features**: Computed from aggregated values
- **Urgency spread**: Market Cap Weighted Average

**Usage:**

```python
from metan.stock.trading.domain.feature.aggregator.vn30 import VN30WhaleFootprintAggregator

aggregator = VN30WhaleFootprintAggregator(
    start_date="2025-01-01",
    end_date="2025-01-05",
)
df = aggregator.calculate()  # Returns pandas DataFrame
```

**Output Features:**

- `vn30_shark{450,900}_{buy,sell}_value`: Sum of values
- `vn30_shark{450,900}_{buy,sell}_ratio_5d_pc`: Sum/Sum ratio
- `vn30_percent_shark{450,900}_buy_sell`: Computed from aggregated values
- `vn30_accum_percent_*`: Accumulated percent features
- `vn30_shark{450,900}_urgency_spread`: Market cap weighted average

**Pipeline:**

```python
from metan.stock.testbed.calculate_vn30_aggregate import run_full_pipeline

# Run full pipeline: calculate features → aggregate
df = run_full_pipeline(
    start_date="2025-01-01",
    end_date="2025-01-05",
    skip_symbol_calculation=False,  # Set True to reuse existing features
)
```

---

### IntradayTimepointsService (info.domain.candle)

> **File:** `packages/stock/metan/stock/info/domain/candle/intraday_timepoints_service.py`

- Builds trading session timepoints for an exchange/interval by fetching TCBS candles for a symbol/date and extracting HH:MM values.
- Persists `{key: INTRADAY_CANDLE_TIMEPOINTS_<EXCHANGE>_<INTERVAL>, value: {...}}` into Supabase `stock_common_configuration` via upsert.

---

### Feature Calculators (trading.domain.feature)

- `WhaleFootprintFeatureCalculator`: consumes tick candles + price baselines; classifies trades into shark/sheep across thresholds (default 450/900 million VND), tracks cumulative avg prices, computes per-candle value ratios vs 5D baseline, and urgency spreads using VWAP.
- **NEW**: `intermediate_values.py` computes intermediate values for VN30 aggregation:
  - `pc_value_5d`: baseline per-candle value for ratio_5d_pc calculation
  - `close_price`: for market cap weight calculation in urgency_spread aggregation
  - `accum_*_value`: accumulated values for accum_percent calculation

- Shared helpers enforce:
  - Consistent candle counts per day (`validate_and_get_base_candle_count_strict`).
  - Day-set equality between prices and tick candles.
  - Monetary units in millions; prices/volumes in raw units.

---

### Persistence (trading.domain.feature.persistor.intraday_symbol_feature_persistor)

- Builds base candle rows from `tick_candles_by_date`.
- Runs feature calculators (currently only Whale Footprint) and merges namespace-scoped feature frames.
- Upserts to Supabase table `stock_trading_feature_candles` with unique `(symbol, time)` constraint; logs written row count.

---

## Key Notes

### 1. Shark/Sheep Classification by Threshold

**Input**: Trade value (in raw units)

```python
trade_value_raw = price × volume  # unit: VND
```

**Classification Logic** (per threshold T):

- T is defined in **millions** (e.g., 450 = 450 million VND)
- Comparison: `trade_value_raw >= T * 1_000_000`
  - ✅ → **shark**: Large trade (institutional investor)
  - ❌ → **sheep**: Small trade (retail investor)

**Default Thresholds**: `[450, 900]` (450M and 900M VND)

### 2. Sides (Trade Direction)

From `TickAction.side`:

- `'B'` (Buy): BUY order
- `'S'` (Sell): SELL order
- `'Undefined'`: ATO/ATC session (NOT counted in whale footprint)

### 3. Point-in-Time vs Accumulative vs Moving-Window

**Naming Convention in Code**:

| Type              | Prefix     | Example                    | Description                                      |
| ----------------- | ---------- | -------------------------- | ------------------------------------------------ |
| **Point-in-time** | _(none)_   | `high`, `low`, `close`     | Value at a specific moment within the candle     |
| **Accumulative**  | `accum_`   | `accum_shark450_buy_value` | Cumulative sum over a period (e.g., intraday)    |
| **Moving-window** | `mov_{N}_` | `mov_15_shark_ratio`       | Rolling average over N periods                   |

**In WhaleFootprintFeatureCalculator Phase 1**:

- Current features are **point-in-time** (per candle)
- Average prices are tracked **cumulatively** within the day

### 4. Monetary Units - IMPORTANT ⚠️

**All monetary values (value) in the application use MILLION (millions) as the unit**

```python
# ✅ CORRECT - Flow in code
trade_value_raw = price × volume          # raw units (VND)
threshold_scaled = 450 * 1_000_000        # scale threshold to raw
is_shark = trade_value_raw >= threshold_scaled
value_in_millions = trade_value_raw / 1_000_000  # convert to millions

# 📊 Output
"shark450_buy_value": 1250  # = 1,250 million VND = 1.25 billion VND
```

**Rationale**:

- Prevents overflow when working with large numbers
- Easier to read and understand in reports
- Consistency across the entire application

### 5. VN30 Free-Float Ratios 🆕

For VN30 Index calculation, free-float ratios are applied to each component:

```python
effective_shares = total_shares × free_float_ratio
market_cap = close_price × effective_shares
```

Ratios range from 0.04 (BID, BCM, GVR) to 1.00 (DGC, LPB, STB). Default is 1.0 for unknown symbols.

## External Dependencies & Cross-Service Contracts

### Supabase (metan.supabase.client)

- Tables used:
  - `stock_info_stocks` (stock metadata incl. `exchange`, `total_shares`).
  - `stock_info_prices` (daily OHLC + foreign flow; fields priceOpen/priceClose/... mapped to `Price`).
  - `stock_info_ticks` (intraday ticks; `meta` lists [ts, volume, price, side]).
  - `stock_common_configuration` (intraday timepoint configs; keys `INTRADAY_CANDLE_TIMEPOINTS_*`).
  - `stock_trading_feature_candles` (feature persistence upsert target).
- Operations: select/order/limit with filters; upsert with `on_conflict` keys for configuration and feature rows.

### TCBS REST (apiextaws.tcbs.com.vn/stock-insight/v2/stock/bars)

- Used by `TcbsSymbolCandleFetcher`; paginated pulls with `resolution` derived from `IntradayInterval` (5m/60m).
- Auth via `StockInfoConfiguration.tcbs_token` (Bearer); filters out post-close trades (>= 14:30) and midday 11:30 artifacts.
- Also leveraged by `TcbsVN30IndexCalculator` for intraday price candles across 30 VN30 symbols.
- Also leveraged indirectly by `IntradayTimepointsService` to derive trading slot schedules.

### Workspace Dependencies

- `metan-core`: logging (`Logger`), environment settings base class.
- `metan-supabase`: provides configured `supabase` client shared across helpers and collectors.
- `pendulum`, `pandas`, `requests`, `pydantic`: time handling, DataFrame features, HTTP, and typed models.
