## TL;DR

- TinyBots spans telemetry ingest, automation triggers, business-facing Wonkers apps, and shared tooling that keeps contracts aligned.
- This overview lists all 22 repositories grouped by role with links to their repo-level overviews.
- Use it to spot missing docs and route work to the right codebase quickly.

## Platform Purpose & Landscape

- Telemetry from robots and partners is ingested, evaluated, and dispatched to downstream automation and order flows.
- Customer and admin surfaces expose REST and GraphQL gateways backed by shared schemas and validators.
- Shared libraries standardize middleware, HTTP clients, and schemas for consistent contracts across services.

## Services

### Automation Core

- megazord-events: Event intake and fan-out for robot telemetry (devdocs/tinybots/megazord-events/OVERVIEW.md)
- m-o-triggers: Trigger scheduling and queue fan-out (devdocs/tinybots/m-o-triggers/OVERVIEW.md)
- sensara-adaptor: Sensara telemetry bridge into TinyBots events (devdocs/tinybots/sensara-adaptor/OVERVIEW.md)
- azi-3-status-check: Status check API and evaluation workflows (devdocs/tinybots/azi-3-status-check/OVERVIEW.md)
- azi-3-status-check-jobs: Background jobs for status checks and follow-ups (devdocs/tinybots/azi-3-status-check-jobs/OVERVIEW.md)
- eve: Schedule management service for Tessa robots (devdocs/tinybots/eve/OVERVIEW.md)
- micro-manager: Robot script lifecycle and execution service (devdocs/tinybots/micro-manager/OVERVIEW.md)
- wadsworth: Speech interaction service mapping voice commands to scripts (devdocs/tinybots/wadsworth/OVERVIEW.md)

### Experience & Business Apps

- wonkers-api: Dashboard REST API for customers and admins (devdocs/tinybots/wonkers-api/OVERVIEW.md)
- wonkers-accounts: Accounts, permissions, and login flows (devdocs/tinybots/wonkers-accounts/OVERVIEW.md)
- wonkers-buy-subscriptions: Subscription purchase flows (devdocs/tinybots/wonkers-buy-subscriptions/OVERVIEW.md)
- wonkers-robots: Robot inventory and admin management (devdocs/tinybots/wonkers-robots/OVERVIEW.md)
- wonkers-taas-orders: TaaS order lifecycle management (devdocs/tinybots/wonkers-taas-orders/OVERVIEW.md)
- wonkers-taas-order-activation: Activation jobs for TaaS orders (devdocs/tinybots/wonkers-taas-order-activation/OVERVIEW.md)
- wonkers-graphql: GraphQL gateway for reporting across TinyBots data (devdocs/tinybots/wonkers-graphql/OVERVIEW.md)
- wonkers-ecd: Ecare and ZSP notification bridge into TaaS order flows (devdocs/tinybots/wonkers-ecd/OVERVIEW.md)

### Shared Libraries, Tooling & Schemas

- tiny-backend-tools: Shared Node.js service scaffolding and middleware (devdocs/tinybots/tiny-backend-tools/OVERVIEW.md)
- tiny-backend-testing-tools: MySQL + Kong integration-test fixtures and helpers (devdocs/tinybots/tiny-backend-testing-tools/OVERVIEW.md)
- tiny-internal-services: Shared DTOs and HTTP clients for TinyBots services (devdocs/tinybots/tiny-internal-services/OVERVIEW.md)
- tiny-internal-services-mocks: Mocks and stubs for integration tests (devdocs/tinybots/tiny-internal-services-mocks/OVERVIEW.md)
- tiny-specs: OpenAPI specs and generated validators (devdocs/tinybots/tiny-specs/OVERVIEW.md)
- typ-e: Java-based MySQL schema defined via Flyway migrations in `src/main/resources/db/migration` (devdocs/tinybots/typ-e/OVERVIEW.md)
- wonkers-db: Dashboard and TaaS MySQL schema managed by Flyway migrations in `src/main/resources/db/migration` (devdocs/tinybots/wonkers-db/OVERVIEW.md)

## Cross-Service Data Flows

- Telemetry from Sensara and robots enters sensara-adaptor then megazord-events, persisting events and dispatching triggers to m-o-triggers and status workflows.
- Status evaluations publish queue messages and database updates consumed by azi-3-status-check-jobs and surfaced through wonkers APIs, GraphQL, and micro-manager script executions.
- Robot schedules are managed by eve, storing cron-based task definitions in typ-e database and notifying downstream consumers via SQS when schedules change.
- Voice commands from robots are resolved by wadsworth, matching spoken phrases to script references; supported scripts are validated against micro-manager before execution.
- External notifications from Ecare and ZSP pass through wonkers-ecd to TaaS order services and dashboards using shared schemas and clients.
- Customer and admin apps query wonkers-api and wonkers-graphql using schemas from tiny-specs, typ-e, and wonkers-db via shared clients and middleware.

## Operational Notes & Testing

- Node.js and TypeScript services use Yarn 3; schema repositories use Java and Maven.
- Kong middleware from tiny-backend-tools provides auth and request context; tiny-internal-services supplies HTTP clients and DTOs; tiny-internal-services-mocks supports isolated tests.
- SQS queues, cron jobs, and long-lived connectors power automation; validate OpenAPI and schema compatibility against tiny-specs and database schemas before deploys.
- Keep repo-specific runbooks under devdocs/tinybots/<repo>/OVERVIEW.md up to date, especially for repos currently missing overviews.

## Database Context Guide

For tasks requiring database work (schema changes, queries, migrations), you **MUST** query the database directly to get current schema context before proposing changes.

### TinyBots Databases

The project uses MySQL databases running via Docker Compose (`devtools/docker-compose.yaml`):

| Database | Service Name | Host (from host machine) | Port | Database Name | Root Password |
|----------|-------------|--------------------------|------|---------------|---------------|
| **typ-e-db** | `mysql-typ-e-db` | `localhost` | `1123` | `tinybots` | `ICgVcbpYW731vY3UjexgAnuQ69Wv2DdN` |
| **wonkers-db** | `mysql-wonkers-db` | `localhost` | `1124` | `dashboard` | `ICgVcbpYW731vY3UjexgAnuQ69Wv2DdN` |
| **atlas-intelligence-db** | `mysql-atlas-intelligence-db` | `localhost` | `1126` | `analytics` | `ICgVcbpYW731vY3UjexgAnuQ69Wv2DdN` |

### How to Query Database Context

**Step 1: Ensure Docker services are running**

```bash
cd devtools && docker compose ps
```

If not running, start them:

```bash
cd devtools && docker compose up -d mysql-typ-e-db mysql-wonkers-db
```

**Step 2: Query database schema/data**

Use `docker exec` to run MySQL commands inside the container:

```bash
# Connect to typ-e-db (tinybots database) - interactive mode
docker exec -it mysql-typ-e-db mysql -u root -pICgVcbpYW731vY3UjexgAnuQ69Wv2DdN tinybots

# Connect to wonkers-db (dashboard database) - interactive mode
docker exec -it mysql-wonkers-db mysql -u root -pICgVcbpYW731vY3UjexgAnuQ69Wv2DdN dashboard
```

**Step 3: One-liner examples (non-interactive)**

```bash
# List all tables in typ-e-db
docker exec mysql-typ-e-db mysql -u root -pICgVcbpYW731vY3UjexgAnuQ69Wv2DdN tinybots -e "SHOW TABLES;"

# Describe a specific table
docker exec mysql-typ-e-db mysql -u root -pICgVcbpYW731vY3UjexgAnuQ69Wv2DdN tinybots -e "DESCRIBE users;"

# Get full CREATE TABLE statement
docker exec mysql-typ-e-db mysql -u root -pICgVcbpYW731vY3UjexgAnuQ69Wv2DdN tinybots -e "SHOW CREATE TABLE users\G"

# List all tables in wonkers-db
docker exec mysql-wonkers-db mysql -u root -pICgVcbpYW731vY3UjexgAnuQ69Wv2DdN dashboard -e "SHOW TABLES;"
```

**Step 4: Common queries for context gathering**

```sql
-- List all tables
SHOW TABLES;

-- Get table schema
DESCRIBE table_name;

-- Show CREATE statement (full schema with indexes, constraints)
SHOW CREATE TABLE table_name;

-- List all columns of a table with details
SELECT COLUMN_NAME, DATA_TYPE, IS_NULLABLE, COLUMN_DEFAULT, COLUMN_KEY
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_SCHEMA = 'tinybots' AND TABLE_NAME = 'your_table';

-- Find tables by name pattern
SHOW TABLES LIKE '%pattern%';

-- Check foreign key relationships
SELECT 
    TABLE_NAME, COLUMN_NAME, CONSTRAINT_NAME,
    REFERENCED_TABLE_NAME, REFERENCED_COLUMN_NAME
FROM INFORMATION_SCHEMA.KEY_COLUMN_USAGE
WHERE REFERENCED_TABLE_NAME IS NOT NULL
  AND TABLE_SCHEMA = 'tinybots';
```

### When to Query Database

- **Schema changes:** Before modifying tables, always check current schema
- **New migrations:** Understand existing structure before adding migrations
- **Query optimization:** Check indexes and table structure
- **Data model understanding:** Verify relationships between tables
- **Debugging:** Verify actual data state

## Testing Environment (CRITICAL)

> **All repositories in TinyBots can ONLY run tests inside Docker containers.** Do NOT attempt to run tests directly on your local machine—they will fail due to missing infrastructure dependencies.

### Running Tests

Tests must be executed using the centralized DevTools infrastructure via `just` commands:

```bash
just -f devtools/Justfile test-<REPO_NAME>
```

**Available Repositories:**

| Repository | Command |
|------------|---------|
| `atlas` | `just -f devtools/Justfile test-atlas` |
| `azi-3-status-check-jobs` | `just -f devtools/Justfile test-azi-3-status-check-jobs` |
| `m-o-triggers` | `just -f devtools/Justfile test-m-o-triggers` |
| `megazord-events` | `just -f devtools/Justfile test-megazord-events` |
| `micro-manager` | `just -f devtools/Justfile test-micro-manager` |
| `wonkers-ecd` | `just -f devtools/Justfile test-wonkers-ecd` |
| `wonkers-graphql` | `just -f devtools/Justfile test-wonkers-graphql` |

### Important Notes

- **Execution Time:** Tests take **1-2 minutes** to complete due to Docker container startup, database migrations, and service initialization. **Be patient and wait for the full output.**
- **Do NOT interrupt:** Let the test command finish completely to get accurate results.
- **Prerequisites:** Ensure Docker is running and you have ECR access (see `devdocs/tinybots/devtools/OVERVIEW.md` for setup).
- **Working Directory:** Commands should be run from the project root (`tinybots/`), not from inside the repository folder.

### What Happens When You Run Tests

1. **Database Reset:** Stateful containers (MySQL databases, LocalStack) are removed and recreated
2. **Migration:** Database migrations run automatically
3. **Service Startup:** Required services (checkpoint, prowl, wonkers, etc.) start up
4. **Test Execution:** The test suite runs inside a Docker container with proper network access to all dependencies
5. **Results:** Test output is displayed in terminal

### Troubleshooting Test Failures

```bash
# View logs for a specific repository's services
just -f devtools/Justfile log-<REPO_NAME>

# Check Docker container status
cd devtools && docker compose ps

# Restart from clean state
cd devtools && docker compose down
just -f devtools/Justfile start-db
just -f devtools/Justfile test-<REPO_NAME>
```

For detailed DevTools documentation, see: `devdocs/tinybots/devtools/OVERVIEW.md`

Repository Coverage Table

| Repository | Service Group | Overview Path | Status |
| --- | --- | --- | --- |
| azi-3-status-check | Automation Core | devdocs/tinybots/azi-3-status-check/OVERVIEW.md | Present |
| azi-3-status-check-jobs | Automation Core | devdocs/tinybots/azi-3-status-check-jobs/OVERVIEW.md | Present |
| eve | Automation Core | devdocs/tinybots/eve/OVERVIEW.md | Present |
| m-o-triggers | Automation Core | devdocs/tinybots/m-o-triggers/OVERVIEW.md | Present |
| megazord-events | Automation Core | devdocs/tinybots/megazord-events/OVERVIEW.md | Present |
| micro-manager | Automation Core | devdocs/tinybots/micro-manager/OVERVIEW.md | Present |
| sensara-adaptor | Automation Core | devdocs/tinybots/sensara-adaptor/OVERVIEW.md | Present |
| wadsworth | Automation Core | devdocs/tinybots/wadsworth/OVERVIEW.md | Present |
| wonkers-api | Experience & Business Apps | devdocs/tinybots/wonkers-api/OVERVIEW.md | Present |
| wonkers-accounts | Experience & Business Apps | devdocs/tinybots/wonkers-accounts/OVERVIEW.md | Present |
| wonkers-buy-subscriptions | Experience & Business Apps | devdocs/tinybots/wonkers-buy-subscriptions/OVERVIEW.md | Missing |
| wonkers-ecd | Experience & Business Apps | devdocs/tinybots/wonkers-ecd/OVERVIEW.md | Present |
| wonkers-graphql | Experience & Business Apps | devdocs/tinybots/wonkers-graphql/OVERVIEW.md | Present |
| wonkers-robots | Experience & Business Apps | devdocs/tinybots/wonkers-robots/OVERVIEW.md | Present |
| wonkers-taas-order-activation | Experience & Business Apps | devdocs/tinybots/wonkers-taas-order-activation/OVERVIEW.md | Missing |
| wonkers-taas-orders | Experience & Business Apps | devdocs/tinybots/wonkers-taas-orders/OVERVIEW.md | Present |
| tiny-backend-tools | Shared Libraries, Tooling & Schemas | devdocs/tinybots/tiny-backend-tools/OVERVIEW.md | Present |
| tiny-backend-testing-tools | Shared Libraries, Tooling & Schemas | devdocs/tinybots/tiny-backend-testing-tools/OVERVIEW.md | Present |
| tiny-internal-services | Shared Libraries, Tooling & Schemas | devdocs/tinybots/tiny-internal-services/OVERVIEW.md | Present |
| tiny-internal-services-mocks | Shared Libraries, Tooling & Schemas | devdocs/tinybots/tiny-internal-services-mocks/OVERVIEW.md | Present |
| tiny-specs | Shared Libraries, Tooling & Schemas | devdocs/tinybots/tiny-specs/OVERVIEW.md | Missing |
| typ-e | Shared Libraries, Tooling & Schemas | devdocs/tinybots/typ-e/OVERVIEW.md | Present |
| wonkers-db | Shared Libraries, Tooling & Schemas | devdocs/tinybots/wonkers-db/OVERVIEW.md | Present |
