# 📋 [PROD-XXX: 2026-01-07] - Sales Order Shipment Information Report

## References

- `devdocs/tinybots/wonkers-graphql/OVERVIEW.md` - Repo architecture, Prisma + Nexus conventions
- `wonkers-graphql/src/graphql/schema/reports/organisationReportsExtension.ts` - OrganisationReports field pattern
- `wonkers-graphql/src/graphql/schema/reports/tessaOrderStatusReport.ts` - objectType pattern
- `wonkers-graphql/src/graphql/schema/reports/tessaOrderStatusReportService.ts` - service mapping pattern
- `wonkers-graphql/src/middlewares/AllowedOrganizationQueryRegistry.ts` - organization access control registry
- `wonkers-graphql/prisma/dashboard/schema.prisma` - Prisma models & relations (dashboard DB)
- `wonkers-graphql/test/graphqlIT/reports/tessaOrderStatusReportIT.ts` - IT test + seeding pattern

## User Requirements

```graphql
query SalesOrderShipmentInformation {
  reports {
    allReports {
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

Implement report field `salesOrderShipmentInformationReport` dưới `reports.allReports` (Prisma + Nexus) để trả về shipment information cho TaaS sales orders từ `dashboard` database.

### ⚠️ Key Considerations

1. **Schema merge constraint:** Root field `reports` nằm ở legacy schema (`src/schema/typeDefs.ts`). Nexus chỉ cần bổ sung `AllReports.salesOrderShipmentInformationReport`.
2. **Security (external org):**
   - Endpoint `/ext/v1/dashboard/graphql` dùng Organization middleware.
   - Cần thêm entry cho field mới vào `AllowedOrganizationQueryRegistry.ts` để không bị 403.
   - Khi request đến từ external organization, resolver phải enforce scoping theo `authenticatedOrganization.relationId` để tránh data leakage.
3. **Data source:** Chỉ dùng `ctx.prisma.dashboard` (không join `tinybots` DB).
4. **GraphQL scalar choices:**
   - `shippedAt` trả `String` ISO 8601 (dùng `flattenDate`).
   - `boxNumber` trả `String` (DB dùng `BigInt`, tránh overflow của GraphQL `Int`).
5. **Sorting:** Default `sortOrder = "desc"`, sort theo `taas_subscription.shipped_at`.
6. **Filtering:** `status: "shipped"` → `taas_subscription.shipped_at IS NOT NULL`; các giá trị status khác (nếu có) cần xác nhận.

### Data Mapping (dashboard DB)

| Field | Prisma Model | Column | Notes |
|-------|--------------|--------|------|
| `deliveryAddressCity` | `taas_order_delivery_address` | `city` | via `taas_order.address_id` |
| `deliveryAddressHomeNumber` | `taas_order_delivery_address` | `home_number` | `Int` |
| `deliveryAddressHomeNumberExtension` | `taas_order_delivery_address` | `home_number_extension` | nullable |
| `deliveryAddressLocationDescription` | `taas_order_delivery_address` | `locationDescription` | nullable (camelCase in Prisma) |
| `deliveryAddressRecipient` | `taas_order_delivery_address` | `recipient` | |
| `deliveryAddressStreet` | `taas_order_delivery_address` | `street` | |
| `deliveryAddressZipcode` | `taas_order_delivery_address` | `zipcode` | |
| `tessaExpertNeeded` | `taas_order` | `tessa_expert_needed` | enum `yes/no/unknown` → `boolean/null` |
| `shippedAt` | `taas_subscription` | `shipped_at` | ISO 8601 |
| `boxNumber` | `dashboard_robot` | `box_number` | via `taas_subscription.serial_id → dashboard_robot.id` |
| `clientNumber` | `taas_subscription` / `taas_order` | `client_id` | prefer subscription, fallback order |
| `trackTraceCode` | `taas_order` | `track_trace_code` | nullable |
| `requesterEmail` | `taas_order_contact` | `email` | via `taas_order.requester_id` |
| `organisationName` | `dashboard_relation` | `name` | via `taas_order.relation_id` |

## 🔄 Implementation Plan

### Phase 1: Analysis & Preparation

- [ ] Validate Prisma relations needed for single-query include
  - **Outcome:** Confirm `taas_order` includes address, requester contact, subscription, robot, relation
- [ ] Define functional scope & defaults
  - **Outcome:** Decide behavior for `status`, `sortOrder`, and whether to support `limit/offset`
- [ ] Define organization scoping strategy
  - **Outcome:** For external org requests, enforce `relation_id = authenticatedOrganization.relationId`

### Phase 2: Implementation (File/Code Structure)

```
wonkers-graphql/src/graphql/schema/reports/
├── allReports/
│   ├── allReports.type.ts                                # 🔄 UPDATE - Add field resolver
│   ├── index.ts                                          # 🔄 UPDATE - Export new type
│   └── salesOrderShipment/                               # ✅ CREATE - New report folder
│       ├── salesOrderShipment.type.ts                    # ✅ CREATE - Nexus objectType
│       ├── salesOrderShipment.service.ts                 # ✅ CREATE - Prisma service
│       └── index.ts                                      # ✅ CREATE - Exports

wonkers-graphql/src/middlewares/
└── AllowedOrganizationQueryRegistry.ts                   # 🔄 UPDATE - Allow new field for orgs

wonkers-graphql/test/graphqlIT/reports/
└── salesOrderShipmentInformationReportIT.ts              # ✅ CREATE - Integration tests

wonkers-db/src/main/resources/db/migration/
└── V59__graphql_report_ro_shipment_permissions.sql       # ✅ CREATE - DB permissions for report
```

### Phase 3: Detailed Implementation Steps

#### Step 1: Define Row Type (`salesOrderShipmentInformationReport.ts`)

Define `SalesOrderShipmentInformationReportRow` bằng `objectType` (nullable fields theo DB reality).

#### Step 2: Implement Service (`salesOrderShipmentInformationReportService.ts`)

- Signature đề xuất (có `relationId` optional để enforce scoping cho external org):
  - `buildReport(params: { relationId?: number; status?: string; sortOrder?: 'asc'|'desc'; limit?: number; offset?: number })`
- Prisma query:
  - `taas_order.findMany({ where, include, orderBy, skip, take })`
  - `where` gồm:
    - `relation_id` nếu `relationId` được set (external org enforced)
    - filter theo `status` (hiện chỉ “shipped”)
- Mapping:
  - address via `taas_order_delivery_address_taas_order_address_idTotaas_order_delivery_address`
  - requester via `taas_order_contact_taas_order_requester_idTotaas_order_contact`
  - subscription/robot via `taas_subscription.dashboard_robot`
  - `shippedAt` dùng `flattenDate(subscription?.shipped_at)`
  - `boxNumber` dùng `robot?.box_number?.toString() ?? null`

#### Step 3: Add Field to `AllReports` (`allReports/allReports.type.ts`)

- Add `salesOrderShipmentInformationReport` với args tối thiểu theo requirement:
  - `relationIds: [Int]` (optional filter by organization)
  - `sortOrder: String` (default: `"desc"`, only accept `"asc"|"desc"`)
  - `status: String` (optional)
  - `limit: Int` (default: 1000)
  - `offset: Int` (default: 0)
- Resolver behavior:
  - Create service với `ctx.prisma.dashboard`
  - Pass `relationIds` if provided for filtering

#### Step 4: Export from `allReports/index.ts`

Export `SalesOrderShipmentInformationReportRow` để Nexus schema include type.

#### Step 5: Allow Organization Access (`AllowedOrganizationQueryRegistry.ts`)

Add new entry:
- key: `SALES_ORDER_SHIPMENT_INFORMATION_REPORT`
- `requiredScope: ['reports:read:self']`
- `argRewriter`: optional (không bắt buộc nếu resolver tự enforce `relationId` từ context)

## 🧪 Test Cases (Integration)

### Test File: `wonkers-graphql/test/graphqlIT/reports/salesOrderShipmentInformationReportIT.ts`

Reuse pattern của `tessaOrderStatusReportIT.ts`:

- Seeding helpers:
  - `seedRelation()`, `seedIntegration()`, `seedRequesterContact()`
  - `seedDeliveryAddress()`
  - `seedRobot(serial, { boxNumber? })`
  - `seedSubscription(serial, { shippedAt?, clientId? })`
  - `seedOrder(subscriptionId|null, { addressId?, trackTraceCode?, tessaExpertNeeded? })`
- Cleanup seeded entities after each test

### IT Scenarios (Implemented)

- [x] **1. Basic mapping (full chain)**
  - Seed: relation → requester → address → robot(boxNumber) → subscription(shipped_at) → order(track_trace_code, tessa_expert_needed)
  - Expect all fields populated; `boxNumber` is string; `shippedAt` ISO 8601
- [x] **2. Nullable handling - Order without address**
  - Order has no delivery address → all address fields null
- [x] **3. Nullable handling - Order without robot**
  - Subscription exists but no robot linked → `boxNumber = null`
- [x] **4. Nullable handling - Order without subscription**
  - `shippedAt = null`, `boxNumber = null`, `clientNumber` fallback to order.client_id
- [x] **5. Filter by status: shipped**
  - Mix shipped and non-shipped orders → `status: "shipped"` only returns shipped
- [x] **6. Filter by status: delivered**
  - Filter `status: "delivered"` returns only delivered orders
- [x] **7. Sort order**
  - Default sort is desc by shipped_at; `sortOrder: "asc"` flips order
- [x] **8. Pagination (limit/offset)**
  - Test `limit: 2, offset: 1` returns correct subset
- [x] **9. Organization auth - valid scopes**
  - External org with `reports:read:self` returns only orders for `x-relation-id`
- [x] **10. Organization auth - invalid scopes**
  - Missing/invalid scope returns 403

## 🚧 Outstanding Issues & Follow-up

### ⚠️ Clarifications Needed (ask before implementation)

- [x] **Pagination:** Có cần `limit/offset` (hoặc default limit) để tránh trả về quá nhiều rows không? => Có
- [x] **Status contract:** `status` chỉ hỗ trợ `"shipped"` hay có thêm values khác? Nếu có, định nghĩa mapping như thế nào? => Dựa vào database để bổ sung thêm các kiểu khác đi
- [ ] **Internal scope:** Internal users có được query cross-relations (all relations) không, hay phải bắt buộc filter theo `relationId(s)`? => chưa hiểu cái này, cứ làm như cũ chuẩn để response về đúng graphql là được.
- [x] **External org exposure:** Field này có cần expose cho `/ext/v1/dashboard/graphql` không? Nếu có, scope nào áp dụng ngoài `reports:read:self`? => không thay đổi endpoint, dùng lại endpoint cũ, chỉ là permission có thể sẽ cần nhưng chúng ta cũng đã có cơ chế để thêm permission rồi. Nhưng sẽ bổ sung sau
