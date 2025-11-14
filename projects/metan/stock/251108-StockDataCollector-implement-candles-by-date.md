# 📋 [251108: 2025-11-08] - StockDataCollector Candles By Date

## 🎯 Objective
> Implement `StockDataCollector.candles_by_date` so downstream agents can request TCBS candles grouped by `YYYY-MM-DD`, leveraging the documented fetcher flow in `devdocs/projects/metan/stock/251108-fetch-candle-from-tcbs.md`.

## 🔄 Implementation Plan

### Phase 1: Analysis & Preparation
- [x] Study TCBS fetcher behavior (pagination, filtering, validation).
  - **Outcome**: Confirmed `TcbsSymbolCandleFetcher.fetch()` already returns deduplicated, intraday-sorted `PriceCandle` objects with ISO timestamps suitable for daily grouping.
- [x] Inspect `StockDataCollector` caches and public API (packages/stock/metan/stock/info/domain/stock_data_collector/stock_data_collector.py).
  - **Outcome**: Identified need for a new cache bucket (e.g., `price_candles_by_date`) plus logging patterns that mirror `ticks()`/`tick_candles_by_date()`.
- [x] Clarify response contract and fail-fast policy.
  - **Outcome**: Callers expect `{date -> [PriceCandle,...]}` sorted chronologically, and any anomaly (missing timestamps, parse errors, fetch failures) must raise `Exception` immediately to stop execution—no silent skips.

### Phase 2: Implementation (File/Code Structure)
> Current + planned touch-points with status markers.

````
packages/stock/
└── metan/stock/info/domain/
    ├── candle/models.py                                  # ✅ PriceCandle definition
    ├── stock_data_collector/abstract.py                  # ✅ CandleFetcher helpers (timestamp, resolution, token)
    ├── stock_data_collector/external/tcbs/
    │   └── tcbs_symbol_candle_fetcher.py                 # ✅ Provides TCBS paginated fetch logic
    └── stock_data_collector/stock_data_collector.py      # 🚧 Implement `candles_by_date` + cache + logging + fail-fast guards
````
Supporting references:
- `devdocs/projects/metan/stock/251108-fetch-candle-from-tcbs.md`   # ✅ Canonical fetch behavior
- `packages/cli/metan/cli/app.py` test cmd                         # ✅ Usage example for fetcher instantiation

### Phase 3: Detailed Implementation Steps
1. **Imports & Cache**: Import `TcbsSymbolCandleFetcher` at the top (if not already) and extend `_cached_data` with a `price_candles_by_date` dict keyed by `symbol|start|end|interval`.
2. **Method Skeleton**: In `candles_by_date()`, follow the cache lookup/early-return style used elsewhere; log cache hits/misses for observability.
3. **Fetcher Invocation**: Instantiate `TcbsSymbolCandleFetcher(symbol=self.symbol, start_date=self.start_date, end_date=self.end_date, interval=self.interval)` and call `fetch()`. Wrap exceptions only to add context, then re-raise `Exception` so the caller halts immediately.
4. **Grouping Logic**: Iterate over the returned `PriceCandle` list, parse `candle.time` via `pendulum.parse(...).to_date_string()`, and append to `defaultdict(list)`. If `time` is missing or parsing fails, raise `Exception` (do not skip records).
5. **Sorting & Serialization**: Sort the grouped dict keys ascending; within each date, sort candles by `c.time`. Optionally enforce ints for OHLCV (they already are) but revalidate before caching.
6. **Caching & Return**: Persist the grouped dict in `_cached_data["price_candles_by_date"]`, then return it. Include summary logging (e.g., number of days/candles).
7. **Unit Hooks (Future)**: Outline how to cover via tests once fixtures exist (group ordering, filter coverage) even if immediate test automation is deferred.
8. **Operational Fail-Fast**: Ensure any downstream consumer sees raised `Exception`s for malformed data so finance workflows stop rather than continue with incomplete candles.

## 📊 Summary of Results

### ✅ Completed Achievements
- Captured a step-by-step plan to wire TCBS candles into `StockDataCollector` with fail-fast guarantees.
- Documented dependencies, cache requirements, and grouping/ordering rules expected by consumers.
- Highlighted error-handling expectations to keep the collector resilient and deterministic under TCBS failures.

## 🚧 Outstanding Issues & Follow-up

### ⚠️ Known Issues (Optional)
- [ ] TCBS bearer token (`CandleFetcher.get_token`) is static; rotation strategy still undefined.
- [ ] No automated tests exist for `candles_by_date`; plan references manual verification only.
- [ ] Logging currently relies on plain info/warning lines—structured context (symbol, range, interval) must accompany raised exceptions for observability.

### 🔮 Future Improvements (Optional)
- [ ] Share grouping utility between CLI experimentation and `StockDataCollector` to avoid duplicate `pendulum` parsing.
- [ ] Introduce offline cache/backfill to minimize API pressure when multiple agents request overlapping ranges.
- [ ] Add retry/backoff policy around `TcbsSymbolCandleFetcher.get_page` once instrumentation is available.

---
**Last updated**: 2025-11-08
---
