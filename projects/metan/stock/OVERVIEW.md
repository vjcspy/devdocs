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
│   │           ├── constants.py  # Expected 5-minute candle counts per exchange (consumed by tests)
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
│   └── testbed/
│       ├── compare_candle.py       # Manual script to diff TCBS candle timestamps across symbols
│       └── calculate_feature.py    # Quick WhaleFootprint runner for manual inspection
├── tests/metan/stock/info/domain/stock_data_collector/test_candles_by_date.py
│       # Live integration tests hitting Supabase + TCBS to assert candle ordering/counts via StockInfoContains constants
└── doc/.gitkeep                    # Placeholder for future docs
```

## Data & Integration Map
| Entity | Source | Fields & Shape | Relationships / Notes |
| --- | --- | --- | --- |
| `Stock` (`stock/models.py`) | Supabase `stock` | `code`, `exchange`, industry codes, historic metadata | Used to determine exchange-specific expectations (e.g., expected candle counts) |
| `Price` (`price/models.py`) | Supabase `stock_info_prices` | Daily OHLC, `volume`, `value`, foreign flow metrics | Sorted chronologically; `_effective_start_date()` counts rows in-range, then fetches through `end_date` plus five extra trading days and maps Supabase `dealVolume` → `volume` |
| `Tick` & `TickAction` (`tick/models.py`) | Supabase `stock_info_ticks` | `meta` array of `[timestamp, volume, price, side]` entries; `side` filtered to `B/S` in practice | Feed for intraday buckets/whale classification; fetch uses the same `_effective_start_date()` window so ticks exist for the five historical days needed for rolling baselines |
| `PriceCandle` (`candle/models.py`) | TCBS API | 5-minute or hourly OHLCV per `time` (ISO) | Defines authoritative intraday slots per trading day; sanitized to skip lunch and after-hours trades |
| `TickCandle` (`candle/models.py`) | Derived | Combines aggregated tick actions with aligned TCBS time slots; stores `value` already scaled to millions | Downstream calculators treat this as the base OHLCV per interval |
| `FeatureBaseCandleRow` (`trading/.../models.py`) | Derived | Flattened `TickCandle` + `date/time` | Carrier row sorted by ISO `time` before feature namespaces are merged and persisted |

**Data flow**
1. `StockDataCollector` pulls Supabase ticks (per-day buckets keyed by `date`), optional daily prices, and TCBS interval candles for the same window/interval, always expanding the request window with `_effective_start_date()` so both ticks and prices include the five prior trading days required by downstream baselines. All results are cached per symbol+date/interval key to avoid repeated requests during feature runs.
2. `tick_candles_by_date()` replays tick actions into `interval` buckets, computes OHLC + traded value per bucket (stored in millions), and enforces the TCBS candle grid as the canonical schedule—every TCBS slot must have a corresponding tick aggregate or the collector raises immediately.
3. Feature calculators consume those grouped candles to compute analytics (volume/value splits above specific VND thresholds, rolling baselines, etc.).
4. `IntradaySymbolFeaturePersistor` merges per-candle features into JSON namespaces and upserts them into Supabase so other services can stream/query them.

## Key Logic
### StockDataCollector (`metan/stock/info/domain/stock_data_collector/stock_data_collector.py`)

> Really important class to fetch and build stock info data from database

- `stock()` reads a single row from Supabase `stock`, validates it with `Stock.model_validate`, and stores it in cache.
- `ticks()` loads `stock_info_ticks` rows between `_effective_start_date()` and `end_date`, filters `TickAction` to buy/sell sides, converts epoch seconds to ISO strings (UTC), and sorts by trade date while logging aggregate stats along the way.
- `prices()` first counts rows `>= start_date`, then fetches up to `end_date` plus five extra historic days (to ensure rolling baselines) and converts each dict into a `Price`. Sorting is ascending before caching.
- `price_candles_by_date()` instantiates `TcbsSymbolCandleFetcher`, groups returned `PriceCandle`s by trading date, sorts them, and rejects missing `time` values.
- `tick_candles_by_date()` pulls both ticks and price candles, groups tick actions per day, buckets them into fixed `IntradayInterval`s, computes OHLC/volume/value (aggregate trade value divided by 1e6) for each bucket, and iterates TCBS price slots in order so the exchange timeline stays authoritative. Any missing tick buckets are logged and raise immediately so downstream analytics never silently diverge from price data

### IntradaySymbolFeaturePersistor (`metan/stock/trading/domain/feature/persistor/intraday/intraday_symbol_feature_persistor.py`)
- Accepts `symbol`, `start_date`, `end_date`, `IntradayInterval` and reuses a single `StockDataCollector` for base candles + calculators.
- `_build_base_candles()` flattens TickCandles (already containing OHLCV/value) into dataclasses sorted by time.
- `_features` currently contains only `WhaleFootprintFeatureCalculator`, but the list is designed to be extended with additional `BaseFeatureCalculator` subclasses.
- `_infer_feature_frame()` normalizes each calculator's `DataFrame` onto the `time` index and drops non-feature columns, enabling consistent merges.
- Each calculator execution lives inside a try/except so a single failing namespace is logged and skipped instead of aborting the entire persistence run.
- `_merge_features_into_rows()` injects `{namespace: feature_map}` JSON per candle; missing namespace data for any time raises immediately to keep persisted rows consistent.
- `_persist_rows()` batches rows (default 500) and upserts into Supabase `stock_trading_feature_candles` via `on_conflict="symbol,time"`. Errors are logged and the persistor keeps processing remaining chunks.
- `persist()` returns a summary dict containing `written`, `candles`, and `namespaces`, which downstream services can use for monitoring or retries.

## External Dependencies & Cross-Service Contracts
| Dependency | Scope | Purpose | Notes |
| --- | --- | --- | --- |
| `metan-core` | Shared | Supplies the `Logger`, environment loader (`BaseEnvSettings`), and common utilities leveraged by configuration + logging calls | Lives in `packages/core`; no direct import cycles with `stock` |
| `metan-supabase` | Shared | Wraps Supabase client creation/config via env vars (`APP_SUPABASE_URL`, `APP_SUPABASE_KEY`) | Used wherever `.table(...)` is called |
| `pendulum` | Runtime | Timezone-aware parsing, timestamp math, ISO formatting | Heavy use inside `StockDataCollector` + fetchers; ensures Asia/Ho_Chi_Minh offsets are applied |
| `pydantic` | Runtime | Data models (`Stock`, `Price`, `Tick`, `TickCandle`) and env settings schemas | **Not declared** in `packages/stock/pyproject.toml`; currently satisfied transitively via other workspace packages |
| `requests` | Runtime | TCBS HTTP client | **Not declared** in `packages/stock/pyproject.toml`; pulled transitively from the workspace root today |
| `pandas` | Runtime | Tabular feature computation, DataFrame merging/persistence | Imported across calculator modules but also missing from the package's direct dependency list |
| `pytest` | Dev | Executes the integration tests under `tests/` | Requires network + Supabase/TCBS credentials |
