> **Branch:** master
> **Last Commit:** 981a953
> **Last Updated:** Fri Oct 3 03:52:35 2025 +0000

## Wonkers API TL;DR
- Dashboard-facing REST API for account auth, subscriptions, robots, relations, contacts, and overview dashboards; also exposes admin/internal endpoints consumed by gateways and automation jobs.
- Backed by MySQL `dashboard` schema, with Kong OAuth/headers for auth, Nodemailer for transactional email, and cross-service calls to Checkpoint robot service and Tinman robots-online heartbeat.
- Uses awilix DI to wire repositories/services/controllers; request validation via `express-joi-validation` and Joi schemas in `api/schemas`.
- Includes an `activation/` job runner that syncs pending registrations, subscriptions, and Teamleader CRM data and triggers robot/account activation workflows.
- Tests run with Mocha/Chai/Supertest + NYC coverage gates (≥90% statements/functions/lines, ≥80% branches).

## Repo Purpose & Bounded Context
- Provides authenticated Dashboard user and admin APIs to manage accounts, MFA, relations, robots, subscriptions, and high-level overviews for “Tessa owner” customers.
- Serves as the contract surface for other TinyBots services (e.g., wonkers-graphql, automation pipelines) through `/internal` endpoints and the OpenAPI spec at `docs/wonkers-api.yaml`.
- Separates web API runtime from batch activation/sync jobs housed under `activation/`.

## Project Structure
- `server.js` – Express app wiring awilix container, route bindings, global validators, and HTTP server tuning.
- `api/controller/` – Controllers for accounts, login/session, relations, subscriptions, robots, overview, chains, contacts.
- `api/service/` – Domain services (account, subscription, robot, relation, chain, link, contact, email, password/TOTP, key generation).
- `api/repository/` – MySQL data access for accounts/activations, relations, robots, subscriptions/overview, payments/transfers, contacts.
- `api/connection/` – MySQL pool + transaction helper; Nodemailer transporter factory.
- `api/validate/` – Kong client + header validator for role-aware authentication.
- `api/schemas/routes/` – Joi request schemas for headers/params/bodies.
- `api/model/` – Domain models and DTO mappers for DB rows to API shapes.
- `api/errors/` – Error types and HTTP mapping helper.
- `activation/` – Job runner and supporting libs for activation, Teamleader sync, relation cleanup, and subscription/robot linking (uses its own `config/`).
- `docs/wonkers-api.yaml` – OpenAPI contract for external/internal consumers.
- `ci/` – Concourse/local test scripts, docker-compose, and entrypoint used in pipelines.
- `test/` – Unit (service/connection), repository integration, and API integration suites with fixtures/setup.

## Controllers & Public Surface
### Account & Auth
- `POST /v2/dashboard/accounts/login` – Username/password(+OTP) login for users; rejects admins (redirects to admin flow).
- `POST /v1/dashboard/accounts/token` – Refresh token to access token using Kong.
- `POST /v1/dashboard/accounts/forgotten` → email reset; `POST /v1/dashboard/accounts/reset` to set new password.
- `POST /v1/dashboard/accounts/` – Create dashboard user from registration invite.
- `GET /v1/dashboard/accounts/activate` and `/create` – Activation and invite validation.
- `POST /v2/dashboard/accounts/mfa` – Issue TOTP secret; `PATCH /v2/dashboard/accounts/self` – enable/disable MFA & profile edits (requires password & OTP when enabling).
- `GET /v2/dashboard/accounts/self` – Fetch v2 account profile; `POST /v2/dashboard/accounts/verifypassword` – password check.
- Internal: `POST /internal/v2/dashboard/accounts/:dashboardUserId/authenticate` – credential+OTP verification for service-to-service.
### Admin
- `PUT /internal/v1/dashboard/admin` – Create admin account.
- `POST /v2/admin/invite` – Invite user for relation.
- Robot admin: `GET /v2/admin/robots`, `GET /v2/admin/robots/:serialId`, `PATCH /v2/admin/robots/:serialId`, `POST /v2/admin/robots/:robotId/status`.
- Subscription admin: `GET /v2/admin/subscriptions` with filters; `POST /v2/admin/subscriptions`; `PATCH /v2/admin/subscriptions/:subscriptionId`; counts via `/internal/v2/admin/subscriptions/count`.
- Chain admin: `GET /v2/admin/chains/:chainId`, `PATCH /v2/admin/chains/:chainId`.
- Overview admin: `GET /v2/admin/overview`, `/internal/v2/admin/overview`; link/transfer via `POST /v2/admin/link` and `POST /v2/admin/dashtransfer`.
### User (“Tessa owner”) Facing
- `GET /v1/v2 dashboard/accounts/self` (v1 deprecated; v2 used) – Account profile; `GET /v1/dashboard/accounts/self/subscriptions` – subscription info; `PATCH /v1/dashboard/accounts/self/subscriptions` – update contact info for subscriptions.
- `GET /v1/dashboard/accounts/self/online` – Robots online for user’s relation.
- `GET /v2/dashboard/chains/:chainId` – Chain details for owning relation.
- `GET /v2/dashboard/robots/:serialId` – Robot details for owning relation.
- `GET /v2/dashboard/contacts/:contactId` / `PATCH ...` – Contact retrieval/update within relation.
- `GET /v2/dashboard/overview` – Overview for relation.

## Core Services & Logic
- **accountService** – Handles invites, creation, activation, password reset, MFA secret storage/validation (node-2fa), role enforcement, last-active tracking.
- **loginController + kong** – Delegates token issuance/refresh to Kong (`kong-js`), enforces role-based login path; refresh uses Kong’s OAuth endpoints via `kongService`.
- **subscriptionService** – CRUD for subscriptions with validation on robot switching, payment references (`paymentReferenceRepository`), and relation consistency; aggregates overview and counts; provides per-relation subscription info.
- **robotService** – Wraps robot repository plus remote Checkpoint robot service for account status/metadata and Tinman robots-online heartbeat; enforces shippable/assignment rules and payment reference requirements.
- **relationService** – Filters relations by attributes for admin use.
- **chainService** – Retrieves/updates chain compositions, delegating per-subscription validation via subscriptionService.
- **linkService** – Links relations, robots, and subscriptions; performs “dash transfer” between relations/robots, ensuring robot+subscription constraints and invoking robotService updates.
- **contactService** – CRUD for relation contacts with authorization via kongValidator.
- **emailService + keyGeneratorService + passwordService** – Generates secure keys, hashes passwords, renders EJS templates, and sends via Nodemailer transporter.
- **totpService** – Generates and validates TOTP codes with drift tolerance.
- **errors** – Centralized HTTP error mapping with DB duplicate handling.
- **activation jobs** – `activation/index.js` runs sequential jobs: activationJob (auto-activates after configurable days), tessaOwnerRegistrationJob, teamleaderSyncJob (pulls Teamleader CRM), cleanRelationsJob; uses its own repositories and mailer plus Checkpoint base URL.

## External Dependencies & Cross-Service Contracts
- **MySQL (`dashboard` schema)** – Primary datastore accessed through repositories; transactions via `dbInteraction`.
- **Kong Gateway** – Authentication/authorization for dashboard users/admins; OAuth token issuance/refresh; integrates with external `userService` & `robotService` addresses from `kongConfig`.
- **Checkpoint Robot Service (`robotServiceAddress`)** – Robot account status changes, robot/account lookups, and app-user associations; also provides serial lookups from robot IDs.
- **Tinman Robots Online Service (`robotsOnlineServiceAddress`)** – Heartbeat endpoint to determine currently online robots for a relation.
- **SMTP (mailcatcher/dev)** – Transport for activation/reset/invite emails; templates in `api/emails/*`.
- **Teamleader API** – Used by activation jobs for CRM sync (`teamleaderConfig`).
- **OpenAPI contract (`docs/wonkers-api.yaml`)** – Shared with downstream consumers (e.g., gateways, wonkers-graphql); keep aligned when routes change.

## Testing & Quality Gates
- Test runner: `yarn test` (or `npm test`) executes StandardJS lint, then Mocha with NYC coverage and enforces ≥90% statements/functions/lines, ≥80% branches; HTML coverage report emitted.
- Unit tests in `test/service`, `test/connection`; repository integration tests in `test/repository`; API/integration flows in `test/integration` using Supertest and Nock; fixtures under `test/fixtures`, setup in `test/setup/dbSetup.js`.
- Additional activation-specific tests live under `activation/test` aligned with job logic.

## Operational Notes
- Config via `config/default.json` (API) and `activation/config/default.json` (jobs); override with environment-specific files. Key entries: DB creds/pool, frontEndAddress links, registration/validation/password reset templates, Kong/Checkpoint/Tinman addresses, relation-type rules for shipping/payment references.
- HTTP server tuned with keep-alive and header timeouts; body parsing limits set to 50mb.
- DI container wires singletons for repos/services/controllers; wrap requests with Joi validation; central error handler returns 400/401/403/404/409/500 codes as appropriate.
