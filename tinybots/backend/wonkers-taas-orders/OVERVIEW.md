> **Branch:** develop  
> **Last Commit:** ff8bbab  
> **Last Updated:** Thu Sep 18 06:43:49 2025 +0000

# Wonkers-TaaS Orders

## TL;DR
- Node/Express service that handles the full TaaS order lifecycle (create, patch, status transitions, returns) for both external partners and dashboard admins.  
- Provides multiple API versions (v3–v6) plus internal endpoints, with Kong-authenticated access control and DTO validation.  
- Orchestrates MySQL persistence, TinyBots TaaS subscriptions, PostNL delivery checks, Monday TEx board creation, device status updates, and rich email notifications.  
- Runs a cron-driven delivery checker and depends on config-driven SMTP, Kong, TaaS, Nedap Ons, Monday, and Device Signals services.

## Repo Purpose & Bounded Context
- Source of truth for creating, updating, and tracking TaaS hardware orders and related concept orders/returns.  
- Acts as the glue between integration users (external), dashboard admins, and downstream services (TaaS service, device signals, email/Slack/Monday).  
- Owns persistence in `dashboard` MySQL schema via repositories; DTOs enforce contract stability across versions.

## Project Structure
- `src/App.ts` – Composition root (Awilix container), route wiring, cron scheduling, DI registrations.  
- `src/server.ts` – Bootstraps `App` and starts the HTTP server.  
- `src/controller/` – Route handlers: external, admin (v3/v4/v5/v6), internal, concept orders/returns, installations, default requester.  
- `src/service/` – Business logic: order lifecycle, status transitions, concept workflows, TaaS subscription orchestration, email, phone, PostNL, Nedap Ons, Monday integration, installations.  
- `src/repository/` – MySQL access (manual SQL, transactions) for orders, concepts, installations, hardware types, default requesters, statuses.  
- `src/model/` – Domain models and DTOs (class-validator), including V5/V6 variants and integration shapes.  
- `src/middleware/` – DTO validation wrappers, access validation, Kong header checking.  
- `src/job/CheckDeliveryJob.ts` – Cron task polling PostNL for delivered shipments.  
- `config/default.json` (+ env overrides) – MySQL, Kong, PostNL, SMTP, email templates, Monday, environment flags.  
- `email/` – Handlebars templates for requester/receiver/returner/pickup/tessa-expert flows.  
- `test/` – Mocha/nyc suites across controllers, services, repositories, jobs, and models with fixture data.

## Controllers & Public Surface
### External (integration, Kong auth via `tinybots-integration-users`)
- `POST /ext/v1/taas-orders/:organisationId` – Create order (phone normalization).  
- `GET /ext/v1/taas-orders/:organisationId` – List orders with query filters.  
- `GET/PATCH/DELETE /ext/v1/taas-orders/:organisationId/:orderId` – Read/update/delete order with access guard.  
- `POST /ext/v1/taas-orders/:organisationId/:orderId/return` – Start a return.

### Admin v3 (dashboard users)
- `GET /v3/admin/taas-orders[:orderId]` – Read orders or single order.  
- `PATCH /v3/admin/taas-orders/:orderId` – Update track & trace / notes (v3 DTO).  
- `POST /v3/admin/taas-orders/:orderId/status/{accept|decline|delivered|accept-return|reject-return}` – Status transitions.

### Admin v4
- `GET /v4/admin/taas-orders[:orderId]` – Read orders (v4 shapes).  
- `POST /v4/admin/taas-orders` – Create order (admin DTO v2).  
- `PATCH /v4/admin/taas-orders/:orderId` – Patch (track & trace, internal notes, tessa expert).  
- `POST /v4/admin/taas-orders/from-taas` – Create orders from existing TaaS subs.  
- `POST /v4/admin/taas-orders/:orderId/return` – Initiate admin return.

### Admin v5
- `GET /v5/admin/taas-orders[:orderId]` – Read v5 orders; pairs with `/internal/v5/taas-orders` raw view.

### Admin v6 (current)
- `GET /v6/admin/taas-orders[:orderId]` – Search by id/uuid/relation/hardwareType/serial; hydrates installations into notes.  
- `POST /v6/admin/taas-orders` – Create order V6 with optional concept link and phone validation.  
- `PATCH /v6/admin/taas-orders/:orderId` – Patch using v2 patch DTO.  
- `POST /v6/admin/taas-orders/from-taas` – Bulk create from TaaS subs (V6).  
- `POST /v6/admin/taas-orders/:orderId/return` – Admin return with concept archiving.  
- `POST /v6/admin/taas-orders/:orderId/status/{accept|decline|delivered|accept-return|reject-return}` – Status transitions (async handler + error middleware).

### Internal service endpoints
- `/internal/v1/taas-orders/...` – Create/read/delete/return orders for back-office.  
- `/internal/v4/taas-orders/:orderId/status/activated` – Activation hook.  
- `/internal/v5/taas-orders` – Raw V5 orders.  
- `/internal/v6/taas-orders/:orderId/status/activated` – Activation for V6.  
- `/internal/v5/taas-orders/installations` (+search/failures) – Installation ingest and queries.

### Concept orders & returns
- `/v4/admin/taas-orders/concepts/orders|returns` (+`:id`) – List/read concepts (v4).  
- `/internal/v1/taas-orders/concepts/orders|returns` – Create concepts internally.  
- `/v6/admin/taas-orders/concepts/orders|returns` (+`:id`) – V6 list/read.  
- `/internal/v6/taas-orders/concepts/orders|returns` – Create V6 concepts.

### Default requester
- `PUT /v3/admin/taas-orders/relation/:relationId/default/requester` – Upsert default requester contact.

## Core Services & Logic
- **TaasOrderService** – Creates admin/external orders (v2/v6), enforces duplicate checks, returns handling, patching, track & trace, pickup address, V5/V6 query fan-out.  
- **TaasOrderStatusService** – Accept/decline/deliver/activate/return flows; creates TaaS subscription via `CustomTaasService`; triggers email notifications; updates Device Signals; pushes TEx pulses to Monday board; handles return accept/reject.  
- **ConceptOrderService** – CRUD/archive for concept orders/returns; links/archives concepts when used; validates phone numbers; V6 DTO mapping.  
- **CreateFromTaasSubsService** – Validates subscriptions, materializes orders (v2/v6) from existing TaaS subs, sets status dates.  
- **InstallationService** – Resolves eligible orders, records success/failure installs, links devices via TaaS service, sets delivered status.  
- **CustomTaasService** – HTTP client to TaaS service (`/internal/v3/admin/taas`), create/delete subscriptions.  
- **TaasOrderRepository/AdminTaasOrderRepository/...** – Raw SQL over `dashboard` schema with manual transactions; handles contacts, addresses, status rows, hardware types.  
- **EmailStatusService** – Template-driven emails (requester/receiver/health-professional/returner/pickup/tessa-expert), per-relation config, resend on contact changes.  
- **KongValidationService** – AuthN/AuthZ for integration vs dashboard users with role checks.  
- **PostNLService** – Fetches shipment status; used by cron job to auto-mark delivered with timestamps.  
- **NedapOnsService** – Retrieves concepts from wonkers-nedap bridge.  
- **PhoneNumberService** – Normalizes/validates numbers (E.164, NL fallback).  
- **AccessValidator** – Middleware enforcing integration ownership per order.

## External Dependencies & Cross-Service Contracts
- **Kong Gateway** – Validates `KongHeader` for integration/dashboard consumers; enforces org membership and admin role.  
- **MySQL (`dashboard` schema)** – Primary persistence via `tiny-tools` Database/Transaction.  
- **TaaS Service** – HTTP calls to create/delete subscriptions and link devices (via `tiny-internal-services`+`CustomTaasService`).  
- **Device Signals Service** – `updateOrderStatus` after delivery/returns.  
- **PostNL API** – Shipment status polling (barcode endpoint, API key).  
- **Monday (tiny-monday)** – Creates TEx appointment pulses; requires `mondayApiToken`, board id, path.  
- **SMTP / Email templates** – Nodemailer transport built from config; relation-specific templates & pickup providers.  
- **Slack (tb-ts-slack-notification)** – SlackService registered (future/optional notifications).  
- **Nedap Ons bridge** – Retrieves concepts for admin flow.  
- **Config loading** – Uses `loadConfigValueLegacy` and `config/default.json` for addresses, cron, keys, environment toggles, default contact.

## Testing & Quality Gates
- Test runner: `nyc mocha` with ts-node/tsx; coverage gates default 90/90/80/90 for full test script.  
- Suites cover controllers (admin v2/v5/v6, external, internal, concept, installation), services (status, email, install, create-from-subs, phone, PostNL), repositories (orders, concepts, installations, default requester), job scheduling, and model validation.  
- Fixtures include PostNL responses and patch payloads; integration tests expect MySQL test database and mocked tiny-internal-services where configured.  
- Lint: `test-standardx` script using @typescript-eslint parser.  
- Build: `tsc`; runtime entry `node dist/server.js`.
