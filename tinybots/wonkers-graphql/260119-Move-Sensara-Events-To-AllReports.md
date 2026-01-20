# 📋 260119 - Move Sensara Events Report to `allReports`

## References

- Existing implementation plan: `devdocs/tinybots/wonkers-graphql/251223-Sensara-Events-GraphQL.md`

## User Requirements

- Trước đó đã implement theo plan: `devdocs/tinybots/wonkers-graphql/251223-Sensara-Events-GraphQL.md`.
- Stackholder muốn move (Sensara events report) vào trong `allReport` (code hiện tại là `allReports`).
- Chưa rõ khác nhau giữa `allReports` và `organisationReports` (có vẻ liên quan permission).
- Tạo plan để update và trả lời phần thắc mắc trong file plan.

## 🎯 Objective

Move Sensara Events report into `reports { allReports { ... } }` (legacy namespace) and remove it from `reports { organisationReports { ... } }`, while keeping behavior consistent and permissioned.

### ⚠️ Key Considerations

#### 1) Khác nhau giữa `allReports` và `organisationReports` là gì?

- **Về mặt API shape**
  - `allReports` và `organisationReports` đều là “namespace objects” dưới `reports` (thường query là `reports { allReports { ... } }` vs `reports { organisationReports { ... } }`).
  - Chúng không chỉ là “tên khác nhau”, mà hiện đang thuộc **2 cơ chế implement khác nhau** trong repo: legacy SDL+resolver vs Nexus/Prisma.

- **Về permission/scoping (điểm quan trọng nhất)**
  - `allReports` (legacy) đang được dùng theo hướng **admin/global reporting**: một số field dưới `allReports` có guard rõ ràng bằng permission hoặc super-user check.
  - `organisationReports` được thiết kế để hỗ trợ **org-scoped reporting** (đặc biệt cho endpoint external org) qua cơ chế middleware allowlist + scope.

- **Về routes và enforcement thực tế trong code hiện tại**
  - Với endpoint Dashboard GraphQL (`/v4/dashboard/graphql`):
    - `allReports`: nhiều report field được enforce bằng permission tại resolver.
    - `organisationReports`: resolver Nexus hiện tại chủ yếu gọi service/Prisma trực tiếp và **không có permission guard tương đương** (trừ khi từng field tự check).
  - Với endpoint external org (`/ext/v1/dashboard/graphql`):
    - Request bị middleware bọc query về `reports { organisationReports { ... } }`, chặn introspection, và chỉ cho phép một allowlist field + scope `reports:read:self`.
    - Vì vậy `organisationReports` ở đây không chỉ là “tên”, mà là **điểm neo của policy (allowlist + scope + rewrite args/response)**.

- **Kết luận thực hành**
  - Nếu stakeholder muốn “move vào `allReports`”, rất có khả năng mục tiêu là:
    - đồng nhất với cách dashboard đang query legacy,
    - hoặc để field này được quản lý quyền theo kiểu “read all / admin”.
  - Nếu use-case cần phục vụ external org endpoint, việc chuyển sang `allReports` sẽ **đòi hỏi sửa middleware/allowlist** vì hiện middleware chỉ bọc vào `organisationReports`.

#### 2) Backward compatibility

- Hiện Sensara events report đã được expose dưới `organisationReports` (theo plan cũ). Việc “move hẳn” sẽ là **breaking change** nếu có client đang dùng field cũ.
- Vì stakeholder yêu cầu “move hẳn”, plan này sẽ **xóa field dưới `organisationReports`** sau khi đã expose được field tương đương dưới `allReports`.

#### 3) Permission model cho field mới dưới `allReports`

- Mục tiêu hợp lý nhất là match với legacy permission của report tương ứng (ví dụ `SENSARA_EVENTS_READ_ALL`).
- Đồng thời đảm bảo query vẫn đọc từ read replica (`ctx.prisma.tinybots`) như implementation hiện tại.

## 🔄 Implementation Plan

### Phase 1: Analysis & Preparation

- [ ] Map current Sensara events resolver entrypoint(s)
  - **Outcome**: Xác định field name hiện đang nằm ở `organisationReports` và signature args.
- [ ] Identify existing legacy `allReports` Sensara-related fields
  - **Outcome**: Xác định có đang tồn tại `sensaraEventsReport` legacy hay không, và nó trả data từ REST datasource hay Prisma.
- [ ] Decide migration mode: alias vs true move
  - **Outcome**: Chốt approach: **move hẳn** (chỉ còn dưới `allReports`).

### Phase 2: Implementation (File/Code Structure)

```
wonkers-graphql/src/schema/
├── typeDefs.ts                        # 🚧 TODO - Add field to AllReports (legacy SDL)

wonkers-graphql/src/resolvers/
├── QueryResolver.ts                   # 🚧 TODO - Implement AllReports.<newField> resolver + permission guard

wonkers-graphql/src/graphql/schema/reports/
├── sensaraEventReportService.ts       # ✅ EXISTING - Reuse Prisma query logic
├── sensaraEventReport.ts              # ✅ EXISTING - Reuse types (if needed)
└── organisationReportsExtension.ts    # 🚧 TODO - Remove sensaraEventReport field

(optional, only if external-org must use allReports)
wonkers-graphql/src/middleware/
├── organisationRequestMiddleware.ts   # 🚧 TODO - Change query wrapper target
wonkers-graphql/src/middlewares/
├── AllowedOrganizationQueryRegistry.ts# 🚧 TODO - Update allowlist/scope rules
```

### Phase 3: Detailed Implementation Steps

#### Step 1: Add a new field under legacy `AllReports`

- [ ] Update legacy SDL to expose a Sensara events report field under `AllReports`
  - Name decision (default): `sensaraEventReport` (match current Prisma/Nexus field)
  - Args: reuse the implemented filter args (`createdSince`, `timePeriod`, `eventType`, `event`, `robotId`, `limit`, `offset`) to avoid rework.
  - Return: reuse the existing row type shape (or define a legacy-compatible type if schema requires it).

#### Step 2: Implement resolver in legacy `QueryResolver` delegating to Prisma service

- [ ] Add resolver `AllReports.sensaraEventReport` in legacy resolver map
  - Instantiate/reuse `SensaraEventReportService` and call the same `buildReport`/query function.
  - Ensure it uses `ctx.prisma.tinybots` (read replica) and returns ISO strings as per existing behavior.

#### Step 3: Enforce permission for `allReports` variant

- [ ] Wrap the resolver with the same permission model expected for “all org” access
  - Default: require `Permission.SENSARA_EVENTS_READ_ALL` (consistent with legacy pattern).
  - Ensure error code is `FORBIDDEN` when missing permission.

#### Step 4: Decide what to do with `organisationReports` field

- [ ] Remove `organisationReports.sensaraEventReport`
  - Outcome: Sensara events report chỉ còn truy cập qua `allReports`.
  - Note: Nếu external org endpoint cần report này thì phải làm bước 5 để allowlist/scope lại.

#### Step 5 (Conditional): If external org endpoint must use this report

Chỉ làm bước này nếu requirement là “org self-service cũng cần Sensara events report”:

- [ ] Add allowlist entry for Sensara events report in org middleware registry with scope enforcement
  - Ensure relation scoping: enforce `relationIds = [authenticatedOrganization.relationId]` (hoặc tương đương filter) để không leak cross-org.
- [ ] Update org request middleware wrapper behavior if it must target `allReports` instead of `organisationReports`
  - Cân nhắc kỹ vì đây là change lớn: nó ảnh hưởng tất cả external org consumers do response flattening logic hiện dựa trên `organisationReports`.

### Phase 4: Validation

- [ ] Verify schema compiles and merged schema exposes the new field under `allReports`
- [ ] Manual query checks (dashboard route):
  - Missing permission → `FORBIDDEN`
  - With permission → returns rows
  - Filters/pagination behave same as current implementation

## 📊 Summary of Results

> Do not summarize the results until the implementation is done and I request it

## 🚧 Outstanding Issues & Follow-up

### ⚠️ Issues/Clarifications (Optional)

- [ ] Confirm which clients currently consume `organisationReports.sensaraEventReport` to decide deprecation timing.
- [ ] Confirm whether external org endpoint needs this report (impacts middleware/allowlist scope work).
