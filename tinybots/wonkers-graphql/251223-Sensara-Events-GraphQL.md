# 📋 251223-Sensara-Events-GraphQL - Add Sensara Events Query to GraphQL

## References

### Source Files
- **Sensara Adaptor Implementation (Reference):**
  - `sensara-adaptor/src/repository/SensaraEventRepository.ts` - Current query implementation
  - `sensara-adaptor/src/model/sensara/SensaraEvent.ts` - Event model definitions
  - `sensara-adaptor/src/model/dto/EventQueryDto.ts` - Query DTO
  - `sensara-adaptor/src/controller/InternalInformationController.ts` - Controller exposing events endpoint

### Target Files
- **GraphQL Implementation:**
  - `wonkers-graphql/src/graphql/schema/reports/sensaraEventReport.ts` - New type definition (to be created)
  - `wonkers-graphql/src/graphql/schema/reports/sensaraEventReportExtension.ts` - New resolver extension (to be created)
  - `wonkers-graphql/src/graphql/schema/reports/sensaraEventReportService.ts` - New service with query logic (to be created)
  - `wonkers-graphql/src/graphql/schema/reports/index.ts` - Export new types
  - `wonkers-graphql/src/graphql/schema/index.ts` - Add to schema types
  - `wonkers-graphql/src/graphql/schema/reports/organisationReportsExtension.ts` - Add query field

### Database Schema
- **Prisma Schema:** `wonkers-graphql/prisma/tinybots/schema.prisma`
  - Tables: `sensara_event`, `sensara_event_schema`, `sensara_resident_robot`, `robot_account`

### Documentation
- **Global Standard:** `devdocs/tinybots/OVERVIEW.md`
- **Repository Standard:** `devdocs/tinybots/wonkers-graphql/OVERVIEW.md`

## User Requirements

```
Sensara has a lot of events. We want to read it from the read replica db in graphql. Recreate the current events report in graphql. And add the following query parameters:

- createdSince: when was the event created
- timePeriod: last hour | last 3 hours | last 6 hours | today | yesterday | lastweek
- eventType: string Note (NotificationResponse, AdlEventResponse, StateExtramuralResponse)
- event: string (technically all possible events in AdlEventType, SensaraNotificationType, StatusExtramuralResponseType_StatusExtramuralResponseState. But prefer to leave it as a string, easier when new events are added)
- robotId: the robotId

See current report query in:
- repo: sensara-adaptor
- file: SensaraEventReport.ts
```

## 🎯 Objective

Implement a GraphQL query endpoint in `wonkers-graphql` to read Sensara events from the read replica database with filtering capabilities, matching the current report structure from `sensara-adaptor` while adding enhanced filtering parameters.

### ⚠️ Key Considerations

1. **Read Replica Access:** Query must use Prisma's `ctx.prisma.tinybots` (read replica) to avoid impacting write performance
2. **Backward Compatibility:** Maintain the existing `SensaraEventReport` structure from legacy GraphQL schema
3. **Migration Pattern:** Follow the new Nexus + Prisma pattern documented in `wonkers-graphql/OVERVIEW.md`, not the legacy REST-based approach
4. **Performance:** Events table can be large - ensure proper indexing and pagination support
5. **Flexibility:** Keep `event` and `eventType` as strings rather than enums to support future event types without schema changes
6. **Time Period Calculation:** Convert enum time periods to actual datetime ranges at runtime
7. **Filter Combination:** Support combining multiple filters (AND logic)

## 🔄 Implementation Plan

### Phase 1: Analysis & Preparation

- [ ] Analyze current Sensara event structure in database
  - **Outcome**: Database uses joined tables: `sensara_event` → `sensara_event_schema` (event name/type) → `sensara_resident_robot` (robot mapping)
  - **Outcome**: Current implementation only supports `createdSince` filter
  - **Outcome**: Need to add indexes for performance on commonly filtered fields

- [ ] Define scope and edge cases
  - **Outcome**: Handle optional filters (all nullable)
  - **Outcome**: Time period calculations based on server time
  - **Outcome**: Empty results when no events match filters
  - **Outcome**: Pagination support for large result sets
  - **Outcome**: String-based event types prevent breaking changes when new events added

- [ ] Review existing GraphQL report patterns
  - **Outcome**: Follow `tessaOrderStatusReport` pattern in `reports/` directory
  - **Outcome**: Use service layer for business logic separation
  - **Outcome**: Use Nexus `objectType` and `extendType` for type definitions
  - **Outcome**: Mount under `Report.organisationReports` tree

### Phase 2: Implementation (File/Code Structure)

```
wonkers-graphql/src/graphql/schema/reports/
├── index.ts                              # 🔄 UPDATE - Export new types
├── organisationReportsExtension.ts       # 🔄 UPDATE - Add sensaraEventReport query field
├── sensaraEventReport.ts                 # ✅ CREATE - Type definitions
├── sensaraEventReportExtension.ts        # ✅ CREATE - Query resolver
├── sensaraEventReportService.ts          # ✅ CREATE - Business logic & DB queries
└── tessaOrderStatusReport.ts             # (existing, reference pattern)

wonkers-graphql/src/graphql/schema/
└── index.ts                              # 🔄 UPDATE - Add types to makeSchema

wonkers-graphql/src/generated/graphql/
├── nexus-typegen.ts                      # 🔄 AUTO-GENERATED
└── schema.graphql                        # 🔄 AUTO-GENERATED
```

### Phase 3: Detailed Implementation Steps

#### Step 1: Create GraphQL Type Definition
**File:** `src/graphql/schema/reports/sensaraEventReport.ts`

- [ ] Define `SensaraEventReportRow` objectType
  - Fields: `sensaraId`, `residentId`, `robotId`, `event`, `eventType`, `createdAt`
  - All fields non-null strings/ints matching existing structure
  - Add field descriptions for API documentation

- [ ] Define `SensaraEventTimePeriod` enumType
  - Values: `LAST_HOUR`, `LAST_3_HOURS`, `LAST_6_HOURS`, `TODAY`, `YESTERDAY`, `LAST_WEEK`
  - Add descriptions explaining exact calculation logic

**Rationale:** Nexus requires explicit type definitions. Enum helps frontend with autocomplete while backend converts to datetime ranges.

#### Step 2: Create Service Layer
**File:** `src/graphql/schema/reports/sensaraEventReportService.ts`

- [ ] Create `SensaraEventReportService` class
  - Constructor accepts `PrismaClient` (tinybots)
  - Private method `calculateTimePeriodRange(period)` converts enum to DateTime range
  - Public method `buildReport(filters, limit?, offset?)` returns Promise<SensaraEventReportRow[]>

- [ ] Implement query logic with Prisma
  - Use `ctx.prisma.tinybots.sensara_event.findMany()`
  - Join with `sensara_event_schema` and `sensara_resident_robot`
  - Build WHERE clause dynamically based on provided filters
  - Handle `createdSince` vs `timePeriod` (timePeriod takes precedence if both provided)
  - Filter by `eventType` on `sensara_event_schema.type`
  - Filter by `event` on `sensara_event_schema.name`
  - Filter by `robotId` on `sensara_resident_robot.robot_id`
  - Apply pagination with limit/offset
  - Order by `id DESC` (newest first)

- [ ] Map database rows to GraphQL response format
  - Transform snake_case to camelCase
  - Convert DateTime to ISO 8601 strings
  - Handle nullable robot_id (some residents may not have robots)

**Rationale:** Service layer separates business logic from GraphQL resolver, making it testable and maintainable. Prisma provides type-safe DB access to read replica.

#### Step 3: Create GraphQL Resolver Extension
**File:** `src/graphql/schema/reports/sensaraEventReportExtension.ts`

- [ ] Define query arguments using Nexus args
  - `createdSince`: nullable `stringArg()` - ISO 8601 datetime string
  - `timePeriod`: nullable `arg({ type: SensaraEventTimePeriod })` - Enum
  - `eventType`: nullable `stringArg()` - e.g., "NotificationResponse"
  - `event`: nullable `stringArg()` - e.g., "FALL_DETECTED"
  - `robotId`: nullable `intArg()` - Robot ID to filter by
  - `limit`: nullable `intArg()` - Max results (default 1000)
  - `offset`: nullable `intArg()` - Pagination offset (default 0)

- [ ] Add field to `OrganisationReports` objectType
  - Field name: `sensaraEventReport`
  - Return type: `nonNull(list(nonNull(SensaraEventReportRow)))`
  - Resolver instantiates service and calls `buildReport()`

**Rationale:** Follows existing pattern from `tessaOrderStatusReport`. All filters optional for flexibility. Pagination prevents memory issues with large datasets.

#### Step 4: Update Export Files
**File:** `src/graphql/schema/reports/index.ts`

- [ ] Export new types
  ```typescript
  export * from './sensaraEventReport'
  export * from './sensaraEventReportExtension'
  ```

**File:** `src/graphql/schema/index.ts`

- [ ] Add types to makeSchema
  ```typescript
  import { SensaraEventReportRow, SensaraEventTimePeriod, OrganisationReports } from './reports'
  // Add to types array: SensaraEventReportRow, SensaraEventTimePeriod
  ```

**Rationale:** Required for Nexus to include types in generated schema. Missing exports cause "unknown type" errors.

#### Step 5: Update OrganisationReports Extension
**File:** `src/graphql/schema/reports/organisationReportsExtension.ts`

- [ ] Import new types and service
- [ ] Add `sensaraEventReport` field to existing `OrganisationReports` objectType definition
- [ ] Implement resolver following same pattern as `tessaOrderStatusReport`

**Rationale:** Keeps all organization reports under unified namespace. Matches existing API structure.

#### Step 6: Testing & Validation

- [ ] Start development server to trigger Nexus typegen
  - **Command:** `npm run dev` (or equivalent)
  - **Verify:** Generated files updated in `src/generated/graphql/`

- [ ] Test query via GraphQL Playground
  - **Endpoint:** `/v4/dashboard/graphql`
  - **Sample Query:**
    ```graphql
    query {
      report {
        organisationReports {
          sensaraEventReport(
            timePeriod: TODAY
            eventType: "NotificationResponse"
            robotId: 12345
            limit: 50
          ) {
            sensaraId
            residentId
            robotId
            event
            eventType
            createdAt
          }
        }
      }
    }
    ```

- [ ] Validate filter combinations
  - Test with no filters (returns recent events)
  - Test `createdSince` alone
  - Test `timePeriod` alone (should override createdSince if both provided)
  - Test `eventType` filter
  - Test `event` filter
  - Test `robotId` filter
  - Test all filters combined
  - Test pagination (limit/offset)

- [ ] Performance validation
  - Check query execution time with large datasets
  - Verify read replica is used (check Prisma client config)
  - Consider adding database index on `created_at` if queries slow

**Rationale:** GraphQL requires runtime validation. Manual testing ensures filters work correctly before production deployment.

### Phase 4: Documentation & Cleanup

- [ ] Update this plan's Summary section with results
- [ ] Document new query in repository documentation if needed
- [ ] Remove legacy `SensaraEventReportGQL` from `src/schema/SensaraEventReport.ts` once confirmed unused

## 📊 Summary of Results
> Do not summarize the results until the implementation is done and I request it

### ✅ Completed Achievements
- (To be filled after implementation)

### 🧪 Test Results
- (To be filled after testing)

## 🚧 Outstanding Issues & Follow-up

### ⚠️ Questions to Clarify
- [ ] **Time Period Calculation Base:** Should "TODAY" be midnight-to-now, or last 24 hours? Should calculations use UTC or organization timezone?
- [ ] **Default Limit:** Is 1000 records a reasonable default max, or should it be configurable?
- [ ] **Empty `robotId` Handling:** Should we filter out events where `sensara_resident_robot.robot_id IS NULL`, or include them?
- [ ] **Deprecation Timeline:** When should the legacy REST endpoint in `sensara-adaptor` be deprecated?
- [ ] **Index Optimization:** Should we add composite index on `(created_at, sensara_event_schema_id)` for performance?

### 🔮 Future Enhancements (Out of Scope)
- [ ] Real-time subscriptions for new events (GraphQL subscriptions)
- [ ] Aggregation queries (event counts by type, hourly buckets)
- [ ] Export functionality (CSV/JSON download)
- [ ] Event detail drill-down with full payload
