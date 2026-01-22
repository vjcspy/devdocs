# TinyBots DevTools - Centralized Integration Test Infrastructure

## AI Agent Context Document

**Version**: 1.0  
**Last Updated**: 2025-12-31  
**Purpose**: Enable AI agents to understand, maintain, and extend the devtools centralized infrastructure

---

## 1. Executive Summary

### What is DevTools?

The `devtools/` directory provides a **centralized Docker Compose infrastructure** for running integration tests across multiple TinyBots repositories. It replaces the decentralized approach where each repository managed its own complete Docker setup in its `ci/` folder.

### Why It Exists

**Problem with Default CI Approach:**

```bash
# Each repo's ci/test.sh has a trap that destroys EVERYTHING:
trap '{
    docker-compose -f ci/docker-compose.yml down     # Kills ALL containers
    docker volume rm $(docker volume ls -q)          # Removes ALL volumes
}' EXIT
```

**DevTools Solution:**

```bash
# Only removes stateful containers (databases), keeps services running:
{{compose}} rm -sf mysql-typ-e-db mysql-wonkers-db   # Only DBs
{{compose}} up -d <required-services>                 # Starts what's needed
{{compose}} run --rm --use-aliases <repo>            # Runs tests
```

With just command

```dotnetcli
just start-atlas 2>&1 | head -50
```

### Key Benefits

| Aspect | Default CI | DevTools |
|--------|-----------|----------|
| Container Management | Destroys ALL containers | Only removes stateful (DB) containers |
| Volume Management | Removes ALL volumes | Docker manages automatically |
| Service Reuse | Rebuilds everything each run | Reuses running services |
| Test Speed | Slow (full rebuild ~5-10min) | Fast (selective restart ~1-2min) |
| Configuration | Duplicated per repo | Single source of truth |

---

## 2. Architecture

### 2.1 Directory Structure

```
tinybots/
├── devtools/                               # Centralized infrastructure
│   ├── docker-compose.yaml                 # ALL service definitions
│   ├── Justfile                           # Main orchestration (imports)
│   ├── justfiles/                         # Per-repo command files
│   │   ├── database.just
│   │   ├── wonkers-graphql.just
│   │   ├── megazord-events.just
│   │   ├── m-o-triggers.just
│   │   ├── wonkers-ecd.just
│   │   └── azi-3-status-check-jobs.just
│   └── localstack/                        # LocalStack init scripts
│
├── <repo>/ci/                             # Legacy (per-repo) CI setup
│   ├── docker-compose.yml                 # Repo-specific containers
│   ├── test.sh                            # Test runner with TRAP cleanup
│   ├── local-test.sh                      # Wrapper script
│   └── node-verify.sh                     # Container entrypoint
```

### 2.2 Service Layer Architecture

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                     devtools/docker-compose.yaml                              │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  LAYER 1: Database Containers (Stateful - Reset Between Tests)              │
│  ┌─────────────────────┐ ┌─────────────────────┐ ┌─────────────────────┐    │
│  │  mysql-typ-e-db     │ │  mysql-wonkers-db   │ │     localstack      │    │
│  │  Port: 1123:3306    │ │  Port: 1124:3306    │ │  Port: 4566-4599    │    │
│  │  DB: tinybots       │ │  DB: dashboard      │ │  Services: SQS      │    │
│  └─────────────────────┘ └─────────────────────┘ └─────────────────────┘    │
│                                                                              │
│  LAYER 2: Migration Runners (Run Once, Wait for Completion)                 │
│  ┌─────────────────────┐ ┌─────────────────────┐                            │
│  │       typ-e         │ │     wonkers-db      │                            │
│  │  Flyway migrations  │ │  Flyway migrations  │                            │
│  │  depends: mysql-typ │ │  depends: mysql-won │                            │
│  └─────────────────────┘ └─────────────────────┘                            │
│                                                                              │
│  LAYER 3: Application Services (Stateless - Reusable)                       │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐│
│  │ checkpoint │ │   prowl    │ │  wonkers   │ │  wonkers-  │ │  kryten    ││
│  │ Port: 3002 │ │ Port: 3001 │ │ Port: 8080 │ │  account   │ │ Port: 8080 ││
│  └────────────┘ └────────────┘ └────────────┘ └────────────┘ └────────────┘│
│                                                                              │
│  LAYER 4: Test Runner Services (Ephemeral - Created Per Test Run)           │
│  ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐                │
│  │ wonkers-graphql │ │ megazord-events │ │  micro-manager  │ ...           │
│  │ node:22-alpine  │ │ node:22-alpine  │ │ node:22-alpine  │                │
│  │ mounts: /repo   │ │ mounts: /repo   │ │ mounts: /repo   │                │
│  └─────────────────┘ └─────────────────┘ └─────────────────┘                │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Service Dependency Reference

### 3.1 Container Name Mapping

When migrating from `ci/docker-compose.yml` to `devtools/docker-compose.yaml`:

| CI Container Name | DevTools Container Name | Purpose | Schema |
|-------------------|------------------------|---------|--------|
| `mysql-db` | `mysql-typ-e-db` | TinyBots main database | `tinybots` |
| `mysql-type-db` | `mysql-typ-e-db` | Same as above (variant) | `tinybots` |
| `mysql-dashboard-db` | `mysql-wonkers-db` | Wonkers/Dashboard database | `dashboard` |
| `wonkers-mysql-db` | `mysql-wonkers-db` | Same as above (variant) | `dashboard` |
| `typ-e` | `typ-e` | Migration runner | - |
| `wonkers-db` | `wonkers-db` | Migration runner | - |
| `checkpoint` | `checkpoint` | Service | - |
| `prowl` | `prowl` | Service | - |
| `wonkers` | `wonkers` | Wonkers API | - |
| `wonkers-account` | `wonkers-account` | Wonkers Account service | - |
| `localstack` | `localstack` | AWS mock (SQS) | - |
| `mailcatcher` | `mailcatcher` | Email testing | - |
| `kryten` | `kryten` | User service | - |

### 3.2 Environment Variable Hostname Updates

When copying environment variables from CI to DevTools, update hostnames:

```diff
# Database hosts
- DB_RW_HOST: mysql-db
+ DB_RW_HOST: mysql-typ-e-db

- DB_HOST: mysql-type-db
+ DB_HOST: mysql-typ-e-db

- DB_WONKERS_HOST: mysql-dashboard-db
+ DB_WONKERS_HOST: mysql-wonkers-db

- DB_WONKERS_ACCOUNT_RW_HOST: wonkers-mysql-db
+ DB_WONKERS_ACCOUNT_RW_HOST: mysql-wonkers-db
```

### 3.3 Repository Dependency Matrix

| Repository | Databases | Migrations | Services | LocalStack |
|------------|-----------|------------|----------|------------|
| `wonkers-graphql` | typ-e, wonkers | typ-e, wonkers-db | wonkers, wonkers-account | ❌ |
| `megazord-events` | typ-e, wonkers | typ-e, wonkers-db | checkpoint, prowl, wonkers, wonkers-account | ✅ |
| `micro-manager` | typ-e | typ-e | checkpoint, prowl | ❌ |
| `m-o-triggers` | typ-e | typ-e | checkpoint, prowl | ✅ |
| `wonkers-ecd` | wonkers | wonkers-db | kryten, mailcatcher | ❌ |
| `azi-3-status-check-jobs` | typ-e, wonkers | typ-e, wonkers-db | - | ❌ |

---

## 4. Implementation Guide

### 4.1 Creating Commands for a New Repository

#### Step 1: Read CI Configuration Files

Always read these files first:

```
<repo>/ci/docker-compose.yml    # Service definitions & dependencies
<repo>/ci/test.sh               # Startup sequence & cleanup logic
<repo>/ci/local-test.sh         # Entry point wrapper (if exists)
<repo>/ci/node-verify.sh        # Container entrypoint commands
```

#### Step 2: Extract Dependency Information

From `ci/docker-compose.yml`, identify:

1. **Database containers** (under `links:` or `depends_on:`)

   ```yaml
   links:
     - mysql-db           # → mysql-typ-e-db
     - mysql-dashboard-db # → mysql-wonkers-db
   ```

2. **Service dependencies** (from `links:` or environment variables)

   ```yaml
   environment:
     CHECKPOINT_ADDRESS: http://checkpoint:8080  # → needs checkpoint
     PROWL_ADDRESS: http://prowl:8080           # → needs prowl
   ```

3. **AWS/External services**

   ```yaml
   AWS_ENDPOINT: http://localstack:4566         # → needs localstack
   ```

4. **Test entrypoint**

   ```yaml
   entrypoint: ci/node-verify.sh
   # or
   entrypoint: ["sh", "-c", "yarn test"]
   ```

From `ci/test.sh`, identify startup sequence:

```bash
docker-compose up -d mysql-db typ-e           # DBs first
docker attach $(docker ps -q --filter=...)    # Wait for migrations
docker-compose up -d checkpoint prowl         # Then services
docker-compose up -d                          # Finally test runner
```

#### Step 3: Verify Service Definition in docker-compose.yaml

Check if the test runner service already exists in `devtools/docker-compose.yaml`:

```yaml
  <repo-name>:
    image: node:22-alpine
    volumes:
      - /Users/kai/work/tinybots/tinybots/backend/<repo-name>:/usr/src/app
    labels:
      - <repo-name>
    environment:
      # All required env vars with CORRECTED hostnames
    working_dir: /usr/src/app
    entrypoint: ["sh", "-c", "yarn test:only"]  # or yarn test
    depends_on:
      typ-e:
        condition: service_completed_successfully
```

If not, add it.

#### Step 4: Create Justfile Commands

Create `devtools/justfiles/<repo-name>.just`:

```just
# ------------------------------------- <Repository Name> -------------------------------------

# Start all dependencies (for development/debugging)
start-<repo-name>:
    {{compose}} up -d \
      <db-containers> <migration-runners> <service-containers>
    -{{compose}} logs -f \
      <db-containers> <migration-runners> <service-containers>

# Run integration tests (resets DBs for clean state)
test-<repo-name>:
    {{compose}} rm -sf <stateful-containers-only>
    {{compose}} up -d \
      <all-required-services-except-test-runner>
    {{compose}} run --rm --use-aliases <repo-name>

# View logs without starting containers
log-<repo-name>:
    {{compose}} logs -f \
      <all-required-services>

# Development mode with hot-reload (optional)
dev-<repo-name>:
    {{compose}} run --rm --service-ports --use-aliases \
      --entrypoint "sh -c 'corepack enable && yarn dev'" <repo-name>

# Debug mode with inspector (optional)
debug-<repo-name>:
    {{compose}} run --rm --service-ports --use-aliases \
      --entrypoint "sh -c 'corepack enable && yarn debug'" <repo-name>
```

#### Step 5: Import in Main Justfile

Add to `devtools/Justfile`:

```just
import 'justfiles/<repo-name>.just'
```

---

## 5. Existing Implementations (Reference Examples)

### 5.1 wonkers-graphql.just

**Dependencies**: Both databases, migration runners, wonkers services

```just
#. ------------------------------------- Wonkers GraphQL -------------------------------------
start-wonkers-graphql:
    {{compose}} up -d \
      mysql-typ-e-db typ-e mysql-wonkers-db wonkers-db wonkers wonkers-account
    -{{compose}} logs -f \
      mysql-typ-e-db typ-e mysql-wonkers-db wonkers-db wonkers wonkers-account

test-wonkers-graphql:
    {{compose}} rm -sf mysql-typ-e-db mysql-wonkers-db
    {{compose}} up -d \
          mysql-typ-e-db typ-e mysql-wonkers-db wonkers-db wonkers wonkers-account
    {{compose}} run --rm  --use-aliases wonkers-graphql

log-wonkers-graphql:
    {{compose}} logs -f \
      mysql-typ-e-db typ-e mysql-wonkers-db wonkers-db wonkers wonkers-account
```

**Analysis**:

- Uses both `mysql-typ-e-db` and `mysql-wonkers-db` (both DBs)
- Needs `typ-e` and `wonkers-db` migration runners
- Needs `wonkers` and `wonkers-account` services
- **Removes**: Both MySQL containers (stateful)

### 5.2 megazord-events.just

**Dependencies**: Both databases, LocalStack, checkpoint, prowl, wonkers services

```just
# ------------------------------------- Megazord Events. -------------------------------------
dev-megazord-events:
    {{compose}} run --rm --service-ports  --use-aliases --entrypoint "sh -c 'corepack enable && yarn start'" megazord-events

debug-megazord-events:
    {{compose}} run --rm --service-ports  --use-aliases --entrypoint "sh -c 'corepack enable && yarn dev'" megazord-events

start-megazord-events:
    {{compose}} up -d \
      mysql-typ-e-db typ-e mysql-wonkers-db wonkers-db localstack checkpoint prowl wonkers wonkers-account

test-megazord-events:
    {{compose}} rm -sf mysql-typ-e-db mysql-wonkers-db localstack
    {{compose}} up -d \
      mysql-typ-e-db mysql-wonkers-db localstack checkpoint prowl wonkers wonkers-account wonkers-db
    {{compose}} run --rm --use-aliases megazord-events

log-megazord-events:
    -{{compose}} logs -f \
          mysql-typ-e-db typ-e mysql-wonkers-db wonkers-db localstack checkpoint prowl wonkers wonkers-account
```

**Analysis**:

- Uses both databases + LocalStack (SQS)
- Needs checkpoint and prowl services
- **Removes**: Both MySQL containers AND localstack (stateful queue)
- **Note**: Includes dev/debug commands for development

### 5.3 m-o-triggers.just

**Dependencies**: typ-e database, LocalStack, checkpoint, prowl

```just
# ------------------------------------- m-o-triggers -------------------------------------
test-m-o-triggers:
    {{compose}} rm -sf mysql-typ-e-db mysql-wonkers-db localstack
    {{compose}} up -d \
      mysql-typ-e-db mysql-wonkers-db localstack checkpoint prowl
    {{compose}} run --rm --use-aliases m-o-triggers
```

**Analysis**:

- Minimal dependencies (typ-e schema only)
- Uses LocalStack for SQS
- **Removes**: Both MySQL containers AND localstack

### 5.4 wonkers-ecd.just

**Dependencies**: wonkers database only (not typ-e)

```just
# ------------------------------------- wonkers-ecd -------------------------------------
test-wonkers-ecd:
    {{compose}} rm -sf mysql-wonkers-db
    {{compose}} up -d \
     mysql-wonkers-db wonkers-db mailcatcher kryten
    {{compose}} run --rm --use-aliases wonkers-ecd
```

**Analysis**:

- Only uses `mysql-wonkers-db` (dashboard schema)
- Needs `mailcatcher` and `kryten` services
- **Removes**: Only `mysql-wonkers-db` (not typ-e)

---

## 6. micro-manager Implementation

### 6.1 CI Analysis

From `micro-manager/ci/docker-compose.yml`:

```yaml
services:
  mysql-db:                    # → mysql-typ-e-db
    image: mysql:8.0
  typ-e:                       # Migration runner
    depends_on: mysql-db
  checkpoint:                  # Required service
    links: [mysql-db]
  prowl:                       # Required service  
    links: [mysql-db]
  node:
    labels: [micro-manager]
    environment:
      DB_RW_HOST: mysql-db     # → mysql-typ-e-db
      CHECKPOINT_ADDRESS: http://checkpoint:8080
      PROWL_ADDRESS: http://prowl:8080
      EVE_ADDRESS: http://eve:8080
      WADSWORTH_ADDRESS: http://wadsworth:8080
      COMMANDER_DATA_ADDRESS: http://commander-data:8080
      REPORTING_ADDRESS: http://reporting:8080
      SIGMUND_ADDRESS: http://sigmund:8080
      WONKERS_ROBOTS_ADDRESS: http://wonkers-robots:8080
      ROBOCOP_ADDRESS: http://robocop:8080
      PUBLIC_BOT_ID: 999999
    entrypoint: ci/node-verify.sh
    depends_on:
      typ-e: {condition: service_completed_successfully}
    links: [mysql-db, checkpoint]
```

From `micro-manager/ci/test.sh`:

```bash
docker-compose up -d mysql-db typ-e
docker attach $(docker ps -q --filter=label=service.name=typ-e)
docker-compose up -d checkpoint prowl
docker-compose up -d
docker attach $(docker ps -q --filter=label=micro-manager)
```

### 6.2 Already in docker-compose.yaml

```yaml
  micro-manager:
    image: node:22-alpine
    volumes:
      - /Users/kai/work/tinybots/tinybots/backend/micro-manager:/usr/src/app
    labels:
      - micro-manager
    environment:
      DB_RW_HOST: mysql-typ-e-db
      DB_PORT: 3306
      CHECKPOINT_ADDRESS: http://checkpoint:8080
      PROWL_ADDRESS: http://prowl:8080
      EVE_ADDRESS: http://eve:8080
      WADSWORTH_ADDRESS: http://wadsworth:8080
      COMMANDER_DATA_ADDRESS: http://commander-data:8080
      REPORTING_ADDRESS: http://reporting:8080
      SIGMUND_ADDRESS: http://sigmund:8080
      WONKERS_ROBOTS_ADDRESS: http://wonkers-robots:8080
      ROBOCOP_ADDRESS: http://robocop:8080
      PUBLIC_BOT_ID: 999999
    working_dir: /usr/src/app
    entrypoint: ["sh", "-c", "yarn test:only"]
    depends_on:
      typ-e:
        condition: service_completed_successfully
```

### 6.3 Required Justfile Commands

Create `devtools/justfiles/micro-manager.just`:

```just
# ------------------------------------- Micro Manager -------------------------------------

# Start dependencies for development
start-micro-manager:
    {{compose}} up -d \
      mysql-typ-e-db typ-e checkpoint prowl
    -{{compose}} logs -f \
      mysql-typ-e-db typ-e checkpoint prowl

# Run integration tests
test-micro-manager:
    {{compose}} rm -sf mysql-typ-e-db
    {{compose}} up -d \
      mysql-typ-e-db typ-e checkpoint prowl
    {{compose}} run --rm --use-aliases micro-manager

# View logs
log-micro-manager:
    {{compose}} logs -f \
      mysql-typ-e-db typ-e checkpoint prowl

# Development mode
dev-micro-manager:
    {{compose}} run --rm --service-ports --use-aliases \
      --entrypoint "sh -c 'corepack enable && yarn dev'" micro-manager

# Debug mode with inspector
debug-micro-manager:
    {{compose}} run --rm --service-ports --use-aliases \
      --entrypoint "sh -c 'corepack enable && yarn debug'" micro-manager
```

### 6.4 Justfile Import

Add to `devtools/Justfile`:

```just
import 'justfiles/micro-manager.just'
```

---

## 7. Command Patterns & Best Practices

### 7.1 Standard Commands

| Command | Purpose | When to Use |
|---------|---------|-------------|
| `start-<repo>` | Start all dependencies, tail logs | Development, debugging |
| `test-<repo>` | Run tests with clean DB state | CI, local testing |
| `log-<repo>` | View logs for all services | Debugging |
| `dev-<repo>` | Hot-reload development mode | Active development |
| `debug-<repo>` | With Node inspector attached | Debugging code |

### 7.2 Selective Container Removal Rules

**ALWAYS Remove**:

- `mysql-typ-e-db` - Contains test data
- `mysql-wonkers-db` - Contains test data

**SOMETIMES Remove**:

- `localstack` - If tests use SQS queues that need reset

**NEVER Remove in test command**:

- `typ-e`, `wonkers-db` - Migration runners (will re-run)
- `checkpoint`, `prowl`, `wonkers` - Stateless services

### 7.3 Docker Compose Flags

```bash
# rm -sf: Remove containers and their anonymous volumes (force)
{{compose}} rm -sf mysql-typ-e-db

# up -d: Detached mode (background)
{{compose}} up -d service1 service2

# run --rm: Auto-remove container after exit
# --use-aliases: Enable service DNS resolution in network
{{compose}} run --rm --use-aliases service

# --service-ports: Expose ports defined in compose file
{{compose}} run --rm --service-ports --use-aliases service
```

---

## 8. Troubleshooting

### 8.1 Common Issues

| Issue | Symptom | Solution |
|-------|---------|----------|
| Tests fail before DB ready | Connection errors | Check `depends_on` with `service_completed_successfully` |
| Can't connect to service | Host not found | Verify hostname matches service name in docker-compose |
| Port conflicts | "port already in use" | Check unique host ports (1123, 1124, etc.) |
| Stale test data | Inconsistent results | Ensure DB containers are in `rm -sf` command |
| ECR auth fails | "authentication required" | Run `just start-db` which includes ECR login |
| Container "already in use" | Name conflict | Add `--rm` flag to `docker compose run` |

### 8.2 Debugging Commands

```bash
# View all running containers
docker ps

# View logs for specific service
docker compose -f devtools/docker-compose.yaml logs -f <service>

# Get shell into running container
docker compose -f devtools/docker-compose.yaml exec <service> sh

# Force recreate containers
docker compose -f devtools/docker-compose.yaml up -d --force-recreate <service>

# Remove all devtools containers
docker compose -f devtools/docker-compose.yaml down
```

---

## 9. File Templates

### 9.1 Justfile Template

```just
# ------------------------------------- <Repository Name> -------------------------------------

# Start dependencies for development
start-<repo>:
    {{compose}} up -d \
      <db-container> <migration-runner> <services>
    -{{compose}} logs -f \
      <db-container> <migration-runner> <services>

# Run integration tests with clean DB
test-<repo>:
    {{compose}} rm -sf <db-containers>
    {{compose}} up -d \
      <all-dependencies>
    {{compose}} run --rm --use-aliases <repo>

# View logs
log-<repo>:
    {{compose}} logs -f \
      <all-dependencies>

# Development mode
dev-<repo>:
    {{compose}} run --rm --service-ports --use-aliases \
      --entrypoint "sh -c 'corepack enable && yarn dev'" <repo>
```

### 9.2 Docker Compose Service Template

```yaml
  <repo-name>:
    image: node:22-alpine
    volumes:
      - /Users/kai/work/tinybots/tinybots/backend/<repo-name>:/usr/src/app
    labels:
      - <repo-name>
    environment:
      DB_RW_HOST: mysql-typ-e-db          # or mysql-wonkers-db
      DB_PORT: 3306
      # Add all required environment variables
      # Update hostnames to match devtools service names
    working_dir: /usr/src/app
    entrypoint: ["sh", "-c", "yarn test:only"]
    depends_on:
      typ-e:                              # or wonkers-db
        condition: service_completed_successfully
```

---

## 10. Quick Reference Checklist

When adding a new repository to devtools:

- [ ] Read `<repo>/ci/docker-compose.yml`
- [ ] Read `<repo>/ci/test.sh`
- [ ] Identify database dependencies (typ-e, wonkers, or both)
- [ ] Identify service dependencies (checkpoint, prowl, localstack, etc.)
- [ ] Map CI container names to devtools names (see Section 3.1)
- [ ] Verify test runner service exists in `devtools/docker-compose.yaml`
- [ ] If not, add service definition with corrected hostnames
- [ ] Create `devtools/justfiles/<repo>.just` with standard commands
- [ ] Add import to `devtools/Justfile`
- [ ] Test with `just test-<repo>`
- [ ] Verify DB containers are removed in test command
- [ ] Document any special requirements

---

## 11. Related Files

| File | Purpose |
|------|---------|
| `devtools/Justfile` | Main orchestration file |
| `devtools/docker-compose.yaml` | Complete service definitions |
| `devtools/justfiles/*.just` | Per-repo command files |
| `devdocs/tinybots/OVERVIEW.md` | Global TinyBots standards |
| `<repo>/ci/docker-compose.yml` | Original repo CI configs (reference) |
| `<repo>/ci/test.sh` | Original test scripts (reference) |

---

## 12. Changelog

| Date | Version | Changes |
|------|---------|---------|
| 2025-12-31 | 1.0 | Initial documentation for AI agents |
