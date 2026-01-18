# 📋 [PROD-437: 2026-01-17] - Sensara Endpoints Review & Update Plan

## References

- **Branch:** `feature/PROD-437-sensara-endpoints` in `sensara-adaptor`
- **Base Branch:** `origin/develop`
- **Repo Overview:** `devdocs/tinybots/sensara-adaptor/OVERVIEW.md`
- **Global Overview:** `devdocs/tinybots/OVERVIEW.md`

## User Requirements

> Original requirements from stakeholder:
>
> - For now ignore authentication
> - Resident endpoints you can get from the database directly
> - For event triggers, use internal endpoints in the tiny internal services to be created by Kai (sample of the current external endpoints)
> - Use the residentId to get the robotId

**API Specification (từ hình ảnh):**

**Resident Endpoints:**
| Method | Path | Description |
|--------|------|-------------|
| GET | `/v1/ext/sensara/residents` | Get all residents and their robot |
| GET | `/v1/ext/sensara/residents/{residentId}` | Get a resident and their robot |
| PATCH | `/v1/ext/sensara/residents/{residentId}` | Update a resident and their robot |

**Events-Triggers Endpoints:**
| Method | Path | Description |
|--------|------|-------------|
| POST | `/v1/ext/sensara/residents/{residentId}/events/subscriptions/triggers` | Create trigger subscription |
| GET | `/v1/ext/sensara/residents/{residentId}/events/subscriptions/triggers` | Get robot trigger subscriptions |
| DELETE | `/v1/ext/sensara/residents/{residentId}/events/subscriptions/triggers/{subscriptionId}` | Delete trigger subscription |

---

## 🎯 Objective

Review implementation trên branch `feature/PROD-437-sensara-endpoints`, so sánh với requirements, xác định gaps và tạo plan cập nhật để align với specification.

### ⚠️ Key Considerations

1. **Path naming convention** phải match với spec (`/v1/ext/sensara/*`)
2. **Authentication** cần ignore theo yêu cầu (hiện tại một số endpoints vẫn có auth)
3. **Missing endpoints** cần được implement
4. **Code quality issues** cần được fix trước khi merge
5. **Backward compatibility** cần được maintain trong quá trình migration

---

## 📊 Phần 1: Analysis - Những Gì Đã Implement

### 1.1 Commits Overview

Branch có **27 commits** với các task chính:
- PROD-474: Create GET endpoint to get all residents
- PROD-506: Create POST trigger endpoint
- PROD-507: Create GET trigger endpoint
- PROD-508: Create DELETE trigger endpoint
- PROD-640: Add authentication for organisations

### 1.2 Endpoints Đã Implement

| Endpoint | Method | Mô tả | Auth |
|----------|--------|-------|------|
| `/v1/sensara/residents` | PUT | Register resident↔robot + hearableLocations | ✅ Required |
| `/v1/sensara/residents/{residentId}` | DELETE | Delete resident mapping | ✅ Required |
| `/v1/sensara/residents` | GET | List residents by organization (via `x-relation-id` header) | ❌ None |
| `/internal/v1/events/residents/{residentId}/subscriptions/triggers` | POST | Create trigger subscription | ❌ None |
| `/internal/v1/events/residents/{residentId}/subscriptions/triggers` | GET | Get trigger subscriptions | ❌ None |
| `/internal/v1/events/residents/{residentId}/subscriptions/triggers/{subscriptionId}` | DELETE | Delete trigger subscription | ❌ None |

### 1.3 Files Changed

```
src/
├── App.ts                                    # 🔄 Modified - routing + DI
├── constants/Container.ts                    # 🔄 Modified - new service names
├── controller/ResidentController.ts          # 🔄 Modified - 4 new methods
├── model/
│   ├── ResidentRobot.ts                      # 🔄 Modified - ResidentRobotWithSerial
│   ├── ServiceConfig.ts                      # 🔄 Modified - new service addresses
│   ├── dto/
│   │   ├── TriggerSubscriptionDto.ts         # ✅ NEW
│   │   └── index.ts                          # 🔄 Modified - export
│   └── mapper/
│       ├── ResidentRobotMapper.ts            # ✅ NEW
│       └── TriggerSubscriptionMapper.ts      # ✅ NEW
├── repository/ResidentRepository.ts          # 🔄 Modified - new queries
└── service/ResidentService.ts                # 🔄 Modified - new methods

test/
├── controller/ResidentControllerIT.ts        # 🔄 Modified - comprehensive tests
├── model/Mapper/
│   ├── ResidentRobotMapperTest.ts            # ✅ NEW
│   └── TriggerSubscriptionMapperTest.ts      # ✅ NEW
└── service/ResidentServiceTest.ts            # ✅ NEW
```

### 1.4 Implementation Details

**GET /v1/sensara/residents Flow:**
```
Request (x-relation-id header)
    ↓
DashboardRobotService.getRobots({relationId})
    ↓
RobotAccountService.getRobotAndUserAccountDetailsBySerials(serials)
    ↓
ResidentRepository.getResidentsWithRobots(robotIds)
    ↓
RobotAccountService.getRobotAccountById(robotId) [per resident]
    ↓
Response: ResidentRobotWithSerial[]
```

**Trigger Endpoints Flow:**
```
Request (residentId in path)
    ↓
ResidentService.getRobotIdByResidentId(residentId)
    ↓
EventService.[post|get|delete]TriggerSubscription(ctx, robotId, ...)
    ↓
TriggerSubscriptionMapper.mapToResponse(subscription)
    ↓
Response
```

---

## 📊 Phần 2: So Sánh Với Requirements - Gaps Analysis

### 2.1 Path Naming Gaps

| Spec | Implementation | Gap |
|------|----------------|-----|
| `/v1/ext/sensara/residents` | `/v1/sensara/residents` | ❌ Missing `/ext` prefix |
| `/v1/ext/sensara/residents/{residentId}/events/subscriptions/triggers` | `/internal/v1/events/residents/{residentId}/subscriptions/triggers` | ❌ Path structure khác hoàn toàn |

### 2.2 Missing Endpoints

| Endpoint | Status | Priority |
|----------|--------|----------|
| `GET /v1/ext/sensara/residents/{residentId}` | ❌ NOT IMPLEMENTED | High |
| `PATCH /v1/ext/sensara/residents/{residentId}` | ❌ NOT IMPLEMENTED | High |

### 2.3 Authentication Gaps

| Requirement | Implementation | Gap |
|-------------|----------------|-----|
| "Ignore authentication for now" | PUT/DELETE residents require `SENSARA_RESIDENT_WRITE_ALL` permission | ❌ Auth vẫn enabled cho write operations |
| | GET residents không có auth | ✅ OK |
| | Trigger endpoints không có auth | ✅ OK |

### 2.4 Functional Gaps

| Requirement | Implementation | Gap |
|-------------|----------------|-----|
| "Resident endpoints from database directly" | GET-all depends on external services (DashboardRobotService, RobotAccountService) | ⚠️ Không hoàn toàn DB-only, sẽ fail nếu services down |
| "Use residentId to get robotId" | ✅ Implemented via `getRobotIdByResidentId()` | None |

### 2.5 Code Quality Issues

| Issue | Location | Severity | Description |
|-------|----------|----------|-------------|
| `console.log` in production | `src/service/ResidentService.ts:L57` | 🔴 High | Should use Logger |
| Missing validation decorator | `src/model/ServiceConfig.ts:L11` | 🟡 Medium | `robotAccountServiceAddress` missing `@IsString()` |
| `describe.only` left in test | `test/controller/ResidentControllerIT.ts:L76` | 🔴 High | Will skip other tests in CI |
| Wrong HTTP status code | `ResidentController.ts:L199` | 🟡 Medium | Returns 500 for invalid subscriptionId, should be 400 |
| Breaking change | `ResidentRepository.ts` | 🟡 Medium | Changed from soft delete to hard delete |

---

## 📊 Phần 3: Kết Luận & Đề Xuất Thay Đổi

### 3.1 Must Fix (Bắt buộc trước khi merge)

#### 3.1.1 Remove `describe.only`

**File:** `test/controller/ResidentControllerIT.ts`

```typescript
// ❌ Current (Line 76)
describe.only('GET /v1/sensara/residents', () => {

// ✅ Fix
describe('GET /v1/sensara/residents', () => {
```

#### 3.1.2 Replace `console.log` with Logger

**File:** `src/service/ResidentService.ts`

```typescript
// ❌ Current (Line 57)
console.log('No residents found for robotIds:', robotIds)

// ✅ Fix
import { Logger } from 'tiny-backend-tools'
// In method:
Logger.loggerFromCtx(ctx).info('No residents found for robotIds', { robotIds })
```

#### 3.1.3 Add missing validation decorator

**File:** `src/model/ServiceConfig.ts`

```typescript
// ❌ Current
robotAccountServiceAddress: string

// ✅ Fix
@IsString()
@MinLength(1)
robotAccountServiceAddress: string
```

#### 3.1.4 Fix HTTP status code

**File:** `src/controller/ResidentController.ts`

```typescript
// ❌ Current (Line 199)
res.status(500).send({ message: 'Invalid subscriptionId' })

// ✅ Fix
res.status(400).send({ message: 'Invalid subscriptionId' })
```

### 3.2 Should Fix (Cần clarify với stakeholder)

#### 3.2.1 Path Naming - Cần quyết định

**Option A:** Đổi sang `/v1/ext/sensara/*` theo spec
- Pros: Match với spec, consistent với external API convention
- Cons: Breaking change nếu có client đang dùng

**Option B:** Giữ `/v1/sensara/*` hiện tại
- Pros: Không breaking change
- Cons: Không match spec

**Recommendation:** Implement cả hai paths, deprecate `/v1/sensara/*` sau khi migrate xong

#### 3.2.2 Trigger Endpoints Path - Cần quyết định

**Spec yêu cầu:** `/v1/ext/sensara/residents/{residentId}/events/subscriptions/triggers`
**Hiện tại:** `/internal/v1/events/residents/{residentId}/subscriptions/triggers`

**Câu hỏi cần clarify:**
1. Trigger endpoints có cần expose external không? (hiện là internal)
2. Nếu expose external, có cần auth không?

#### 3.2.3 Authentication - Cần quyết định

**Câu hỏi:** "Ignore authentication" có nghĩa là:
- A) Remove auth hoàn toàn? (risky for security)
- B) Add feature flag để toggle? (recommended)
- C) Chỉ áp dụng cho development/testing?

**Recommendation:** Implement feature flag `features.ignoreAuth` trong config

### 3.3 Missing Endpoints - Implementation Needed

#### 3.3.1 GET /v1/ext/sensara/residents/{residentId}

**Flow:**
```
residentId (path param)
    ↓
ResidentRepository.getResidentByResidentId(residentId)
    ↓
ResidentRepository.getHearableLocations(robotId)
    ↓
Response: { id, residentId, robotId, hearableLocations }
```

#### 3.3.2 PATCH /v1/ext/sensara/residents/{residentId}

**Flow:**
```
residentId (path param) + body: { hearableLocations: string[] }
    ↓
ResidentRepository.getResidentByResidentId(residentId)
    ↓
Delete existing hearable locations
    ↓
Insert new hearable locations
    ↓
Response: { id, residentId, robotId, hearableLocations }
```

---

## 🔄 Implementation Plan

### Phase 1: Code Quality Fixes (Immediate)

- [ ] **1.1** Remove `describe.only` from `test/controller/ResidentControllerIT.ts:L76`
  - **Outcome:** All tests run in CI
- [ ] **1.2** Replace `console.log` with Logger in `src/service/ResidentService.ts:L57`
  - **Outcome:** Proper logging with context
- [ ] **1.3** Add `@IsString() @MinLength(1)` to `robotAccountServiceAddress` in `src/model/ServiceConfig.ts`
  - **Outcome:** Config validation complete
- [ ] **1.4** Change status 500 → 400 for invalid subscriptionId in `src/controller/ResidentController.ts:L199`
  - **Outcome:** Correct HTTP semantics

### Phase 2: Missing Endpoints (After stakeholder confirmation)

- [ ] **2.1** Implement `GET /v1/ext/sensara/residents/{residentId}`
  - **Files:** `ResidentController.ts`, `ResidentService.ts`, `App.ts`
  - **Test:** Add IT for 200 success and 404 not found
- [ ] **2.2** Implement `PATCH /v1/ext/sensara/residents/{residentId}`
  - **Files:** `ResidentController.ts`, `ResidentService.ts`, `ResidentRepository.ts`, `App.ts`
  - **DTO:** Create `ResidentPatchDto.ts`
  - **Test:** Add IT for 200 success, 404 not found, 400 validation

### Phase 3: Path Normalization (After stakeholder confirmation)

- [ ] **3.1** Add route aliases under `/v1/ext/sensara/*`
  - **Files:** `App.ts`
  - **Approach:** Mount same handlers on both paths
- [ ] **3.2** Decide on trigger endpoints path
  - **Options:** Keep internal OR expose external
  - **Outcome:** Document decision

### Phase 4: Authentication Toggle (After stakeholder confirmation)

- [ ] **4.1** Add `features.ignoreAuth` config
  - **Files:** `config/default.json`, `config/custom-environment-variables.json`
- [ ] **4.2** Conditionally skip auth middleware
  - **Files:** `App.ts`
  - **Logic:** If `ignoreAuth=true`, skip KongHeader + permission validators

### Phase 5: DB-Only Fallback for GET-all

- [ ] **5.1** Wrap service calls in try/catch
  - **Files:** `src/service/ResidentService.ts`
  - **Logic:** If services fail, return DB data without serial
- [ ] **5.2** Add test for fallback scenario
  - **Files:** `test/controller/ResidentControllerIT.ts`

### Phase 6: Testing & Documentation

- [ ] **6.1** Run full test suite: `just -f devtools/Justfile test-sensara-adaptor`
- [ ] **6.2** Ensure lint/typecheck pass
- [ ] **6.3** Update `devdocs/tinybots/sensara-adaptor/OVERVIEW.md` with new endpoints

---

## 📊 File Structure After Updates

```
src/
├── App.ts                                    # 🔄 Add new routes + auth toggle
├── controller/
│   └── ResidentController.ts                 # 🔄 Add GET-by-id, PATCH-by-id
├── model/
│   ├── ServiceConfig.ts                      # 🔄 Add validation + ignoreAuth config
│   └── dto/
│       ├── ResidentPatchDto.ts               # ✅ NEW
│       └── index.ts                          # 🔄 Export new DTO
├── repository/
│   └── ResidentRepository.ts                 # 🔄 Add update hearable locations
└── service/
    └── ResidentService.ts                    # 🔄 Add methods + fix logging + fallback

config/
├── default.json                              # 🔄 Add features.ignoreAuth
└── custom-environment-variables.json         # 🔄 Map env var

test/
└── controller/
    └── ResidentControllerIT.ts               # 🔄 Remove .only + add new tests
```

---

## 📊 Summary of Results

> Chưa thực hiện - sẽ cập nhật sau khi implementation hoàn tất

### ✅ Completed Achievements

_Pending implementation_

---

## 🚧 Outstanding Issues & Follow-up

### ⚠️ Issues/Clarifications Needed

- [ ] **Q1:** Confirm path naming: `/v1/ext/sensara/*` hay `/v1/sensara/*`?
- [ ] **Q2:** Trigger endpoints expose external hay giữ internal?
- [ ] **Q3:** "Ignore authentication" có nghĩa cụ thể là gì? Feature flag hay remove hoàn toàn?
- [ ] **Q4:** Soft delete → Hard delete change có intentional không? Có cần rollback?
- [ ] **Q5:** GET-all có bắt buộc phải có `x-relation-id` header không? Spec không mention này

### 📝 Notes

- Branch đã implement phần lớn logic đúng (residentId → robotId mapping)
- Test coverage khá tốt nhưng có issue với `.only`
- Cần coordinate với team phát triển `tiny-internal-services` nếu có thay đổi EventService contract
