## TL;DR

- TinyBots is a set of backend services and Wonkers apps for telemetry, automation, and TaaS order flows.
- This overview lists all 23 discovered repositories grouped by role with links to their repo overviews.
- Use the coverage table to spot missing repo overviews quickly.

## Platform Purpose & Landscape

- Telemetry from robots and partners is ingested, evaluated, and dispatched to downstream automation and order flows.
- Customer and admin surfaces expose REST and GraphQL gateways backed by shared schemas and database migrations.
- Shared libraries and tooling standardize middleware, HTTP clients, and contracts for consistent integrations.

## Services:

### Automation Core

- azi-3-status-check: Status check API and evaluation workflows (devdocs/tinybots/azi-3-status-check/OVERVIEW.md)
- eve: Schedule management service for Tessa robots (devdocs/tinybots/eve/OVERVIEW.md)
- herbie: Background cleanup of expired user and robot accounts (devdocs/tinybots/herbie/OVERVIEW.md)
- m-o-triggers: Trigger scheduling and queue fan-out (devdocs/tinybots/m-o-triggers/OVERVIEW.md)
- megazord-events: Event intake and fan-out for robot telemetry (devdocs/tinybots/megazord-events/OVERVIEW.md)
- micro-manager: Robot script lifecycle and execution service (devdocs/tinybots/micro-manager/OVERVIEW.md)
- sensara-adaptor: Sensara telemetry bridge into TinyBots events (devdocs/tinybots/sensara-adaptor/OVERVIEW.md)
- wadsworth: Speech interaction service mapping voice commands to scripts (devdocs/tinybots/wadsworth/OVERVIEW.md)
- wonkers-nedap: Nedap ONS integration syncing orders and returns into TaaS flows (devdocs/tinybots/wonkers-nedap/OVERVIEW.md)

### Experience & Business Apps

- wonkers-api: Dashboard REST API for customers and admins (devdocs/tinybots/wonkers-api/OVERVIEW.md)
- wonkers-accounts: Accounts, permissions, and login flows (devdocs/tinybots/wonkers-accounts/OVERVIEW.md)
- wonkers-robots: Robot inventory and admin management (devdocs/tinybots/wonkers-robots/OVERVIEW.md)
- wonkers-taas-orders: TaaS order lifecycle management (devdocs/tinybots/wonkers-taas-orders/OVERVIEW.md)
- wonkers-graphql: GraphQL gateway for reporting across TinyBots data (devdocs/tinybots/wonkers-graphql/OVERVIEW.md)
- wonkers-ecd: Ecare and ZSP notification bridge into TaaS order flows (devdocs/tinybots/wonkers-ecd/OVERVIEW.md)

### Shared Libraries, Tooling & Schemas

- atlas: Batch jobs that anonymise and copy typ-e data to the intelligence database (devdocs/tinybots/atlas/OVERVIEW.md)
- cves-scan: Vulnerability scan tooling and CI scripts (devdocs/tinybots/cves-scan/OVERVIEW.md)
- tiny-backend-tools: Shared Node.js service scaffolding and middleware (devdocs/tinybots/tiny-backend-tools/OVERVIEW.md)
- tiny-internal-services: Shared DTOs and HTTP clients for TinyBots services (devdocs/tinybots/tiny-internal-services/OVERVIEW.md)
- tiny-internal-services-mocks: Mocks and stubs for integration tests (devdocs/tinybots/tiny-internal-services-mocks/OVERVIEW.md)
- tiny-specs: OpenAPI specs and generated validators (devdocs/tinybots/tiny-specs/OVERVIEW.md)
- typ-e: MySQL schema and Flyway migrations for robot scheduling and automation data (devdocs/tinybots/typ-e/OVERVIEW.md)
- wonkers-db: MySQL schema and Flyway migrations for dashboard and TaaS order data (devdocs/tinybots/wonkers-db/OVERVIEW.md)

## Cross-Service Data Flows

- Telemetry from Sensara and robots enters sensara-adaptor and megazord-events, then dispatches triggers to m-o-triggers and status workflows.
- Status evaluations update databases and publish queue messages surfaced through wonkers-api, wonkers-graphql, and micro-manager script executions.
- Robot schedules are managed by eve and stored in typ-e database, with downstream notifications to keep automation in sync.
- Voice commands from robots are resolved by wadsworth, mapping spoken phrases to scripts validated and executed by micro-manager.
- External notifications from Ecare and ZSP pass through wonkers-ecd into TaaS order and dashboard flows using shared schemas and clients.
- Nedap ONS survey results are polled by wonkers-nedap, mapped into concept DTOs, and pushed into wonkers-taas-orders.
- Script and execution data is copied from typ-e into the intelligence database by atlas with anonymisation applied for analytics use.

## Operational Notes & Testing

- Most services are Node.js/TypeScript (Yarn); schema repos use Java/Maven.
- Shared middleware and scaffolding live in tiny-backend-tools; shared clients and DTOs live in tiny-internal-services.
- Run tests via the centralized DevTools commands from the workspace root: `just -f devtools/tinybots/local/Justfile test-<repo>`.
- Keep repo overviews under `devdocs/tinybots/<repo>/OVERVIEW.md` up to date; missing overviews should be added when the repo is actively worked on.

## DevTools Infrastructure

TinyBots uses a centralized DevTools infrastructure located at `devtools/tinybots/local/`. Key capabilities:

| Feature | Description |
|---------|-------------|
| **Docker Compose** | Single `docker-compose.yaml` defining all databases, migration runners, and shared services |
| **Just Commands** | Repository-specific commands for starting dependencies, running tests, and dev mode |
| **Seed Data** | NPM scripts to populate databases with realistic test data |
| **Database Services** | MySQL containers for typ-e-db, wonkers-db, and atlas-intelligence-db |

**Quick Commands:**

```bash
# Run tests for a repository
just -f devtools/tinybots/local/Justfile test-<repo>

# Start dependencies for local development
just -f devtools/tinybots/local/Justfile start-<repo>

# View service logs
just -f devtools/tinybots/local/Justfile log-<repo>
```

> **For detailed setup and all available commands**, see: `devdocs/misc/devtools/tinybots/OVERVIEW.md`

## Database Access

For tasks requiring database work (schema changes, queries, migrations), query the database directly to get current schema context before proposing changes.

The project uses MySQL databases running via Docker Compose (`devtools/tinybots/local/docker-compose.yaml`):

| Database | Service Name | Host (from host machine) | Port | Database Name | Root Password |
|----------|-------------|--------------------------|------|---------------|---------------|
| typ-e-db | mysql-typ-e-db | localhost | 1123 | tinybots | ICgVcbpYW731vY3UjexgAnuQ69Wv2DdN |
| wonkers-db | mysql-wonkers-db | localhost | 1124 | dashboard | ICgVcbpYW731vY3UjexgAnuQ69Wv2DdN |
| atlas-intelligence-db | mysql-atlas-intelligence-db | localhost | 1126 | analytics | ICgVcbpYW731vY3UjexgAnuQ69Wv2DdN |

How to query database context:

Step 1: ensure Docker services are running

```bash
cd devtools/tinybots/local && docker compose ps
```

If not running, start them:

```bash
cd devtools/tinybots/local && docker compose up -d mysql-typ-e-db mysql-wonkers-db
```

Step 2: query database schema/data

```bash
# Connect to typ-e-db (tinybots database) - interactive mode
docker exec -it mysql-typ-e-db mysql -u root -pICgVcbpYW731vY3UjexgAnuQ69Wv2DdN tinybots

# Connect to wonkers-db (dashboard database) - interactive mode
docker exec -it mysql-wonkers-db mysql -u root -pICgVcbpYW731vY3UjexgAnuQ69Wv2DdN dashboard
```

Step 3: one-liner examples (non-interactive)

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

Step 4: common queries for context gathering

```sql
SHOW TABLES;

DESCRIBE table_name;

SHOW CREATE TABLE table_name;

SELECT COLUMN_NAME, DATA_TYPE, IS_NULLABLE, COLUMN_DEFAULT, COLUMN_KEY
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_SCHEMA = 'tinybots' AND TABLE_NAME = 'your_table';

SHOW TABLES LIKE '%pattern%';

SELECT
    TABLE_NAME, COLUMN_NAME, CONSTRAINT_NAME,
    REFERENCED_TABLE_NAME, REFERENCED_COLUMN_NAME
FROM INFORMATION_SCHEMA.KEY_COLUMN_USAGE
WHERE REFERENCED_TABLE_NAME IS NOT NULL
  AND TABLE_SCHEMA = 'tinybots';
```

## Test Execution

> **CRITICAL:** All TinyBots tests can ONLY run inside Docker containers via `just` commands. Do NOT run tests directly on your local machine—they will fail due to missing infrastructure dependencies.

```bash
# Run tests for a repository
just -f devtools/tinybots/local/Justfile test-<repo>
```

**For complete test execution instructions, troubleshooting, and available repositories**, see: `devdocs/agent/rules/tinybots/run-tests.md`

## Writing Tests

When writing new tests for TinyBots repositories, follow the testing guidelines skill for assertion patterns and best practices:

- **Skill:** `devdocs/agent/skills/tinybots/testing-guidelines/SKILL.md`
- Use `deep.include` for object assertions (not individual field checks)
- Follow Arrange-Act-Assert pattern

Repository Coverage Table

| Repository | Service Group | Overview Path | Status |
| --- | --- | --- | --- |
| atlas | Shared Libraries, Tooling & Schemas | devdocs/tinybots/atlas/OVERVIEW.md | Present |
| azi-3-status-check | Automation Core | devdocs/tinybots/azi-3-status-check/OVERVIEW.md | Present |
| cves-scan | Shared Libraries, Tooling & Schemas | devdocs/tinybots/cves-scan/OVERVIEW.md | Missing |
| eve | Automation Core | devdocs/tinybots/eve/OVERVIEW.md | Present |
| herbie | Automation Core | devdocs/tinybots/herbie/OVERVIEW.md | Missing |
| m-o-triggers | Automation Core | devdocs/tinybots/m-o-triggers/OVERVIEW.md | Present |
| megazord-events | Automation Core | devdocs/tinybots/megazord-events/OVERVIEW.md | Present |
| micro-manager | Automation Core | devdocs/tinybots/micro-manager/OVERVIEW.md | Present |
| sensara-adaptor | Automation Core | devdocs/tinybots/sensara-adaptor/OVERVIEW.md | Present |
| wadsworth | Automation Core | devdocs/tinybots/wadsworth/OVERVIEW.md | Present |
| wonkers-nedap | Automation Core | devdocs/tinybots/wonkers-nedap/OVERVIEW.md | Present |
| wonkers-api | Experience & Business Apps | devdocs/tinybots/wonkers-api/OVERVIEW.md | Present |
| wonkers-accounts | Experience & Business Apps | devdocs/tinybots/wonkers-accounts/OVERVIEW.md | Present |
| wonkers-ecd | Experience & Business Apps | devdocs/tinybots/wonkers-ecd/OVERVIEW.md | Present |
| wonkers-graphql | Experience & Business Apps | devdocs/tinybots/wonkers-graphql/OVERVIEW.md | Present |
| wonkers-robots | Experience & Business Apps | devdocs/tinybots/wonkers-robots/OVERVIEW.md | Present |
| wonkers-taas-orders | Experience & Business Apps | devdocs/tinybots/wonkers-taas-orders/OVERVIEW.md | Present |
| tiny-backend-tools | Shared Libraries, Tooling & Schemas | devdocs/tinybots/tiny-backend-tools/OVERVIEW.md | Present |
| tiny-internal-services | Shared Libraries, Tooling & Schemas | devdocs/tinybots/tiny-internal-services/OVERVIEW.md | Present |
| tiny-internal-services-mocks | Shared Libraries, Tooling & Schemas | devdocs/tinybots/tiny-internal-services-mocks/OVERVIEW.md | Present |
| tiny-specs | Shared Libraries, Tooling & Schemas | devdocs/tinybots/tiny-specs/OVERVIEW.md | Missing |
| typ-e | Shared Libraries, Tooling & Schemas | devdocs/tinybots/typ-e/OVERVIEW.md | Present |
| wonkers-db | Shared Libraries, Tooling & Schemas | devdocs/tinybots/wonkers-db/OVERVIEW.md | Present |
