# Overview
>
> **Branch:** projects/cli  
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

### Testbed Scripts

- `testbed/calculate_feature.py`: quick-run whale footprint calculation for a hardcoded symbol/date.
- `testbed/compare_candle.py`: compares TCBS candle time grids between two symbols to spot schedule gaps.

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

## Observability & Error Handling

- Collector and feature layers use `Logger` for info/error events and intentionally fail fast on missing data (e.g., mismatched day sets, missing candles, invalid config payloads).
- Cache dictionaries in `StockDataCollector` reduce repeated Supabase/TCBS traffic but are cleared manually in tests.

## Testing Notes

- `tests/metan/stock/info/domain/stock_data_collector/test_candles_by_date.py` hits live Supabase + TCBS to verify:
  - TCBS candles return sorted dates/times within requested window.
  - Candle count matches exchange-specific expectations (`StockInfoContains` constants).
- No isolated unit fakes; tests assume credentials/connectivity are available.

## Next Steps for Onboarding

- Ensure `stock_info_config.tcbs_token` is set in environment for TCBS access.
- Seed `stock_common_configuration` with intraday timepoints via `IntradayTimepointsGenerator` before running feature calculations.
- When extending features, follow naming/unit conventions from Whale Footprint and reuse `BaseFeatureCalculator` + helpers to enforce data integrity.
