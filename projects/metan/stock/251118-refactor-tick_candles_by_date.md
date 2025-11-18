# Refactor tick_candles_by_date

## User Requirement
### Description
In `packages/stock/metan/stock/info/domain/stock_data_collector/stock_data_collector.py` class we have a method called `tick_candles_by_date`.
we have been combining by price_candles_map and tick candle in this method to build tick_candles.
Now, I want to change use candle_timepoints from configuration; we don't need to use price_candles_map anymore

### Steps
1. Read carefully and fully `devdocs/agent/TEMPLATE.md` to understand the template.
2. Read carefully and fully `devdocs/projects/metan/stock/OVERVIEW.md` to understand the project.
3. Read carefully and fully `packages/stock/metan/stock/info/domain/stock_data_collector/stock_data_collector.py` to understand the implementation.
4. Read carefully and fully `devdocs/projects/metan/stock/251117-init-intraday-candle-timepoints.md` to understand the previous implementation for getting intraday candle timepoints.
5. Give me a details plan and update to this file. I will review.

## 🎯 Objective
- Refactor `tick_candles_by_date` to use configured intraday candle timepoints (`HH:mm` in UTC) from `stock_common_configuration` instead of TCBS `price_candles_by_date`. Preserve per-day grouping, strict schedule enforcement, OHLC/volume/value computation (value in millions), and caching semantics.

### ⚠️ Key Considerations
- Schedule source: use `INTRADAY_CANDLE_TIMEPOINTS_{EXCHANGE}_{INTERVAL}` produced by `IntradayTimepointsGenerator` (`packages/stock/metan/stock/info/helper/intraday_timepoints_generator.py:7`).
- Exchange derivation: fetch via `stock()` to read `Stock.exchange` and map to `HSX`/`HNX` key segment (`packages/stock/metan/stock/info/domain/stock_data_collector/stock_data_collector.py:39`).
- Time handling: timepoints are `HH:mm` extracted from ISO UTC; construct per-day slot timestamps at UTC. Do not convert timezone.
- Enforcement: every configured slot must have a corresponding tick aggregate; missing buckets raise immediately (current behavior mirrored from TCBS enforcement).
- Units: monetary `value` stored in millions, computed from raw `price × volume` and divided by `1_000_000` (`candle/models.py` conventions; current implementation already follows this).
- Backwards compatibility: keep `price_candles_by_date()` unchanged for other consumers; only `tick_candles_by_date()` stops depending on it.

## 🔄 Implementation Plan
[Don't require running any test]

### Phase 1: Analysis & Preparation
- [ ] Analyze detailed requirements
  - **Outcome**: Replace TCBS-driven schedule with configuration-driven `HH:mm` timepoints keyed by exchange+interval; group ticks per day and align strictly to configured slots; raise on gaps; maintain caches and logs.
- [ ] Define scope and edge cases
  - **Outcome**: Handle missing configuration key, empty `timepoints`, or mismatch between tick buckets and configured slots; ensure deterministic ordering; validate input interval matches configuration; support at least `300s`.

### Phase 2: Implementation (File/Code Structure)
> Status markers: 🚧 To-Do, ✅ Implemented

```
packages/stock/metan/stock/info/
├── domain/stock_data_collector/stock_data_collector.py   # 🚧 Update: fetch config timepoints, refactor tick_candles_by_date
├── helper/intraday_timepoints_generator.py               # ✅ Implemented: produces config records (HH:mm list)
```

- Add cached store for config timepoints: `_cached_data["intraday_timepoints_by_exchange"]` (per `exchange|interval`).
- Move configuration access to `packages/stock/metan/stock/info/configuration.py` as `get_intraday_timepoints(exchange, interval)` with internal cache.
- Refactor `tick_candles_by_date()` (`packages/stock/metan/stock/info/domain/stock_data_collector/stock_data_collector.py:153`):
  - Remove dependency on `price_candles_by_date()` and `price_candles_map` integration.
  - Build per-day `schedule_epoch_map` from configuration timepoints by combining `date + HH:mm` in UTC to epoch seconds.
  - Bucket ticks by interval as today, compute `TickCandle` OHLC/volume/value per bucket.
  - Iterate configured `schedule_epoch_map` and require a corresponding tick candle at each slot; raise with clear error/context if missing.
  - Preserve logging: include `{date}: ticks, buckets, schedule_slots` and detailed missing-slot ISO time in UTC.

### Phase 3: Detailed Implementation Steps
1. Determine `exchange` via `self.stock().exchange`; normalize to `HSX`/`HNX` strings used in configuration keys.
2. Build key: `key = f"INTRADAY_CANDLE_TIMEPOINTS_{exchange}_{int(self.interval)}"`.
3. Fetch configuration: `supabase.table("stock_common_configuration").select("value").eq("key", key).limit(1).execute()`; validate `value.timepoints` is a non-empty list of `HH:mm` strings.
4. Cache timepoints per `exchange|interval` and return sorted unique list.
5. Group `TickAction`s per trading `date` (existing logic).
6. For each `date`, construct `schedule_epoch_map` by parsing `date` at UTC, then for each `HH:mm` create epoch seconds (UTC) and collect into a sorted dict.
7. Build `tick_candles_map` from tick buckets exactly as current implementation: compute `open/close/high/low`, `volume`, and `value = sum(price×volume)/1_000_000`; use bucket floor `epoch // interval * interval` to key.
8. Iterate `schedule_epoch_map` in ascending order; for each epoch, emit `TickCandle` from `tick_candles_map[epoch]`; on missing, log error with ISO UTC and raise `ValueError` to keep schedule authoritative.
9. Store `result[date] = candles`, cache under `"tick_candles_by_date"`, and return.
10. Do not modify `price_candles_by_date()`; leave intact for other pathways.

## 🚧 Outstanding Issues & Follow-up
- Validate the configuration records exist for all relevant exchanges/intervals before feature runs; otherwise, raise error immediately with context including expected key.
- Consider enhancing error message to suggest missing config key and show example key string.

## References
- `packages/stock/metan/stock/info/domain/stock_data_collector/stock_data_collector.py:153` — current `tick_candles_by_date()` implementation and integration points to refactor.
- `packages/stock/metan/stock/info/helper/intraday_timepoints_generator.py:7` — generator that persists `HH:mm` timepoints.
- `devdocs/projects/metan/stock/OVERVIEW.md:145` — describes current TCBS-driven enforcement; new flow mirrors strict schedule using configuration.