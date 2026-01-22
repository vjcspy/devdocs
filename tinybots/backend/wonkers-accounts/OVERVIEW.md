# wonkers-accounts Overview

## TL;DR
- TypeScript/Express service that authenticates dashboard admins, brokers OAuth tokens via Kong, and exposes CRUD APIs for dashboard accounts, MFA, groups, permissions, and SSO linkages.
- Persists all identity and RBAC data inside the `dashboard` MySQL schema, enriches requests with TinyTools middlewares (validation, context logging), and offers `/internal/v3/admin/...` endpoints other TinyBots services call for permission checks.

## Table of Contents
- [Repo Purpose & Interactions](#repo-purpose--interactions)
- [Repo Structure](#repo-structure)
- [Controllers / Public Surface](#controllers--public-surface)
- [Key Services & Repositories](#key-services--repositories)
- [Domain Data & Integration Map](#domain-data--integration-map)
- [Tests & Quality Signals](#tests--quality-signals)
- [External Dependencies & Cross-Service Contracts](#external-dependencies--cross-service-contracts)
- [Gaps & Risks](#gaps--risks)

## Repo Purpose & Interactions
wonkers-accounts backs the Tinybots dashboard login and RBAC experience. It owns admin account lifecycle (creation, password & MFA management, deletion), SSO account linking, group and permission assignment, and the issuance/refresh of OAuth tokens. Requests are served through an Express app that extends `TinyDatabaseAppAuthenticated`, giving it a MySQL pool, Kong validation helpers, and standard middlewares (context propagation, logging, serializer, error handler). Dependency injection uses Awilix so controllers/services are singleton-scoped.

Primary interaction patterns:
- Dashboard UI hits `/v3/admin/...` endpoints behind Kong. The service validates Kong headers (`x-consumer-*`, `x-authenticated-*`) to ensure callers are real dashboard admins before mutating accounts or verifying passwords.
- Other Tinybots backends (e.g. wonkers-graphql, wonkers-api, robotics) call `/internal/v3/admin/...` routes to list groups/permissions, validate whether an account has a permission, and translate external IdP identifiers into dashboard account IDs.
- Account login delegates token issuance to Kong (`TokenService`), updates `last_active_at`, and enforces multi-factor authentication with TinyTools’ TOTP service.

Runtime & ops notes:
- Config under `config/default.json` specifies MySQL host/user and Kong endpoints; `config/custom-environment-variables.json` maps env vars such as `DB_RW_HOST`, `KONG_ADMIN_ADDRESS`, `KONG_API_ADDRESS`, and `SUPER_USERS`.
- `server.ts` loads config via `loadConfigValue(V2)` helpers, instantiates `App`, and starts the HTTP server plus MySQL pool.
- Package scripts: `yarn build` compiles TypeScript to `dist/`, `yarn start` runs `dist/server.js`, `yarn test` runs `standardx` + full mocha integration suite (needs a reachable MySQL instance), and `yarn unit-test` runs only `*Test.ts`.
- Dockerfile builds a production image on Node 22 Alpine with a two-stage build, embedding only the compiled `dist` bundle plus runtime configs.

## Repo Structure
```text
src/
  App.ts                  Express + TinyTools bootstrap, DI container wiring, middleware + route registration
  server.ts               Entry point that loads config and starts the App
  controller/             Account, Login, Group, Permission controllers (HTTP handlers)
  service/                Business logic for accounts, tokens, groups, permissions
  repository/             MySQL repositories and transactions per aggregate
  model/                  DTOs, domain entities, validation schemas, constants
  errors/                 Custom WonkersAccountErrors wrapping HTTP error classes
config/                   Node-config definitions and env mappings for MySQL/Kong/super users
docs/                     `wonkers-accounts.yaml` OpenAPI contract + resources
ci/                       Concourse pipelines, docker entrypoint, local test/build scripts
test/                     Mocha/Supertest integration suites, DB fixtures, setup helpers
Dockerfile                Multi-stage docker build producing the production runtime image
```

## Controllers / Public Surface
All controllers are registered in `App.setEndpoints`, use TinyTools `ValidationMiddleware` for DTO validation, and share TinyTools context logger & error middleware. Routes fall into two categories: `/v3/admin/...` (guarded by Kong headers) and `/internal/v3/admin/...` (assumed to be called from trusted services).

### AccountController (`src/controller/AccountController.ts`)
- `DELETE /v3/admin/accounts?email=` – Validates Kong headers and `EmailQueryDto`, verifies the caller is listed in the `superUsers` config, and deletes the target account through `AccountService.deleteUser`.
- `GET /v3/admin/accounts?email=` – Authenticated admin fetches a dashboard account by email; returns a `DashboardAccountDto` or `404`.
- `PATCH /v3/admin/accounts/mfa` – Body validated with `AdminEnableMfaDto`; re-verifies password + OTP via `AccountService.patchAccount` to enable MFA for an admin on behalf of support.
- `POST /v3/admin/accounts/verifypassword` – Kong-authenticated admin can prove their password without retrieving tokens; returns `204` when `PasswordDto` matches.
- `GET /v3/admin/accounts/relations/:relationId` – Lists owner accounts tied to a relation, using `RelationIdPathDto` and `AccountService.getOwnerAccounts`.
- `GET /internal/v3/admin/accounts/:accountId` – Fetches any account by id; used by other services without Kong auth.
- `POST /internal/v3/admin/accounts/:accountId/validate` – Validates whether an account has every permission listed in `AccountPermissionDto`; returns `200` or a `ForbiddenError`.
- `GET /internal/v3/admin/accounts` (query `issuer`, `issuer_user`) – Looks up the dashboard account id linked to an external IdP user, logs via TinyTools context logger, and propagates errors through the shared serializer + error middleware.
- `POST /internal/v3/admin/accounts` – Creates or links an admin account from external IdP data (`CreateUserFromExternalUserDto`), returning the dashboard id; rejects duplicates with `409`.

### LoginController (`src/controller/LoginController.ts`)
- `POST /v3/admin/accounts/login` – Calls `AccountService.verifyAdmin` to check username/password/OTP, enforces MFA enrollment (returns `{ token: null, needMfa: true }` or `403` with secret), and on success requests an access/refresh token from Kong via `TokenService.getToken` before updating `last_active_at`.
- `POST /v3/admin/accounts/token` – Accepts a `RefreshTokenDto`, refreshes tokens via `TokenService.getRefreshToken`, reloads the related account, verifies the account isn’t blocked, and returns the new access token payload.

### GroupController (`src/controller/GroupController.ts`)
- `/internal/v3/admin/groups` – `GET` lists groups, `POST` creates new groups using `CreateGroupDto`.
- `/internal/v3/admin/groups/:groupName` – `GET` fetches group metadata (case-insensitive), `PATCH` updates name/description via `UpdateGroupDto`, and `DELETE` removes the group if unused. All validate `GroupPathDto`.
- `/internal/v3/admin/groups/:groupName/permissions` – `GET` lists permissions inside a group; `POST` adds a permission using `CreateGroupPermissionDto`.
- `/internal/v3/admin/groups/:groupName/permissions/:permissionName` – `DELETE` removes a permission from the group.
- `/internal/v3/admin/groups/:groupName/users` – `GET` lists members; `POST` adds a user email via `CreateGroupUserDto`.
- `/internal/v3/admin/groups/:groupName/users/:userEmail` – `DELETE` detaches a user from the group.
All group routes are unauthenticated HTTP endpoints intended for internal automation and rely on `GroupService` to enforce referential integrity and error semantics (`NotFound`, `UnprocessableEntity`).

### PermissionController (`src/controller/PermissionController.ts`)
- `GET /internal/v3/admin/permissions` – Lists every permission record.
- `GET /internal/v3/admin/permissions/:permissionId` – Fetches a permission by id via `PermissionService.getPermissionById`.
- `POST /internal/v3/admin/permissions` – Creates a permission from `CreatePermissionDto`; duplicate names raise `409`.

## Key Services & Repositories

### AccountService (`src/service/AccountService.ts`)
- Depends on `AccountRepository`, `PasswordService`, `TotpService`, `RelationRepository`, and `AccountPermissionRepository`.
- Core flows: fetch/delete accounts, validate passwords/MFA, enforce admin roles (`AccountRole`) and statuses (`AccountStatus`), and update `last_active_at`.
- `verifyAdmin` prevents SSO-managed accounts from using password login, enforces role = ADMIN, validates password + OTP, handles MFA enrollment (issuing shared secrets via TinyTools TOTP service), and throws typed `WonkersAccountErrors`.
- `patchAccount` re-validates password/OTP before enabling MFA, then re-fetches the updated account.
- `validatePermission` joins `group_users` and `permissions` via `AccountPermissionRepository`; missing permissions produce a `ForbiddenError`.
- `getUserIdByExternalApplicationUser` and `createUserFromExternalApplicationUser` back the `/internal` SSO endpoints, obtaining the admin relation id and upserting `dashboard_external_account` records in a transaction.

### TokenService (`src/service/TokenService.ts`)
- Thin wrapper around `kong-js` `KongService`; issues tokens scoped to `primary`, and refreshes tokens while translating Kong responses into `{ integrationId, token }`.
- Converts any Kong refresh failure into `UnauthorizedError('Invalid Credentials')`.

### GroupService (`src/service/GroupService.ts`)
- Depends on `GroupRepository`, `GroupPermissionRepository`, `PermissionRepository`, `GroupUserRepository`, and `AccountRepository`.
- Handles creation/update/deletion of groups (with conflict handling), manages membership of permissions/users (fairly strict validation + error taxonomy), and normalizes email/user lookups through `AccountRepository`.
- Encapsulates idempotent operations (adding a permission/user twice yields `422`) and ensures 404 vs 422 semantic differences.

### PermissionService (`src/service/PermissionService.ts`)
- Wraps `PermissionRepository` to provide consistent `NotFoundError` semantics, plus duplicate-handling when inserting.

### Repository layer highlights
- `AccountRepository`: raw SQL queries for `dashboard_user_account`, `dashboard_relation`, `dashboard_user_profile`, and `dashboard_external_account`. Provides transactional delete and external-account upsert flows, MFA secret storage, and `getOwnerAccountsByRelationId`.
- `AccountPermissionRepository`: joins `group_users -> group_permissions -> permissions` to fetch distinct permissions for a user; used by the permission validation endpoint.
- `GroupRepository`, `GroupPermissionRepository`, and `GroupUserRepository`: CRUD around `groups`, `group_permissions`, and `group_users` tables with explicit conflict handling and referential integrity errors (e.g., `ER_NO_REFERENCED_ROW_2`, `ER_ROW_IS_REFERENCED_2`).
- `PermissionRepository`: CRUD for `permissions`, with duplicate detection.
- `RelationRepository`: looks up the admin relation (`relation_number = 0`) and can insert it if missing.

## Domain Data & Integration Map
- `DashboardAccount` aggregates account id, username, hashed password (`Password` value object), MFA info, relation metadata (`dashboard_relation` + `dashboard_relation_type`), and status/role IDs. `DashboardAccountDto` exposes API-friendly fields.
- Account statuses (`ACTIVE`, `BLOCKED`, `UNCONFIRMED`) and roles (`USER`, `ADMIN`) are defined both as enums (`model/constants/AccountConstant.ts`) and value objects (`AccountStatus`, `AccountRole`) to translate between database ids and human-readable names.
- External identity records live in `dashboard_external_account` (issuer + unique_id) and enable `/internal/v3/admin/accounts` to map IdP claims to dashboard ids. Duplicate issuer/unique combinations raise `409` during creation.
- Groups, permissions, and memberships are stored in `groups`, `permissions`, `group_permissions`, and `group_users`. These tables back the RBAC endpoints and the permission-validation API used by other services just-in-time.
- MFA secrets and timestamps are stored per account in `dashboard_user_account`. `AccountRepository.setSecret` writes TOTP seeds generated by TinyTools `TotpService`.
- Transactions (TinyTools `Transaction`) ensure multi-step operations (account deletion, SSO account creation) either fully succeed or roll back.
- Kong integration: `/v3/admin/...` endpoints expect Kong to authenticate requests and supply `KongHeader` metadata; tokens are issued/validated via Kong admin & API URLs (see `kongConfig`).
- Logging/observability: `contextMiddleware` seeds a correlation id via `randomUUID`, `contextLoggerMiddleware` attaches a winston logger, and account controllers log inbound/outbound payloads for SSO endpoints.

## Tests & Quality Signals
- `test/controller/*IT.ts` uses `supertest` against a real `App` instance and a seeded MySQL database. They cover happy paths and failure modes for account lookup/deletion, MFA enablement, owner account retrieval, login flows (including MFA enrollment/needMfa branches), permission validation, and all group/permission endpoints. `AuthenticationTester` verifies Kong header requirements on admin routes.
- `test/repository/*IT.ts` exercises each repository directly (Account, AccountPermission, Group, GroupPermission, GroupUser, Permission, Relation) to ensure SQL queries behave as expected with real MySQL data, duplicate protection, and referential integrity errors.
- `test/helpers/DbSetup.ts` and `test/setup/DashboardAccountSetup.ts` extend `tiny-testing` helpers to seed relations, accounts, permissions, and to clean tables in a deterministic order. Tests expect environment variables such as `DB_RW_HOST` and use hard-coded credentials to connect to the `dashboard` database.
- `yarn test` runs `standardx` linting plus the full mocha suite under NYC with thresholds (Statements ≥88%, Functions ≥90%, Branches ≥60%, Lines ≥87%). `yarn unit-test` narrows execution to files with the `*Test.ts` suffix.
- CI scripts under `ci/` (e.g., `local-test.sh`, `concourse-test.sh`) wrap the same commands inside docker-compose to ensure MySQL dependencies are available in pipelines.

## External Dependencies & Cross-Service Contracts
- **MySQL (`dashboard` schema):** accessed through TinyTools’ `Database`/`Repository` abstractions. Configurable via `mysql` node-config object and environment. Holds dashboard user accounts, relations, profile data, groups, permissions, and SSO links. Transactions are required for destructive actions (delete account, create external accounts).
- **Kong (`kong-js`):** `KongService` manages OAuth clients, issues access/refresh tokens (`TokenService`), refreshes tokens, and derives the authenticated user id. `KongValidationService` authenticates admin requests via headers. Config requires admin/api base URLs and consumer/scopes.
- **TinyTools (`tiny-backend-tools`):** supplies cross-cutting infrastructure—`TinyDatabaseAppAuthenticated`, Awilix container wiring, `ValidationMiddleware`, `PasswordService`, `TotpService`, `Cron` logger integration, serializer + error middlewares, and context propagation utilities. `TinyDatabaseAppAuthenticated` also injects a `Database` instance into repositories.
- **Tiny Testing (`tiny-testing`):** provides DB setup helpers and the `AuthenticationTester` used in integration tests.
- **Other libraries:** `class-validator`/`class-transformer` enforce DTO constraints; `google-libphonenumber` formats Dutch phone numbers; `moment` and `winston` support tests/logging.
- **Cross-service contracts:** `docs/wonkers-accounts.yaml` describes the API and is consumed by other repos. Internal callers depend on `/internal/v3/admin/groups`, `/internal/v3/admin/permissions`, `/internal/v3/admin/accounts/:accountId/validate`, and the SSO lookup/create endpoints to make authorization decisions; breaking these contracts has cascading effects on wonkers-graphql, wonkers-api, and automation services.

## Gaps & Risks
- `src/service/AccountService.ts`: `patchAccount` assigns `const success = this.validPassword(...)` without `await`, so the Promise is always truthy and invalid passwords can slip through MFA enabling. This is a security bug and should be fixed by awaiting the password check.
- `src/service/GroupService.ts`: `updateGroup` calls `return this.getGroupByName(name)` with the original path-param name even after renaming, which will throw `NotFoundError` or return stale data when `update.name` differs. It should fetch by the new name (or id) after updates.
- `/internal/v3/admin/...` routes (groups, permissions, account validation/creation) currently lack any authentication middleware, so any caller within the network can mutate RBAC data. Confirm that network-level ACLs exist or add service-to-service auth.
- `test/controller/AccountControllerIT.ts` and other fixtures hard-code the root MySQL password (`ICgVcbpYW731vY3UjexgAnuQ69Wv2DdN`). This is effectively a secret in source control—move credentials to env vars or a secrets manager.
- `AccountService.verifyAdmin` logs `console.log('check if external account', isExternalAccount, username, dashboardAccount)` which can leak sensitive account payloads (including password hashes) to stdout. Replace with structured logging that redacts secrets.
