# 📋 [PROD-984: 2025-12-30] - Add Fields to Tessa Order Status Report

## References

- Current implementation:
  - [wonkers-graphql/src/graphql/schema/reports/tessaOrderStatusReport.ts](../../../wonkers-graphql/src/graphql/schema/reports/tessaOrderStatusReport.ts)
  - [wonkers-graphql/src/graphql/schema/reports/tessaOrderStatusReportService.ts](../../../wonkers-graphql/src/graphql/schema/reports/tessaOrderStatusReportService.ts)
  - [wonkers-graphql/src/graphql/schema/reports/organisationReportsExtension.ts](../../../wonkers-graphql/src/graphql/schema/reports/organisationReportsExtension.ts)
- Standards:
  - [devdocs/tinybots/OVERVIEW.md](../OVERVIEW.md)
  - [devdocs/tinybots/wonkers-graphql/OVERVIEW.md](./OVERVIEW.md)

## User Requirements

Stakeholder requests to expose the following data fields in the `tessaOrderStatusReport` response:

1. **relation name** - The name of the relation associated with the order
2. **tessaExpertNeeded** - Whether expert help is needed (yes/no/unknown)
3. **sales order id** - The specific sales order ID

> [!IMPORTANT]
> **Correction**: Previously this was planned as filter parameters. The requirement has been updated to **add these fields to the response object** instead. The input parameters (filters) should remain unchanged.

## 🎯 Objective

Enhance the `tessaOrderStatusReport` GraphQL query by adding three new fields to the return type. This allows the frontend to display relation names, expert requirement status, and sales order IDs in the report.

### ⚠️ Key Considerations

1. **Database Schema Confirmed** (✅ Verified from `devtools/prisma/dashboard/schema.prisma`):
   - **Relation name**: `taas_order` -> `dashboard_relation.name` (String, nullable)
   - **Expert needed flag**: `taas_order.tessa_expert_needed` (enum: `yes`, `no`, `unknown`)
   - **Sales order ID**: `taas_order.id` (Int, primary key)

2. **Backward Compatibility**: Adding fields to the response is non-breaking.

3. **Performance**: Ensure the necessary data is fetched efficiently. `dashboard_relation` should be joined/included if not already.

4. **Enum Handling**: `tessa_expert_needed` is an enum `yes`, `no`, `unknown`. We should expose this, potentially mapping to a Boolean or keeping as String/Enum in GraphQL. Given the potential for 'unknown', String or Enum is safer, or we map to Boolean if business logic dictates.
   - *Consideration*: `tessaExpertNeeded` as `Boolean`?
   - *Plan*: `tessaExpertNeeded: Boolean` (nullable). `yes` -> true, `no` -> false, `unknown` -> null.

## 🔄 Implementation Plan

### Phase 1: Analysis & Preparation

- [x] ✅ **Investigate Prisma schema for field mappings**
  - **Schema Confirmed** from `devtools/prisma/dashboard/schema.prisma`:
    ```prisma
    model taas_order {
      id                  Int                            @id
      relation_id         Int                            @db.UnsignedInt
      tessa_expert_needed taas_order_tessa_expert_needed @default(unknown) // enum: yes, no, unknown
      dashboard_relation  dashboard_relation             @relation(...)
      ...
    }
    
    model dashboard_relation {
      id   Int     @id
      name String? @db.VarChar(128)  // nullable
      ...
    }
    ```

- [x] ✅ **Define GraphQL Response Fields**
  - `relationName`: `String` (nullable)
  - `tessaExpertNeeded`: `Boolean` (nullable)
  - `salesOrderId`: `Int`

### Phase 2: Implementation (File/Code Structure)

**Files to modify:**
```
wonkers-graphql/
├── src/graphql/schema/reports/
│   ├── tessaOrderStatusReport.ts        # 🔄 [MODIFY] Add new fields to GraphQL definition
│   ├── tessaOrderStatusReportService.ts # 🔄 [MODIFY] Map data in `buildReport`
│   └── organisationReportsExtension.ts  # ✅ No changes
```

### Phase 3: Detailed Implementation Steps

#### Step 1: Update Output Type (`tessaOrderStatusReport.ts`)
- [ ] **Add fields to `TessaOrderStatusReportRow` (or equivalent)**
  ```typescript
  export const TessaOrderStatusReportRow = objectType({
    name: 'TessaOrderStatusReportRow',
    definition(t) {
      // ... existing fields
      t.int('salesOrderId', { description: 'The unique ID of the sales order' })
      t.nullable.string('relationName', { description: 'Name of the relation' })
      t.nullable.boolean('tessaExpertNeeded', { description: 'Whether expert help is needed' })
    },
  })
  ```

#### Step 2: Update Service Layer (`tessaOrderStatusReportService.ts`)
- [ ] **Ensure data fetching**
  - Add `include: { dashboard_relation: true }` to `this.dashboardPrisma.taas_order.findMany` if not present.

- [ ] **Map results**
  - Map `taas_order.id` -> `salesOrderId`
  - Map `taas_order.dashboard_relation?.name` -> `relationName`
  - Map `taas_order.tessa_expert_needed`:
    - `case 'yes': return true`
    - `case 'no': return false`
    - `default: return null`

#### Step 3: Schema Verification
- [ ] **Regenerate Nexus types**
  - `npm run generate:nexus`

#### Step 4: Verification
- [ ] **Test Query**
  ```graphql
  query {
    analytics {
      reports(organisationId: 123) {
        tessaOrderStatusReport(relationIds: [...]) {
          clientId
          salesOrderStatus
          salesOrderId
          relationName
          tessaExpertNeeded
        }
      }
    }
  }
  ```

## 📊 Summary of Results
> Do not summarize the results until the implementation is done and I request it

## 🚧 Outstanding Issues & Follow-up

- [ ] **Confirm Boolean mapping**: Is `unknown` -> `null` acceptable?
