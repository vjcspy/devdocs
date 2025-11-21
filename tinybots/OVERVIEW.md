## TL;DR
- TinyBots spans automation services for robot telemetry and triggers, Wonkers-facing business apps, and shared tooling and schema repos.
- This overview lists 18 repositories, grouped for routing work to the right surface.
- Repo-specific docs live under devdocs/tinybots/<repo>/OVERVIEW.md; missing entries should be backfilled.

## Platform Purpose & Landscape
- Robots and external feeds emit data that is ingested, scheduled, and surfaced through internal APIs and dashboards.
- Automation services handle event intake, trigger orchestration, and status pipelines; business apps power subscriptions, orders, and reporting.
- Shared libraries provide HTTP scaffolding, DTOs, and schema definitions so services stay aligned on contracts and testing.

## Services:
### Automation Core
- megazord-events: Event intake and fan-out for robot telemetry; overview: devdocs/tinybots/megazord-events/OVERVIEW.md.
- m-o-triggers: Trigger scheduling and SQS fan-out; overview: devdocs/tinybots/m-o-triggers/OVERVIEW.md.
- sensara-adaptor: Sensara telemetry bridge into TinyBots events; overview: devdocs/tinybots/sensara-adaptor/OVERVIEW.md.
- azi-3-status-check: Status check API and evaluation workflows; overview: devdocs/tinybots/azi-3-status-check/OVERVIEW.md.
- azi-3-status-jobs: Background jobs for status checks and follow-ups; overview: devdocs/tinybots/azi-3-status-jobs/OVERVIEW.md.

### Experience & Business Apps
- wonkers-api: Dashboard REST API for customers and admins; overview: devdocs/tinybots/wonkers-api/OVERVIEW.md.
- wonkers-accounts: Accounts, permissions, and login flows; overview: devdocs/tinybots/wonkers-accounts/OVERVIEW.md.
- wonkers-buy-subscriptions: Purchase flows for subscriptions; overview: devdocs/tinybots/wonkers-buy-subscriptions/OVERVIEW.md.
- wonkers-robots: Robot inventory and admin management; overview: devdocs/tinybots/wonkers-robots/OVERVIEW.md.
- wonkers-taas-orders: TaaS order lifecycle management; overview: devdocs/tinybots/wonkers-taas-orders/OVERVIEW.md.
- wonkers-taas-order-activation: Activation jobs for TaaS orders; overview: devdocs/tinybots/wonkers-taas-order-activation/OVERVIEW.md.
- wonkers-graphql: GraphQL gateway for reporting across TinyBots data; overview: devdocs/tinybots/wonkers-graphql/OVERVIEW.md.

### Shared Libraries, Tooling & Schemas
- tiny-backend-tools: Shared Node.js service scaffolding and middleware; overview: devdocs/tinybots/tiny-backend-tools/OVERVIEW.md.
- tiny-internal-services: Shared DTOs and HTTP clients for TinyBots services; overview: devdocs/tinybots/tiny-internal-services/OVERVIEW.md.
- tiny-internal-services-mocks: Mocks and stubs for integration tests; overview: devdocs/tinybots/tiny-internal-services-mocks/OVERVIEW.md.
- tiny-specs: OpenAPI specs and generated validators; overview: devdocs/tinybots/tiny-specs/OVERVIEW.md.
- typ-e: Java-based MySQL schema definitions for core robots; overview: devdocs/tinybots/typ-e/OVERVIEW.md.
- wonkers-db: Dashboard and TaaS MySQL schema management; overview: devdocs/tinybots/wonkers-db/OVERVIEW.md.

## Cross-Service Data Flows
- Telemetry flows from Sensara and robots into sensara-adaptor and megazord-events, which persist events and dispatch triggers to m-o-triggers and status workflows.
- Triggered actions and status evaluations publish queue messages and database updates that downstream jobs in azi-3-status-jobs and experience services consume.
- Customer and admin apps query wonkers services and wonkers-graphql, pulling data shaped by schemas defined in typ-e and wonkers-db and contracts from tiny-specs.

## Operational Notes & Testing
- Services are Node.js or TypeScript with Yarn 3; schema repos use Java and Maven.
- Kong headers and shared middleware from tiny-backend-tools provide auth and request context.
- SQS queues, cron jobs, and long-lived connectors power automation; mocks under tiny-internal-services-mocks support isolated tests.
- Repo-specific runbooks and endpoints live under devdocs/tinybots/<repo>/OVERVIEW.md and should be added where missing.

Repository Coverage Table
| Repository | Service Group | Overview Path | Status |
| --- | --- | --- | --- |
| azi-3-status-check | Automation Core | devdocs/tinybots/azi-3-status-check/OVERVIEW.md | Present |
| azi-3-status-jobs | Automation Core | devdocs/tinybots/azi-3-status-jobs/OVERVIEW.md | Present |
| m-o-triggers | Automation Core | devdocs/tinybots/m-o-triggers/OVERVIEW.md | Present |
| megazord-events | Automation Core | devdocs/tinybots/megazord-events/OVERVIEW.md | Present |
| sensara-adaptor | Automation Core | devdocs/tinybots/sensara-adaptor/OVERVIEW.md | Present |
| tiny-backend-tools | Shared Libraries, Tooling & Schemas | devdocs/tinybots/tiny-backend-tools/OVERVIEW.md | Present |
| tiny-internal-services | Shared Libraries, Tooling & Schemas | devdocs/tinybots/tiny-internal-services/OVERVIEW.md | Present |
| tiny-internal-services-mocks | Shared Libraries, Tooling & Schemas | devdocs/tinybots/tiny-internal-services-mocks/OVERVIEW.md | Missing |
| tiny-specs | Shared Libraries, Tooling & Schemas | devdocs/tinybots/tiny-specs/OVERVIEW.md | Missing |
| typ-e | Shared Libraries, Tooling & Schemas | devdocs/tinybots/typ-e/OVERVIEW.md | Missing |
| wonkers-accounts | Experience & Business Apps | devdocs/tinybots/wonkers-accounts/OVERVIEW.md | Present |
| wonkers-api | Experience & Business Apps | devdocs/tinybots/wonkers-api/OVERVIEW.md | Missing |
| wonkers-buy-subscriptions | Experience & Business Apps | devdocs/tinybots/wonkers-buy-subscriptions/OVERVIEW.md | Missing |
| wonkers-db | Shared Libraries, Tooling & Schemas | devdocs/tinybots/wonkers-db/OVERVIEW.md | Missing |
| wonkers-graphql | Experience & Business Apps | devdocs/tinybots/wonkers-graphql/OVERVIEW.md | Present |
| wonkers-robots | Experience & Business Apps | devdocs/tinybots/wonkers-robots/OVERVIEW.md | Missing |
| wonkers-taas-order-activation | Experience & Business Apps | devdocs/tinybots/wonkers-taas-order-activation/OVERVIEW.md | Missing |
| wonkers-taas-orders | Experience & Business Apps | devdocs/tinybots/wonkers-taas-orders/OVERVIEW.md | Missing |
