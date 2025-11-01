# Stock Web Overview

Purpose: Focused guidance for the `projects/stock/apps/web` app — how we structure widgets, charts, and UI, and conventions to extend safely.

## Project Structure (Stock Web)

- App root: `projects/stock/apps/web/`
- Pages (Next.js, pages router):
  - `src/pages/_app.tsx` — global CSS imports for grid/resize
  - `src/pages/_document.tsx` — HTML shell
  - `src/pages/index.tsx` — landing page
  - `src/pages/dashboard.tsx` — dashboard grid (RGL) with Cards and charts
- Styles: `src/styles/globals.css` — Tailwind base/theme and app styles
- Dev server: `pnpm --filter @stock/apps-web dev` → `http://localhost:3000` (`/dashboard` main view)

## Folders and Purpose

- `src/components/ui/` — shadcn UI primitives only (e.g., `card.tsx`, `button.tsx`).
  - Avoid putting components with business or data logic here.
- `src/components/chart/` — presentational chart components (pure render, no fetch/persistence).
  - Example: `CandleFeatureLineChart.tsx` renders line chart from provided props.
- `src/components/dashboard/` — container widgets that fetch, map data, and wire UI.
  - Example: `StockCandleFeatureChart.tsx` handles filters, Supabase queries, persistence, then passes props to chart components.

## Dashboard Pattern

- Layout: `react-grid-layout` responsive grid, imported dynamically with SSR disabled.
- Card: shadcn `Card` with draggable header (`.rgl-drag-handle`) and interactive content in `CardContent`.
- Charts: `react-chartjs-2` atop Chart.js; maintain responsive sizing using flex and `maintainAspectRatio: false`.

## Conventions & Best Practices

- Use `es-toolkit` for utilities like `debounce`; avoid hand-rolled timing logic.
- Disable zoom/pan for intraday charts unless explicitly required; remove reset buttons where not needed.
- Persistence: use `src/lib/user-config.ts` with `getUserConfig`, `getUserConfigKey`, `putUserConfig(key, data)` to store `user_config` in `localStorage`.
- SSR safety: access `window/localStorage` only inside client effects.
- Dragging vs interaction: apply drag handle on headers; cancel drag on interactive elements.

## Extending the Dashboard

- Add a widget:
  - Define grid layout items per breakpoint in `dashboard.tsx`.
  - Create a container in `src/components/dashboard/` with filters and data fetching.
  - Render a presentational chart/table from `src/components/chart/` in `CardContent`.
- Optional: persist grid layout to `localStorage` for user customization.