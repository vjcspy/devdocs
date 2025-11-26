## TL;DR
- TinyBots spans automation ingest/trigger services, Wonkers-facing apps, and shared tooling and schema repos.
- This overview lists all 19 repositories with their service groups and overview paths to route work quickly.
- Repo-specific docs live under devdocs/tinybots/<repo>/OVERVIEW.md; add any missing ones.

## Platform Purpose & Landscape
- Telemetry from robots and partners is ingested, evaluated, and fanned out to downstream apps and queues.
- Wonkers customer and admin surfaces rely on REST/GraphQL gateways backed by shared schemas and validators.
- Shared libraries and schemas keep contracts, scaffolding, and mock tooling aligned across services.

## Services:
### Automation Core
- megazord-events: Event intake and fan-out for robot telemetry (devdocs/tinybots/megazord-events/OVERVIEW.md)
- m-o-triggers: Trigger scheduling and queue fan-out (devdocs/tinybots/m-o-triggers/OVERVIEW.md)
- sensara-adaptor: Sensara telemetry bridge into TinyBots events (devdocs/tinybots/sensara-adaptor/OVERVIEW.md)
- azi-3-status-check: Status check API and evaluation workflows (devdocs/tinybots/azi-3-status-check/OVERVIEW.md)
- azi-3-status-check-jobs: Background jobs for status checks and follow-ups (devdocs/tinybots/azi-3-status-check-jobs/OVERVIEW.md)
- micro-manager: Robot script lifecycle and execution service (devdocs/tinybots/micro-manager/OVERVIEW.md)

### Experience & Business Apps
- wonkers-api: Dashboard REST API for customers and admins (devdocs/tinybots/wonkers-api/OVERVIEW.md)
- wonkers-accounts: Accounts, permissions, and login flows (devdocs/tinybots/wonkers-accounts/OVERVIEW.md)
- wonkers-buy-subscriptions: Subscription purchase flows (devdocs/tinybots/wonkers-buy-subscriptions/OVERVIEW.md)
- wonkers-robots: Robot inventory and admin management (devdocs/tinybots/wonkers-robots/OVERVIEW.md)
- wonkers-taas-orders: TaaS order lifecycle management (devdocs/tinybots/wonkers-taas-orders/OVERVIEW.md)
- wonkers-taas-order-activation: Activation jobs for TaaS orders (devdocs/tinybots/wonkers-taas-order-activation/OVERVIEW.md)
- wonkers-graphql: GraphQL gateway for reporting across TinyBots data (devdocs/tinybots/wonkers-graphql/OVERVIEW.md)

### Shared Libraries, Tooling & Schemas
- tiny-backend-tools: Shared Node.js service scaffolding and middleware (devdocs/tinybots/tiny-backend-tools/OVERVIEW.md)
- tiny-internal-services: Shared DTOs and HTTP clients for TinyBots services (devdocs/tinybots/tiny-internal-services/OVERVIEW.md)
- tiny-internal-services-mocks: Mocks and stubs for integration tests (devdocs/tinybots/tiny-internal-services-mocks/OVERVIEW.md)
- tiny-specs: OpenAPI specs and generated validators (devdocs/tinybots/tiny-specs/OVERVIEW.md)
- typ-e: Java-based MySQL schema definitions for core robots (devdocs/tinybots/typ-e/OVERVIEW.md)
- wonkers-db: Dashboard and TaaS MySQL schema management (devdocs/tinybots/wonkers-db/OVERVIEW.md)

## Cross-Service Data Flows
- Telemetry from Sensara and robots enters sensara-adaptor then megazord-events, persisting events and dispatching triggers to m-o-triggers and status workflows.
- Status evaluations publish queue messages and database updates consumed by azi-3-status-check-jobs and surfaced through wonkers APIs, GraphQL, and micro-manager script executions.
- Customer and admin apps query wonkers-api and wonkers-graphql using schemas from tiny-specs, typ-e, and wonkers-db via shared clients.

## Operational Notes & Testing
- Node.js and TypeScript services use Yarn 3; schema repositories use Java and Maven.
- Kong headers and middleware from tiny-backend-tools provide auth and request context; tiny-internal-services supplies HTTP clients and DTOs; tiny-internal-services-mocks supports isolated tests.
- SQS queues, cron jobs, and long-lived connectors power automation; validate OpenAPI and schema compatibility against tiny-specs and database schemas before deploys.
- Ensure repo-specific runbooks and endpoints stay current under devdocs/tinybots/<repo>/OVERVIEW.md, especially for missing overview files.

Repository Coverage Table

| Repository | Service Group | Overview Path | Status |
| --- | --- | --- | --- |
| megazord-events | Automation Core | devdocs/tinybots/megazord-events/OVERVIEW.md | Present |
| m-o-triggers | Automation Core | devdocs/tinybots/m-o-triggers/OVERVIEW.md | Present |
| sensara-adaptor | Automation Core | devdocs/tinybots/sensara-adaptor/OVERVIEW.md | Present |
| azi-3-status-check | Automation Core | devdocs/tinybots/azi-3-status-check/OVERVIEW.md | Present |
| azi-3-status-check-jobs | Automation Core | devdocs/tinybots/azi-3-status-check-jobs/OVERVIEW.md | Present |
| micro-manager | Automation Core | devdocs/tinybots/micro-manager/OVERVIEW.md | Present |
| wonkers-api | Experience & Business Apps | devdocs/tinybots/wonkers-api/OVERVIEW.md | Present |
| wonkers-accounts | Experience & Business Apps | devdocs/tinybots/wonkers-accounts/OVERVIEW.md | Present |
| wonkers-buy-subscriptions | Experience & Business Apps | devdocs/tinybots/wonkers-buy-subscriptions/OVERVIEW.md | Missing |
| wonkers-robots | Experience & Business Apps | devdocs/tinybots/wonkers-robots/OVERVIEW.md | Present |
| wonkers-taas-orders | Experience & Business Apps | devdocs/tinybots/wonkers-taas-orders/OVERVIEW.md | Present |
| wonkers-taas-order-activation | Experience & Business Apps | devdocs/tinybots/wonkers-taas-order-activation/OVERVIEW.md | Missing |
| wonkers-graphql | Experience & Business Apps | devdocs/tinybots/wonkers-graphql/OVERVIEW.md | Present |
| tiny-backend-tools | Shared Libraries, Tooling & Schemas | devdocs/tinybots/tiny-backend-tools/OVERVIEW.md | Present |
| tiny-internal-services | Shared Libraries, Tooling & Schemas | devdocs/tinybots/tiny-internal-services/OVERVIEW.md | Present |
| tiny-internal-services-mocks | Shared Libraries, Tooling & Schemas | devdocs/tinybots/tiny-internal-services-mocks/OVERVIEW.md | Present |
| tiny-specs | Shared Libraries, Tooling & Schemas | devdocs/tinybots/tiny-specs/OVERVIEW.md | Missing |
| typ-e | Shared Libraries, Tooling & Schemas | devdocs/tinybots/typ-e/OVERVIEW.md | Missing |
| wonkers-db | Shared Libraries, Tooling & Schemas | devdocs/tinybots/wonkers-db/OVERVIEW.md | Missing |
