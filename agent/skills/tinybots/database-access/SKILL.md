---
name: database-access
description: TinyBots database connection info, schema queries, and context gathering commands for MySQL databases
---

# TinyBots Database Access

When working on tasks requiring database context (schema changes, queries, migrations), use this guide to query databases directly.

## Database Connection Info

The project uses MySQL databases running via Docker Compose (`devtools/tinybots/local/docker-compose.yaml`):

| Database | Service Name | Host | Port | Database Name | Root Password |
|----------|-------------|------|------|---------------|---------------|
| typ-e-db | mysql-typ-e-db | localhost | 1123 | tinybots | ICgVcbpYW731vY3UjexgAnuQ69Wv2DdN |
| wonkers-db | mysql-wonkers-db | localhost | 1124 | dashboard | ICgVcbpYW731vY3UjexgAnuQ69Wv2DdN |
| atlas-intelligence-db | mysql-atlas-intelligence-db | localhost | 1126 | analytics | ICgVcbpYW731vY3UjexgAnuQ69Wv2DdN |

## Quick Start

### Step 1: Ensure Docker Services Are Running

```bash
cd devtools/tinybots/local && docker compose ps
```

If not running, start them:

```bash
cd devtools/tinybots/local && docker compose up -d mysql-typ-e-db mysql-wonkers-db
```

### Step 2: Connect to Databases (Interactive)

```bash
# typ-e-db (tinybots database)
docker exec -it mysql-typ-e-db mysql -u root -pICgVcbpYW731vY3UjexgAnuQ69Wv2DdN tinybots

# wonkers-db (dashboard database)
docker exec -it mysql-wonkers-db mysql -u root -pICgVcbpYW731vY3UjexgAnuQ69Wv2DdN dashboard
```

## One-Liner Commands (Non-Interactive)

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

## Common Queries for Context Gathering

```sql
-- List all tables
SHOW TABLES;

-- Describe table structure
DESCRIBE table_name;

-- Get full CREATE TABLE statement
SHOW CREATE TABLE table_name;

-- Get detailed column info
SELECT COLUMN_NAME, DATA_TYPE, IS_NULLABLE, COLUMN_DEFAULT, COLUMN_KEY
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_SCHEMA = 'tinybots' AND TABLE_NAME = 'your_table';

-- Find tables by pattern
SHOW TABLES LIKE '%pattern%';

-- List all foreign key relationships
SELECT
    TABLE_NAME, COLUMN_NAME, CONSTRAINT_NAME,
    REFERENCED_TABLE_NAME, REFERENCED_COLUMN_NAME
FROM INFORMATION_SCHEMA.KEY_COLUMN_USAGE
WHERE REFERENCED_TABLE_NAME IS NOT NULL
  AND TABLE_SCHEMA = 'tinybots';
```

## Database Ownership

| Database | Used By | Purpose |
|----------|---------|---------|
| typ-e-db | Robot automation services | Robot scheduling, scripts, executions |
| wonkers-db | Dashboard & TaaS services | User accounts, orders, business data |
| atlas-intelligence-db | Atlas batch jobs | Anonymised analytics data |
