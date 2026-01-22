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

## 📝 Updated Requirements (from Stakeholder)

1. **Remove fields completely** (not mask) - breaking change accepted
2. **For address and healthcare fields:** 
   - Add permission checks: `TAAS_ORDER_HEALTHCARE_INFO_READ_ALL` and `TAAS_ORDER_ADDRESS_READ_ALL`
   - If missing either permission → throw error (standard BaseResolver pattern)
   - Keep address fields in schema for users with both permissions
3. **Healthcare info fields always removed:**
   - `discipline`
   - `healthCareDemand`
   - `returnReason`
   - `healthcareProfessional`
4. **Simplified implementation:** Remove directly in code, no masking utility needed

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

### Phase 1: Analysis & Scope Verification ✅

**Completed Analysis Results:**

#### PHI Fields to Remove Completely:
1. `discipline` (String)
2. `healthCareDemand` (String)
3. `returnReason` (String)
4. `healthcareProfessional` (Contact type)

#### Address Fields Requiring Permission:
- `deliveryAddress` (Address type)
- `pickupAddress` (Address type)
- **Permissions Required:** 
  - `Permission.TAAS_ORDER_HEALTHCARE_INFO_READ_ALL`
  - `Permission.TAAS_ORDER_ADDRESS_READ_ALL`
- **Behavior:** Throw error if user lacks either permission (using BaseResolver.Wrap pattern)

#### Affected Queries & Types:
1. **Legacy Schema:**
   - `RawData.salesOrders` query
   - `Organisation.salesOrders` query
   - `Report.inUseTessaReport` query
   - `AllOrganisationReports.inUseTessaReport` query
   - `SalesOrder` GraphQL type
   - `InUseTessaReportRow` GraphQL type

2. **Reports:**
   - `InUseTessaReportBuilder.ts` - removes discipline/healthCareDemand

3. **Verified Clean:**
   - ✅ `SalesOrderInstallationReportRow` - No PHI or address data
   - ✅ `TessaOrderStatusReport` (Nexus) - No PHI fields

### Phase 2: Implementation (Simplified Approach)

**Strategy:** Remove fields directly in code, no masking utility needed.

#### 🚧 Files to Modify:

```
wonkers-graphql/
├── src/
│   ├── schema/
│   │   ├── RawTaasOrderModel.ts         # 🔄 Remove PHI fields from GraphQL type
│   │   └── InUseTessaReportModel.ts     # 🔄 Remove discipline, healthCareDemand from GraphQL type
│   ├── resolvers/
│   │   └── QueryResolver.ts             # 🔄 Add permission check for salesOrders queries
│   └── reports/
│       └── InUseTessaReportBuilder.ts   # 🔄 Remove PHI field assignments
```

### Phase 3: Detailed Implementation Steps

#### Step 3.1: Remove PHI Fields from SalesOrder GraphQL Type
- [x] Modify `src/schema/RawTaasOrderModel.ts`
  - **Remove from class definition:**
    - `discipline: string | null`
    - `healthCareDemand: string | null`
    - `returnReason: string | null`
    - `healthcareProfessional: TaasOrderContact | null`
  - **Remove from GraphQL type definition:**
    - `discipline: String`
    - `healthCareDemand: String`
    - `returnReason: String`
    - `healthcareProfessional: Contact`
  - **Outcome:** SalesOrder type no longer exposes PHI

#### Step 3.2: Add Permission Check to RawData.salesOrders
- [x] Wrap resolver with `BaseResolver.Wrap` in `src/resolvers/QueryResolver.ts`
  - **Location:** RawData.salesOrders resolver (line ~47)
  - **Implementation:**
    ```typescript
    salesOrders: BaseResolver.Wrap(
      {
        selectUserId: (ctx) => ctx.dashboardUser?.id,
        permissions: [
          Permission.TAAS_ORDER_HEALTHCARE_INFO_READ_ALL,
          Permission.TAAS_ORDER_ADDRESS_READ_ALL
        ]
      },
      async (_, args, { dataSources }) => {
        return dataSources.taasOrderApi.getRawTaasOrders({
          ...args,
          updatedSince: args.updatedSince == null ? null : moment(args.updatedSince).toISOString()
        })
      }
    )
    ```
  - **Outcome:** Throws error if user lacks either TAAS_ORDER_HEALTHCARE_INFO_READ_ALL or TAAS_ORDER_ADDRESS_READ_ALL permission

#### Step 3.3: Add Permission Check to Organisation.salesOrders
- [x] Wrap resolver with `BaseResolver.Wrap` in `src/resolvers/QueryResolver.ts`
  - **Location:** Organisation.salesOrders resolver (line ~302)
  - **Implementation:** Same pattern as Step 3.2
  - **Outcome:** Consistent permission enforcement across all salesOrders queries

#### Step 3.4: Remove PHI Fields from InUseTessaReportRow Type
- [x] Modify `src/schema/InUseTessaReportModel.ts`
  - **Remove from class definition:**
    - `discipline: string | null`
    - `healthCareDemand: string | null`
  - **Remove from GraphQL type definition:**
    - `discipline: String`
    - `healthCareDemand: String`
  - **Outcome:** InUseTessaReportRow type no longer exposes PHI

#### Step 3.5: Remove PHI Assignments in InUseTessaReportBuilder
- [x] Modify `src/reports/InUseTessaReportBuilder.ts`
  - **Location:** `createInUseTessaReport` method (line ~86-87)
  - **Remove lines:**
    - `discipline: rawTaasOrder?.discipline ?? null,`
    - `healthCareDemand: rawTaasOrder?.healthCareDemand ?? null,`
  - **Outcome:** Report builder no longer includes PHI in output

### Phase 4: Testing & Validation

- [x] **Unit Tests Created:**
  - **File:** `test/SalesOrderPermissionTest.ts`
    - Tests permission check for `RawData.salesOrders`
    - Tests permission check for `Organisation.salesOrders`
    - Tests FORBIDDEN error when user lacks `TAAS_ORDER_HEALTHCARE_INFO_READ_ALL`
    - Tests FORBIDDEN error when user lacks `TAAS_ORDER_ADDRESS_READ_ALL`
    - Tests FORBIDDEN error when user lacks both permissions
    - Tests that PHI fields (discipline, healthCareDemand, returnReason, healthcareProfessional) cannot be queried
  - **File:** `test/InUseTessaReportPhiRemovalTest.ts`
    - Tests that discipline and healthCareDemand are removed from InUseTessaReportRow
    - Tests report builder does not include PHI fields in output
    - Tests GraphQL schema validation fails when trying to query removed fields

- [ ] Manual testing of affected queries
  - **Test Cases:**
    1. **Query `rawData.salesOrders` without permissions:**
       - Should throw GraphQLError with FORBIDDEN code
       - Verify error message indicates missing permissions (either TAAS_ORDER_HEALTHCARE_INFO_READ_ALL or TAAS_ORDER_ADDRESS_READ_ALL)
    2. **Query `rawData.salesOrders` with both permissions:**
       - Should return sales orders successfully
       - Verify no `discipline`, `healthCareDemand`, `returnReason`, `healthcareProfessional` fields in response
       - Verify `deliveryAddress` and `pickupAddress` are present
    3. **Query `organisation.salesOrders` with/without both permissions:**
       - Same behavior as RawData.salesOrders
    4. **Query `reports.inUseTessaReport`:**
       - Verify no `discipline` and `healthCareDemand` fields in response
       - Verify report builds successfully
    5. **Query organisation-level `inUseTessaReport`:**
       - Same as above
  - **Outcome:** All PHI fields removed, permission enforcement working

- [ ] Run test suite
  - **Command:** `npm test` or `pnpm test`
  - **Expected:** All tests pass including new permission tests
  - **Outcome:** Test suite passes without errors

- [ ] Verify GraphQL schema changes
  - **Task:** Generate and review GraphQL schema
  - **Check:** Removed fields no longer appear in schema introspection
  - **Outcome:** Schema reflects compliance requirements

- [ ] Review with compliance/security team
  - **Task:** Present implementation to team for audit review
  - **Outcome:** Sign-off that implementation meets NEN 7510 A.9.4.1 (2)

- [ ] Document changes
  - **Update:** `devdocs/tinybots/wonkers-graphql/OVERVIEW.md` with PHI removal information
  - **Outcome:** Clear documentation for future developers

### Phase 5: Deployment & Monitoring

- [ ] Deploy to staging environment
  - **Note:** Breaking changes expected - analytics consumers will need updates
  - **Outcome:** Staging validation complete

- [ ] Coordinate with analytics tool users
  - **Task:** Notify users that healthcare information fields have been removed
  - **Action:** Provide list of removed fields and permission requirements
  - **Outcome:** Stakeholders informed and prepared

- [ ] Deploy to production
  - **Task:** Standard deployment process
  - **Monitor:** Check logs for permission errors
  - **Outcome:** Production deployment successful

- [ ] Confirm audit compliance
  - **Task:** Update audit documentation showing compliance implementation
  - **Outcome:** Audit NC closed

## 📊 Summary of Results
> Do not summarize the results until the implementation is done

### ✅ Completed Achievements
- [To be filled after implementation]

## 🚧 Outstanding Issues & Follow-up

### ⚠️ Known Impact & Decisions

- [x] **Breaking Change Confirmed:** Fields removed from GraphQL schema
  - **Decision:** Stakeholder confirmed no need to care about breaking changes
  - **Impact:** Analytics tools querying these fields will receive errors

- [x] **Permission Enforcement Approach:** Use existing BaseResolver.Wrap pattern
  - **Decision:** Keep current permission error behavior (throw GraphQLError)
  - **Behavior:** Users without both `TAAS_ORDER_HEALTHCARE_INFO_READ_ALL` and `TAAS_ORDER_ADDRESS_READ_ALL` cannot query salesOrders

- [x] **Scope Confirmed:** Remove PHI from all queries that expose SalesOrder or InUseTessaReport
  - **Queries affected:**
    - `RawData.salesOrders`
    - `Organisation.salesOrders`
    - `Report.inUseTessaReport`
    - `AllOrganisationReports.inUseTessaReport`

### 📋 Post-Implementation Tasks

- [ ] **Update API Documentation:** Notify consumers about removed fields
  - Fields removed: `discipline`, `healthCareDemand`, `returnReason`, `healthcareProfessional`
  - Permissions required for `salesOrders` queries: `TAAS_ORDER_HEALTHCARE_INFO_READ_ALL` and `TAAS_ORDER_ADDRESS_READ_ALL`

- [ ] **Monitor Permission Errors:** Track users hitting permission errors
  - Review logs for `FORBIDDEN` errors on salesOrders queries
  - Work with stakeholders to grant permissions as needed

- [ ] **Analytics Tool Migration:** Coordinate with tool owners
  - Update queries to remove references to deleted fields
  - Ensure tools handle permission errors gracefully

