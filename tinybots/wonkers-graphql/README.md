# Wonkers GraphQL

This README explains the project architecture, the two data-access mechanisms (legacy via REST and new via Prisma + Nexus), how schemas are merged, and the key points to follow when contributing. It is written to be simple and clear for developers and AI agents.

## Purpose
- Provide a unified GraphQL API for the Dashboard.
- Run legacy REST-based resolvers alongside a new direct DB approach.
- Gradually deprecate legacy and keep only the Prisma + Nexus implementation.

## Architecture Overview
- Legacy (REST):
  - SDL schema: `src/schema/*.ts` (e.g., `typeDefs.ts`).
  - Resolvers: `src/resolvers/QueryResolver.ts` using `src/datasources/*` (REST endpoints).
- New (Prisma + Nexus):
  - Root: `src/graphql`.
  - Schema entry: `src/graphql/schema/index.ts` exports `graphqlSchema` via Nexus `makeSchema`.
  - Context: `src/graphql/context/prismaContext.ts` adds `ctx.prisma.tinybots` and `ctx.prisma.dashboard`.
  - Prisma lifecycle: `src/graphql/lib/prisma.ts` handles connect/disconnect and builds MySQL URLs from config.
  - Types & resolvers: `src/graphql/schema/**` (e.g., `rawData/*`, `kpi/*`).
  - Example: `rawData.robotProfiles` queries `tinybots.robot_profile`; KPI `retentionKpi` uses `dashboard.taas_subscription`.

## Schema Merge
- In `src/App.ts`:
  - `legacySchema = makeExecutableSchema({ typeDefs, resolvers })`.
  - `graphqlSchema` is imported from `src/graphql/schema`.
  - `combinedSchema = mergeSchemas({ schemas: [legacySchema, graphqlSchema] })`.
- Result: one unified GraphQL schema, serving both legacy and new fields.
- Guideline: add new features only in `src/graphql` to minimize legacy changes.

## Routes & Context
- `'/v4/dashboard/graphql'`: Kong header validation, uses `authenticatedContext` plus Prisma context.
- `'/internal/v4/dashboard/graphql'`: no Kong validation, uses `unauthenticatedContext` plus Prisma context.
- Context provides legacy `dataSources`, user info, headers, and `ctx.prisma` for DB access.

## Prisma Lifecycle & Config
- `connectPrisma()` called in `App.start()`; `disconnectPrisma()` in `App.stop()`.
- Runtime DB URLs are built from `config/default.json` (overridable via `config/custom-environment-variables.json`).
- Prisma schemas:
  - Tinybots: `prisma/tinybots/schema.prisma` (env `TINYBOTS_DATABASE_URL` for generate).
  - Dashboard: `prisma/dashboard/schema.prisma` (env `DASHBOARD_DATABASE_URL` for generate).

## Development Guidelines (New Approach)
- Define types/resolvers in `src/graphql/schema/**` via Nexus (`objectType`, `extendType`).
- Use `ctx.prisma.dashboard` / `ctx.prisma.tinybots` for queries.
- Enforce validation/permissions with `BaseResolver.Wrap` for consistency.
- Map dates via `src/graphql/utils/flattenDate.ts` when returning ISO strings.
- Avoid changing `src/schema` and `src/resolvers` unless maintaining compatibility.

## Error Handling
- `formatError` in `App.ts` masks internal errors (500) and only exposes safe details.
- Pass-through for certain GraphQL error codes: `BAD_USER_INPUT`, `GRAPHQL_VALIDATION_FAILED`, `GRAPHQL_PARSE_FAILED`, `BAD_REQUEST`, `OPERATION_RESOLUTION_FAILURE`, `FORBIDDEN`.

## Deprecation Plan
- Run both schemas for now.
- Migrate each legacy field/use-case into `src/graphql` with Prisma + Nexus.
- Stop updating legacy resolvers/dataSources once migrated; gradually remove legacy code.

## Quick References
- Nexus schema: `src/graphql/schema/index.ts`
- Prisma context: `src/graphql/context/prismaContext.ts`
- Prisma lifecycle: `src/graphql/lib/prisma.ts`
- Legacy resolvers: `src/resolvers/QueryResolver.ts`
- Legacy SDL: `src/schema/typeDefs.ts`
- Server entrypoints: `src/server.ts`, app config in `src/App.ts`