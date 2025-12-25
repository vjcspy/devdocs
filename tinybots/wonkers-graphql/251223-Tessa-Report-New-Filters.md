# 📋 [PROD-XXX: 2025-12-23] - Add Filters to Tessa Order Status Report

## References

- Current implementation:
  - [wonkers-graphql/src/graphql/schema/reports/tessaOrderStatusReport.ts](../../../wonkers-graphql/src/graphql/schema/reports/tessaOrderStatusReport.ts)
  - [wonkers-graphql/src/graphql/schema/reports/tessaOrderStatusReportService.ts](../../../wonkers-graphql/src/graphql/schema/reports/tessaOrderStatusReportService.ts)
  - [wonkers-graphql/src/graphql/schema/reports/organisationReportsExtension.ts](../../../wonkers-graphql/src/graphql/schema/reports/organisationReportsExtension.ts)
- Standards:
  - [devdocs/tinybots/OVERVIEW.md](../OVERVIEW.md)
  - [devdocs/tinybots/wonkers-graphql/OVERVIEW.md](./OVERVIEW.md)

## User Requirements

Stakeholder requests to add the following filter parameters to `tessaOrderStatusReport` in the analytics query:

1. **relation name** - filter by relation name
2. **tessaExpertNeeded** - yes/no filter (boolean)
3. **sales order id** - filter by specific sales order ID

## 🎯 Objective

Enhance the `tessaOrderStatusReport` GraphQL query with three new optional filter parameters to allow users to narrow down report results by relation name, expert requirement flag, and specific sales order IDs.

### ⚠️ Key Considerations

1. **Database Schema Confirmed** (✅ Verified from `devtools/prisma/dashboard/schema.prisma`):
   - **Relation name**: `taas_order.relation_id` → `dashboard_relation.name` (String, nullable)
   - **Expert needed flag**: `taas_order.tessa_expert_needed` (enum: `yes`, `no`, `unknown`)
   - **Sales order ID**: `taas_order.id` (Int, primary key)

2. **Backward Compatibility**: New parameters must be optional to avoid breaking existing queries

3. **Filter Logic**: Filters should be additive (AND logic) when multiple are provided

4. **Type Safety**: Follow Nexus type patterns for GraphQL arguments

5. **Performance**: Ensure filters are applied at the database query level (Prisma `where` clause), not in-memory filtering

6. **Enum Handling**: `tessa_expert_needed` is an enum with values `yes`, `no`, `unknown` - need to map boolean input to enum values

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
    
    enum taas_order_tessa_expert_needed {
      yes
      no
      unknown
    }
    ```

- [x] ✅ **Determine filter parameter types**
  - **Confirmed GraphQL argument types**:
    - `relationName: String` (nullable) - filters by `dashboard_relation.name`
    - `tessaExpertNeeded: Boolean` (nullable) - maps to enum: `true` → `yes`, `false` → `no`
    - `salesOrderId: Int` (nullable) - filters by `taas_order.id`

- [ ] **Identify edge cases**
  - **Edge cases to handle**:
    - Empty/null filter values → ignored (no filtering applied)
    - Invalid relation names → returns empty results (database handles)
    - Non-existent sales order IDs → returns empty results (database handles)
    - `tessaExpertNeeded` boolean mapping: need to decide if `unknown` should be included
    - Combinations of filters → all applied with AND logic

### Phase 2: Implementation (File/Code Structure)

**Files to modify:**
```
wonkers-graphql/
├── src/graphql/schema/reports/
│   ├── organisationReportsExtension.ts  # 🔄 Add new args to resolver
│   ├── tessaOrderStatusReportService.ts # 🔄 Add filter logic to buildReport
│   └── tessaOrderStatusReport.ts        # ✅ No changes needed (row type)
```

**No new files required** - this is a pure enhancement to existing code.

### Phase 3: Detailed Implementation Steps

#### Step 1: Schema Investigation ✅ COMPLETED
- [x] **Prisma schema confirmed** from `devtools/prisma/dashboard/schema.prisma`
  - `taas_order` model:
    - `id` (Int) - primary key for sales order ID filter
    - `relation_id` (Int) - foreign key to `dashboard_relation`
    - `tessa_expert_needed` (enum) - values: `yes`, `no`, `unknown`
  - `dashboard_relation` model:
    - `name` (String?, nullable) - relation name field
  - **Relationship**: `taas_order` → `dashboard_relation` via `relation_id`

#### Step 2: Update Service Layer (`tessaOrderStatusReportService.ts`)
- [ ] **Modify `buildReport` method signature**
  - Add new optional parameters:
    ```typescript
    async buildReport(
      relationIds: number[],
      limit?: number,
      offset?: number,
      relationName?: string | null,
      tessaExpertNeeded?: boolean | null,
      salesOrderId?: number | null
    ): Promise<TessaOrderStatusReportRow[]>
    ```

- [ ] **Enhance Prisma query with filters**
  - Update `this.dashboardPrisma.taas_order.findMany` where clause:
    ```typescript
    where: {
      relation_id: { in: relationIds },
      // Add conditional filters:
      ...(relationName && { 
        dashboard_relation: { 
          name: relationName 
        } 
      }),
      ...(tessaExpertNeeded !== null && { 
        tessa_expert_needed: tessaExpertNeeded ? 'yes' : 'no'  // Map boolean to enum
      }),
      ...(salesOrderId && { id: salesOrderId })
    }
    ```
  - Note: `dashboard_relation` is already included in the existing query's `include` clause (implied by the relation), so no additional join needed

#### Step 3: Update GraphQL Resolver (`organisationReportsExtension.ts`)
- [ ] **Add new GraphQL arguments**
  - Import necessary Nexus types: `stringArg`, `booleanArg`, `intArg`
  - Add to `tessaOrderStatusReport` field args:
    ```typescript
    args: {
      relationIds: nonNull(list(nonNull(intArg()))),
      limit: intArg({ description: 'Maximum number of records to return (optional)' }),
      offset: intArg({ description: 'Number of records to skip (optional)' }),
      relationName: stringArg({ description: 'Filter by relation name (optional)' }),
      tessaExpertNeeded: booleanArg({ description: 'Filter by expert needed flag (optional)' }),
      salesOrderId: intArg({ description: 'Filter by specific sales order ID (optional)' })
    }
    ```

- [ ] **Pass arguments to service**
  - Update resolver call:
    ```typescript
    return service.buildReport(
      args.relationIds,
      args.limit ?? undefined,
      args.offset ?? undefined,
      args.relationName ?? undefined,
      args.tessaExpertNeeded ?? undefined,
      args.salesOrderId ?? undefined
    )
    ```

#### Step 4: Schema Generation & Validation
- [ ] **Regenerate Nexus types**
  - Run: `npm run generate:nexus` (or equivalent command)
  - Verify: `.schema.graphql` contains new arguments
  - Check: `nexus-typegen.ts` has correct TypeScript types

- [ ] **Manual GraphQL query validation**
  - Test query with each filter individually:
    ```graphql
    query {
      analytics {
        reports(organisationId: 123) {
          tessaOrderStatusReport(
            relationIds: [1, 2, 3]
            relationName: "Test Relation"
            tessaExpertNeeded: true
            salesOrderId: 456
          ) {
            clientId
            serial
            salesOrderStatus
          }
        }
      }
    }
    ```

#### Step 5: Integration Test (Optional Enhancement)
- [ ] **Create or update integration test**
  - File: `test/graphql/schema/reports/tessaOrderStatusReportIT.test.ts`
  - Add test cases for:
    - Filter by relation name
    - Filter by expert needed (true/false)
    - Filter by sales order ID
    - Combination of filters
    - No filters (backward compatibility)

## 📊 Summary of Results
> Do not summarize the results until the implementation is done and I request it

### ✅ Completed Achievements
- TBD after implementation

## 🚧 Outstanding Issues & Follow-up

### ⚠️ Issues/Clarifications

- [x] ✅ **Field Names Confirmed** (Resolved):
  - Expert flag: `tessa_expert_needed` (enum with values: `yes`, `no`, `unknown`)
  - Relation name: `dashboard_relation.name` (String?, nullable)
  - Sales order ID: `taas_order.id` (Int, primary key)
  
- [ ] **Stakeholder Clarification Needed**:
  1. **Enum Mapping Strategy**: When `tessaExpertNeeded` filter is provided:
     - `true` → filter for `tessa_expert_needed = 'yes'`
     - `false` → filter for `tessa_expert_needed = 'no'`
     - **Question**: Should `null`/undefined include all values (including `unknown`)?
     - **Recommendation**: `null` = no filter (returns all), only filter when explicitly set
  
  2. **Filter Interaction**:
     - Should `salesOrderId` filter work with `relationIds`, or should they be mutually exclusive?
     - **Recommendation**: Allow both - if `salesOrderId` is provided, it adds an additional AND condition
     - If order doesn't match the relation, no results returned (database enforces)
  
  3. **Relation Name Matching**:
     - Should `relationName` be exact match (case-sensitive) or support partial/fuzzy matching?
     - **Recommendation**: Start with exact match for simplicity; can enhance later if needed

---

**Next Actions:**
1. ✅ Schema investigation complete
2. Get stakeholder confirmation on the 3 clarification points above (or proceed with recommendations)
3. Implement the changes in service layer and GraphQL resolver
4. Test with real data to validate filter behavior
