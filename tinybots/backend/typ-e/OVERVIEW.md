## typ-e TL;DR
- Flyway-managed MySQL schema for core robot accounts, schedules, scripts, and status-check data used by micro-manager, triggers, and status services.
- Source of truth is versioned SQL migrations only; no application logic lives here.
- To inspect or modify tables, read the migration files under `src/main/resources/db/migration`.

## Schema Layout & How to Navigate
- Flyway migration files are ordered lexicographically (`V01__...`, `V02__...`); later files override/extend earlier tables. Always scan the highest-numbered migration for the final shape of a table.
- Use `rg "CREATE TABLE" src/main/resources/db/migration` or `rg "ALTER TABLE <table>" src/main/resources/db/migration` to locate definitions and changes quickly.
- Common domains covered: robot/user accounts, schedules/scripts, status-check workflows, permissions, speech interactions, and telemetry-related event tables. Confirm exact fields in the latest migration touching each table.

## Editing Guidelines
- Add new schema changes by creating a new Flyway migration file in `src/main/resources/db/migration` with the next version prefix (e.g., `V94__short_description.sql`). Avoid modifying existing migrations to preserve history.
- Keep migrations idempotent where possible and include backfills/data migrations in the same file when needed.
- Coordinate changes with dependent services (micro-manager, trigger service, status-check stack) to align their expectations and database access permissions.

## Verification
- Run Flyway via Maven/CI as defined in `pom.xml`; ensure migrations apply cleanly against a fresh database and upgrade path from current production version.
