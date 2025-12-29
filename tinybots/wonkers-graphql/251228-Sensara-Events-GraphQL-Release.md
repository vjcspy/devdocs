# Sensara Events GraphQL Release Document

## Feature Description

### Overview
Added a new GraphQL query endpoint `sensaraEventReport` to the `wonkers-graphql` service, enabling querying of Sensara events from the read replica database with comprehensive filtering capabilities.

### What Was Changed

**New Files Created:**
- `wonkers-graphql/src/graphql/schema/reports/sensaraEventReport.ts` - GraphQL type definitions (SensaraEventReportRow, SensaraEventTimePeriod)
- `wonkers-graphql/src/graphql/schema/reports/sensaraEventReportService.ts` - Business logic service with time period calculation and filtering
- `wonkers-graphql/test/graphql/schema/reports/sensaraEventReportService.test.ts` - Unit tests (20+ test cases)
- `wonkers-graphql/test/graphqlIT/reports/sensaraEventReportIT.ts` - Integration tests (15 test scenarios)
- `typ-e/src/main/resources/db/migration/V97_1__grant_sensara_tables_to_graphql_report_ro.sql` - Database migration for read permissions

**Files Modified:**
- `wonkers-graphql/src/graphql/schema/reports/organisationReportsExtension.ts` - Added `sensaraEventReport` query field
- `wonkers-graphql/src/graphql/schema/reports/index.ts` - Exported new types
- `wonkers-graphql/src/graphql/schema/index.ts` - Added types to Nexus schema
- `devdocs/tinybots/wonkers-graphql/update-mysql-user-permission-local.md` - Updated permission documentation

### Key Features

1. **Query Parameters:**
   - `createdSince` (ISO 8601 datetime) - Filter events by creation timestamp
   - `timePeriod` (enum) - Predefined time ranges: LAST_HOUR, LAST_3_HOURS, LAST_6_HOURS, TODAY, YESTERDAY, LAST_WEEK
   - `eventType` (string) - Filter by event type (NotificationResponse, AdlEventResponse, StateExtramuralResponse)
   - `event` (string) - Filter by specific event name (e.g., FALL_DETECTED, MEDICATION_REMINDER)
   - `robotId` (integer) - Filter by robot ID
   - `limit` (integer, default: 1000) - Maximum records to return
   - `offset` (integer, default: 0) - Pagination offset

2. **Response Fields:**
   - `sensaraId` - Sensara event identifier
   - `residentId` - Sensara resident identifier
   - `robotId` - Robot ID (nullable)
   - `event` - Event name
   - `eventType` - Event type classification
   - `createdAt` - Event creation timestamp (ISO 8601)

3. **Technical Implementation:**
   - Uses Prisma with read replica (`ctx.prisma.tinybots`)
   - Proper joins with `sensara_event_schema` and `sensara_resident_robot` tables
   - `timePeriod` takes precedence over `createdSince` when both provided
   - All filters use AND logic
   - Results ordered by `id DESC` (newest first)
   - Handles null `robotId` gracefully

### Comparison with Plan

The implementation follows the plan document (`devdocs/tinybots/wonkers-graphql/251223-Sensara-Events-GraphQL.md`) with all specified features delivered:

✅ All query parameters implemented as specified
✅ Read replica database access configured
✅ Time period enum with 6 predefined ranges
✅ String-based event filtering for flexibility
✅ Comprehensive unit tests (20+ tests)
✅ Full integration test coverage (15 tests)
✅ Database permissions configured via Flyway migration
✅ Documentation updated

---

## Pull Requests (PRs) for Merge

**Repository: wonkers-graphql**
- [ ] PR: Add Sensara Events GraphQL Query with Filtering and Pagination
  - Implementation of sensaraEventReport query
  - Unit and integration tests
  - GraphQL schema updates

**Repository: typ-e**
- [ ] PR: Grant SELECT Permissions on Sensara Tables to graphql_report_ro
  - Database migration V97_1
  - Required for wonkers-graphql to query sensara events

**Note:** PRs must be merged in order: `typ-e` first, then `wonkers-graphql`

---

## Build Verification

### Prerequisites
- Access to the internal GraphQL endpoint: `/internal/v4/dashboard/graphql`
- Or external endpoint with organization headers: `/ext/v1/dashboard/graphql`

### Verification Commands

#### 1. Basic Query - Get Recent Events
```bash
curl -X POST 'http://localhost:8080/internal/v4/dashboard/graphql' \
  -H 'Content-Type: application/json' \
  -d '{
    "query": "query { reports { organisationReports { sensaraEventReport(limit: 10) { sensaraId residentId robotId event eventType createdAt } } } }"
  }'
```

**Expected Response:**
```json
{
  "data": {
    "reports": {
      "organisationReports": {
        "sensaraEventReport": [
          {
            "sensaraId": "SENSARA-123",
            "residentId": "RES-456",
            "robotId": 789,
            "event": "FALL_DETECTED",
            "eventType": "NotificationResponse",
            "createdAt": "2024-12-28T10:30:00.000Z"
          }
        ]
      }
    }
  }
}
```

#### 2. Filter by Time Period - TODAY
```bash
curl -X POST 'http://localhost:8080/internal/v4/dashboard/graphql' \
  -H 'Content-Type: application/json' \
  -d '{
    "query": "query { reports { organisationReports { sensaraEventReport(timePeriod: TODAY, limit: 50) { sensaraId residentId robotId event eventType createdAt } } } }"
  }'
```

#### 3. Filter by Event Type - NotificationResponse
```bash
curl -X POST 'http://localhost:8080/internal/v4/dashboard/graphql' \
  -H 'Content-Type: application/json' \
  -d '{
    "query": "query { reports { organisationReports { sensaraEventReport(eventType: \"NotificationResponse\", limit: 20) { sensaraId residentId robotId event eventType createdAt } } } }"
  }'
```

#### 4. Filter by Specific Event Name
```bash
curl -X POST 'http://localhost:8080/internal/v4/dashboard/graphql' \
  -H 'Content-Type: application/json' \
  -d '{
    "query": "query { reports { organisationReports { sensaraEventReport(event: \"FALL_DETECTED\", limit: 20) { sensaraId residentId robotId event eventType createdAt } } } }"
  }'
```

#### 5. Filter by Robot ID
```bash
curl -X POST 'http://localhost:8080/internal/v4/dashboard/graphql' \
  -H 'Content-Type: application/json' \
  -d '{
    "query": "query { reports { organisationReports { sensaraEventReport(robotId: 12345, limit: 20) { sensaraId residentId robotId event eventType createdAt } } } }"
  }'
```

#### 6. Filter by Created Since (ISO 8601)
```bash
curl -X POST 'http://localhost:8080/internal/v4/dashboard/graphql' \
  -H 'Content-Type: application/json' \
  -d '{
    "query": "query { reports { organisationReports { sensaraEventReport(createdSince: \"2024-12-20T00:00:00.000Z\", limit: 100) { sensaraId residentId robotId event eventType createdAt } } } }"
  }'
```

#### 7. Combined Filters
```bash
curl -X POST 'http://localhost:8080/internal/v4/dashboard/graphql' \
  -H 'Content-Type: application/json' \
  -d '{
    "query": "query { reports { organisationReports { sensaraEventReport(timePeriod: TODAY, eventType: \"NotificationResponse\", event: \"FALL_DETECTED\", robotId: 12345, limit: 10) { sensaraId residentId robotId event eventType createdAt } } } }"
  }'
```

#### 8. Pagination Test
```bash
# First page
curl -X POST 'http://localhost:8080/internal/v4/dashboard/graphql' \
  -H 'Content-Type: application/json' \
  -d '{
    "query": "query { reports { organisationReports { sensaraEventReport(timePeriod: LAST_WEEK, limit: 10, offset: 0) { sensaraId createdAt } } } }"
  }'

# Second page
curl -X POST 'http://localhost:8080/internal/v4/dashboard/graphql' \
  -H 'Content-Type: application/json' \
  -d '{
    "query": "query { reports { organisationReports { sensaraEventReport(timePeriod: LAST_WEEK, limit: 10, offset: 10) { sensaraId createdAt } } } }"
  }'
```

#### 9. External Endpoint (with Organization Headers)
```bash
curl -X POST 'http://localhost:8080/ext/v1/dashboard/graphql' \
  -H 'Content-Type: application/json' \
  -H 'x-tiny-organization-identity-type: external' \
  -H 'x-tiny-organization-identity-value: tessa' \
  -H 'x-relation-id: 63' \
  -H 'x-tiny-organization-scopes: reports:read' \
  -d '{
    "query": "query { reports { organisationReports { sensaraEventReport(timePeriod: TODAY, limit: 10) { sensaraId residentId robotId event eventType createdAt } } } }"
  }'
```

### Verification Checklist

- [ ] Basic query returns sensara events successfully
- [ ] Time period filters (TODAY, YESTERDAY, LAST_WEEK, etc.) work correctly
- [ ] Event type filtering works (NotificationResponse, AdlEventResponse, StateExtramuralResponse)
- [ ] Specific event name filtering works
- [ ] Robot ID filtering works
- [ ] Created since timestamp filtering works
- [ ] Combined filters work with AND logic
- [ ] Pagination works (limit and offset)
- [ ] Results are ordered by newest first (DESC)
- [ ] Null robotId values are handled correctly
- [ ] Query works on both internal and external endpoints
- [ ] No database permission errors in logs

### Performance Notes

- Default limit is 1000 to prevent large result sets
- Query uses read replica database to avoid impacting write performance
- Results are ordered by ID DESC for efficient newest-first retrieval
- Database has indexes on commonly queried fields

---

## Test Results

**Unit Tests:** ✅ All 20+ tests passing
- Field mapping and null handling
- Time period calculations (all 6 periods)
- Individual filter tests
- Combined filter tests
- Pagination and ordering

**Integration Tests:** ✅ All 15 tests passing
- Full GraphQL query execution
- All filter combinations with real database
- Pagination verification
- Empty result handling
- Null robotId handling

**Total Test Coverage:** 35+ test cases covering all scenarios
