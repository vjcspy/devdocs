# 📋 [251108: 2025-11-08] - Fetch Candle From TCBS

## 🎯 Objective
> Document how `TcbsSymbolCandleFetcher` ingests historical price candles from TCBS, including pagination, filtering, and authentication so future agents can safely extend or reuse the worker.

## 🔄 Implementation Plan

### Phase 1: Analysis & Preparation
- [x] Analyze detailed requirements
  - **Outcome**: Candle ingestion must respect user-provided `start_date`/`end_date`, deduplicate `tradingDate` records, skip negotiated-trade samples after 14:30 Asia/Ho_Chi_Minh, and produce validated `PriceCandle` models.
- [x] Define scope and edge cases
  - **Outcome**: Handle pagination via the TCBS `to` cursor (Unix timestamp) with `countBack=300`, enforce inclusive date boundaries, guard against missing fields or type coercion errors, and degrade gracefully when the REST call or bearer token fails.

### Phase 2: Implementation (File/Code Structure)
> Status markers show current state per component.

```
packages/stock/
└── metan/stock/info/domain/
    ├── candle/models.py                         # ✅ Defines PriceCandle dataclass used downstream
    ├── stock_data_collector/abstract.py         # ✅ CandleFetcher base: date helpers, resolution + token hooks
    └── stock_data_collector/external/tcbs/
        └── tcbs_symbol_candle_fetcher.py        # ✅ Implements TCBS client (requests + pagination)
```
Supporting scripts:
- `scripts/new-service.sh` / `scripts/new-package.sh` # ✅ Scaffolding if another provider is added.
- `devdocs/projects/metan/stock/*`                   # ✅ Knowledge base for data collectors.

### Phase 3: Detailed Implementation Steps
1. Convert `start_date`/`end_date` strings to Unix timestamps via `_date_to_timestamp`, capturing full-day bounds (start-of-day vs end-of-day).
2. Initialize a `dict` keyed by `tradingDate` to avoid duplicates as TCBS pages overlap.
3. Page backward using `get_page(to_param)`, where `to_param` starts at the end timestamp and is updated to the smallest `tradingDate` seen (`_iso_to_timestamp`).
4. For each returned item:
   - Drop samples outside the requested window.
   - Parse Asia/Ho_Chi_Minh time and skip any candle recorded at/after 14:30 because negotiated trades distort the daily close.
   - Keep the raw TCBS payload in the dict for later normalization.
5. Exit paging when the smallest timestamp in the current batch is older than `from_ts` or when the API returns no data.
6. Sort retained items by `tradingDate`, then instantiate `PriceCandle` objects (int-cast OHLCV fields) inside a guarded block to skip malformed rows while logging via `print`.
7. `get_page` builds the REST request to `https://apiextaws.tcbs.com.vn/stock-insight/v2/stock/bars` with:
   - Headers: user-agent spoofing, `Authorization: Bearer {token}`, `Accept-language: vi`, etc.
   - Params: `{ticker, type=stock, resolution=_get_resolution(), to=<cursor>, countBack=300}`.
   - Error handling: `requests.get(..., timeout=30)` + `raise_for_status`; RequestException falls back to `[]`.

## 📊 Summary of Results

### ✅ Completed Achievements
- Captured end-to-end flow for `TcbsSymbolCandleFetcher`, including pagination cursor logic and market-specific time filtering.
- Documented how `PriceCandle` normalization and `requests` headers interplay with helper methods in the abstract base class.
- Highlighted operational safeguards (duplicate removal, negotiation-trade exclusion, graceful degradation on HTTP failures).

## 🚧 Outstanding Issues & Follow-up

### ⚠️ Known Issues (Optional)
- [ ] Token acquisition via `get_token()` is external to this module; outages or expirations surface only as RequestExceptions and `print` logs.

---
**Last updated**: 2025-11-08
---
