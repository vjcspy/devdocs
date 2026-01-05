# DevTools - Centralized Development Infrastructure

> **Purpose:** Centralized Docker Compose infrastructure for local development and integration testing across TinyBots repositories
> **Last Updated:** 2026-01-05

## DevTools TL;DR

- **Centralized Docker Infrastructure:** Single `docker-compose.yaml` defining all databases, migration runners, and shared services (Checkpoint, Prowl, LocalStack, Wonkers, etc.) to eliminate duplicated CI setups across repos.
- **Just Command Orchestration:** Repository-specific `just` commands in `justfiles/` provide one-liner commands to start dependencies, run integration tests, or launch dev mode for any repo.
- **Seed Data Management:** NPM scripts (`seed`, `seed:clean`) populate MySQL databases with realistic test data for local development and testing workflows.
- **Speed & Safety:** Only resets stateful containers (databases) between test runs; reuses running services for faster iteration (~1-2min vs full rebuild ~5-10min).

---

## Repo Purpose & Bounded Context

DevTools provides the **local development and integration testing infrastructure** for the entire TinyBots monorepo. It replaces the decentralized approach where each repository managed its own `ci/docker-compose.yml` with a **single source of truth** for:

- Database containers (typ-e-db, wonkers-db, atlas-intelligence-db)
- Migration runners (typ-e, wonkers-db, intelligence)
- Shared backend services (checkpoint, prowl, wonkers, wonkers-account, kryten, etc.)
- AWS mocks (LocalStack for SQS)
- Email testing (mailcatcher)

This eliminates configuration drift, speeds up test cycles, and provides consistent development environments across all repositories.

---

## Key Feature 1: Just Commands for Rapid Development

### Overview

The `Justfile` orchestrates Docker Compose operations through repository-specific command files in `justfiles/`. Each repo gets dedicated commands to:

1. **Start dependencies** (`start-<repo>`) - Launch all required services for development
2. **Run integration tests** (`test-<repo>`) - Reset databases and execute test suites
3. **View logs** (`log-<repo>`) - Monitor service outputs
4. **Dev/Debug modes** (`dev-<repo>`, `debug-<repo>`) - Interactive development with hot-reload

### Available Commands

**Database Management:**
```bash
just start-db  # Start both MySQL databases and run migrations
```

**Per-Repository Commands (Pattern):**
```bash
just start-<repo>    # Start all dependencies for local dev
just test-<repo>     # Run integration tests (resets DBs)
just log-<repo>      # View logs for all services
just dev-<repo>      # Start in dev mode with hot-reload
just debug-<repo>    # Start with Node debugger attached
```

**Available Repos:**
- `atlas`
- `azi-3-status-check-jobs`
- `m-o-triggers`
- `megazord-events`
- `micro-manager`
- `wonkers-ecd`
- `wonkers-graphql`

### Example Usage

```bash
# Local development workflow
cd /path/to/tinybots/devtools
just start-megazord-events  # Starts DBs, migrations, and services
# Make code changes in megazord-events/
just test-megazord-events   # Run tests with clean database state

# View service logs
just log-micro-manager

# Debug mode for interactive development
just debug-wonkers-graphql  # Starts with Node inspector on port 9229
```

### Command Patterns Explained

**`start-<repo>`**: Starts all dependencies but not the test runner itself. Useful for:
- Running the app manually in your IDE with debugger attached
- Iterating on code without running full test suite
- Keeping services running while switching between repos

**`test-<repo>`**: Resets databases to clean state, starts services, and runs tests:
```bash
# Pattern from justfiles/megazord-events.just
test-megazord-events:
    {{compose}} rm -sf mysql-typ-e-db mysql-wonkers-db localstack  # Reset stateful containers
    {{compose}} up -d mysql-typ-e-db mysql-wonkers-db localstack checkpoint prowl wonkers wonkers-account wonkers-db
    {{compose}} run --rm --use-aliases megazord-events  # Run tests
```

**Key Design Decision:** Only stateful containers (databases, LocalStack) are removed between test runs. Application services (checkpoint, prowl, etc.) are reused, dramatically reducing startup time.

### Creating New Just Commands

When adding integration test support for a new repository, follow the detailed guide in:

**→ [devdocs/tinybots/devtools/create-just-commands.md](create-just-commands.md)**

The guide covers:
- Reading CI configuration to extract dependencies
- Mapping legacy container names to DevTools names
- Creating repository-specific `.just` files
- Importing commands into main Justfile
- Service dependency matrix and troubleshooting

---

## Key Feature 2: Database Seeding for Local Testing

### Overview

The `src/seed/` directory provides a TypeScript-based seeding system to populate databases with realistic test data. This eliminates manual SQL imports and ensures consistent baseline data for local development.

### Available Seed Scripts

```bash
# Generate Prisma clients (required before seeding)
npm run generate

# Seed data into databases
npm run seed

# Clean all seeded data
npm run seed:clean

# Seed with specific scope (partial seeding)
npm run seed -- --scope=<scope-name>

# Dry-run mode (preview changes without applying)
npm run seed -- --dry-run
```

### Seed Architecture

**Orchestrator Pattern:** `src/seed/core/Orchestrator.ts` manages execution order and dependencies between seed units.

**Seed Units** (`src/seed/units/`):
- `RobotAccountStatusSeed` - Robot account statuses
- `RobotAccountSeed` - Robot accounts with references
- `EventProviderSeed` - Event providers for megazord-events
- `EventSchemaSeed` - Event schemas
- `IncomingEventCleanerSeed` - Cleans incoming events
- `ScriptReferenceSeed` - Script references for micro-manager
- `ScriptVersionSeed` - Script versions
- `TaskScheduleSeed` - Scheduled tasks
- `ScriptExecutionSeed` - Script execution history

**Context Management:** `src/seed/core/SeedContext.ts` provides database connections via Prisma clients:
- `tinybots` database → `mysql-typ-e-db` (port 1123)
- `dashboard` database → `mysql-wonkers-db` (port 1124)

### Common Workflows

**Initial Setup:**
```bash
cd /path/to/tinybots/devtools
npm install
npm run generate  # Generate Prisma clients
npm run seed      # Populate databases
```

**Reset to Clean State:**
```bash
npm run seed:clean  # Remove all seeded data
npm run seed        # Re-populate
```

**Working with Specific Data:**
```bash
# Seed only robots and related data
npm run seed -- --scope=robots

# Preview what would be seeded
npm run seed -- --dry-run
```

### Database Access

Databases are accessible from host machine:

| Database | Host | Port | Database Name | Username | Password |
|----------|------|------|---------------|----------|----------|
| typ-e-db | localhost | 1123 | tinybots | root | ICgVcbpYW731vY3UjexgAnuQ69Wv2DdN |
| wonkers-db | localhost | 1124 | dashboard | root | ICgVcbpYW731vY3UjexgAnuQ69Wv2DdN |
| atlas-intelligence-db | localhost | 1126 | analytics | root | ICgVcbpYW731vY3UjexgAnuQ69Wv2DdN |

**Direct MySQL Access:**
```bash
# Connect to tinybots database
mysql -h 127.0.0.1 -P 1123 -u root -pICgVcbpYW731vY3UjexgAnuQ69Wv2DdN tinybots

# Or via Docker exec
docker exec -it mysql-typ-e-db mysql -u root -pICgVcbpYW731vY3UjexgAnuQ69Wv2DdN tinybots
```

### Prisma Schema Management

Schemas are located in `prisma/`:
- `prisma/tinybots/schema.prisma` - typ-e-db schema
- `prisma/dashboard/schema.prisma` - wonkers-db schema

**Sync schema from database:**
```bash
npm run db:pull:tinybots   # Pull schema from typ-e-db
npm run db:pull:dashboard  # Pull schema from wonkers-db
```

---

## Project Structure

```
devtools/
├── docker-compose.yaml          # All service definitions (single source of truth)
├── Justfile                     # Main orchestrator (imports repo-specific files)
├── package.json                 # Seed scripts and Prisma commands
├── justfiles/                   # Per-repo command definitions
│   ├── database.just           # Database startup commands
│   ├── atlas.just
│   ├── azi-3-status-check-jobs.just
│   ├── m-o-triggers.just
│   ├── megazord-events.just
│   ├── micro-manager.just
│   ├── wonkers-ecd.just
│   └── wonkers-graphql.just
├── localstack/                  # LocalStack initialization scripts
│   └── *.sh                    # SQS queue setup, etc.
├── prisma/                      # Prisma schemas and generated clients
│   ├── tinybots/
│   │   └── schema.prisma
│   └── dashboard/
│       └── schema.prisma
└── src/
    ├── config/                  # Database connection configs
    ├── generated/              # Prisma generated clients
    └── seed/                   # Seeding system
        ├── core/               # Orchestrator and context
        │   ├── Orchestrator.ts
        │   └── SeedContext.ts
        └── units/              # Individual seed modules
            ├── RobotAccountStatusSeed.ts
            ├── RobotAccountSeed.ts
            ├── EventProviderSeed.ts
            ├── EventSchemaSeed.ts
            ├── ScriptReferenceSeed.ts
            └── ...
```

---

## Docker Compose Service Architecture

### Service Layers

**Layer 1: Databases (Stateful - Reset Between Tests)**
- `mysql-typ-e-db` - TinyBots main database (port 1123)
- `mysql-wonkers-db` - Dashboard/Wonkers database (port 1124)
- `mysql-atlas-intelligence-db` - Atlas analytics database (port 1126)
- `localstack` - AWS services mock (SQS, ports 4566-4599)

**Layer 2: Migration Runners (Run Once Per Database Reset)**
- `typ-e` - Flyway migrations for typ-e-db
- `wonkers-db` - Flyway migrations for wonkers-db
- `intelligence` - Flyway migrations for atlas-intelligence-db

**Layer 3: Application Services (Stateless - Reusable)**
- `checkpoint` - Permission and scheduling service (port 3002)
- `prowl` - Robot coordination service (port 3001)
- `wonkers` - Wonkers API (port 8080)
- `wonkers-account` - Account management service
- `kryten` - User service (port 8080)
- `mailcatcher` - Email testing service

**Layer 4: Test Runners (Ephemeral - Created Per Test)**
- `wonkers-graphql`, `megazord-events`, `micro-manager`, etc.
- Mount local repo into container and run test suite
- Use `--use-aliases` for service discovery via container names

### Why This Architecture Is Fast

**Traditional CI Approach (Slow):**
```bash
trap 'docker-compose down && docker volume rm $(docker volume ls -q)' EXIT
# ☠️ Destroys EVERYTHING on exit, rebuild takes 5-10min
```

**DevTools Approach (Fast):**
```bash
docker compose rm -sf mysql-typ-e-db mysql-wonkers-db  # Only remove DBs
docker compose up -d <services>                         # Reuse if already running
# ⚡ Selective reset takes 1-2min
```

Services remain running between test runs. Only databases are reset to ensure clean state.

---

## Prerequisites & Setup

### Required Tools

- **Docker & Docker Compose** (v2.x+)
- **Just** - Command runner (`brew install just` on macOS)
- **Node.js 18+** - For seed scripts
- **AWS ECR Access** - For pulling service images (typ-e, wonkers-db, checkpoint, prowl, etc.)

### Initial Setup

1. **Authenticate with ECR:**
   ```bash
   aws ecr get-login-password --region eu-central-1 | \
     docker login --username AWS --password-stdin \
     https://693338167548.dkr.ecr.eu-central-1.amazonaws.com
   ```

2. **Start databases and migrations:**
   ```bash
   cd devtools
   just start-db
   ```

3. **Install dependencies and seed data:**
   ```bash
   npm install
   npm run generate
   npm run seed
   ```

4. **Verify services:**
   ```bash
   docker compose ps
   # Should show typ-e-db, wonkers-db running
   # Migration runners (typ-e, wonkers-db) will show "exited (0)"
   ```

---

## Common Workflows

### Daily Development

```bash
# Start dependencies for your repo
just start-<your-repo>

# Make code changes in ../your-repo/

# Run tests with clean database
just test-<your-repo>

# View logs if something fails
just log-<your-repo>
```

### Debugging Integration Tests

```bash
# Start services without running tests
just start-<repo>

# In another terminal, run tests manually
cd ../<repo>
yarn test

# Or attach debugger from IDE to existing containers
```

### Resetting Everything

```bash
# Stop all services
docker compose down

# Remove volumes (nuclear option)
docker volume rm $(docker volume ls -q | grep tinybots-devtools)

# Fresh start
just start-db
npm run seed
```

### Adding Test Data

```bash
# Modify seed units in src/seed/units/
# Then re-run seeding
npm run seed:clean
npm run seed
```

---

## Environment Variables

Most services use default environment variables defined in `docker-compose.yaml`. Override per-service if needed:

```bash
# Example: Override typ-e version
TYP_E_VERSION=v2.3.4 just start-db

# Example: Override checkpoint version
CHECKPOINT_VERSION=develop just start-megazord-events
```

**Common Variables:**
- `TYP_E_VERSION` - typ-e migration runner image tag (default: latest)
- `WONKERS_DB_VERSION` - wonkers-db migration runner image tag (default: latest)
- `CHECKPOINT_VERSION` - checkpoint service image tag (default: latest)
- `INTELLIGENCE_VERSION` - intelligence migration runner image tag (default: latest)

---

## Troubleshooting

### Services Won't Start

```bash
# Check if containers are running
docker compose ps

# View logs for specific service
docker compose logs -f <service-name>

# Restart a specific service
docker compose restart <service-name>
```

### Migration Failures

```bash
# Check migration runner logs
docker compose logs typ-e
docker compose logs wonkers-db

# Manually reset database
docker compose rm -sf mysql-typ-e-db
docker compose up -d mysql-typ-e-db
# Wait for MySQL to be ready
docker compose up typ-e
```

### Seed Script Fails

```bash
# Regenerate Prisma clients
npm run generate

# Check database connectivity
mysql -h 127.0.0.1 -P 1123 -u root -pICgVcbpYW731vY3UjexgAnuQ69Wv2DdN tinybots

# Run seed with verbose logging
DEBUG=* npm run seed
```

### Port Conflicts

If ports 1123, 1124, or 1126 are already in use:

```bash
# Find process using port
lsof -i :1123

# Kill process or modify docker-compose.yaml ports
```

---

## Related Documentation

- **[create-just-commands.md](create-just-commands.md)** - Detailed guide for adding integration test support for new repositories
- **[AGENTS.md](/Users/kai/work/tinybots/AGENTS.md)** - TinyBots engineering protocols and AI agent guidelines
- **[devdocs/tinybots/OVERVIEW.md](../OVERVIEW.md)** - Global TinyBots project overview

---

## External Dependencies

### Docker Images (ECR)

All application images are hosted in AWS ECR (`693338167548.dkr.ecr.eu-central-1.amazonaws.com`):
- `typ-e:<version>` - TinyBots main API and migration runner
- `wonkers-db:<version>` - Wonkers database migration runner
- `intelligence:<version>` - Atlas intelligence migration runner
- `checkpoint:<version>` - Permission and scheduling service
- `prowl:<version>` - Robot coordination service
- `wonkers:<version>` - Wonkers API
- `wonkers-account:<version>` - Account management service
- `kryten:<version>` - User service

### Public Images
- `mysql:8.0` - MySQL database
- `localstack/localstack` - AWS services mock
- `node:22-alpine` - Node.js runtime for test runners

---

## Design Principles

1. **Single Source of Truth:** All service definitions live in `devtools/docker-compose.yaml`, not scattered across repos.

2. **Stateful vs Stateless:** Only databases and LocalStack are reset between tests. Application services are reused for speed.

3. **Explicit Dependencies:** Just commands clearly declare which services each repo needs, making dependency graphs transparent.

4. **Volume Management:** Docker manages volumes automatically. No manual volume cleanup needed between test runs.

5. **Service Discovery:** Test runners use `--use-aliases` to connect to services via container names (e.g., `mysql-typ-e-db`, `checkpoint`).

6. **Migration Safety:** Migration runners use `depends_on` to wait for databases, and `service_completed_successfully` to ensure they run before tests.

7. **Developer Experience:** One-liner commands (`just test-<repo>`) hide Docker Compose complexity while remaining transparent and debuggable.
