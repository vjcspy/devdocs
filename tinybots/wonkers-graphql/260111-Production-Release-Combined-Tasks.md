# Production Release - Combined Tasks: wonkers-graphql

**Release Date:** 2025-01-11  
**Environment:** Production  
**Service:** wonkers-graphql

---

## Release Overview

This release combines 3 implemented features for production deployment:

| # | Task | Plan Document | Type |
|---|------|---------------|------|
| 1 | Sensara Events GraphQL Query | `251223-Sensara-Events-GraphQL.md` | New Feature |
| 2 | Analytics Data Masking (NEN 7510 Compliance) | `251223-PROD-Analytics-Data-Masking.md` | Breaking Change |
| 3 | Tessa Order Status Report - New Fields | `tessaOrderStatusReport/PROD-984-Tessa-Report-New-Filters.md` | Enhancement |

---

## Task 1: Sensara Events GraphQL Query

### Feature Description

New GraphQL query endpoint `sensaraEventReport` under `Report.organisationReports` tree for querying Sensara events from the read replica database.

### Key Changes

**New Files:**
- `src/graphql/schema/reports/sensaraEventReport.ts` - Type definitions
- `src/graphql/schema/reports/sensaraEventReportService.ts` - Business logic
- Tests: unit tests + integration tests

**Query Parameters:**
- `createdSince` - ISO 8601 datetime
- `timePeriod` - Enum: LAST_HOUR, LAST_3_HOURS, LAST_6_HOURS, TODAY, YESTERDAY, LAST_WEEK
- `eventType` - String (NotificationResponse, AdlEventResponse, StateExtramuralResponse)
- `event` - String (event name)
- `robotId` - Integer
- `limit`/`offset` - Pagination

**Response Fields:**
- `sensaraId`, `residentId`, `robotId`, `event`, `eventType`, `createdAt`

### Dependencies

**⚠️ REQUIRED:** `typ-e` migration must be deployed FIRST:
- `V97_1__grant_sensara_tables_to_graphql_report_ro.sql`

---

## Task 2: Analytics Data Masking (NEN 7510 Compliance)

### Feature Description

Remove Personal Health Information (PHI) from analytics reports to comply with NEN 7510 A.9.4.1 (2).

### Key Changes (⚠️ Breaking Change)

**PHI Fields Removed Completely:**
- `discipline`
- `healthCareDemand`
- `returnReason`
- `healthcareProfessional`

**Permission Requirements Added:**
- `RawData.salesOrders` and `Organisation.salesOrders` now require:
  - `TAAS_ORDER_HEALTHCARE_INFO_READ_ALL`
  - `TAAS_ORDER_ADDRESS_READ_ALL`
- Users without both permissions will receive `FORBIDDEN` error

**Affected Types/Queries:**
- `SalesOrder` GraphQL type
- `InUseTessaReportRow` GraphQL type
- `RawData.salesOrders` query
- `Organisation.salesOrders` query
- `Report.inUseTessaReport` query

### Impact

⚠️ **Analytics consumers** must update queries to remove references to deleted fields.

---

## Task 3: Tessa Order Status Report - New Fields

### Feature Description

Added 3 new fields to `tessaOrderStatusReport` response:

| Field | Type | Description |
|-------|------|-------------|
| `salesOrderId` | `Int` | Sales order ID |
| `relationName` | `String?` | Name of the relation |
| `tessaExpertNeeded` | `Boolean?` | Expert help needed (yes→true, no→false, unknown→null) |

### Key Changes

**Modified Files:**
- `src/graphql/schema/reports/tessaOrderStatusReport.ts` - Added new fields
- `src/graphql/schema/reports/tessaOrderStatusReportService.ts` - Data mapping

---

## Pull Requests for Merge

### Repository: typ-e (MERGE FIRST)
- [ ] PR: Grant SELECT Permissions on Sensara Tables to graphql_report_ro
  - Migration: `V97_1__grant_sensara_tables_to_graphql_report_ro.sql`

### Repository: wonkers-graphql (MERGE AFTER typ-e)
- [ ] PR: Combined release - Sensara Events, Data Masking, Tessa Report Fields
  - All 3 features in single deployment

---

## Production Verification

### Prerequisites

- Internal endpoint: `https://production-api/internal/v4/dashboard/graphql`
- External endpoint: `https://production-api/ext/v1/dashboard/graphql`
- Valid authentication headers

---

### Task 1 Verification: Sensara Events GraphQL

#### Test 1.1: Basic Query - Get Recent Events

```bash
curl -X POST 'https://production-api/internal/v4/dashboard/graphql' \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer <token>' \
  -d '{
    "query": "query { reports { organisationReports { sensaraEventReport(timePeriod: TODAY, limit: 5) { sensaraId residentId robotId event eventType createdAt } } } }"
  }'
```

**Expected Result:**
- Returns array of sensara events
- Each event has: sensaraId, residentId, robotId (nullable), event, eventType, createdAt
- Events are ordered newest first

#### Test 1.2: Filter by Event Type

```bash
curl -X POST 'https://production-api/internal/v4/dashboard/graphql' \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer <token>' \
  -d '{
    "query": "query { reports { organisationReports { sensaraEventReport(eventType: \"NotificationResponse\", limit: 5) { sensaraId event eventType } } } }"
  }'
```

**Expected Result:**
- Returns only events where `eventType = "NotificationResponse"`

---

### Task 2 Verification: Analytics Data Masking

#### Test 2.1: Permission Check - Without Required Permissions

```bash
# User WITHOUT TAAS_ORDER_HEALTHCARE_INFO_READ_ALL or TAAS_ORDER_ADDRESS_READ_ALL
curl -X POST 'https://production-api/internal/v4/dashboard/graphql' \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer <user-without-permissions>' \
  -d '{
    "query": "query { rawData { salesOrders(limit: 1) { id } } }"
  }'
```

**Expected Result:**
```json
{
  "errors": [{
    "extensions": { "code": "FORBIDDEN" },
    "message": "User does not have permission..."
  }]
}
```

#### Test 2.2: PHI Fields Removed

```bash
# User WITH required permissions
curl -X POST 'https://production-api/internal/v4/dashboard/graphql' \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer <user-with-permissions>' \
  -d '{
    "query": "query { rawData { salesOrders(limit: 1) { id deliveryAddress { city } } } }"
  }'
```

**Expected Result:**
- Query succeeds (has both permissions)
- `deliveryAddress` field is present
- Attempting to query `discipline`, `healthCareDemand`, `returnReason`, `healthcareProfessional` should return GraphQL schema error (field does not exist)

---

### Task 3 Verification: Tessa Report New Fields

#### Test 3.1: Query New Fields

```bash
curl -X POST 'https://production-api/internal/v4/dashboard/graphql' \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer <token>' \
  -d '{
    "query": "query { analytics { reports(organisationId: 123) { tessaOrderStatusReport { clientId salesOrderStatus salesOrderId relationName tessaExpertNeeded } } } }"
  }'
```

**Expected Result:**
- `salesOrderId`: Integer value (not null)
- `relationName`: String or null
- `tessaExpertNeeded`: true, false, or null

#### Test 3.2: Verify Field Values

```bash
# Pick a known order and verify field mappings
curl -X POST 'https://production-api/internal/v4/dashboard/graphql' \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer <token>' \
  -d '{
    "query": "query { analytics { reports(organisationId: 123) { tessaOrderStatusReport(limit: 3) { salesOrderId relationName tessaExpertNeeded } } } }"
  }'
```

**Expected Result:**
- `tessaExpertNeeded` maps correctly:
  - Database `yes` → GraphQL `true`
  - Database `no` → GraphQL `false`
  - Database `unknown` → GraphQL `null`

---

## Verification Checklist

### Task 1: Sensara Events GraphQL
- [ ] Query returns sensara events successfully
- [ ] Time period filter works (TODAY)
- [ ] Event type filter works

### Task 2: Analytics Data Masking
- [ ] Users without permissions receive FORBIDDEN error on salesOrders
- [ ] PHI fields (discipline, healthCareDemand, returnReason, healthcareProfessional) not in schema
- [ ] Users with permissions can query salesOrders with address fields

### Task 3: Tessa Report New Fields
- [ ] `salesOrderId` field returns integer
- [ ] `relationName` field returns string or null
- [ ] `tessaExpertNeeded` returns boolean or null with correct mapping

---

## Rollback Plan

If issues occur:

1. **Rollback wonkers-graphql** to previous version
2. **typ-e migration** (V97_1) can remain - it only grants read permissions

---

## Post-Deployment Monitoring

- [ ] Check application logs for errors
- [ ] Monitor GraphQL error rates
- [ ] Watch for permission-related errors (Task 2)
- [ ] Verify Sensara events are being returned from read replica

---

## Stakeholder Communication

### Breaking Changes (Task 2)

Notify analytics tool users:
- **Removed fields:** `discipline`, `healthCareDemand`, `returnReason`, `healthcareProfessional`
- **New permission requirements:** `TAAS_ORDER_HEALTHCARE_INFO_READ_ALL` + `TAAS_ORDER_ADDRESS_READ_ALL` for salesOrders queries

### New Features

- **Sensara Events query** available for internal monitoring dashboards
- **Tessa Report** now includes `salesOrderId`, `relationName`, `tessaExpertNeeded`
