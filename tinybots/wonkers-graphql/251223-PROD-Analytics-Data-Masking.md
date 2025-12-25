# 📋 [PROD: 251223] - Analytics Data Masking for Healthcare Information Compliance

## References

- Monday.com: https://tinybots.monday.com/boards/1369365082/pulses/2547347523
- Repo-specific standard: `devdocs/tinybots/wonkers-graphql/OVERVIEW.md`
- Global standard: `devdocs/tinybots/OVERVIEW.md`
- Compliance Requirement: A.9.4.1 (2) Information access restriction (NEN 7510)

## User Requirements

```
During the internal audit an NC was found that we did not comply with A.9.4.1 (2) Information access restriction (NEN 7510):

'Access to functions related to the processing of personal health information should be isolated (and separated) from access to information processing infrastructure that is not related to the processing of personal health information.'

Currently prod. team has implemented data masking for sales orders. However data masking for analytics tool is not implemented.

We would like to mask the data in relevant analytics tools reports / raw data.

Remove HealthcareInformation and Addresses from reports in Analytics

Find which reports have healthcare information (return reason, healthcare demand and discipline) or an address

Remove these information from the reports
```

## 🎯 Objective

Implement data masking for personal health information (PHI) in wonkers-graphql analytics reports to comply with NEN 7510 A.9.4.1 (2) requirements. Specifically, mask healthcare information fields (return reason, healthcare demand, discipline) and address data from all analytics endpoints and raw data queries.

### ⚠️ Key Considerations

1. **Compliance Priority**: This is a regulatory compliance issue (NEN 7510) that must be addressed to pass internal audit
2. **Scope of PHI**: The following fields are classified as Personal Health Information:
   - `healthcare_demand` / `healthCareDemand`
   - `discipline` / `discipline_id`
   - `return_reason` / `returnReason`
   - Address fields (delivery and pickup addresses)
3. **Existing Masking Implementation**: Sales orders already have data masking implemented - use this as a reference pattern
4. **Two Data Access Mechanisms**: wonkers-graphql has both legacy REST-based resolvers and new Prisma+Nexus direct DB access
5. **Affected Reports**: 
   - `inUseTessaReport` - contains `discipline`, `healthCareDemand`
   - `salesOrderInstallationReport` - may contain address data
   - `salesOrders` (raw data query) - contains `returnReason`, `healthCareDemand`, `discipline`, `deliveryAddress`, `pickupAddress`
   - Any organization-level reports that expose this data

## 🔄 Implementation Plan

### Phase 1: Analysis & Preparation

- [ ] Analyze existing data masking implementation for sales orders
  - **Outcome**: Document the current masking pattern and approach used for sales orders
  - **Location**: Check `src/resolvers/QueryResolver.ts`, `src/datasources/*`, and any masking utilities
  
- [ ] Identify all GraphQL queries and types that expose PHI
  - **Outcome**: Complete list of queries/types to modify:
    - `RawData.salesOrders` query
    - `Report.inUseTessaReport` query  
    - `Report.salesOrderInstallationReport` query
    - `SalesOrder` type (in legacy schema)
    - `InUseTessaReportRow` type
    - `SalesOrderInstallationReportRow` type
    - `Address` type
  
- [ ] Define masking rules and scope
  - **Outcome**: Decision on masking approach:
    - Complete removal (null) vs. masked placeholder (e.g., "***")
    - Whether to apply at resolver level or data source level
    - Permissions/roles that should bypass masking (if any)
  
- [ ] Review database schema to ensure understanding of PHI fields
  - **Outcome**: Verified mapping between Prisma models and GraphQL types
  - **Files**: `prisma/dashboard/schema.prisma`, `devtools/prisma/dashboard/schema.prisma`

### Phase 2: Implementation (File/Code Structure)

#### 🚧 Files to Modify:

```
wonkers-graphql/
├── src/
│   ├── schema/
│   │   ├── RawTaasOrderModel.ts         # 🔄 Update to mask PHI fields
│   │   ├── InUseTessaReportModel.ts     # 🔄 Update to mask discipline, healthCareDemand
│   │   └── SalesOrderInstallationReportModel.ts  # 🔄 Update if contains address data
│   ├── resolvers/
│   │   └── QueryResolver.ts             # 🔄 Apply masking in resolvers
│   ├── reports/
│   │   ├── InUseTessaReportBuilder.ts   # 🔄 Mask PHI in report building
│   │   └── SalesOrderInstallationReportBuilder.ts  # 🔄 Mask address data if present
│   ├── datasources/
│   │   └── TaasOrderApi.ts              # 🔄 Review and potentially apply masking at data source
│   └── utils/                            # 🆕 NEW - Create masking utilities
│       └── phiMasking.ts                # 🆕 NEW - PHI masking helper functions
```

### Phase 3: Detailed Implementation Steps

#### Step 3.1: Create PHI Masking Utility
- [ ] Create `src/utils/phiMasking.ts` with masking functions
  - **Implementation**:
    ```typescript
    // Utility to mask PHI fields in objects
    export function maskHealthcareInformation<T extends Record<string, any>>(
      data: T,
      fieldsToMask: (keyof T)[]
    ): T {
      const masked = { ...data }
      fieldsToMask.forEach(field => {
        if (masked[field] !== null && masked[field] !== undefined) {
          masked[field] = null // or '***' depending on requirement
        }
      })
      return masked
    }
    
    export function maskAddress(address: any): any {
      if (!address) return null
      return null // or return masked object with all fields nulled
    }
    
    // Fields that contain PHI
    export const PHI_FIELDS = {
      healthcareDemand: true,
      healthCareDemand: true,
      healthcare_demand: true,
      discipline: true,
      disciplineId: true,
      discipline_id: true,
      returnReason: true,
      return_reason: true,
      deliveryAddress: true,
      pickupAddress: true,
      address: true,
      healthcareProfessional: true
    }
    ```
  - **Outcome**: Reusable masking utility for consistent PHI handling

#### Step 3.2: Update RawData.salesOrders Query
- [ ] Modify resolver in `src/resolvers/QueryResolver.ts`
  - **Implementation**: Apply masking to sales orders returned from `getRawTaasOrders`
  - **Location**: `Query.salesOrders` resolver (around line 47)
  - **Changes**:
    ```typescript
    import { maskHealthcareInformation, maskAddress } from '../utils/phiMasking'
    
    salesOrders: async (_, args, { dataSources }) => {
      const orders = await dataSources.taasOrderApi.getRawTaasOrders(args)
      // Mask PHI from each order
      return orders.map(order => ({
        ...order,
        healthCareDemand: null,
        discipline: null,
        returnReason: null,
        deliveryAddress: maskAddress(order.deliveryAddress),
        pickupAddress: maskAddress(order.pickupAddress),
        healthcareProfessional: null
      }))
    }
    ```

#### Step 3.3: Update InUseTessaReport
- [ ] Modify `src/reports/InUseTessaReportBuilder.ts`
  - **Implementation**: Mask `discipline` and `healthCareDemand` in report rows
  - **Location**: `createInUseTessaReports` method
  - **Changes**: Before returning report rows, apply masking:
    ```typescript
    return reportRows.map(row => ({
      ...row,
      discipline: null,
      healthCareDemand: null
    }))
    ```
  - **Outcome**: InUseTessaReport no longer exposes PHI

#### Step 3.4: Update SalesOrderInstallationReport (if needed)
- [ ] Review `src/reports/SalesOrderInstallationReportBuilder.ts`
  - **Task**: Check if this report exposes any PHI or address data
  - **Action**: If yes, apply appropriate masking similar to Step 3.3
  - **Outcome**: Confirmation that report is compliant

#### Step 3.5: Update GraphQL Type Definitions (Optional)
- [ ] Consider updating `src/schema/RawTaasOrderModel.ts` and `InUseTessaReportModel.ts`
  - **Task**: Update comments/documentation to indicate fields are masked
  - **Changes**: Add comments like `@deprecated PHI - Masked for compliance`
  - **Outcome**: Clear documentation for API consumers

#### Step 3.6: Handle Legacy vs New Architecture
- [ ] Ensure masking applies to both legacy and Nexus schemas
  - **Legacy**: Handled in resolvers (Step 3.2)
  - **New (Nexus)**: Check if `src/graphql/schema/**` exposes any of these reports
  - **Task**: Search for any Prisma-based queries that might expose PHI
  - **Outcome**: Consistent masking across both architectures

### Phase 4: Testing & Validation

- [ ] Manual testing of affected queries
  - **Test Cases**:
    1. Query `rawData.salesOrders` - verify all PHI fields are null/masked
    2. Query `reports.inUseTessaReport` - verify discipline and healthCareDemand are masked
    3. Query `reports.salesOrderInstallationReport` - verify no address leaks
    4. Check organization-level reports if they expose these fields
  - **Outcome**: All PHI fields properly masked in responses

- [ ] Review with compliance/security team
  - **Task**: Present implementation to team for audit review
  - **Outcome**: Sign-off that implementation meets NEN 7510 A.9.4.1 (2)

- [ ] Document changes
  - **Update**: `devdocs/tinybots/wonkers-graphql/OVERVIEW.md` with PHI masking information
  - **Outcome**: Clear documentation for future developers

### Phase 5: Deployment & Monitoring

- [ ] Deploy to staging environment
  - **Verify**: No breaking changes to downstream analytics consumers
  - **Outcome**: Staging validation complete

- [ ] Coordinate with analytics tool users
  - **Task**: Notify users that healthcare information will no longer be available in reports
  - **Outcome**: Stakeholders informed and prepared

- [ ] Deploy to production
  - **Task**: Standard deployment process
  - **Monitor**: Check logs for any errors related to masking
  - **Outcome**: Production deployment successful

- [ ] Confirm audit compliance
  - **Task**: Update audit documentation showing compliance implementation
  - **Outcome**: Audit NC closed

## 📊 Summary of Results
> Do not summarize the results until the implementation is done

### ✅ Completed Achievements
- [To be filled after implementation]

## 🚧 Outstanding Issues & Follow-up

### ⚠️ Issues/Clarifications

- [ ] **Decision Required**: Should PHI fields return `null` or a masked placeholder like `"[MASKED]"`?
  - **Impact**: API contract with analytics consumers
  - **Recommendation**: `null` is cleaner and more explicit

- [ ] **Permission Consideration**: Should certain user roles (e.g., healthcare professionals) have access to unmasked data?
  - **Impact**: Requires permission checking in resolvers
  - **Recommendation**: Discuss with compliance team before implementing role-based access

- [ ] **Data Access for Internal Tools**: Are there internal Tinybots tools that require access to this data?
  - **Impact**: May need separate internal-only endpoints
  - **Recommendation**: Use separate internal GraphQL endpoints if needed

- [ ] **Backwards Compatibility**: Will masking break existing analytics dashboards?
  - **Impact**: Downstream consumers may need updates
  - **Recommendation**: Communication plan with analytics users before deployment
