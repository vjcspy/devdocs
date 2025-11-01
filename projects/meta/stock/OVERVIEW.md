# Stock Project Dashboard Overview

Purpose: Provide a concise overview so any developer or AI agent can quickly understand how the dashboard and chart containers are structured, and how to extend them.

## 1) Project Structure

- Monorepo and tooling (root)
  - pnpm workspace: e:/js/meta/pnpm-workspace.yaml
  - Turborepo orchestration: e:/js/meta/turbo.json
- Stock web app (Next.js, pages router)
  - App root: e:/js/meta/projects/stock/apps/web/
  - Pages
    - App entry (global CSS imports for grid/resize): e:/js/meta/projects/stock/apps/web/src/pages/_app.tsx
    - Dashboard page (RGL grid + shadcn Cards + Chart.js): e:/js/meta/projects/stock/apps/web/src/pages/dashboard.tsx
    - Document (HTML shell): e:/js/meta/projects/stock/apps/web/src/pages/_document.tsx
    - Landing page: e:/js/meta/projects/stock/apps/web/src/pages/index.tsx
  - UI (shadcn/ui)
    - Card primitives: e:/js/meta/projects/stock/apps/web/src/components/ui/card.tsx
    - Button: e:/js/meta/projects/stock/apps/web/src/components/ui/button.tsx
  - Styles
    - Tailwind base/theme and app-wide styles: e:/js/meta/projects/stock/apps/web/src/styles/globals.css
- Dev server
  - Start: pnpm --filter @stock/apps-web dev
  - Local: http://localhost:3000 (Dashboard at /dashboard)

## 2) Core Idea (Dashboard + Chart Containers)

- Layout engine
  - Uses react-grid-layout’s Responsive grid wrapped by a WidthProvider for responsive, draggable, and resizable layout.
  - The grid is dynamically imported with SSR disabled to avoid server-side rendering issues.
  - Card headers act as drag handles via a CSS selector (e.g., .rgl-drag-handle), so dragging is intentional while content stays interactive.
  - Drag is canceled on interactive elements (e.g., buttons) so clicks don’t trigger drags.

- Widget/container pattern
  - Each dashboard widget is a shadcn/ui Card:
    - Header: title + actions (e.g., “Reset zoom”), carries the drag handle class.
    - Content: flex container where the chart/table fills available height and width.
  - Cards are resizable; charts auto-resize with the card.

- Charts
  - Rendered with react-chartjs-2 on top of Chart.js.
  - chartjs-plugin-zoom is dynamically imported and registered on the client (inside useEffect) to avoid SSR import/registration errors.
  - For actions like reset zoom, a React ref is kept to access the chart instance and call chartInstance.resetZoom().

- Scrolling and sizing
  - Page container uses min-h-screen w-full overflow-x-hidden overflow-y-auto to allow vertical scrolling when the grid exceeds viewport height.
  - Grid rowHeight and margins are tuned; CardContent and inner wrappers use flex and h-full for proper canvas sizing.

- Extensibility
  - To add a new widget:
    1. Define layout items for each breakpoint (x, y, w, h, i) in the Responsive grid.
    2. Render a new Card with a draggable header (drag handle class) and optional action buttons.
    3. Place a chart/table component inside CardContent that stretches to fill the container.
  - Optional enhancements: layout persistence to localStorage; toolbar to add/remove widgets dynamically; table widgets via @tanstack/react-table.

## 3) Key Dependencies

- Core app
  - Next.js, React, ReactDOM (pages router)
  - Tailwind CSS (configured in globals.css)
- UI
  - shadcn/ui (Card, Button; Table available for data grids)
- Grid and resize
  - react-grid-layout (Responsive grid)
  - react-resizable (resize handles)
  - Their global CSS is imported in _app.tsx
- Charts and tables
  - chart.js (chart engine)
  - react-chartjs-2 (React bindings)
  - chartjs-plugin-zoom (zoom/pan, registered client-side)
  - @tanstack/react-table (advanced table rendering and data modeling)
- Tooling
  - pnpm (monorepo package manager)
  - Turborepo (task orchestration)

## Operational Notes & Best Practices

- SSR safety
  - Import the Responsive grid via dynamic import with ssr: false.
  - Register chartjs-plugin-zoom inside a client-only effect (useEffect) to avoid SSR issues.

- Dragging vs. interaction
  - Put the drag handle class on the Card header; configure draggableCancel for interactive elements (e.g., buttons, inputs).

- Sizing for charts
  - Ensure the grid item and CardContent use flex and h-full so the canvas fills and resizes correctly.
  - Keep a ref to the chart instance when you need imperative actions (e.g., reset zoom).

- Persistence (optional)
  - Persist layouts per breakpoint to localStorage; on mount, restore and feed the layout back into ResponsiveReactGridLayout.

- Where to work
  - Main integration for widgets: e:/js/meta/projects/stock/apps/web/src/pages/dashboard.tsx
  - UI primitives: e:/js/meta/projects/stock/apps/web/src/components/ui/
  - Global CSS and Tailwind theme: e:/js/meta/projects/stock/apps/web/src/styles/globals.css