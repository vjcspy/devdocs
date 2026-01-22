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
  - Types & resolvers: organized in feature-based folder structure under `src/graphql/schema/**`.

## Schema Merge

- In `src/App.ts`:
  - `legacySchema = makeExecutableSchema({ typeDefs, resolvers })`.
  - `graphqlSchema` is imported from `src/graphql/schema`.
  - `combinedSchema = mergeSchemas({ schemas: [legacySchema, graphqlSchema] })`.
- Result: one unified GraphQL schema, serving both legacy and new fields.
- **Guideline: add new features only in `src/graphql` using Prisma + Nexus. Do NOT modify legacy code.**

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

## New Folder Structure (Feature-Based)

> **IMPORTANT**: All new GraphQL implementations MUST follow this folder structure.

The schema folder is organized to mirror the GraphQL hierarchy:

```
src/graphql/schema/
├── index.ts                                # Main schema entry (Nexus makeSchema)
├── rawData/                                # Query.rawData
│   ├── index.ts
│   ├── rawData.ts
│   ├── rawDataQuery.ts
│   └── robotProfile.ts
│
└── reports/                                # Query.reports
    ├── index.ts                            # Barrel exports for reports
    ├── report.type.ts                      # Report root type
    │
    ├── kpi/                                # Query.reports.kpi
    │   ├── index.ts
    │   ├── kpi.type.ts
    │   ├── config.ts
    │   ├── operationKpi/
    │   │   ├── index.ts
    │   │   └── operationKpi.type.ts
    │   └── retentionKpi/
    │       ├── index.ts
    │       ├── retentionKpi.type.ts
    │       └── retentionKpi.service.ts
    │
    ├── organisationReports/                # Query.reports.organisationReports
    │   ├── index.ts
    │   ├── organisationReports.type.ts
    │   ├── salesOrderShipment/
    │   │   ├── index.ts
    │   │   ├── salesOrderShipment.type.ts
    │   │   └── salesOrderShipment.service.ts
    │   └── tessaOrderStatus/
    │       ├── index.ts
    │       ├── tessaOrderStatus.type.ts
    │       └── tessaOrderStatus.service.ts
    │
    └── allReports/                         # Query.reports.allReports
        ├── index.ts
        ├── allReports.type.ts
        └── sensaraEvent/
            ├── index.ts
            ├── sensaraEvent.type.ts
            └── sensaraEvent.service.ts
```

### File Naming Conventions

| File Type | Naming Pattern | Example |
|-----------|----------------|---------|
| GraphQL Type Definition | `<feature>.type.ts` | `retentionKpi.type.ts` |
| Business Logic/Service | `<feature>.service.ts` | `retentionKpi.service.ts` |
| Barrel Export | `index.ts` | `index.ts` |
| Config | `config.ts` | `config.ts` |

### Test Structure (mirrors source)

```
test/graphql/schema/
├── rawData/
│   └── robotProfile.test.ts
└── reports/
    ├── organisationReports/
    │   └── tessaOrderStatus/
    │       └── tessaOrderStatus.service.test.ts
    └── allReports/
        └── sensaraEvent/
            └── sensaraEvent.service.test.ts
```

## Development Guidelines (New Approach)

> **CRITICAL**: When implementing new GraphQL queries/mutations, you MUST:
> 1. Use **Prisma + Nexus** (NOT legacy REST approach)
> 2. Follow the **feature-based folder structure** above
> 3. Place files in the correct location matching the GraphQL hierarchy

### Implementation Steps for New Features

1. **Determine GraphQL path**: Where does your feature sit in the schema? (e.g., `Query.reports.organisationReports.newReport`)
2. **Create folder structure**: Mirror the GraphQL path (e.g., `src/graphql/schema/reports/organisationReports/newReport/`)
3. **Create files with correct naming**:
   - `index.ts` - Barrel export
   - `<feature>.type.ts` - Nexus type definitions (`objectType`, `extendType`)
   - `<feature>.service.ts` - Business logic (if needed)
4. **Update parent index.ts**: Export new types from parent barrel files
5. **Use Prisma for DB access**: `ctx.prisma.dashboard` / `ctx.prisma.tinybots`
6. **Enforce validation**: Use `BaseResolver.Wrap` for consistency
7. **Create matching test structure**: `test/graphql/schema/<path>/<feature>.service.test.ts`

### Code Patterns

```typescript
// ✅ CORRECT: New approach with Nexus
// src/graphql/schema/reports/organisationReports/newReport/newReport.type.ts
import { objectType, extendType } from 'nexus'
import { NewReportService } from './newReport.service'

export const NewReportType = objectType({
  name: 'NewReport',
  definition(t) {
    t.nonNull.string('id')
    t.nonNull.string('name')
  },
})

export const NewReportQuery = extendType({
  type: 'OrganisationReports',
  definition(t) {
    t.field('newReport', {
      type: 'NewReport',
      resolve: async (_parent, _args, ctx) => {
        return NewReportService.getReport(ctx.prisma.dashboard)
      },
    })
  },
})
```

```typescript
// ❌ WRONG: Do NOT add new features to legacy
// src/resolvers/QueryResolver.ts - DO NOT MODIFY
// src/schema/typeDefs.ts - DO NOT MODIFY
```

### Additional Guidelines

- Map dates via `src/graphql/utils/flattenDate.ts` when returning ISO strings.
- Avoid changing `src/schema` and `src/resolvers` unless maintaining legacy compatibility.

## Error Handling

- `formatError` in `App.ts` masks internal errors (500) and only exposes safe details.
- Pass-through for certain GraphQL error codes: `BAD_USER_INPUT`, `GRAPHQL_VALIDATION_FAILED`, `GRAPHQL_PARSE_FAILED`, `BAD_REQUEST`, `OPERATION_RESOLUTION_FAILURE`, `FORBIDDEN`.

## Deprecation Plan

- Run both schemas for now.
- Migrate each legacy field/use-case into `src/graphql` with Prisma + Nexus.
- Stop updating legacy resolvers/dataSources once migrated; gradually remove legacy code.

## Quick References

| Category | Path |
|----------|------|
| Nexus schema entry | `src/graphql/schema/index.ts` |
| Prisma context | `src/graphql/context/prismaContext.ts` |
| Prisma lifecycle | `src/graphql/lib/prisma.ts` |
| Reports folder | `src/graphql/schema/reports/` |
| RawData folder | `src/graphql/schema/rawData/` |
| Legacy resolvers (DO NOT MODIFY) | `src/resolvers/QueryResolver.ts` |
| Legacy SDL (DO NOT MODIFY) | `src/schema/typeDefs.ts` |
| Server entrypoints | `src/server.ts`, `src/App.ts` |
| Restructure plan | `devdocs/tinybots/wonkers-graphql/260120-Schema-Folder-Restructure.md` |