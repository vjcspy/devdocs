## wonkers-db TL;DR
- Flyway-managed MySQL schema for dashboard and TaaS domains (accounts, relations, subscriptions, chains/orders, devices, ECD/Teamleader links).
- Only versioned SQL migrations live here; no application/runtime code.
- To inspect table structures or relationships, read the migration files under `src/main/resources/db/migration`.

## Schema Layout & How to Navigate
- Migration files are ordered lexicographically (`V01__...` → `Vnn__...`); later versions define the effective schema. Check the highest-numbered migration touching a table to understand its current shape.
- Fast lookups: `rg "CREATE TABLE" src/main/resources/db/migration` for definitions; `rg "ALTER TABLE <table>" src/main/resources/db/migration` to trace evolutions.
- Key domains: dashboard users/admins, relations and contact data, subscriptions and chains, TaaS orders and hardware types, device management, ECD/Teamleader integration artifacts, and permissions tokens for dashboards/overview access.

## Editing Guidelines
- Add new changes as a new Flyway migration file in `src/main/resources/db/migration` with the next version prefix (e.g., `V55__describe_change.sql`). Do not edit existing migrations.
- Include data backfills or permission grants in the same migration when altering contract-critical tables (subscriptions, orders, permissions) to keep environments consistent.
- Coordinate with wonkers-api, wonkers-graphql, wonkers-taas-orders, wonkers-buy-subscriptions, and dashboard services so DB schema changes are reflected in their clients/specs.

## Verification
- Run Flyway as wired in `pom.xml`/CI to verify migrations on clean and upgrade paths; ensure backward compatibility for rolling deploys.
