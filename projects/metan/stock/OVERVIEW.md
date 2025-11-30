# Overview
>
> **Branch:** master  
> **Last Commit:** 15a8728  
> **Last Updated:** Tue Nov 18 22:39:24 2025 +0700

## Stock Package Overview (TL;DR)

- Stock workspace focuses on ingesting intraday ticks/prices, normalizing them into candle shapes, and computing trading features (currently Whale Footprint) for persistence to Supabase.
- Data comes from two places: TCBS REST for raw candles and Supabase tables for ticks, prices, stock metadata, and configuration.
- Core flows live in `packages/stock/metan/stock`: `info` prepares market data, `trading` turns that data into features, and `testbed` offers quick scripts to sanity-check outputs.

## Repo Purpose & Bounded Context

- Part of `metan-workspace` (see `pyproject.toml` workspace members), this package (`metan-stock`) delivers stock-specific data collection and feature generation used by trading analytics and derivative products (e.g., VN30F1M aggregation described in feature doc).
- Bounded to intraday equities data (Vietnam exchanges HSX/HNX) and downstream feature persistence; relies on other workspace packages for logging (`metan-core`) and database connectivity (`metan-supabase`).

## Project Structure

```
packages/stock/
├─ pyproject.toml                # package metadata (deps: metan-supabase, pendulum)
├─ metan/stock/
│  ├─ main.py                    # CLI stub
│  ├─ common/helper/config_data.py           # fetch intraday timepoint configs from Supabase
│  ├─ info/
│  │  ├─ configuration.py        # env settings (tcbs_token)
│  │  ├─ helper/intraday_timepoints_generator.py   # builds & persists intraday schedule configs via TCBS+Supabase
│  │  ├─ domain/
│  │  │  ├─ candle/models.py     # IntradayInterval enum, TickCandle, PriceCandle
│  │  │  ├─ price/models.py      # daily Price schema
│  │  │  ├─ tick/models.py       # Tick & TickAction
│  │  │  ├─ stock/models.py      # Stock metadata model
│  │  │  └─ stock_data_collector/
│  │  │     ├─ abstract.py       # CandleFetcher base with interval/time helpers
│  │  │     ├─ constants.py      # expected candle counts per exchange
│  │  │     ├─ stock_data_collector.py   # central data loader/cacher
│  │  │     └─ external/tcbs/...
│  │  │        ├─ tcbs_symbol_candle_fetcher.py    # REST client for TCBS bars endpoint
│  │  │        └─ tcbs_contract_candle_fetcher.py  # placeholder
│  ├─ trading/
│  │  ├─ domain/feature/
│  │  │  ├─ calculator/base_feature_calculator.py   # abstract calculator contract
│  │  │  ├─ calculator/common/base.py               # shared validation & aggregation helpers
│  │  │  ├─ calculator/whale_footprint/*.py         # feature logic (values, ratios, urgency, averages)
│  │  │  ├─ models.py                               # FeatureBaseCandleRow dataclass
│  │  │  └─ persistor/intraday/intraday_symbol_feature_persistor.py  # merges features + upserts to Supabase
│  └─ testbed/                                      # quick-run scripts (feature calc, candle compare)
└─ tests/metan/stock/info/domain/stock_data_collector/test_candles_by_date.py
```

## Core Services & Logic

### StockDataCollector (info.domain.stock_data_collector.stock_data_collector)

- Loads per-symbol data between `start_date` and `end_date` at a given `IntradayInterval`.
- Sources:
  - Supabase `stock_info_stocks` → `Stock` model (exchange drives schedule selection).
  - Supabase `stock_info_prices` → `Price` list (includes OHLC and foreign flow fields).
  - Supabase `stock_info_ticks` → per-day `TickAction` lists, filtered to sides B/S, timestamps normalized to ISO UTC.
  - TCBS REST (`TcbsSymbolCandleFetcher`) → intraday `PriceCandle` series, filtered to trading session hours.
- Caches results per symbol/date/interval to reduce database/API calls.
- Produces:
  - `tick_candles_by_date()`: buckets tick actions into OHLCV candles per schedule slot; fails fast if gaps occur.
  - `price_candles_by_date()`: TCBS price candles grouped per trading date with strict time validation.

### IntradayTimepointsGenerator (info.helper.intraday_timepoints_generator)

- Builds trading session timepoints for an exchange/interval by fetching TCBS candles for a symbol/date and extracting HH:MM values.
- Persists `{key: INTRADAY_CANDLE_TIMEPOINTS_<EXCHANGE>_<INTERVAL>, value: {...}}` into Supabase `stock_common_configuration` via upsert.

### Feature Calculators (trading.domain.feature)

- `WhaleFootprintFeatureCalculator`: consumes tick candles + price baselines; classifies trades into shark/sheep across thresholds (default 450/900 million VND), tracks cumulative avg prices, computes per-candle value ratios vs 5D baseline, and urgency spreads using VWAP.
- Shared helpers enforce:
  - Consistent candle counts per day (`validate_and_get_base_candle_count_strict`).
  - Day-set equality between prices and tick candles.
  - Monetary units in millions; prices/volumes in raw units.

### Persistence (trading.domain.feature.persistor.intraday_symbol_feature_persistor)

- Builds base candle rows from `tick_candles_by_date`.
- Runs feature calculators (currently only Whale Footprint) and merges namespace-scoped feature frames.
- Upserts to Supabase table `stock_trading_feature_candles` with unique `(symbol, time)` constraint; logs written row count.

## Key Notes

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

| Loại              | Prefix     | Ví dụ                      | Mô tả                                            |
| ----------------- | ---------- | -------------------------- | ------------------------------------------------ |
| **Point-in-time** | _(none)_   | `high`, `low`, `close`     | Giá trị tại thời điểm trong candle               |
| **Accumulative**  | `accum_`   | `accum_shark450_buy_value` | Cộng dồn trong khoảng thời gian (e.g., intraday) |
| **Moving-window** | `mov_{N}_` | `mov_15_shark_ratio`       | Trung bình trượt N periods                       |

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

## External Dependencies & Cross-Service Contracts

### Supabase (metan.supabase.client)

- Tables used:
  - `stock_info_stocks` (stock metadata incl. `exchange`).
  - `stock_info_prices` (daily OHLC + foreign flow; fields priceOpen/priceClose/... mapped to `Price`).
  - `stock_info_ticks` (intraday ticks; `meta` lists [ts, volume, price, side]).
  - `stock_common_configuration` (intraday timepoint configs; keys `INTRADAY_CANDLE_TIMEPOINTS_*`).
  - `stock_trading_feature_candles` (feature persistence upsert target).
- Operations: select/order/limit with filters; upsert with `on_conflict` keys for configuration and feature rows.

### TCBS REST (apiextaws.tcbs.com.vn/stock-insight/v2/stock/bars)

- Used by `TcbsSymbolCandleFetcher`; paginated pulls with `resolution` derived from `IntradayInterval` (5m/60m).
- Auth via `StockInfoConfiguration.tcbs_token` (Bearer); filters out post-close trades (>= 14:30) and midday 11:30 artifacts.
- Also leveraged indirectly by `IntradayTimepointsGenerator` to derive trading slot schedules.

### Workspace Dependencies

- `metan-core`: logging (`Logger`), environment settings base class.
- `metan-supabase`: provides configured `supabase` client shared across helpers and collectors.
- `pendulum`, `pandas`, `requests`, `pydantic`: time handling, DataFrame features, HTTP, and typed models.

