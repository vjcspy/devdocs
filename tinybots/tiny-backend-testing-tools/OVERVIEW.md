> **Branch:** master
> **Last Commit:** 0d17cfd
> **Last Updated:** 2025-12-04

# tiny-backend-testing-tools

## TL;DR
- Shared integration-test kit for TinyBots Node.js services: MySQL fixture builders, Kong auth probes, and assertion helpers.
- Targets two schemas: `dashboard` (accounts, relations, subscriptions, permissions) and `tinybots` (scripts, interactions).
- Ships as a git dependency consumed by megazord-events, m-o-triggers, wonkers-* apps, and sensara-adaptor to keep DB test setup consistent.

## Repo Purpose & Bounded Context
- Provides deterministic MySQL seeding helpers so services can spin up representative dashboard/app/script data in integration tests.
- Supplies `AuthenticationTester` to assert Kong header enforcement across admin and integration flows without duplicating boilerplate.
- Exposes minimal test utility surface (`expect`, `createRobot`, DTOs) to keep downstream tests consistent with TinyBots domain models.

## Project Structure
```
lib/
  index.ts                      # Public exports for consumers
  controller/AuthenticationTester.ts
  database/
    DbSetup.ts                  # MySQL wrapper on tiny-tools Database
    DashboardAccountDbSetup.ts  # Relations, users, activations, robots, integrations
    DashboardRobotSetup.ts      # Payment reference + robot relation helpers
    BuySubscriptionDbSetup.ts   # Subscription chains, extensions, cancellations
    AppAccountDbSetup.ts        # App users, robots, anonymised UUIDs
    ScriptsDbSetup.ts           # Script references, versions, steps, nodes, schedules
    InteractionDbSetup.ts       # Speech interactions + enable/disable toggles
    PermissionDbSetup.ts        # Groups, permissions, and admin seeding
  utils/test-util.ts            # chai config, wait(), Robot factory, DTOs
test/
  controller/AuthenticationTesterTest.ts
  repository/*.ts               # DB setup integration tests
  fixtures/fullv2Script.json    # Example script graph for ScriptsDbSetup
ci/
  docker-compose.yml, test.sh   # Spins MySQL schemas (wonkers-db, typ-e) for CI runs
package.json, tsconfig.json, yarn.lock
```

## Controllers & Public Surface
- `AuthenticationTester` builds supertest requests with Kong headers and asserts role-specific access:
  - `testAdminAccess(adminId, userId)` expects dashboard-user-only access; rejects app/integration users.
  - `testIntegrationSingleAccess(adminId, userId)` expects integration-user-only access; rejects dashboard/app users.
- `index.ts` re-exports the tester, DB setup classes, `expect`, and `createRobot` so consumers import from the package root.

## Core Services & Logic
- **DbSetup base**: wraps `tiny-tools` `Database` over `mysql2` pools, offering `query`/`close`.
- **DashboardAccountDbSetup**: seeds relations (creates type rows on demand), dashboard users (role-aware), activations, integrations, and robots.
- **DashboardRobotSetup**: aligns robots to relations and payment references; ensures reference records exist.
- **BuySubscriptionDbSetup**: creates subscription chains, extensions, cancellations; auto-calculates periods with `moment`; ties chains to robots and relations, optionally provisioning a new robot.
- **AppAccountDbSetup**: seeds `user_account`/`robot_account`, anonymised UUIDs, and user-robot links with default hashed credentials.
- **ScriptsDbSetup**: constructs full script graphs (references, versions, steps, nodes, next relations, categories, schedules); validates language tags and categories; supports multiple node/next types (`say`, `wait`, `goto`, `then`, `closedQuestion`, `multipleChoice`).
- **InteractionDbSetup**: builds speech interactions, default interaction versions, enable/disable toggles per robot/base ref, and command lists; leverages `tiny-types` `SpeechInteraction`.
- **PermissionDbSetup**: seeds permissions, groups, mappings, and an admin user with specified permissions.
- **Utilities**: `createRobot` fabricates dashboard robot payloads; `wait` helper for async timing; `expect` preconfigured with `chai` + `chai-as-promised`.

## External Dependencies & Cross-Service Contracts
- Relies on `tiny-tools` (DB wrapper, controller base classes) and `kong-js` for Kong header schemas; consumers must provide Kong backends for validation in live environments.
- Depends on MySQL schemas:
  - `dashboard` for account, relation, subscription, and permission tables.
  - `tinybots` for script and interaction tables.
- Uses `tiny-types` domain DTOs (e.g., `SpeechInteraction`) to stay aligned with shared models.
- CI scripts pull AWS ECR images and run `ci/docker-compose.yml` services (`wonkers-db`, `typ-e`, `tiny-backend-testing-tools`) to provision schemas before tests.

## Testing & Quality Gates
- Test runner: `mocha` via `nyc` with `ts-node/register`; commands: `yarn test` (all) or `yarn unit-test` (Test-suffixed).
- Coverage thresholds: statements 55%, functions 55%, branches 30/45%, lines 60%; HTML report emitted by nyc.
- Integration tests expect live MySQL endpoints provided via env:
  - `DB_RW_HOST_DASHBOARD` for dashboard schema.
  - `DB_RW_HOST_TINYBOTS` for script/interaction schema.
- Key suites:
  - `test/controller/AuthenticationTesterTest.ts` mocks Kong upstreams with `nock` and validates role enforcement.
  - Repository specs seed and clean data across dashboard, subscription, permission, script, and interaction flows; fixture `fullv2Script.json` drives end-to-end script creation.
- CI helpers (`ci/test.sh`, `ci/local-test.sh`) boot docker-compose services and attach to containers; require AWS and Docker credentials when running outside local dev.
