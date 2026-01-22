> **Branch:** develop  
> **Last Commit:** 2dff32f  
> **Last Updated:** 2025-11-21

## Wonkers Robots Overview
- API surface for managing hardware robots in the Wonkers admin/dashboard domain: lookup, ownership checks, pairing, configuration retrieval, and creation of dashboard robot records.
- Exposes v3/v4/v5 REST endpoints, validates requests via Kong headers, and persists robot metadata in the dashboard MySQL schema.

## Repo Purpose & Bounded Context
- Owns authoritative dashboard-facing robot records (serial, box number, hardware version/type, relation linkage, ICC, payment refs, TaaS flags).
- Provides admin and Tessa-owner views plus internal endpoints used by downstream services for linking/migration and robot creation.
- Delegates robot account state, pairing invitations, config blobs, and credential checks to specialist services while enforcing access control and request validation.

## Project Structure
```text
src/
  server.ts                      # bootstrap -> App
  App.ts                         # DI container, route wiring, Kong validators, config loader
  controller/                    # HTTP entrypoints (Robot*, Pairing*)
  service/                       # domain logic, external service clients, healthcheck
  repository/                    # MySQL access (+internal mappers, hardware types)
  model/                         # DTOs, DB mappers, domain shapes
  middleware/validationMiddleware.ts
  error/tbError.ts
docs/wonkers-robots.yaml         # OpenAPI 3.0 spec for published endpoints
config/{default.json, custom-environment-variables.json, reset.sh}
test/                            # mocha/nyc suites (controller IT, service unit, repo IT)
Dockerfile, tsconfig.json, prettier.config.js, yarn.lock
```

## Controllers & Public Surface
### Health & Context
- `GET /healthcheck` – express-healthcheck using DB ping.
- Global middlewares: request/context tracing (`tiny-tools`), morgan logging, JSON/urlencoded parsing, class-validator DTO guards.

### Robot Accounts & Config
- `PUT /v3/dashboard/robots/:serialId/account/deleterequest` – owner auth (Kong USER) + credential check; create delete request.
- `GET /v3/dashboard/robots/:serialId/account/deleterequest` – owner auth; fetch pending delete request.
- `GET /v3/dashboard/robots/:serialId/account/config` – owner auth; fetch robot config blob.
- `GET /v3/dashboard/robots/:serialId/account` – owner auth; fetch robot account.
- `GET /v4/dashboard/robots/:serialId/account` – owner auth; returns robot account + primary user.
- `GET /v3/admin/robots/:serialId/account` – admin auth; same payload as owner variant.

### Pairing & User Management
- `PUT /v3/dashboard/robots/:serialId/pairing/invites` – owner auth; body `PutPairingInviteDto`; creates invite via robot account service.
- `GET /v3/dashboard/robots/:serialId/pairing/invites` – owner auth; list invites.
- `DELETE /v3/dashboard/robots/:serialId/pairing/invites/:pairingInviteId` – owner auth; revoke invite.
- `POST /v3/dashboard/robots/:serialId/pairing/users/:userId/delete` – owner auth; validates password/OTP then unpairs user.
- `PATCH /v3/dashboard/robots/:serialId/pairing/users/:userId` – owner auth; change role via `PatchUserRobotDto`.
- `GET /v3/dashboard/robots/:serialId/pairing/users` – owner auth; list paired user accounts.

### Robot Retrieval & Mutation
- `GET /v3/admin/robots` – admin auth; filter by serial/boxNr/relationId/iccId/taas (v3 response).
- `GET /v3/admin/robots/:serialId` – admin auth; fetch v3 robot by serialId.
- `PATCH /v3/admin/robots/:serialId` – admin auth; patch excludeOkr/notes/deactivated/shippable/paymentReference/zendeskTicketId/iccId/taas then reload.
- `GET /v4/dashboard/robots/:serialId` – owner auth; v4 view constrained to owner relation.
- `GET /v4/admin/robots/:robotId` – admin validator; v4 robot by robotId (serialId semantics).
- `PATCH /v4/admin/robots/:serialId` – admin validator; v4 patch using `PatchRobotDto`.
- `GET /v4/admin/robots` – admin validator; v4 filter with optional hardwareType and serializer middleware.
- `GET /v5/dashboard/robots/:robotId` – Tessa-owner validator; v4 domain JSON with serializer + context group.

### Internal (Service-to-Service) Endpoints
- `GET /internal/v3/admin/robots/count` – robot counts per organisation or specific relation.
- `POST /internal/v3/admin/robots/:serialId/link` – re-link TaaS robots to new relation (with business guard).
- `GET /internal/v3/admin/robots/:serialId` – raw robot by serialId (v3 shape).
- `GET /internal/v4/admin/robots` – filter v4 robots without auth middleware.
- `GET /internal/v4/admin/robots/:serialId` – v4 robot by serialId.
- `POST /internal/v4/dashboard/robots` – create dashboard robot (serial/box/hardwareVersion/assembledAt).
- `GET /internal/v4/dashboard/robots/accounts/:robotId/robot` – resolve robot by robot account id.
- `GET /internal/v5/admin/robots/:serialId` – v4 domain by serialId with serializer pipeline.
- `GET /internal/v5/dashboard/robots/accounts/:robotId/robot` – v4 domain by robot account id.
- `POST /internal/v5/dashboard/robots` – create robot including hardwareType.

### Validation & Auth Layers
- Kong header auth via `KongValidationService` (role-enforced: USER/ADMIN).
- Additional validators from `tiny-tools`: `adminValidator`, `tessaOwnerValidator`, serializer/context middlewares for v4/v5 routes.
- DTO validation with `class-validator` for body/path/query; rejects non-whitelisted fields by default.

## Core Services & Logic
- **RobotService**: wraps RobotRepository + RobotAccountService; enforces ownership, retrieves robots in v3/v4 shapes, links robots to relations (with depot/TaaS guard), patches shippable/deactivated timestamps with `moment`, and creates dashboard robots (v4, v5 adds hardwareType).
- **RobotAccountService**: Axios client to robot-account service (`/internal/v2/robots/accounts`), robot-config service (`/internal/v1/config/{robotId}`), and contact info endpoint; maps responses to DeleteRequest/RobotAccount/RobotConfig/UserAccount classes with error mapping.
- **PairingService**: orchestrates invites/removals via `tiny-internal-services` RobotAccountService + DashboardAccountService to assemble contact info and roles.
- **CredentialService**: posts credentials to Dashboard service `/internal/v2/dashboard/accounts/{id}/authenticate`; maps 401 to UnauthorizedError.
- **KongValidationService**: Thin wrapper around `kong-js` authentication enforcing consumer `tinybots-dashboard-users` and role matching.
- **HealthCheck**: MySQL connectivity probe (pool.getConnection + ping).
- **Repositories**:  
  - `RobotRepository` executes SQL for robot reads/filters (v3/v4), patching, linking, counts, ICC/payment reference upsert, insertion; maps to domain classes and relation mapper (`repository/internal/RawRelationV3`).  
  - `HardwareTypeRepository` implements `IHardwareTypeProvider` to list hardware types from `hardware_type` table.  
  - `Database` abstracts mysql2 pool query/queryOne with simple error handling.
- **App/DI**: awilix container registers services, repositories, config values, and async container for hardware types; wraps `tiny-tools` cron context for logging.

## External Dependencies & Cross-Service Contracts
- **MySQL (dashboard database)** via `mysql2` pool; queries tables `dashboard_robot`, `dashboard_relation`, `dashboard_payment_reference`, `dashboard_robot_subscription_chain`, `taas_subscription`, `hardware_type`, `icc`.
- **Kong API Gateway** using `kong-js` with config `kong.adminBaseUrl`, `kong.apiBaseUrl`, and service addresses (`robotService`, `userService`, `dashboardService`).
- **Robot Account Service** (`robotAccountServiceAddress`) for accounts, delete requests, pairing, contact info; v2 internal paths.
- **Robot Config Service** (`robotsConfigServiceAddress`) for `internal/v1/config/{robotId}`.
- **Dashboard Service** (`dashboardServiceAddress`) for credential verification.
- **Dashboard Account Service & User Account Service** (from `tiny-internal-services` clients, addresses from Kong config) for fetching dashboard user details and robot account management.
- **Tiny shared libs**: `tiny-tools` (context, validators, logger, serializer), `tiny-internal-services` (typed DTOs/clients, hardware type loader), `tiny-backend-tools` logger helpers.
- **Docs**: OpenAPI spec in `docs/wonkers-robots.yaml` references per-path resource YAMLs.

## Testing & Quality Gates
- Test command runs lint (`standardx` with `@typescript-eslint`), then `nyc mocha` across `test/**/**/*.ts`; coverage gates: statements 90%, functions 90%, branches 65%, lines 90. `unit-test` subset has branches 70%.
- Integration tests cover controllers (`test/controller/*IT.ts`), repositories (`RobotRepositoryIT`), services (`RobotServiceTest`, `RobotAccountServiceTest`, `HeathCheckIT`, `KongValidationServiceIT`), middleware validation, and pairing/account flows with fixtures in `test/fixtures/testSetup.sql`.
- Test setup utilities under `test/setup` manage DB fixtures and shared helpers; mocks available via `tiny-internal-services-mocks` dev dependency.

## Operational Notes
- Config is loaded via `loadConfigValueV2` with env overrides defined in `config/custom-environment-variables.json`; required endpoints: DB host/user, Kong admin/API addresses, robot/user/dashboard account services, robot config service.
- Server binds to port 8080 with keepAlive/headers timeouts set; logging via morgan + winston JSON.
- Start command expects compiled artifacts (`dist/server.js`); run `yarn build && yarn start` or `ts-node` via test scripts during development.
