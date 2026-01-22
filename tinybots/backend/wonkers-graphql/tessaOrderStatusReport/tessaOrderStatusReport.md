# Tessa Order Status Report - Implementation Documentation

## Overview

This document provides comprehensive documentation for the `tessaOrderStatusReport` query, which provides comprehensive
order status information for Tessa robots including order lifecycle, subscription details, and device status.

**Status**: ✅ **COMPLETED** - All implementation, unit tests, and integration tests have been successfully implemented and are passing.

## Quick Reference

**GraphQL Query:**
```graphql
query {
  reports {
    organisationReports {
      tessaOrderStatusReport(relationIds: [63]) {
        clientId
        clientUuid
        teamId
        serial
        onlineStatus
        deviceLastSeen
        salesOrderStatus
        salesOrderReceivedAt
        salesOrderAcceptedAt
        salesOrderShippedAt
        salesOrderDeliveredAt
        subscriptionStartAt
        subscriptionEndAt
        # ... plus 6 more fields
      }
    }
  }
}
```

**With Pagination:**
```graphql
query {
  reports {
    organisationReports {
      tessaOrderStatusReport(
        relationIds: [63, 64]
        limit: 100
        offset: 0
      ) {
        serial
        clientId
        clientUuid
        # ...
      }
    }
  }
}
```

**Files:**
- Schema: `src/graphql/schema/reports/tessaOrderStatusReport.ts`
- Service: `src/graphql/schema/reports/tessaOrderStatusReportService.ts`
- Extension: `src/graphql/schema/reports/organisationReportsExtension.ts`
- Report Extension: `src/graphql/schema/reports/reportExtension.ts`
- Unit Tests: `test/graphql/schema/reports/tessaOrderStatusReportService.test.ts` ✅
- Integration Tests: `test/graphqlIT/reports/tessaOrderStatusReportIT.ts` ✅

**Test Status:** All tests passing ✅

## Implementation Summary

### What Was Implemented

The implementation integrates the report into the new `OrganisationReports` type structure (accessed via `reports.organisationReports.tessaOrderStatusReport`), following the project's GraphQL schema organization pattern.

**Key Implementation Details:**
- Query is accessed via `reports { organisationReports { tessaOrderStatusReport() } }`
- Integrated with new `OrganisationReports` type (file: `src/graphql/schema/reports/organisationReportsExtension.ts`)
- Extends the legacy `Report` type via Nexus (file: `src/graphql/schema/reports/reportExtension.ts`)
- Status derivation uses legacy `getTaasOrderStatusString` utility for consistency with existing codebase
- Date formatting uses `flattenDate` utility (returns ISO 8601 strings)
- **[PROD-521]**: Processes ALL orders, even those without robots (serial and onlineStatus are nullable)
- Includes `clientUuid` field from `taas_order.client_uuid` for ECD unique identifier

### Files Created/Modified

1. **Schema Definition**: `src/graphql/schema/reports/tessaOrderStatusReport.ts`
   - Defines `TessaOrderStatusReportRow` objectType with 19 fields
   - All fields properly typed (nonNull/nullable)
   - `serial`, `onlineStatus`, and `deviceLastSeen` are nullable (for orders without robots)
   - Includes `clientUuid` for ECD unique identifier

2. **Service Implementation**: `src/graphql/schema/reports/tessaOrderStatusReportService.ts`
   - Implements `buildReport()` method with Prisma cross-database queries
   - Handles snake_case → camelCase mapping for legacy status utility
   - Implements pagination with optional limit/offset

3. **Organisation Reports Type**: `src/graphql/schema/reports/organisationReportsExtension.ts`
   - Defines `OrganisationReports` type with `tessaOrderStatusReport` field
   - Uses `t.list.field()` to return array of report rows
   - Connects to service layer

4. **Report Extension**: `src/graphql/schema/reports/reportExtension.ts`
   - Extends legacy `Report` type with `organisationReports` field
   - Integrates Nexus schema with legacy GraphQL schema

5. **Exports**: `src/graphql/schema/reports/index.ts`
   - Exports all report types

6. **Schema Integration**: `src/graphql/schema/index.ts`
   - Registers `Report`, `OrganisationReports`, and `TessaOrderStatusReportRow` types

7. **Unit Tests**: `test/graphql/schema/reports/tessaOrderStatusReportService.test.ts`
   - ✅ Tests field mappings (salesOrderReceivedAt, salesOrderAcceptedAt)
   - ✅ Tests clientId/teamId fallback logic
   - ✅ Tests status derivation with legacy utility
   - ✅ Tests pagination behavior
   - ✅ All tests passing

7. **Integration Tests**: `test/graphqlIT/reports/tessaOrderStatusReportIT.ts`
   - ✅ Seeds test data (relation, integration, contacts, orders, robots, subscriptions)
   - ✅ Tests full GraphQL query execution via `reports.allReports.tessaOrderStatusReport`
   - ✅ Validates returned data structure and values
   - ✅ Proper cleanup of seeded data
   - ✅ All tests passing

## Task Requirements

Based on the GraphQL query structure provided:

```graphql
query MyQuery {
    reports {
        organisationReports {
            tessaOrderStatusReport(relationIds: 63) {
                clientId
                clientUuid
                teamId
                serial
                onlineStatus
                deviceLastSeen
                salesOrderStatus
                salesOrderReceivedAt
                salesOrderDeclinedAt
                salesOrderAcceptedAt
                salesOrderShippedAt
                salesOrderDeliveredAt
                salesOrderActivatedAt
                salesOrderReturnReceivedAt
                salesOrderReturnAcceptedAt
                salesOrderReturnRejectedAt
                salesOrderReturnedAt
                subscriptionStartAt
                subscriptionEndAt
            }
        }
    }
}
```

**With pagination:**

```graphql
query MyQueryWithPagination {
    reports {
        organisationReports {
            tessaOrderStatusReport(
                relationIds: [63, 64],
                limit: 100,
                offset: 0
            ) {
                clientId
                clientUuid
                teamId
                serial
                onlineStatus
                deviceLastSeen
                salesOrderStatus
                salesOrderReceivedAt
                salesOrderDeclinedAt
                salesOrderAcceptedAt
                salesOrderShippedAt
                salesOrderDeliveredAt
                salesOrderActivatedAt
                salesOrderReturnReceivedAt
                salesOrderReturnAcceptedAt
                salesOrderReturnRejectedAt
                salesOrderReturnedAt
                subscriptionStartAt
                subscriptionEndAt
            }
        }
    }
}
        salesOrderReturnAcceptedAt
        salesOrderReturnRejectedAt
        salesOrderReturnedAt
        subscriptionStartAt
        subscriptionEndAt
    }
}
```

**Note:** If `limit` and `offset` are not provided, the query will return all matching records.

### Key Mappings

**User clarifications (Updated):**

- `salesOrderReceivedAt` = `salesOrderCreatedAt` → maps to `taas_order.created_at` (when order was created)
- `salesOrderAcceptedAt` = `salesOrderProcessingAt` → maps to `taas_order_status.processing_at` (when order processing starts)
- `salesOrderReturnProcessStartedAt` = `salesOrderReturnReceivedAt` (return initiated)

## Database Schema Analysis

### Dashboard Database Tables

| Database  | Table                | Column                      | Type            | Relation                  | Description                                             |
|-----------|----------------------|-----------------------------|-----------------|---------------------------|---------------------------------------------------------|
| dashboard | `taas_order`         | `id`                        | Int (PK)        | -                         | Unique order identifier, maps to `salesOrderId`         |
| dashboard | `taas_order`         | `relation_id`               | Int (FK)        | → `dashboard_relation.id` | Organization/relation filter                            |
| dashboard | `taas_order`         | `client_id`                 | String          | -                         | Client identifier, maps to `clientId` output            |
| dashboard | `taas_order`         | `client_uuid`               | String          | -                         | ECD unique identifier, maps to `clientUuid` output      |
| dashboard | `taas_order`         | `team_id`                   | String          | -                         | Team identifier, maps to `teamId` output                |
| dashboard | `taas_order`         | `taas_id`                   | Int (FK)        | → `taas_subscription.id`  | Links to subscription                                   |
| dashboard | `taas_order`         | `created_at`                | DateTime        | -                         | Order creation timestamp, maps to `salesOrderReceivedAt` |
| dashboard | `taas_order`         | `updated_at`                | DateTime        | -                         | Last modification timestamp                             |
| dashboard | `taas_order_status`  | `taas_order_id`             | Int (PK, FK)    | → `taas_order.id`         | One-to-one relationship with order                      |
| dashboard | `taas_order_status`  | `ordered_at`                | DateTime        | -                         | Initial order timestamp                                 |
| dashboard | `taas_order_status`  | `processing_at`             | DateTime        | -                         | Maps to `salesOrderAcceptedAt` (when processing starts) |
| dashboard | `taas_order_status`  | `delivered_at`              | DateTime        | -                         | Maps to `salesOrderDeliveredAt`                         |
| dashboard | `taas_order_status`  | `activated_at`              | DateTime        | -                         | Maps to `salesOrderActivatedAt` (robot first online)    |
| dashboard | `taas_order_status`  | `declined_at`               | DateTime        | -                         | Maps to `salesOrderDeclinedAt`                          |
| dashboard | `taas_order_status`  | `return_process_started_at` | DateTime        | -                         | Maps to `salesOrderReturnReceivedAt` (return initiated) |
| dashboard | `taas_order_status`  | `return_accepted_at`        | DateTime        | -                         | Maps to `salesOrderReturnAcceptedAt`                    |
| dashboard | `taas_order_status`  | `return_rejected_at`        | DateTime        | -                         | Maps to `salesOrderReturnRejectedAt`                    |
| dashboard | `taas_order_status`  | `created_at`                | DateTime        | -                         | Status record creation                                  |
| dashboard | `taas_order_status`  | `updated_at`                | DateTime        | -                         | Last status update                                      |
| dashboard | `taas_subscription`  | `id`                        | Int (PK)        | -                         | Unique subscription identifier                          |
| dashboard | `taas_subscription`  | `serial_id`                 | BigInt (FK)     | → `dashboard_robot.id`    | Robot reference                                         |
| dashboard | `taas_subscription`  | `relation_id`               | Int (FK)        | → `dashboard_relation.id` | Organization                                            |
| dashboard | `taas_subscription`  | `client_id`                 | String          | -                         | Client identifier                                       |
| dashboard | `taas_subscription`  | `team_id`                   | String          | -                         | Team identifier                                         |
| dashboard | `taas_subscription`  | `start_at`                  | DateTime        | -                         | Maps to `subscriptionStartAt`                           |
| dashboard | `taas_subscription`  | `end_at`                    | DateTime        | -                         | Maps to `subscriptionEndAt`                             |
| dashboard | `taas_subscription`  | `shipped_at`                | DateTime        | -                         | When robot was shipped                                  |
| dashboard | `taas_subscription`  | `archived_at`               | DateTime        | -                         | Subscription archive date                               |
| dashboard | `dashboard_robot`    | `id`                        | BigInt (PK)     | -                         | Robot ID in dashboard                                   |
| dashboard | `dashboard_robot`    | `serial`                    | String (unique) | -                         | Robot serial number, maps to `serial` output            |
| dashboard | `dashboard_robot`    | `relation_id`               | Int (FK)        | → `dashboard_relation.id` | Organization                                            |
| dashboard | `dashboard_robot`    | `taas`                      | Boolean         | -                         | Is TaaS robot flag                                      |
| dashboard | `dashboard_robot`    | `deactivated_at`            | DateTime        | -                         | Robot deactivation date                                 |
| dashboard | `dashboard_relation` | `id`                        | Int (PK)        | -                         | Organization identifier (filter)                        |
| dashboard | `dashboard_relation` | `name`                      | String          | -                         | Organization name                                       |

### Note on Field Mappings

Based on schema analysis and user clarifications:

- **`salesOrderReceivedAt`**: Maps to `taas_order.created_at` (order creation timestamp).
- **`salesOrderAcceptedAt`**: Maps to `taas_order_status.processing_at` (when processing starts).
- **`salesOrderShippedAt`**: Available in `taas_subscription.shipped_at` (confirmed ✔).
- **`salesOrderReturnedAt`**: Maps to `taas_order_status.end_at` (needs Arno's confirmation).
- **`clientId` / `teamId`**: Use fallback logic: `taas_subscription` values first, then `taas_order` values, then `null`.

### Tinybots Database Tables

| Database | Table                 | Column           | Type            | Relation             | Description                                        |
|----------|-----------------------|------------------|-----------------|----------------------|----------------------------------------------------|
| tinybots | `robot_account`       | `id`             | Int (PK)        | -                    | Robot account identifier                           |
| tinybots | `robot_account`       | `username`       | String (unique) | -                    | Robot serial, join key to `dashboard_robot.serial` |
| tinybots | `robot_account`       | `last_active_at` | DateTime        | -                    | Last activity timestamp                            |
| tinybots | `robot_online_status` | `robot_id`       | Int (PK, FK)    | → `robot_account.id` | Current online status                              |
| tinybots | `robot_online_status` | `status_id`      | Int             | -                    | Status code (0=offline, 1=online)                  |
| tinybots | `robot_online_status` | `updated_at`     | DateTime        | -                    | Maps to `deviceLastSeen`                           |

## Data Flow and Join Strategy

### Primary Query Flow

```
Step 1: Filter by relationIds
  dashboard_relation (where id IN relationIds)
    ↓
Step 2: Get all orders
  taas_order (where relation_id IN filtered relations)
    ↓
Step 3: Get order status timestamps
  taas_order_status (LEFT JOIN on taas_order_id)
    ↓
Step 4: Get subscription details
  taas_subscription (LEFT JOIN on taas_order.taas_id)
    ↓
Step 5: Get robot details
  dashboard_robot (JOIN on taas_subscription.serial_id)
    ↓
Step 6: Cross-database join to tinybots
  robot_account (where username = dashboard_robot.serial)
    ↓
Step 7: Get online status
  robot_online_status (LEFT JOIN on robot_account.id)
```

### Field Mapping Logic

#### Order Status Derivation

The `salesOrderStatus` field should be derived from timestamps in `taas_order_status`:

```typescript
function deriveSalesOrderStatus(status: taas_order_status | null): string {
    if (!status) return 'UNKNOWN';

    if (status.declined_at) return 'DECLINED';
    if (status.return_rejected_at) return 'RETURN_REJECTED';
    if (status.return_accepted_at) return 'RETURN_ACCEPTED';
    if (status.return_process_started_at) return 'RETURN_IN_PROGRESS';
    if (status.activated_at) return 'ACTIVATED';
    if (status.delivered_at) return 'DELIVERED';
    if (status.processing_at) return 'PROCESSING';
    if (status.ordered_at) return 'ORDERED';

    return 'UNKNOWN';
}
```

#### Online Status Mapping

From `robot_online_status.status_id`:

- `0` → `"OFFLINE"`
- `1` → `"ONLINE"`
- `null` → `"OFFLINE"` (default)

## Implementation Details

### Actual Implementation (As Built)

#### File Structure

```
src/graphql/schema/reports/
├── tessaOrderStatusReport.ts          # Schema definition (TessaOrderStatusReportRow objectType)
├── tessaOrderStatusReportService.ts   # Service class with business logic
├── allReportsExtension.ts             # Wrapper type extending AllReports
└── index.ts                           # Exports

test/graphql/schema/reports/
└── tessaOrderStatusReportService.test.ts  # Unit tests (✅ passing)

test/graphqlIT/reports/
└── tessaOrderStatusReportIT.ts           # Integration tests (✅ passing)
```

#### Schema Definition

**File:** `src/graphql/schema/reports/tessaOrderStatusReport.ts`

```typescript
import { objectType } from 'nexus'

export const TessaOrderStatusReportRow = objectType({
  name: 'TessaOrderStatusReportRow',
  definition(t) {
    t.nullable.string('clientId', { description: 'Client identifier' })
    t.nullable.string('clientUuid', { description: 'ECD unique identifier' })
    t.nullable.string('teamId', { description: 'Team identifier' })
    t.nullable.string('serial', { description: 'Robot serial number (null if no robot linked)' })
    t.nullable.string('onlineStatus', { description: 'Current online status: ONLINE or OFFLINE (null if no robot)' })
    t.nullable.string('deviceLastSeen', { description: 'Last time robot was seen online (ISO 8601)' })
    t.nullable.string('salesOrderStatus', { description: 'Current order status' })
    t.nullable.string('salesOrderReceivedAt', { description: 'When order was created (ISO 8601)' })
    t.nullable.string('salesOrderDeclinedAt', { description: 'When order was declined (ISO 8601)' })
    t.nullable.string('salesOrderAcceptedAt', { description: 'When order processing started (ISO 8601)' })
    t.nullable.string('salesOrderShippedAt', { description: 'When robot was shipped (ISO 8601)' })
    t.nullable.string('salesOrderDeliveredAt', { description: 'When robot was delivered (ISO 8601)' })
    t.nullable.string('salesOrderActivatedAt', { description: 'When robot was first activated (ISO 8601)' })
    t.nullable.string('salesOrderReturnReceivedAt', { description: 'When return process started (ISO 8601)' })
    t.nullable.string('salesOrderReturnAcceptedAt', { description: 'When return was accepted (ISO 8601)' })
    t.nullable.string('salesOrderReturnRejectedAt', { description: 'When return was rejected (ISO 8601)' })
    t.nullable.string('salesOrderReturnedAt', { description: 'When robot was returned (ISO 8601)' })
    t.nullable.string('subscriptionStartAt', { description: 'Subscription start date (ISO 8601)' })
    t.nullable.string('subscriptionEndAt', { description: 'Subscription end date (ISO 8601)' })
  }
})
```

#### AllReports Extension

**File:** `src/graphql/schema/reports/organisationReportsExtension.ts`

```typescript
import { intArg, list, nonNull, objectType } from 'nexus'

import { TessaOrderStatusReportRow } from './tessaOrderStatusReport'
import { TessaOrderStatusReportService } from './tessaOrderStatusReportService'

export const OrganisationReports = objectType({
  name: 'OrganisationReports',
  definition(t) {
    t.list.field('tessaOrderStatusReport', {
      type: nonNull(TessaOrderStatusReportRow),
      args: {
        relationIds: nonNull(list(nonNull(intArg()))),
        limit: intArg({ description: 'Maximum number of records to return (optional)' }),
        offset: intArg({ description: 'Number of records to skip (optional)' })
      },
      async resolve(_parent, args, ctx) {
        const service = new TessaOrderStatusReportService(ctx.prisma.dashboard, ctx.prisma.tinybots)
        return service.buildReport(args.relationIds, args.limit ?? undefined, args.offset ?? undefined)
      }
    })
  }
})
```

**Key Points:**
- Uses `t.list.field()` to return array of report rows
- Integrated into `OrganisationReports` type (not direct Query extension)
- Properly typed with `nonNull(TessaOrderStatusReportRow)`
- **[PROD-521]**: Processes all orders even without robots - `serial` and `onlineStatus` are nullable

#### Service Implementation

**File:** `src/graphql/schema/reports/tessaOrderStatusReportService.ts`

Key implementation details:

1. **Cross-Database Query Pattern**:
   ```typescript
   // Step 1: Query dashboard database
   const orders = await this.dashboardPrisma.taas_order.findMany({
     where: { relation_id: { in: relationIds } },
     include: {
       taas_order_status: true,
       taas_subscription: {
         include: { dashboard_robot: true }
       }
     },
     skip: offset,
     take: limit
   })

   // Step 2: Extract serials for cross-database join
   const serials = orders
     .map((order) => order.taas_subscription?.dashboard_robot?.serial)
     .filter((serial): serial is string => serial != null)

   // Step 3: Query tinybots database
   const robotAccounts = await this.tinybotsPrisma.robot_account.findMany({
     where: { username: { in: serials } },
     include: { robot_online_status: true }
   })

   // Step 4: Build map for O(1) lookups
   const robotStatusMap = new Map(
     robotAccounts.map((robot) => [robot.username, robot.robot_online_status])
   )
   ```

2. **Status Derivation with Legacy Utility**:
   ```typescript
   private deriveSalesOrderStatus(status: any): string | null {
     if (status == null) return null

     // Map Prisma fields (snake_case) to RawTaasOrderModel format (camelCase)
     const mappedStatus = {
       orderedAt: status.ordered_at,
       processingAt: status.processing_at,
       deliveredAt: status.delivered_at,
       activatedAt: status.activated_at,
       declinedAt: status.declined_at,
       returnProcessStartedAt: status.return_process_started_at,
       returnAcceptedAt: status.return_accepted_at,
       returnRejectedAt: status.return_rejected_at,
       returnedAt: status.returned_at,
       shippedAt: status.shipped_at
     }

     return getTaasOrderStatusString({ status: mappedStatus } as RawTaasOrderModel)
   }
   ```

3. **Field Mappings**:
   - `salesOrderReceivedAt`: `order.created_at` ✅
   - `salesOrderAcceptedAt`: `status.processing_at` ✅
   - `salesOrderShippedAt`: `subscription.shipped_at` ✅
   - `salesOrderReturnedAt`: `subscription.end_at` ✅
   - `clientId`/`teamId`: `subscription?.value ?? order.value` (fallback logic) ✅

4. **Date Formatting**:
   - Uses `flattenDate()` utility from `src/graphql/utils/datetime.ts`
   - Returns ISO 8601 formatted strings

## Original Implementation Plan (For Reference)

### Step 1: Define GraphQL Schema (Nexus)

**File:** `src/graphql/schema/reports/tessaOrderStatusReport.ts`

```typescript
import { extendType, intArg, list, nonNull, objectType } from 'nexus'

import { TessaOrderStatusReportService } from './tessaOrderStatusReportService'

export const TessaOrderStatusReportRow = objectType({
  name: 'TessaOrderStatusReportRow',
  definition(t) {
    t.nullable.string('clientId', { description: 'Client identifier' })
    t.nullable.string('teamId', { description: 'Team identifier' })
    t.nonNull.string('serial', { description: 'Robot serial number' })
    t.nonNull.string('onlineStatus', { description: 'Current online status: ONLINE or OFFLINE' })
    t.nullable.string('deviceLastSeen', { description: 'Last time robot was seen online (ISO 8601)' })
    t.nullable.string('salesOrderStatus', { description: 'Current order status' })
    t.nullable.string('salesOrderReceivedAt', { description: 'When order was created (ISO 8601)' })
    t.nullable.string('salesOrderDeclinedAt', { description: 'When order was declined (ISO 8601)' })
    t.nullable.string('salesOrderAcceptedAt', { description: 'When order processing started (ISO 8601)' })
    t.nullable.string('salesOrderShippedAt', { description: 'When robot was shipped (ISO 8601)' })
    t.nullable.string('salesOrderDeliveredAt', { description: 'When robot was delivered (ISO 8601)' })
    t.nullable.string('salesOrderActivatedAt', { description: 'When robot was first activated (ISO 8601)' })
    t.nullable.string('salesOrderReturnReceivedAt', { description: 'When return process started (ISO 8601)' })
    t.nullable.string('salesOrderReturnAcceptedAt', { description: 'When return was accepted (ISO 8601)' })
    t.nullable.string('salesOrderReturnRejectedAt', { description: 'When return was rejected (ISO 8601)' })
    t.nullable.string('salesOrderReturnedAt', { description: 'When robot was returned - maps to taas_order_status.end_at (ISO 8601, pending confirmation)' })
    t.nullable.string('subscriptionStartAt', { description: 'Subscription start date (ISO 8601)' })
    t.nullable.string('subscriptionEndAt', { description: 'Subscription end date (ISO 8601)' })
  }
})

export const TessaOrderStatusReportQuery = extendType({
  type: 'Query',
  definition(t) {
    t.list.field('tessaOrderStatusReport', {
      type: TessaOrderStatusReportRow,
      args: {
        relationIds: nonNull(list(nonNull(intArg()))),
        limit: intArg({ description: 'Maximum number of records to return (optional)' }),
        offset: intArg({ description: 'Number of records to skip (optional)' })
      },
      async resolve(_parent, args, ctx) {
        const service = new TessaOrderStatusReportService(ctx.prisma.dashboard, ctx.prisma.tinybots)
        return service.buildReport(args.relationIds, args.limit ?? undefined, args.offset ?? undefined)
      }
    })
  }
})
```

### Step 2: Create Report Service

**File:** `src/graphql/schema/reports/tessaOrderStatusReportService.ts`

```typescript
import { PrismaClient as DashboardPrismaClient } from '../../../generated/prisma/dashboard'
import { PrismaClient as TinybotsPrismaClient } from '../../../generated/prisma/tinybots'
import { formatISODate } from '../../utils/datetime'

export interface TessaOrderStatusReportRow {
  clientId: string | null
  clientUuid: string | null
  teamId: string | null
  serial: string | null
  onlineStatus: string | null
  deviceLastSeen: string | null
  salesOrderStatus: string | null
  salesOrderReceivedAt: string | null
  salesOrderDeclinedAt: string | null
  salesOrderAcceptedAt: string | null
  salesOrderShippedAt: string | null
  salesOrderDeliveredAt: string | null
  salesOrderActivatedAt: string | null
  salesOrderReturnReceivedAt: string | null
  salesOrderReturnAcceptedAt: string | null
  salesOrderReturnRejectedAt: string | null
  salesOrderReturnedAt: string | null
  subscriptionStartAt: string | null
  subscriptionEndAt: string | null
}

export class TessaOrderStatusReportService {
  constructor(
    private dashboardPrisma: DashboardPrismaClient,
    private tinybotsPrisma: TinybotsPrismaClient
  ) {}

  async buildReport(relationIds: number[], limit?: number, offset?: number): Promise<TessaOrderStatusReportRow[]> {
    // Step 1: Query dashboard database for orders with optional pagination
    const orders = await this.dashboardPrisma.taas_order.findMany({
      where: {
        relation_id: { in: relationIds }
      },
      include: {
        taas_order_status: true,
        taas_subscription: {
          include: {
            dashboard_robot: true
          }
        }
      },
      skip: offset,
      take: limit
    })

    // Step 2: Get robot serials for cross-database join
    const serials = orders
      .map((order) => order.taas_subscription?.dashboard_robot?.serial)
      .filter((serial): serial is string => serial != null)

    // Step 3: Query tinybots database for online status
    const robotAccounts = await this.tinybotsPrisma.robot_account.findMany({
      where: {
        username: { in: serials }
      },
      include: {
        robot_online_status: true
      }
    })

    // Step 4: Build indexed map for quick lookups
    const robotStatusMap = new Map(
      robotAccounts.map((robot) => [robot.username, robot.robot_online_status])
    )

    // Step 5: Build report rows
    const reportRows: TessaOrderStatusReportRow[] = []

    for (const order of orders) {
      const subscription = order.taas_subscription
      const robot = subscription?.dashboard_robot
      const status = order.taas_order_status

      if (!robot) continue // Skip orders without robots

      const onlineStatus = robotStatusMap.get(robot.serial)

      reportRows.push({
        // Prefer subscription data, fallback to order data
        clientId: subscription?.client_id ?? order.client_id,
        teamId: subscription?.team_id ?? order.team_id,
        serial: robot.serial,
        onlineStatus: this.deriveOnlineStatus(onlineStatus?.status_id),
        deviceLastSeen: onlineStatus?.updated_at ? formatISODate(onlineStatus.updated_at) : null,
        salesOrderStatus: this.deriveSalesOrderStatus(order, status),
        salesOrderReceivedAt: order.created_at ? formatISODate(order.created_at) : null, // Order creation timestamp
        salesOrderDeclinedAt: status?.declined_at ? formatISODate(status.declined_at) : null,
        salesOrderAcceptedAt: status?.processing_at ? formatISODate(status.processing_at) : null, // When processing starts
        salesOrderShippedAt: subscription?.shipped_at ? formatISODate(subscription.shipped_at) : null,
        salesOrderDeliveredAt: status?.delivered_at ? formatISODate(status.delivered_at) : null,
        salesOrderActivatedAt: status?.activated_at ? formatISODate(status.activated_at) : null,
        salesOrderReturnReceivedAt: status?.return_process_started_at
          ? formatISODate(status.return_process_started_at)
          : null,
        salesOrderReturnAcceptedAt: status?.return_accepted_at ? formatISODate(status.return_accepted_at) : null,
        salesOrderReturnRejectedAt: status?.return_rejected_at ? formatISODate(status.return_rejected_at) : null,
        salesOrderReturnedAt: status?.end_at ? formatISODate(status.end_at) : null, // Maps to taas_order_status.end_at (pending Arno's confirmation)
        subscriptionStartAt: subscription?.start_at ? formatISODate(subscription.start_at) : null,
        subscriptionEndAt: subscription?.end_at ? formatISODate(subscription.end_at) : null
      })
    }

    return reportRows
  }

  private deriveOnlineStatus(statusId: number | null | undefined): string {
    // 1 = ONLINE, 0 = OFFLINE, null/undefined = OFFLINE (default)
    if (statusId === 1) return 'ONLINE'
    return 'OFFLINE'
  }

  private deriveSalesOrderStatus(order: any, status: any): string | null {
    // Reuse existing utility function from legacy code
    // This maps the status to a string representation based on timestamp priority
    // See: src/reports/utils/TaasUtils.ts (getTaasOrderStatusString)

    if (status == null) {
      return null
    }

    // Status priority (most recent first):
    // declined > returned > returnRejected > returnAccepted > returnProcessStarted >
    // activated > delivered > shipped/shipping > processing > ordered

    if (status.declined_at != null) {
      return 'declined'
    } else if (status.return_rejected_at != null) {
      return 'returnRejected'
    } else if (status.return_accepted_at != null) {
      return 'returnAccepted'
    } else if (status.return_process_started_at != null) {
      return 'returnProcessStarted'
    } else if (status.activated_at != null) {
      return 'activated'
    } else if (status.delivered_at != null) {
      return 'delivered'
    } else if (status.processing_at != null) {
      return 'processing'
    } else if (status.ordered_at != null) {
      return 'ordered'
    }

    return null
  }
}
```

**Note:** This implementation follows the new approach pattern used in `src/graphql/schema/kpi/retentionKpiService.ts`:
- Service class in same directory as schema definition
- Uses Prisma clients directly
- Uses `formatISODate` from `src/graphql/utils/datetime.ts` for date formatting
- No dependency on legacy code (reports/utils are for legacy REST endpoints)

### Step 3: Export Schema

**File:** `src/graphql/schema/reports/index.ts`

```typescript
export * from './tessaOrderStatusReport'
```

**Update:** `src/graphql/schema/index.ts`

```typescript
import path from 'node:path'

import { makeSchema } from 'nexus'

import { KPI, OperationKPI, RetentionKpiPoint } from './kpi'
import { RawData, RawDataQuery, RobotProfile } from './rawData'
import { TessaOrderStatusReportQuery, TessaOrderStatusReportRow } from './reports'

export const graphqlSchema = makeSchema({
  types: [
    RobotProfile,
    RawData,
    RawDataQuery,
    KPI,
    OperationKPI,
    RetentionKpiPoint,
    TessaOrderStatusReportQuery,
    TessaOrderStatusReportRow
  ],
  contextType: {
    module: path.join(__dirname, '../context/prismaContext.ts'),
    export: 'PrismaGraphqlContext'
  },
  outputs: {
    typegen: path.join(__dirname, '../../../src/generated/graphql/nexus-typegen.ts'),
    schema: path.join(__dirname, '../../../src/generated/graphql/schema.graphql')
  },
  shouldGenerateArtifacts: process.env.NODE_ENV === 'development'
})
```

## Testing Strategy

### Unit Tests

**File:** `test/graphql/schema/reports/tessaOrderStatusReportService.test.ts`

```typescript
import { describe, it, expect, beforeEach, jest } from '@jest/globals'
import { TessaOrderStatusReportService } from '../../../../src/graphql/schema/reports/tessaOrderStatusReportService'

describe('TessaOrderStatusReportService', () => {
  let mockDashboardPrisma: any
  let mockTinybotsPrisma: any
  let service: TessaOrderStatusReportService

  beforeEach(() => {
    // Setup mock Prisma clients
    mockDashboardPrisma = {
      taas_order: {
        findMany: jest.fn()
      }
    }
    mockTinybotsPrisma = {
      robot_account: {
        findMany: jest.fn()
      }
    }
    service = new TessaOrderStatusReportService(mockDashboardPrisma, mockTinybotsPrisma)
  })

  it('should build report with correct field mappings', async () => {
    // Mock data setup
    mockDashboardPrisma.taas_order.findMany.mockResolvedValue([
      {
        id: 1,
        client_id: 'CLIENT123',
        team_id: 'TEAM456',
        relation_id: 63,
        taas_order_status: {
          processing_at: new Date('2024-01-15'),
          delivered_at: new Date('2024-01-20')
        },
        taas_subscription: {
          client_id: 'CLIENT123',
          team_id: 'TEAM456',
          start_at: new Date('2024-01-20'),
          end_at: null,
          shipped_at: new Date('2024-01-18'),
          dashboard_robot: {
            serial: 'TESSA001'
          }
        }
      }
    ])

    mockTinybotsPrisma.robot_account.findMany.mockResolvedValue([
      {
        username: 'TESSA001',
        robot_online_status: {
          status_id: 1,
          updated_at: new Date('2024-10-21')
        }
      }
    ])

    const result = await service.buildReport([63])

    expect(result).toHaveLength(1)
    expect(result[0]).toMatchObject({
      clientId: 'CLIENT123',
      teamId: 'TEAM456',
      serial: 'TESSA001',
      onlineStatus: 'ONLINE'
    })
  })

  it('should handle orders without subscriptions', async () => {
    mockDashboardPrisma.taas_order.findMany.mockResolvedValue([
      {
        id: 2,
        client_id: 'CLIENT999',
        taas_order_status: null,
        taas_subscription: null // No subscription
      }
    ])

    const result = await service.buildReport([63])
    expect(result).toHaveLength(0) // Should skip orders without robots
  })

  it('should correctly derive online status', async () => {
    const service = new TessaOrderStatusReportService(mockDashboardPrisma, mockTinybotsPrisma)

    // Test via private method behavior
    // Status 1 = ONLINE, 0 or null = OFFLINE
  })

  it('should correctly derive sales order status', async () => {
    // Test status derivation priority
  })

  it('should prefer subscription clientId/teamId over order values', async () => {
    mockDashboardPrisma.taas_order.findMany.mockResolvedValue([
      {
        id: 3,
        client_id: 'ORDER_CLIENT',
        team_id: 'ORDER_TEAM',
        taas_order_status: null,
        taas_subscription: {
          client_id: 'SUB_CLIENT',
          team_id: 'SUB_TEAM',
          dashboard_robot: { serial: 'TESSA002' }
        }
      }
    ])

    mockTinybotsPrisma.robot_account.findMany.mockResolvedValue([])

    const result = await service.buildReport([63])
    expect(result[0].clientId).toBe('SUB_CLIENT')
    expect(result[0].teamId).toBe('SUB_TEAM')
  })

  it('should apply pagination with limit and offset', async () => {
    mockDashboardPrisma.taas_order.findMany.mockResolvedValue([
      // Mock paginated results
    ])

    const result = await service.buildReport([63], 10, 20)

    // Verify Prisma was called with correct pagination params
    expect(mockDashboardPrisma.taas_order.findMany).toHaveBeenCalledWith(
      expect.objectContaining({
        skip: 20,
        take: 10
      })
    )
  })

  it('should query all records when limit and offset are not provided', async () => {
    const result = await service.buildReport([63])

    // Verify Prisma was called without skip/take
    expect(mockDashboardPrisma.taas_order.findMany).toHaveBeenCalledWith(
      expect.objectContaining({
        skip: undefined,
        take: undefined
      })
    )
  })
})
```

### Integration Tests

**File:** `test/graphqlIT/reports/tessaOrderStatusReportIT.ts`

```typescript
import { describe, it, expect } from '@jest/globals'
import { graphqlSchema } from '../../../src/graphql/schema'
import { buildPrismaContext } from '../../../src/graphql/context/prismaContext'

describe('tessaOrderStatusReport GraphQL Query', () => {
  it('should return report for given relationIds', async () => {
    const query = `
      query {
        tessaOrderStatusReport(relationIds: [1, 2]) {
          serial
          clientId
          teamId
          onlineStatus
          deviceLastSeen
          salesOrderStatus
          subscriptionStartAt
          subscriptionEndAt
        }
      }
    `

    // Execute query with test context
    // Assert structure and data types
  })

  it('should handle empty relationIds gracefully', async () => {
    const query = `
      query {
        tessaOrderStatusReport(relationIds: []) {
          serial
        }
      }
    `
    // Should return empty array
  })

  it('should apply pagination with limit and offset', async () => {
    const query = `
      query {
        tessaOrderStatusReport(relationIds: [1, 2], limit: 10, offset: 5) {
          serial
          clientId
        }
      }
    `
    // Should return paginated results
  })

  it('should return all records when pagination is not specified', async () => {
    const query = `
      query {
        tessaOrderStatusReport(relationIds: [1]) {
          serial
        }
      }
    `
    // Should return all matching records
  })

  it('should format dates as ISO 8601', async () => {
    // Verify date formatting
  })
})
```

## Database Field Mapping Summary

| GraphQL Field                | Database Source                               | Notes                                                        | Clarify                                                     |
| ---------------------------- | --------------------------------------------- | ------------------------------------------------------------ | ----------------------------------------------------------- |
| `clientId`                   | `taas_order.client_id`                        | Direct mapping                                               | taas_subscription?.clientId ?? taas_order?.clientId ?? null |
| `clientUuid`                 | `taas_order.client_uuid`                      | ECD unique identifier                                        | ✔                                                           |
| `teamId`                     | `taas_order.team_id`                          | Direct mapping                                               | taas_subscription?.teamId ?? taas_order?.teamId ?? null     |
| `serial`                     | `dashboard_robot.serial`                      | Via subscription join                                        | ✔                                                           |
| `onlineStatus`               | `robot_online_status.status_id`               | Derived: 0→OFFLINE, 1→ONLINE                                 | ✔ (serial -> robot_account->robot_online_status)            |
| `deviceLastSeen`             | `robot_online_status.updated_at`              | Flattened date                                               | ✔                                                           |
| `salesOrderStatus`           | Derived from `taas_order_status.*`            | Need follow src/reports/utils/TaasUtils.ts (getTaasOrderStatusString) | ✔                                                           |
| `salesOrderReceivedAt`       | `taas_order.created_at`                       | **Updated: Order creation timestamp**                        | ✔                                                           |
| `salesOrderDeclinedAt`       | `taas_order_status.declined_at`               |                                                              | ✔                                                           |
| `salesOrderAcceptedAt`       | `taas_order_status.processing_at`             | **Updated: When order processing starts**                    | ✔                                                           |
| `salesOrderShippedAt`        | `taas_subscription.shipped_at`                | Proxy field (not in order_status)                            | ✔                                                           |
| `salesOrderDeliveredAt`      | `taas_order_status.delivered_at`              | Direct mapping                                               | ✔                                                           |
| `salesOrderActivatedAt`      | `taas_order_status.activated_at`              | Direct mapping                                               | ✔                                                           |
| `salesOrderReturnReceivedAt` | `taas_order_status.return_process_started_at` | **Clarified mapping**                                        | ✔                                                           |
| `salesOrderReturnAcceptedAt` | `taas_order_status.return_accepted_at`        | Direct mapping                                               | ✔                                                           |
| `salesOrderReturnRejectedAt` | `taas_order_status.return_rejected_at`        | Direct mapping                                               | ✔                                                           |
| `salesOrderReturnedAt`       | `taas_subscription.end_at`                    | **Maps to end_at**                                           | ✔                                                           |
| `subscriptionStartAt`        | `taas_subscription.start_at`                  | Flattened date                                               | ✔                                                           |
| `subscriptionEndAt`          | `taas_subscription.end_at`                    | Flattened date                                               | ✔                                                           |

## Open Questions and Recommendations

### Fields Pending Confirmation (Check with @Arno Nederlof)

1. `salesOrderReturnedAt`: Currently mapped to `taas_order_status.end_at` ✔
   1. **Implementation**: Mapped to `taas_order_status.end_at` in current implementation
2. **~~`salesOrderAcceptedAt`~~**: ~~`taas_order_status.processing_at`~~ ✔ **CONFIRMED**
3. **~~`salesOrderReceivedAt`~~**: ~~`taas_order.created_at`~~ ✔ **CONFIRMED**
4. `salesOrderShippedAt`:
   1. **Mapping**: `taas_subscription.shipped_at` ✔
5. `clientId` **/** `teamId`:  ✔
   1. **Logic**: `taas_subscription.client_id ?? taas_order.client_id ?? null`
   2. **Logic**: `taas_subscription.team_id ?? taas_order.team_id ?? null`

### Performance Considerations

- **Expected volume**: Number of orders per relation can vary widely
- **Pagination**: Implemented with optional `limit` and `offset` parameters
  - If not provided, query returns all matching records
  - Recommended to use pagination for large datasets (e.g., `limit: 100`)
  - Pagination is applied at database level using Prisma's `skip` and `take`
- **Optimization**: Add database indexes on:
    - `taas_order.relation_id`
    - `taas_subscription.serial_id`
    - `robot_account.username`

### Future Enhancements

1. Add optional filters:
    - `teamIds: [String!]`
    - `clientIds: [String!]`
    - `orderStatus: [String!]`
    - Date range filters

2. Add total count field for pagination:
    - Return both data and total count
    - Useful for building pagination UI

3. Add sorting options:
    - Sort by date fields
    - Sort by status

4. Include additional metrics:
    - Order duration (delivered_at - ordered_at)
    - Subscription duration (end_at - start_at)
    - Device uptime percentage

## Implementation Checklist

- [x] ✅ Create GraphQL schema definition: `src/graphql/schema/reports/tessaOrderStatusReport.ts`
  - [x] Define `TessaOrderStatusReportRow` object type with all 19 fields (including `clientUuid`)
  - [x] Add nullable configuration for unavailable fields
  - [x] Make `serial` and `onlineStatus` nullable for [PROD-521] requirement
- [x] ✅ Create OrganisationReports extension: `src/graphql/schema/reports/organisationReportsExtension.ts`
  - [x] Extend `OrganisationReports` type with `tessaOrderStatusReport` field
  - [x] Add `relationIds`, `limit`, and `offset` arguments
  - [x] Use `t.list.field()` for array return type
- [x] ✅ Implement service class: `src/graphql/schema/reports/tessaOrderStatusReportService.ts`
  - [x] Create service class with Prisma clients in constructor
  - [x] Implement data fetching with proper joins
  - [x] Add pagination support with optional `limit` and `offset` parameters
  - [x] Implement field derivation logic (status, online status)
  - [x] Use `flattenDate` for all date fields (ISO 8601 output)
  - [x] Map `salesOrderReceivedAt` to `taas_order.created_at`
  - [x] Map `salesOrderAcceptedAt` to `taas_order_status.processing_at`
  - [x] Map `salesOrderReturnedAt` to `taas_subscription.end_at`
  - [x] Map `salesOrderShippedAt` to `taas_subscription.shipped_at`
  - [x] Implement snake_case → camelCase mapping for `getTaasOrderStatusString`
  - [x] Implement clientId/teamId fallback: `subscription ?? order ?? null`
- [x] ✅ Schema integration:
  - [x] Create `src/graphql/schema/reports/index.ts`
  - [x] Export new types from `src/graphql/schema/index.ts`
  - [x] Register `AllReports` and `TessaOrderStatusReportRow` in makeSchema
- [x] ✅ Write unit tests: `test/graphql/schema/reports/tessaOrderStatusReportService.test.ts`
  - [x] Test all field mappings
  - [x] Test edge cases (missing data, null subscriptions)
  - [x] Test clientId/teamId fallback logic
  - [x] Test status derivation with camelCase mapping
  - [x] Test pagination (with and without limit/offset)
  - [x] All tests passing ✅
- [x] ✅ Write integration tests: `test/graphqlIT/reports/tessaOrderStatusReportIT.ts`
  - [x] Test GraphQL query execution via `reports.organisationReports.tessaOrderStatusReport`
  - [x] Seed test data (relation, integration, contact, robot, subscription, order, order_status, tinybots)
  - [x] Test with seeded data
  - [x] Verify ISO 8601 date formatting
  - [x] Proper cleanup of seeded data
  - [x] All tests passing ✅
- [x] ✅ Documentation:
  - [x] Update this plan document with actual implementation
  - [x] Add inline code comments for complex logic
- [x] ✅ Verification:
  - [x] Run unit tests: All passing ✅
  - [x] Run integration tests: All passing ✅
  - [x] Verify date formats (ISO 8601) ✅
  - [x] Verify status derivation matches requirements ✅

**Status: ✅ COMPLETE - All tasks finished, tests passing**

## Testing Results

### Unit Tests

**File:** `test/graphql/schema/reports/tessaOrderStatusReportService.test.ts`

**Status:** ✅ All tests passing

**Test Coverage:**
- ✅ Field mappings (salesOrderReceivedAt → created_at, salesOrderAcceptedAt → processing_at)
- ✅ ClientId/teamId fallback logic (subscription → order → null)
- ✅ Status derivation with camelCase mapping for legacy utility
- ✅ Pagination with limit/offset parameters
- ✅ Edge cases (missing subscriptions, null robots)

### Integration Tests

**File:** `test/graphqlIT/reports/tessaOrderStatusReportIT.ts`

**Status:** ✅ All tests passing

**Test Coverage:**
- ✅ Full GraphQL query execution through `reports.organisationReports.tessaOrderStatusReport`
- ✅ Data seeding (relation, integration, contact, order, subscription, robot, online status)
- ✅ Field validation (clientId, teamId, serial, onlineStatus, dates)
- ✅ ISO 8601 date formatting
- ✅ Status derivation ('delivered' status confirmed)
- ✅ Proper cleanup of seeded data

**Integration Test Data Flow:**
1. Seeds dashboard database: relation → integration → requester contact → robot → subscription → order → order_status
2. Seeds tinybots database: robot_account → robot_online_status
3. Executes GraphQL query via supertest
4. Validates response structure and data
5. Cleans up all seeded data

## Estimated Implementation Time

**Original Estimate:** ~7 hours

**Actual Time Spent:**
- Schema definition: ~1 hour ✅
- Service implementation: ~3 hours ✅
- Unit tests creation: ~1.5 hours ✅
- Integration tests creation and debugging: ~2.5 hours ✅
- Bug fixes (field type mismatch, foreign key constraints, snake_case mapping): ~1.5 hours ✅
- Documentation updates: ~0.5 hour ✅
- **Total: ~10 hours**

**Notes on Time Variance:**
- Additional time spent on debugging integration test setup (foreign key constraints for `integration` and `taas_order_contact`)
- Fixed GraphQL type mismatch (`t.field` → `t.list.field`)
- Implemented proper snake_case → camelCase mapping for legacy utility compatibility

## Notes for Future Reference

- ✅ **Implementation Complete**: Report follows the **new GraphQL approach** pattern from `src/graphql/schema/kpi/*`
- ✅ Uses service class pattern (similar to `retentionKpiService.ts`) with Prisma clients
- ✅ Cross-database joins are handled in application layer (Prisma limitation)
- ✅ Date formatting uses `flattenDate` from `src/graphql/utils/datetime.ts` for ISO 8601 output
- ✅ Sales order status derivation uses legacy `getTaasOrderStatusString` utility with proper snake_case → camelCase mapping
- ✅ **Pagination support**: Optional `limit` and `offset` parameters
  - Applied at database level using Prisma's `skip` and `take`
  - If not provided, returns all matching records
  - Recommended for large datasets
- ✅ **Field mappings** (confirmed and tested):
  - `salesOrderReceivedAt`: Maps to `taas_order.created_at` (order creation timestamp)
  - `salesOrderAcceptedAt`: Maps to `taas_order_status.processing_at` (when processing starts)
  - `salesOrderReturnedAt`: Maps to `taas_subscription.end_at`
  - `salesOrderShippedAt`: Maps to `taas_subscription.shipped_at`
  - `clientId`/`teamId`: Fallback logic - prefer subscription values, fallback to order values, then null
- ✅ Integrated into `OrganisationReports` type (not direct Query extension)
- ✅ Follow existing Nexus + Prisma patterns for consistency with new approach

### Key Learnings

1. **Foreign Key Dependencies**: Integration tests require proper seeding of foreign key dependencies (`integration`, `taas_order_contact`)
2. **GraphQL Type Definitions**: Must use `t.list.field()` for array returns, not `t.field()` with array service return type
3. **Legacy Utility Integration**: When reusing legacy utilities, ensure proper data format mapping (snake_case → camelCase)
4. **Cross-Database Queries**: Application-level joins via serial number are effective for cross-database relationships

### Maintenance Notes

- If `getTaasOrderStatusString` logic changes, update the camelCase mapping in `deriveSalesOrderStatus()`
- Integration tests use root database credentials for seeding - ensure proper cleanup
- Consider adding database indexes on frequently queried fields:
  - `taas_order.relation_id`
  - `taas_subscription.serial_id`
  - `robot_account.username`
