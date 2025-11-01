# 📋 [251101: Candle Feature Chart]

## 🎯 Objective
> Render intraday candle features as a responsive line chart with dual Y-axes, persist user filters to localStorage, and keep the UI minimal and fast.

## 🔄 Implementation Plan

- Fetch intraday data from `stock_trading_feature_candles` by `symbol` and `date`.
- Present chart in `components/chart/CandleFeatureLineChart.tsx` (pure presentational).
- Container `components/dashboard/StockCandleFeatureChart.tsx` handles filters, fetch, and persistence.
- Axis rules: datasets with id ending `_price` or `close` → `y` (left); others → `y1` (right).
- Disable zoom/pan entirely; remove reset button.
- Debounce refresh using `es-toolkit/debounce` at 500ms for `symbol`, `date`, `features` changes.
- User config persistence (`localStorage` → `user_config.dashboard.stockCandleFeatureChart`).
  - Utility: `src/lib/user-config.ts` with `getUserConfig`, `getUserConfigKey`, `putUserConfig(key, data)` that merges into nested path.

## 📊 Summary of Results

- Dual Y-axes implemented in `CandleFeatureLineChart` and dataset assignment.
- Intraday line chart (labels `HH:MM`) renders close + selected feature series.
- Zoom/pan disabled; card UI streamlined.
- Filters (`symbol`, `date`, `features`) persisted and restored via `user_config`.
- Debounced refresh via `es-toolkit`, removed `pendingSymbol` state.

## 🚧 Outstanding Issues & Follow-up

- Optional: axis tick format customization per unit (price vs value/ratio).
- Optional: replace plain `<input>` with shadcn `Input` once available locally.

## 🔮 Future Improvements

- Add color presets per feature group.
- Add feature search for long lists (shadcn `command`).
- Persist grid layout positions per breakpoint.