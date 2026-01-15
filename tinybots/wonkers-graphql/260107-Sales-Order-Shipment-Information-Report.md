# 📋 [PROD-XXX: 2026-01-07] - Sales Order Shipment Information Report

## References

- `wonkers-graphql/src/graphql/schema/reports/organisationReportsExtension.ts` - Report extension pattern
- `wonkers-graphql/src/graphql/schema/reports/tessaOrderStatusReport.ts` - Nexus objectType pattern
- `wonkers-graphql/src/graphql/schema/reports/tessaOrderStatusReportService.ts` - Service pattern
- `wonkers-graphql/prisma/dashboard/schema.prisma` - Database schema
- `wonkers-graphql/test/graphqlIT/reports/tessaOrderStatusReportIT.ts` - Test pattern reference

## User Requirements

```graphql
query SalesOrderShipmentInformation {
  reports {
    organisationReports {
      salesOrderShipmentInformationReport(sortOrder: "desc", status: "shipped") {
        deliveryAddressCity
        deliveryAddressHomeNumber
        deliveryAddressHomeNumberExtension
        deliveryAddressLocationDescription
        deliveryAddressRecipient
        deliveryAddressStreet
        deliveryAddressZipcode
        tessaExpertNeeded
        shippedAt
        boxNumber
        clientNumber
        trackTraceCode
        requesterEmail
        organisationName
      }
    }
  }
}
```

## 🎯 Objective

Implement GraphQL report `salesOrderShipmentInformationReport` trong `reports.organisationReports` sử dụng Prisma + Nexus pattern để cung cấp thông tin vận chuyển của các sales orders.

### ⚠️ Key Considerations

1. **Query Path:** `reports.organisationReports.salesOrderShipmentInformationReport` (Prisma + Nexus approach)
2. **Pattern Follow:** Theo pattern của `tessaOrderStatusReport` trong `src/graphql/schema/reports/`
3. **Data Source:** Tất cả từ `dashboard` database, query qua `ctx.prisma.dashboard`
4. **Data Source Mapping:**

| Field | Source Table | Column | Notes |
|-------|-------------|--------|-------|
| `deliveryAddressCity` | `taas_order_delivery_address` | `city` | Via `taas_order.address_id` |
| `deliveryAddressHomeNumber` | `taas_order_delivery_address` | `home_number` | Integer |
| `deliveryAddressHomeNumberExtension` | `taas_order_delivery_address` | `home_number_extension` | Nullable |
| `deliveryAddressLocationDescription` | `taas_order_delivery_address` | `locationDescription` | Nullable |
| `deliveryAddressRecipient` | `taas_order_delivery_address` | `recipient` | |
| `deliveryAddressStreet` | `taas_order_delivery_address` | `street` | |
| `deliveryAddressZipcode` | `taas_order_delivery_address` | `zipcode` | |
| `tessaExpertNeeded` | `taas_order` | `tessa_expert_needed` | Enum: 'yes'/'no'/'unknown' → Boolean/null |
| `shippedAt` | `taas_subscription` | `shipped_at` | ISO 8601 datetime |
| `boxNumber` | `dashboard_robot` | `box_number` | Via `taas_subscription.serial_id` |
| `clientNumber` | `taas_subscription` / `taas_order` | `client_id` | Prefer subscription, fallback to order |
| `trackTraceCode` | `taas_order` | `track_trace_code` | Nullable |
| `requesterEmail` | `taas_order_contact` | `email` | Via `taas_order.requester_id` |
| `organisationName` | `dashboard_relation` | `name` | Via `taas_order.relation_id` |

5. **Filter Logic:**
   - `status: "shipped"` → Filter orders có `taas_subscription.shipped_at IS NOT NULL`
   - `sortOrder: "asc" | "desc"` → Sort theo `shipped_at`

## 🔄 Implementation Plan

### Phase 1: Analysis & Preparation

- [ ] Confirm data availability trong database
  - **Outcome:** Xác nhận tất cả fields có thể lấy được từ database relationships
- [ ] Define edge cases:
  - Orders không có delivery address (`address_id = NULL`) → all address fields = null
  - Orders không có robot assigned → `boxNumber = null`
  - Orders không có subscription → `shippedAt = null`, `clientNumber` từ order

### Phase 2: Implementation (File/Code Structure)

```
wonkers-graphql/src/graphql/schema/reports/
├── index.ts                                              # 🔄 UPDATE - Export new files
├── organisationReportsExtension.ts                       # 🔄 UPDATE - Add field to OrganisationReports
├── salesOrderShipmentInformationReport.ts                # ✅ CREATE - Nexus objectType definition
└── salesOrderShipmentInformationReportService.ts         # ✅ CREATE - Prisma-based service

wonkers-graphql/test/graphqlIT/reports/
└── salesOrderShipmentInformationReportIT.ts              # ✅ CREATE - Integration tests
```

### Phase 3: Detailed Implementation Steps

#### Step 1: Create Nexus ObjectType (`salesOrderShipmentInformationReport.ts`) 🚧

```typescript
import { objectType } from 'nexus'

export const SalesOrderShipmentInformationReportRow = objectType({
  name: 'SalesOrderShipmentInformationReportRow',
  definition(t) {
    // Delivery Address Fields
    t.nullable.string('deliveryAddressCity', { description: 'City of delivery address' })
    t.nullable.int('deliveryAddressHomeNumber', { description: 'Home number of delivery address' })
    t.nullable.string('deliveryAddressHomeNumberExtension', { description: 'Home number extension' })
    t.nullable.string('deliveryAddressLocationDescription', { description: 'Location description' })
    t.nullable.string('deliveryAddressRecipient', { description: 'Recipient name' })
    t.nullable.string('deliveryAddressStreet', { description: 'Street name' })
    t.nullable.string('deliveryAddressZipcode', { description: 'Zipcode' })
    
    // Order Fields
    t.nullable.boolean('tessaExpertNeeded', { description: 'Whether Tessa expert assistance is needed' })
    t.nullable.string('shippedAt', { description: 'When the order was shipped (ISO 8601)' })
    t.nullable.string('boxNumber', { description: 'Robot box number' })
    t.nullable.string('clientNumber', { description: 'Client identifier' })
    t.nullable.string('trackTraceCode', { description: 'Track and trace code for shipment' })
    t.nullable.string('requesterEmail', { description: 'Email of the requester' })
    t.nullable.string('organisationName', { description: 'Name of the organisation' })
  }
})
```

#### Step 2: Create Service (`salesOrderShipmentInformationReportService.ts`) 🚧

```typescript
import { PrismaClient as DashboardPrismaClient } from '../../../generated/prisma/dashboard'
import { flattenDate } from '../../utils/datetime'

export interface SalesOrderShipmentInformationReportRow {
  deliveryAddressCity: string | null
  deliveryAddressHomeNumber: number | null
  deliveryAddressHomeNumberExtension: string | null
  deliveryAddressLocationDescription: string | null
  deliveryAddressRecipient: string | null
  deliveryAddressStreet: string | null
  deliveryAddressZipcode: string | null
  tessaExpertNeeded: boolean | null
  shippedAt: string | null
  boxNumber: string | null
  clientNumber: string | null
  trackTraceCode: string | null
  requesterEmail: string | null
  organisationName: string | null
}

export class SalesOrderShipmentInformationReportService {
  constructor(private dashboardPrisma: DashboardPrismaClient) {}

  async buildReport(
    sortOrder: 'asc' | 'desc' = 'desc',
    status?: string
  ): Promise<SalesOrderShipmentInformationReportRow[]> {
    // Build where clause based on status
    const whereClause: any = {}
    if (status === 'shipped') {
      whereClause.taas_subscription = {
        shipped_at: { not: null }
      }
    }

    // Query orders with all related data
    const orders = await this.dashboardPrisma.taas_order.findMany({
      where: whereClause,
      include: {
        taas_order_delivery_address_taas_order_address_idTotaas_order_delivery_address: true,
        taas_subscription: {
          include: {
            dashboard_robot: true
          }
        },
        dashboard_relation: true,
        taas_order_contact_taas_order_requester_idTotaas_order_contact: true
      },
      orderBy: {
        taas_subscription: {
          shipped_at: sortOrder
        }
      }
    })

    // Map to report rows
    return orders.map(order => {
      const address = order.taas_order_delivery_address_taas_order_address_idTotaas_order_delivery_address
      const subscription = order.taas_subscription
      const robot = subscription?.dashboard_robot
      const requester = order.taas_order_contact_taas_order_requester_idTotaas_order_contact

      return {
        deliveryAddressCity: address?.city ?? null,
        deliveryAddressHomeNumber: address?.home_number ?? null,
        deliveryAddressHomeNumberExtension: address?.home_number_extension ?? null,
        deliveryAddressLocationDescription: address?.locationDescription ?? null,
        deliveryAddressRecipient: address?.recipient ?? null,
        deliveryAddressStreet: address?.street ?? null,
        deliveryAddressZipcode: address?.zipcode ?? null,
        tessaExpertNeeded: order.tessa_expert_needed === 'yes' ? true 
                         : order.tessa_expert_needed === 'no' ? false 
                         : null,
        shippedAt: flattenDate(subscription?.shipped_at),
        boxNumber: robot?.box_number?.toString() ?? null,
        clientNumber: subscription?.client_id ?? order.client_id,
        trackTraceCode: order.track_trace_code ?? null,
        requesterEmail: requester?.email ?? null,
        organisationName: order.dashboard_relation?.name ?? null
      }
    })
  }
}
```

#### Step 3: Update Extension (`organisationReportsExtension.ts`) 🚧

Add new field to `OrganisationReports`:

```typescript
t.list.field('salesOrderShipmentInformationReport', {
  type: nonNull(SalesOrderShipmentInformationReportRow),
  args: {
    sortOrder: stringArg({ 
      description: 'Sort order by shipped date: "asc" or "desc" (default: "desc")' 
    }),
    status: stringArg({ 
      description: 'Filter by status: "shipped" returns only shipped orders' 
    })
  },
  async resolve(_parent, args, ctx) {
    const service = new SalesOrderShipmentInformationReportService(ctx.prisma.dashboard)
    const sortOrder = args.sortOrder === 'asc' ? 'asc' : 'desc'
    return service.buildReport(sortOrder, args.status ?? undefined)
  }
})
```

#### Step 4: Update Index (`index.ts`) 🚧

```typescript
export * from './salesOrderShipmentInformationReport'
```

## 🧪 Test Cases

### Test File: `test/graphqlIT/reports/salesOrderShipmentInformationReportIT.ts`

#### Setup & Teardown

- Use `DashboardPrismaClient` for seeding test data
- Seed helper functions:
  - `seedRelation()` - Create test organisation
  - `seedIntegration()` - Create test integration
  - `seedRequesterContact()` - Create test requester contact
  - `seedDeliveryAddress()` - Create delivery address
  - `seedRobot()` - Create dashboard robot with box_number
  - `seedSubscription()` - Create subscription with shipped_at
  - `seedOrder()` - Create order linking all entities
- Cleanup after each test to ensure isolation

#### Test Cases

##### 1. Basic Data Return Tests

| Test Case | Description | Expected Outcome |
|-----------|-------------|------------------|
| `should return order with all fields populated` | Seed complete order with all relations | All 14 fields returned with correct values |
| `should return correct delivery address fields` | Seed order with delivery address | All 7 address fields match seeded data |
| `should return correct requester email` | Seed order with requester contact | `requesterEmail` matches contact email |
| `should return correct organisation name` | Seed order with relation | `organisationName` matches relation name |

##### 2. Nullable Field Handling

| Test Case | Description | Expected Outcome |
|-----------|-------------|------------------|
| `should return null for missing delivery address` | Order without `address_id` | All address fields = null |
| `should return null for missing robot/boxNumber` | Subscription without robot | `boxNumber = null` |
| `should return null for order without subscription` | Order with `taas_id = null` | `shippedAt = null`, `boxNumber = null` |
| `should return null for missing track trace code` | Order without track_trace_code | `trackTraceCode = null` |
| `should handle null home_number_extension` | Address without extension | `deliveryAddressHomeNumberExtension = null` |

##### 3. tessaExpertNeeded Mapping

| Test Case | Description | Expected Outcome |
|-----------|-------------|------------------|
| `should map tessa_expert_needed 'yes' to true` | Order with `tessa_expert_needed = 'yes'` | `tessaExpertNeeded = true` |
| `should map tessa_expert_needed 'no' to false` | Order with `tessa_expert_needed = 'no'` | `tessaExpertNeeded = false` |
| `should map tessa_expert_needed 'unknown' to null` | Order with `tessa_expert_needed = 'unknown'` | `tessaExpertNeeded = null` |

##### 4. Filter Tests

| Test Case | Description | Expected Outcome |
|-----------|-------------|------------------|
| `should filter by status 'shipped'` | Seed shipped + non-shipped orders | Only shipped orders returned |
| `should return all orders when no status filter` | Seed mixed orders | All orders returned |
| `should handle empty result when no shipped orders` | No shipped orders exist | Empty array `[]` |

##### 5. Sort Order Tests

| Test Case | Description | Expected Outcome |
|-----------|-------------|------------------|
| `should sort by shippedAt DESC by default` | Multiple orders with different shipped_at | Newest shipped first |
| `should sort by shippedAt ASC when specified` | `sortOrder: "asc"` | Oldest shipped first |
| `should handle sort with null shippedAt values` | Mix of shipped and non-shipped | Null values at end/beginning based on DB behavior |

##### 6. clientNumber Priority Tests

| Test Case | Description | Expected Outcome |
|-----------|-------------|------------------|
| `should prefer subscription client_id over order client_id` | Both have different values | Returns subscription's client_id |
| `should fallback to order client_id when subscription is null` | No subscription | Returns order's client_id |

##### 7. Edge Cases

| Test Case | Description | Expected Outcome |
|-----------|-------------|------------------|
| `should return empty array when no orders exist` | Empty database | `[]` |
| `should handle multiple orders correctly` | Seed 5 orders | Returns all 5 with correct data |
| `should handle special characters in fields` | Address with "ë", "ñ" | Characters preserved correctly |

##### 8. Date Format Tests

| Test Case | Description | Expected Outcome |
|-----------|-------------|------------------|
| `should return shippedAt in ISO 8601 format` | Order with shipped_at | Format: `YYYY-MM-DDTHH:mm:ss.sssZ` |

## 📊 Summary of Results

> Do not summarize the results until the implementation is done and I request it

### ✅ Completed Achievements
- [ ] Service layer implemented
- [ ] Nexus types defined
- [ ] Extension updated
- [ ] All tests passing

## 🚧 Outstanding Issues & Follow-up

### ⚠️ Clarifications Needed

- [ ] **Pagination:** Report có cần `limit`/`offset` arguments không? Các report khác có support
- [ ] **Additional status values:** Ngoài "shipped", có cần support status khác không? (e.g., "all", "pending", "delivered")
- [ ] **Organization scoping:** Report có cần filter theo `relationIds` không? Pattern phổ biến trong các report khác

