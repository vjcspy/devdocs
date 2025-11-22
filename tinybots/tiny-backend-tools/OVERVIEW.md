> **Branch:** master  
> **Last Commit:** 46b6202 (Updated from 2cc3467)  
> **Last Updated:** 2025-11-21

# tiny-backend-tools Overview

## Title & TL;DR

- Platform toolkit that every TinyBots Node.js/TypeScript service imports for Express bootstrapping, Kong auth/permission guards, request context propagation, and shared MySQL lifecycle wrappers.
- Ships cross-cutting middleware (validation, serializer, context logger, async and error wrappers), GraphQL permission helpers, and DI-ready base app classes covering unauthenticated, authenticated, and DB-backed services.
- Bundles infrastructure helpers (AWS SQS producer/consumer with context-aware tracing, cron job harnesses, resource pools/timers), security services (password, TOTP, email), config loaders, and Awilix async module orchestration.
- Tests run via `yarn test` with Mocha + NYC; SQS integration depends on Localstack. Node >= 20, Yarn 3.8.7; build emits `dist/`.

## Recent Changes Log

- Dependency and CI hardening from PR “task/PROD-496-fix-cves” (diff vs 2cc3467): `package.json`, `yarn.lock`, and `ci/test.sh` updated to remediate vulnerabilities and refresh pinned versions. No runtime or API surface changes detected; rerun smoke tests if consumers rely on transitive versions.

## Repo Purpose & Bounded Context

- Serves as the shared backend platform layer for TinyBots services (megazord-events, m-o-triggers, sensara-adaptor, Wonkers APIs, etc.). Any change here propagates broadly, so backward compatibility and tracing correctness are critical.
- Defines the standard HTTP/GraphQL/Kong contract (headers, permission checks, error/serialization shape) and tracing semantics (`Call-Ref`/`Svc-Ref` headers).
- Provides reusable infrastructure abstractions so individual services focus on domain logic rather than boilerplate.

## Project Structure

- `lib/` source of truth, re-exported through `lib/index.ts`.
  - `TinyApp*` / `TinyDatabaseApp*`: Express bootstrap classes (port 8080, morgan logger, JSON/body parsers, `/healthcheck`, keep-alive tuning).
  - `controller/`: `HealthCheck`, `DbHealthCheck`, base `Controller` and `AuthenticatedController`.
  - `middleware/`: context + logger propagation, class-validator/transformer validation, serializer, async handler, admin/permission guards, error -> Slack reporter, context groups.
  - `validation/`: `KongValidationService` wrapping `kong-js`.
  - `providers/`: `cron` (ContextCronJob), `sqs` (ContextSQS), `pool` (SimpleContextPool/TimerPool), `PermissionProvider` HTTP client.
  - `modules/`: `AwilixWrapper` for async module init/stop.
  - `repository/`: `Database`, `Transaction`, base `Repository`.
  - `model/`: DTOs and domains (`BaseDomain`, `Robot`, `Relation`, `Password`, `LogConfig`, `MySQLConfig`, `SmtpConfig`, `PermissionsProviderConfig`).
  - `service/`: `PasswordService` (PBKDF2 + zxcvbn), `TotpService` (notp), `EmailService` (EJS + Nodemailer).
  - `api/`: `OauthApiClient` + `ITokenManager` contract.
  - `utils/`: error wrappers, config loaders (`loadConfigValue`, `loadConfigValueV2`), class-validator helpers, map utilities.
  - `constants/`: error codes, permissions, Awilix container keys.
  - `graphql/`: `BaseResolver`, `InputValidator`, permission-aware resolver wrapper.
  - `context/` & `logger/`: request context creation, header setters, and logger patching.
- `ci/docker-compose.yml` spins up Localstack SQS + Node runner executing `ci/node-verify.sh`; `ci/test.sh` handles CI test flow.
- `test/` Mocha suites for apps, middleware, services, providers (cron/pool/sqs), utils, GraphQL helpers; `test/providers/sqs/sqsIT.ts` is the Localstack integration test.
- `dist/` build output (gitignored); `coverage/` NYC reports.

## Controllers & Public Surface

- **Base app classes:** `TinyAppUnauthenticated` and `TinyAppAuthenticated` start Express with morgan logging, JSON/urlencoded parsers, `/healthcheck`, and Awilix container setup. Authenticated variant wires `KongAuthenticationProvider` + `KongValidationService`.
- **Database variants:** `TinyDatabaseAppBase` adds `mysql2` pool + `Database` wrapper; `TinyDatabaseAppUnauthenticated` mounts `DbHealthCheck`; `TinyDatabaseAppAuthenticated` layers Kong auth; `TinyDatabaseAppAuthenticatedPermissions` registers permission provider + admin/permission middleware and exposes `useAdminValidatorMiddleware`, `usePermissionValidatorMiddleware`, `usePermissionRoutes`.
- **Health checks:** `HealthCheck` invokes a supplied callback immediately; `DbHealthCheck` pings MySQL pool and reports connection state.
- **Middleware surface:**  
  - `contextMiddleware` + `contextLoggerMiddleware` attach `IRequestContext` (`callRef`, `serviceRef`) and logger child into requests/responses.  
  - `ValidationMiddleware` offers header/body/query/path validators via class-validator/transformer.  
  - `SerializerMiddleware` serializes `BaseDomain` objects honoring context groups; `ContextGroupMiddleware` tags request with serialization groups.  
  - `AsyncHandlerMiddleware` wraps async route handlers.  
  - `AdminMiddleware` exposes admin/robot/tessa-owner validators, `permissionValidator`, `matchPermissions`, and `userRobotAccessValidator`.  
  - `ErrorMiddleware` normalizes errors, logs via context logger, returns JSON, and notifies Slack on 5xx.
- **GraphQL helper:** `BaseResolver.Wrap` adds optional permission validation and DTO validation; requires `SetPermissionApiProvider` during bootstrap.

## Core Services & Logic

- **Auth & permissions:** `KongValidationService` enforces consumer identity (dashboard users, robots, integrations, users) and roles before attaching entities to the request. `PermissionAPIProvider` posts to Dashboard service (`PermissionsProviderConfig.address`) to validate `Permission.*` constants; used by middleware and GraphQL wrappers.
- **Request context & logging:** `Context.newRequestContext` derives `callRef` from inbound header or URL + UUID; `setHeader` writes `Call-Ref`/`Svc-Ref` to responses; `loggerFromCtx` returns patched child logger (winston-compatible).
- **Data layer:** `Database` and `Transaction` wrap mysql2 pool with `query`, `queryOne`, `ping`, `getConnection`; `Repository` base injects `Database`. `MySQLConfig` DTO validates pool settings.
- **Domains & serialization:** `BaseDomain.FromPlain` validates DTOs, wraps validation errors into `ApplicationError` with error codes; `ContextGroup` controls field exposure.
- **Infrastructure:**  
  - `ContextSQS` produces/consumes messages with JSON bodies + message attributes storing `callRef`; retries sending (note: retry counter increments twice per failure, effectively halving `maxAttempts`).  
  - `ContextCronJob`/`SimpleContextCronJob` schedule work with inherited context/logging and error handling.  
  - `SimpleContextPool` and `TimerPool` manage pooled async resources and timers with graceful stop semantics.  
  - `AwilixWrapper` tracks async modules and runs `init/stop` lifecycle hooks in order.
- **Security & comms:** `PasswordService` hashes/verifies passwords (PBKDF2, strength via zxcvbn, supports Java-style hashes), `TotpService` issues/validates TOTP secrets/tokens, `EmailService` renders EJS templates and sends via Nodemailer transport. `OauthApiClient` wraps Axios to inject access tokens, refresh on 401 once per burst (queued waiters), and retry bounded by `_maxRetries`.
- **Utilities:** `loadConfigValue` and `loadConfigValueV2` load/validate config (supports primitive validation and DTO instantiation); `ApplicationError` supports annotations + Slack-safe logging via `logApplicationError`; map helpers (`MultiIndexedMap`, `Map` wrappers) used in tests.

## External Dependencies & Cross-Service Contracts

- **Kong Gateway (`kong-js`)** for header validation and role detection; depends on upstream Dashboard and robot user stores.
- **Dashboard Permission API** (`PermissionsProviderConfig.address`), called by `PermissionAPIProvider.validate()` for admin permission checks.
- **MySQL** via `mysql2` pooling for any DB-backed TinyApp variants.
- **AWS SQS** via AWS SDK v3; message attributes carry `callRef`; tests rely on Localstack (port 4566).
- **Slack** via `tb-ts-slack-notification` for 5xx error alerts.
- **SMTP** via Nodemailer + EJS templates in consumer repos; `EmailService` expects `html.ejs` and `subject.ejs` under a provided template path.
- **Cron** via `cron` package for scheduled jobs.
- **OAuth/HTTP** via `axios` with token manager interface for downstream APIs.

## Testing & Quality Gates

- Run `yarn test` (or `yarn unit-test`) → `nyc mocha --require ts-node/register --recursive test/**/*.ts`, then HTML coverage report and thresholds: statements ≥55%, functions ≥50%, branches ≥45%, lines ≥60%.
- Integration: `test/providers/sqs/sqsIT.ts` requires Localstack; `ci/docker-compose.yml` starts Localstack + Node runner via `ci/node-verify.sh` with `AWS_ENDPOINT=http://localstack:4566`.
- Other suites cover app bootstraps, middleware (context, serializer, admin/permissions, validation, error), services (KongValidationService, Password/TOTP/Email), cron/pool modules, GraphQL wrappers, and util helpers.
- Build: `yarn build` removes `dist` then compiles via `tsc`; Node engine set to >=20.0.0; package manager locked to Yarn 3.8.7.
