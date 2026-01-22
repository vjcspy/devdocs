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

**API Specification (from image):**

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

Review implementation on branch `feature/PROD-437-sensara-endpoints`, compare with requirements, identify gaps, and create an update plan to align with the specification.

### ⚠️ Key Considerations

1. **Path naming convention** must match the spec (`/v1/ext/sensara/*`)
2. **Authentication** needs to be ignored as requested (currently some endpoints still have auth)
3. **Missing endpoints** need to be implemented
4. **Code quality issues** need to be fixed before merge
5. **Backward compatibility** needs to be maintained during migration

---

## 📊 Part 1: Analysis - What Has Been Implemented

### 1.1 Commits Overview

The branch has **27 commits** with the following main tasks:
- PROD-474: Create GET endpoint to get all residents
- PROD-506: Create POST trigger endpoint
- PROD-507: Create GET trigger endpoint
- PROD-508: Create DELETE trigger endpoint
- PROD-640: Add authentication for organisations

### 1.2 Implemented Endpoints

| Endpoint | Method | Description | Auth |
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

## 📊 Part 2: Comparison with Requirements - Gaps Analysis

### 2.1 Path Naming Gaps

| Spec | Implementation | Gap |
|------|----------------|-----|
| `/v1/ext/sensara/residents` | `/v1/sensara/residents` | ❌ Missing `/ext` prefix |
| `/v1/ext/sensara/residents/{residentId}/events/subscriptions/triggers` | `/internal/v1/events/residents/{residentId}/subscriptions/triggers` | ❌ Path structure is completely different |

### 2.2 Missing Endpoints

| Endpoint | Status | Priority |
|----------|--------|----------|
| `GET /v1/ext/sensara/residents/{residentId}` | ❌ NOT IMPLEMENTED | High |
| `PATCH /v1/ext/sensara/residents/{residentId}` | ❌ NOT IMPLEMENTED | High |

### 2.3 Authentication Gaps

| Requirement | Implementation | Gap |
|-------------|----------------|-----|
| "Ignore authentication for now" | PUT/DELETE residents require `SENSARA_RESIDENT_WRITE_ALL` permission | ❌ Auth is still enabled for write operations |
| | GET residents has no auth | ✅ OK |
| | Trigger endpoints have no auth | ✅ OK |

### 2.4 Functional Gaps

| Requirement | Implementation | Gap |
|-------------|----------------|-----|
| "Resident endpoints from database directly" | GET-all depends on external services (DashboardRobotService, RobotAccountService) | ⚠️ Not completely DB-only; will fail if services are down |
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

## 📊 Part 3: Conclusion & Proposed Changes

### 3.1 Must Fix (Mandatory before merge)

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

### 3.2 Should Fix (Need clarification from stakeholder)

#### 3.2.1 Path Naming - Decision needed

**Option A:** Change to `/v1/ext/sensara/*` according to spec
- Pros: Matches spec, consistent with external API convention
- Cons: Breaking change if clients are already using it

**Option B:** Keep current `/v1/sensara/*`
- Pros: No breaking change
- Cons: Does not match spec

**Recommendation:** Implement both paths, deprecate `/v1/sensara/*` after migration is complete

#### 3.2.2 Trigger Endpoints Path - Decision needed

**Spec requires:** `/v1/ext/sensara/residents/{residentId}/events/subscriptions/triggers`
**Current:** `/internal/v1/events/residents/{residentId}/subscriptions/triggers`

**Questions to clarify:**
1. Do trigger endpoints need to be exposed externally? (currently internal)
2. If exposed externally, is auth required?

#### 3.2.3 Authentication - Decision needed

**Question:** What does "Ignore authentication" specifically mean?
- A) Remove auth completely? (risky for security)
- B) Add feature flag to toggle? (recommended)
- C) Only apply to development/testing?

**Recommendation:** Implement feature flag `features.ignoreAuth` in config

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

> Not executed - will be updated after implementation is complete

### ✅ Completed Achievements

_Pending implementation_

---

## 🚧 Outstanding Issues & Follow-up

### ⚠️ Issues/Clarifications Needed

- [ ] **Q1:** Confirm path naming: `/v1/ext/sensara/*` or `/v1/sensara/*`?
- [ ] **Q2:** Do trigger endpoints need to be exposed externally? (currently internal)
- [ ] **Q3:** What does "Ignore authentication" specifically mean? Feature flag or remove completely?
- [ ] **Q4:** Is the Soft delete → Hard delete change intentional? Is rollback needed?
- [ ] **Q5:** Is the `x-relation-id` header mandatory for GET-all? Spec does not mention this

### 📝 Notes

- Branch has implemented most logic correctly (residentId → robotId mapping)
- Test coverage is quite good but has an issue with `.only`
- Need to coordinate with the `tiny-internal-services` development team if there are changes to EventService contract
