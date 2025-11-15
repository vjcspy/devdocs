# Metan Stock Package Overview

**TL;DR**: Provides Supabase-backed ingestion of Vietnamese equity ticks/prices, normalizes them against TCBS intraday candles, and produces per-candle "whale footprint" features that get persisted back into Supabase for downstream traders and bots.

## Table of Contents
- [Repo Purpose & Interactions](#repo-purpose--interactions)
- [Inventory](#inventory)
- [Data & Integration Map](#data--integration-map)
- [Key Logic](#key-logic)
- [External Dependencies & Cross-Service Contracts](#external-dependencies--cross-service-contracts)

## Repo Purpose & Interactions
The `packages/stock` module is the shared data/feature layer for any AI agent that needs reliable intra-day Vietnamese stock information. It pulls authoritative tick and daily price history from Supabase, fetches interval candles from the third-party TCBS API, replays ticks into aligned `TickCandle`s, computes feature packs (currently "whale footprint"), and upserts those enriched candles back into Supabase. Other workspace packages (CLI/services) call into this module instead of duplicating ingestion logic.

| System | Direction | Protocol & Endpoint | Auth | Timeout/Retry | Entry Points |
| --- | --- | --- | --- | --- | --- |
| Supabase – `stock` table | Read | Supabase Python client → PostgREST `table("stock").select("*")` | `APP_SUPABASE_URL` + `APP_SUPABASE_KEY` (via `metan.supabase`) | Default client timeout, no retries configured | `StockDataCollector.stock()` fetches symbol metadata |
| Supabase – `stock_info_ticks` | Read | `.table("stock_info_ticks").select().eq().gte().lte()` | Same as above | Default timeout, no retries | `StockDataCollector.ticks()` hydrates `Tick` + `TickAction` |
| Supabase – `stock_info_prices` | Read | `.table("stock_info_prices").select()` (count + fetch) | Same | Default timeout, no retries | `StockDataCollector.prices()` loads OHLCV rows |
| Supabase – `stock_info_stocks` | Read | `.table("stock_info_stocks")` | Same | Default timeout | `tests/.../test_candles_by_date.py` validates expected buckets using exchange info |
| Supabase – `stock_trading_feature_candles` | Write | `.table(...).upsert(chunk, on_conflict="symbol,time")` | Same | Default timeout, chunked batches of 500, no retry/backoff | `IntradaySymbolFeaturePersistor._persist_rows()` persists feature JSON |
| TCBS API `https://apiextaws.tcbs.com.vn/stock-insight/v2/stock/bars` | Read | HTTPS GET `ticker`, `type=stock`, `resolution`, `to`, `countBack=300` | Bearer `tcbs_token` from env via `stock_info_config` | `requests.get(..., timeout=30)`, no retry; manual pagination using `to` cursor | `TcbsSymbolCandleFetcher.fetch()` builds price candles that define per-day intraday slots |

## Inventory
```text
packages/stock/
├── pyproject.toml                 # Hatch build metadata; declares only the minimal runtime deps (metan-supabase, pendulum)
├── Makefile & scripts/*.sh        # Thin wrappers that proxy to Poetry for install/lint/test/format
├── metan/stock/
│   ├── info/
│   │   ├── configuration.py      # Loads the TCBS bearer token via BaseEnvSettings
│   │   └── domain/
│   │       ├── candle/models.py  # IntradayInterval enum, TickCandle & PriceCandle schemas
│   │       ├── price/models.py   # Daily OHLCV (Price) model
│   │       ├── stock/models.py   # Reference stock metadata schema
│   │       ├── tick/models.py    # TickAction + Tick models (side B/S/Undefined)
│   │       └── stock_data_collector/
│   │           ├── stock_data_collector.py
│   │           │   # Core data access layer: fetches Supabase ticks/prices/stock + TCBS candles, caches results, builds TickCandle sets
│   │           ├── constants.py  # Expected 5-minute candle counts per exchange
│   │           ├── abstract.py   # CandleFetcher base class (interval mapping, token helpers)
│   │           └── external/tcbs/
│   │               ├── tcbs_symbol_candle_fetcher.py  # Calls TCBS bars API and sanitizes timestamps
│   │               └── tcbs_contract_candle_fetcher.py # Placeholder for derivative contracts (not implemented)
│   ├── trading/
│   │   └── domain/feature/
│   │       ├── calculator/base_feature_calculator.py # Base pandas pipeline contract
│   │       ├── calculator/common/base.py             # Shared aggregation helpers
│   │       ├── calculator/whale_footprint/*.py       # Value, avg price, ratio, urgency feature blocks
│   │       ├── models.py                             # `FeatureBaseCandleRow`
│   │       └── persistor/intraday/intraday_symbol_feature_persistor.py # Runs calculators + upserts into Supabase
│   └── testbed/compare_candle.py    # Manual script to diff TCBS candle timestamps across symbols
├── tests/metan/stock/info/domain/stock_data_collector/test_candles_by_date.py
│       # Live integration tests hitting Supabase + TCBS to assert candle ordering/counts
└── doc/.gitkeep                    # Placeholder for future docs
```

## Data & Integration Map
| Entity | Source | Fields & Shape | Relationships / Notes |
| --- | --- | --- | --- |
| `Stock` (`stock/models.py`) | Supabase `stock` | `code`, `exchange`, industry codes, historic metadata | Used to determine exchange-specific expectations (e.g., expected candle counts) |
| `Price` (`price/models.py`) | Supabase `stock_info_prices` | Daily OHLC, `volume`, `value`, foreign flow metrics | Sorted chronologically; currently fetched up to `end_date` with five extra historical days |
| `Tick` & `TickAction` (`tick/models.py`) | Supabase `stock_info_ticks` | `meta` array of `[timestamp, volume, price, side]` entries; `side` filtered to `B/S` in practice | Feed for building intraday tick buckets and classifying whales vs sheep |
| `PriceCandle` (`candle/models.py`) | TCBS API | 5-minute or hourly OHLCV per `time` (ISO) | Defines authoritative intraday slots per trading day; sanitized to skip lunch and after-hours trades |
| `TickCandle` (`candle/models.py`) | Derived | Combines aggregated tick actions with aligned TCBS time slots; stores `value` already scaled to millions | Downstream calculators treat this as the base OHLCV per interval |
| `FeatureBaseCandleRow` (`trading/.../models.py`) | Derived | Flattened `TickCandle` + `date/time` | Carrier row before feature namespaces are merged and persisted |

**Data flow**
1. `StockDataCollector` pulls Supabase ticks (per-day buckets keyed by `date`), optional daily prices, and TCBS interval candles for the same window/interval. All results are cached per symbol+date window to avoid repeated requests during feature runs.
2. `tick_candles_by_date()` replays tick actions into `interval` buckets, computes OHLC + traded value per bucket, and enforces that every TCBS slot has a corresponding tick aggregate (hard fail if data is missing).
3. Feature calculators consume those grouped candles to compute analytics (volume/value splits above specific VND thresholds, rolling baselines, etc.).
4. `IntradaySymbolFeaturePersistor` merges per-candle features into JSON namespaces and upserts them into Supabase so other services can stream/query them.

## Key Logic
### StockDataCollector (`metan/stock/info/domain/stock_data_collector/stock_data_collector.py`)
- Central cache (`_cached_data`) keyed by symbol/date/interval to avoid redundant Supabase/TCBS calls during the same Python process.
- `stock()` reads a single row from Supabase `stock`, validates it with `Stock.model_validate`, and stores it in cache.
- `ticks()` loads `stock_info_ticks` rows in the requested range, filters `TickAction` to buy/sell sides, converts epoch seconds to ISO strings (UTC), and sorts by trade date while logging aggregate stats along the way.
- `prices()` first counts rows `>= start_date`, then fetches up to `end_date` plus five extra historic days (to ensure rolling baselines) and converts each dict into a `Price`. Sorting is ascending before caching.
- `price_candles_by_date()` instantiates `TcbsSymbolCandleFetcher`, groups returned `PriceCandle`s by trading date, sorts them, and rejects missing `time` values.
- `tick_candles_by_date()` pulls both ticks and price candles, groups tick actions per day, buckets them into fixed `IntradayInterval`s, computes OHLC/volume/value for each bucket, and ensures one-to-one alignment with the TCBS price slots. Any missing tick buckets result in a raised error so downstream analytics never silently diverge from price data.

### TCBS integration (`metan/stock/info/domain/stock_data_collector/external/tcbs/tcbs_symbol_candle_fetcher.py`)
- `fetch()` paginates backwards using the `to` UNIX timestamp and `countBack=300`, deduplicates by `tradingDate`, and filters out candles at 11:30 and at/after 14:30 local time (to avoid lunch/after-hours anomalies).
- Builds `PriceCandle` Pydantic objects, raising `ValueError` when fields are missing/invalid.
- `get_page()` issues the actual HTTP GET with a 30s timeout and the TCBS bearer token pulled from `stock_info_config`. There is no retry/backoff layer, so upstream throttling will raise immediately.

### WhaleFootprintFeatureCalculator (`metan/stock/trading/domain/feature/calculator/whale_footprint/whale_footprint_feature_calculator.py`)
- Inherits `BaseFeatureCalculator`, defaulting to thresholds of 450 M and 900 M VND per trade (customizable via constructor argument).
- `gather_daily_value_and_candle_counts_from_ticks()` + `validate_and_get_base_candle_count_strict()` enforce that each trading day contains the same number of candles; mismatches abort the calculation.
- `compute_per_candle_baselines()` derives 
  - `pc_value_today`: average per-candle traded value for the day, and 
  - `pc_value_5d`: rolling average of the previous five days' per-candle value (must have 5 prior days).
- `_build_rows()` iterates each candle, aggregates tick actions via `aggregate_single_candle()` into shark/sheep buckets, tracks cumulative weighted value to calculate running average buy/sell prices, computes ratios against `pc_value_5d`, and adds a `shark_urgency_spread` derived from the buy/sell average delta normalized by candle VWAP.
- Returns a pandas `DataFrame` indexed by ISO `time`, ready to merge into persistence rows.

### IntradaySymbolFeaturePersistor (`metan/stock/trading/domain/feature/persistor/intraday/intraday_symbol_feature_persistor.py`)
- Accepts `symbol`, `start_date`, `end_date`, `IntradayInterval` and reuses a single `StockDataCollector` for base candles + calculators.
- `_build_base_candles()` flattens TickCandles (already containing OHLCV/value) into dataclasses sorted by time.
- `_features` currently contains only `WhaleFootprintFeatureCalculator`, but the list is designed to be extended with additional `BaseFeatureCalculator` subclasses.
- `_infer_feature_frame()` normalizes each calculator's `DataFrame` onto the `time` index and drops non-feature columns, enabling consistent merges.
- `_merge_features_into_rows()` injects `{namespace: feature_map}` JSON per candle; missing namespace data for any time raises immediately to keep persisted rows consistent.
- `_persist_rows()` batches rows (default 500) and upserts into Supabase `stock_trading_feature_candles` via `on_conflict="symbol,time"`. Errors are logged and the persistor keeps processing remaining chunks.

## External Dependencies & Cross-Service Contracts
| Dependency | Scope | Purpose | Notes |
| --- | --- | --- | --- |
| `metan-core` | Shared | Supplies the `Logger`, environment loader (`BaseEnvSettings`), and common utilities leveraged by configuration + logging calls | Lives in `packages/core`; no direct import cycles with `stock` |
| `metan-supabase` | Shared | Wraps Supabase client creation/config via env vars (`APP_SUPABASE_URL`, `APP_SUPABASE_KEY`) | Used wherever `.table(...)` is called |
| `pendulum` | Runtime | Timezone-aware parsing, timestamp math, ISO formatting | Heavy use inside `StockDataCollector` + fetchers; ensures Asia/Ho_Chi_Minh offsets are applied |
| `requests` | Runtime | TCBS HTTP client | **Not declared** in `packages/stock/pyproject.toml`; pulled transitively from the workspace root today |
| `pandas` | Runtime | Tabular feature computation, DataFrame merging/persistence | Imported across calculator modules but also missing from the package's direct dependency list |
| `rich` | Runtime | Logger console output for the testbed comparison script | Provided by `metan-core`'s logger stack |
| `pytest` | Dev | Executes the integration tests under `tests/` | Requires network + Supabase/TCBS credentials |
