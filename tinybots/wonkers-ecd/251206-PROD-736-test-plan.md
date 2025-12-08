# PROD-736: Test Coverage Plan for Concept Order Implementation

## Current State Analysis

**Branch:** `task/PROD-736-TASK2-implementation`

**Current test file:** `test/controller/EcarePuurNotificationControllerIT.ts` (265 lines)

**Test structure:**
```
describe('eCare Puur notification controller IT')
  describe('general') - 2 tests
    ✅ should test integration security
    ✅ should throw 500 when tinybots fails to authenticate
  
  describe('AanmeldBericht') - 2 tests
    ✅ should register for a Taas Tessa minimal
    ✅ should register for a Taas Tessa full
  
  describe('AfmeldBericht') - 1 test
    ✅ should unsubscribe for a Taas Tessa minimal
```

**Total: 5 tests** (VERY MINIMAL coverage)

---

## Gap Analysis: Missing Critical Tests

### Category 1: Multi-Tenant Scenarios (CRITICAL - Bị xóa nhầm)

**Current coverage:** ❌ NONE
**Risk:** High - Production has multiple integrations

**Missing tests:**

1. **`should subscribe with integrationB (different tenant)`**
   - **Purpose:** Verify concept orders work across different integration tenants
   - **Setup:** Use `integrationIdB`, `relationB`
   - **Why critical:** Production has multiple tenants, must ensure isolation

2. **`should subscribe with integrationB and different clientId`**
   - **Purpose:** Verify clientId mapping works for different integrations
   - **Setup:** Use `integrationIdB` with `clientId = '123456'`
   - **Why critical:** Each integration may have different client ID formats

---

### Category 2: API Enrichment Failures (CRITICAL - Core feature)

**Current coverage:** ❌ NONE
**Risk:** VERY HIGH - Best-effort enrichment is core concept order feature

**Missing tests:**

3. **`should subscribe when requester not in form (API enrichment success)`**
   - **Purpose:** Verify API enrichment works when requester missing from notification
   - **Setup:** Use `createValidSubscribeMinimalNoRequester()`
   - **Mock:** `requesterResponse` returns valid data
   - **Expected:** Order created with enriched requester data
   - **Why critical:** Common scenario - form may not have requester

4. **`should subscribe when requester API returns invalid email (fallback defaults)`** ⚠️ **MOST CRITICAL**
   - **Purpose:** Verify fallback defaults when API returns bad data
   - **Setup:** Use `createValidSubscribeMinimalNoRequester()`
   - **Mock:** `requesterResponseWrongPhoneAndEmail` (email invalid/null)
   - **Expected:** Order created with `operations@tinybots.nl`, `SenderId`, `ERROR RETRIEVING EMPLOYEE`
   - **Status code:** `200` (not 400!)
   - **Why critical:** Verifies best-effort enrichment pattern - system continues even when enrichment fails

5. **`should subscribe when requester API returns no contact info (fallback defaults)`**
   - **Purpose:** Verify fallback when API returns completely empty contact
   - **Setup:** Use `createValidSubscribeMinimalNoRequester()`
   - **Mock:** `requesterResponseNoContact` (all fields null)
   - **Expected:** Order created with default `operations@tinybots.nl`
   - **Status code:** `200` (not 400!)
   - **Why critical:** Extreme case of API failure - system must not crash

6. **`should subscribe when client API fails (use UNKNOWN)`**
   - **Purpose:** Verify fallback client data when client API fails
   - **Mock:** `puurApiScope.get(clientUrl).reply(500)` or `.reply(404)`
   - **Expected:** Order created with client `{ ecareNumber: 'UNKNOWN', name: 'UNKNOWN' }`
   - **Status code:** `200`
   - **Why critical:** External API may be down, system must continue

7. **`should subscribe when careteam API fails (use UNKNOWN)`**
   - **Purpose:** Verify fallback careteam when API fails
   - **Mock:** `puurApiScope.get(careteamUrl).reply(500)`
   - **Expected:** Order created with `teamId = 'UNKNOWN'`
   - **Status code:** `200`
   - **Why critical:** Careteam API may be unreliable

8. **`should subscribe when Ecare auth fails (401) - graceful degradation`** ⚠️ **IMPORTANT**
   - **Purpose:** Verify system behavior when Ecare authentication fails
   - **Mock:** `authScope.post('/token').reply(401)`
   - **Expected:** `500` error with Slack notification
   - **Message:** 'Er ging iets mis met het versturen van de melding...'
   - **Why critical:** External service may deny access, must fail gracefully

---

### Category 3: Edge Cases & Advanced Scenarios

**Current coverage:** ❌ NONE
**Risk:** MEDIUM - Less common but important

**Missing tests:**

9. **`should subscribe with multiCareteamResponse (multiple teams)`**
   - **Purpose:** Verify careteam aggregation when client belongs to multiple teams
   - **Setup:** Use `createValidSubscribeFull()`
   - **Mock:** `multiCareteamResponse` instead of `careteamResponse`
   - **Expected:** `teamId = 'Team A Tinybots'` (aggregated teams)
   - **Why important:** Edge case but happens in production

10. **`should unsubscribe when returner not in form (API enrichment success)`**
    - **Purpose:** Verify returner enrichment for unsubscribe
    - **Setup:** Use `createValidReturnMinimalNoReturner()`
    - **Mock:** `requesterResponse` (same API used for returner)
    - **Expected:** Return created with enriched returner data
    - **Why important:** Common unsubscribe scenario

11. **`should unsubscribe when returner API fails (fallback defaults)`**
    - **Purpose:** Verify fallback returner when API fails
    - **Setup:** Use `createValidReturnMinimalNoReturner()`
    - **Mock:** `requesterResponseNoContact`
    - **Expected:** Return created with default `operations@tinybots.nl`
    - **Status code:** `200`
    - **Why important:** Unsubscribe must not fail due to API issues

---

### Category 4: Validation Edge Cases (LOW PRIORITY)

**Current coverage:** ❌ NONE
**Risk:** LOW - Relaxed validation means most should pass

**Optional tests (nice to have):**

12. **`should subscribe with extreme validation failures (still accept)`**
    - **Purpose:** Verify relaxed validation truly accepts bad data
    - **Setup:** Invalid zipcode, invalid email format, missing required fields
    - **Expected:** `200` - order still created
    - **Why optional:** Already covered by relaxed validation logic, but good to document behavior

---

## Implementation Plan

### Phase 1: CRITICAL Tests (Must have before merge)

**Priority:** P0 - BLOCKER
**Estimate:** 2-3 hours

1. Test #4: `should subscribe when requester API returns invalid email (fallback defaults)`
2. Test #5: `should subscribe when requester API returns no contact info (fallback defaults)`
3. Test #6: `should subscribe when client API fails (use UNKNOWN)`
4. Test #7: `should subscribe when careteam API fails (use UNKNOWN)`
5. Test #8: `should subscribe when Ecare auth fails (401)`

**Why P0:** These tests verify core concept order feature (best-effort enrichment). Without these, we cannot confidently deploy.

---

### Phase 2: Important Tests (Should have)

**Priority:** P1 - HIGH
**Estimate:** 1-2 hours

1. Test #1: `should subscribe with integrationB (different tenant)`
2. Test #2: `should subscribe with integrationB and different clientId`
3. Test #3: `should subscribe when requester not in form (API enrichment success)`
4. Test #10: `should unsubscribe when returner not in form (API enrichment success)`
5. Test #11: `should unsubscribe when returner API fails (fallback defaults)`

**Why P1:** Multi-tenant support and enrichment success paths are important but less critical than failure scenarios.

---

### Phase 3: Edge Cases (Nice to have)

**Priority:** P2 - MEDIUM
**Estimate:** 1 hour

1. Test #9: `should subscribe with multiCareteamResponse (multiple teams)`

**Why P2:** Less common scenario, but good to have.

---

### Phase 4: Optional Validation Tests

**Priority:** P3 - LOW
**Estimate:** 30 mins

1. Test #12: `should subscribe with extreme validation failures (still accept)`

**Why P3:** Behavior already covered by relaxed validation implementation.

---

## Test Structure Refactoring Needed

### Current Issues:

1. **No `integrationIdB` setup** - Currently only `integrationId` exists
2. **No `relationB` setup** - Only `relationA` exists
3. **Helper functions missing:**
   - `nockSuccessIntegrationB()` - Not present
   - `nockFailedEmail()` - Not present
   - `nockFailIntegrationBNoContact()` - Not present
   - `nockAuthFail()` - Not present

### Required Setup Changes:

```typescript
// Add to before() block:
const relationB: Relation = {
  relationId: 12345,
  name: 'RelationB',
  teamleaderType: 'company',
  teamleaderId: 'RelationB',
  type: 'Customer'
}

let integrationIdB: number

// In before():
relationB.relationId = await dbSetup.addRelationWithId(relationB.relationId, null, relationB.teamleaderId, relationB.name)
integrationIdB = await dbSetup.addIntegration('integrationB@email.com', 'myIntegrationB', [{
  relationId: relationB.relationId,
  name: 'testRelationB'
}])
await dbSetup.query('INSERT INTO ecd_oauth (`integration_id`, `client_id`, `client_secret`) VALUES(?,?,?)', [integrationIdB, authClientId, clientSecret])
```

### Required Helper Functions:

```typescript
// Add to AanmeldBericht describe block:

const nockSuccessIntegrationB = async (clientUrl: string, careTeamReturn: any, expectedOrder: ConceptOrderDto, employeeId?: string) => {
  // Same as nockSuccess but uses integrationIdB
  authScope.post('/token', { ... }).reply(200, { ... })
  puurApiScope.get(`${clientUrl}/careteam`).reply(200, careTeamReturn)
  puurApiScope.get(clientUrl).reply(200, clientResponse)
  if (employeeId) {
    scopeEmployee.get(`/employees/${employeeId}/teams`).reply(200, multiEmployeeTeamResponse)
    scopeEmployee.get(`/HealthProfessionals/${employeeId}`).reply(200, requesterResponse)
  }
  const minimalOrder = createValidSubscribeMinimal(integrationIdB, relationB.relationId)
  minimalOrder.expectedOrder.clientId = '123456'
  const orderId = await taasOrderSetup.createTaasOrder(minimalOrder.expectedOrder, integrationIdB, relationB.relationId)
  wonkersTaasOrderApiScope.post('/internal/v1/taas-orders/concepts/orders').reply(200, (uri, requestBody) => {
    try {
      expect(requestBody).to.deep.equal(expectedOrder)
    } catch (error) {
      postOrderError = error as Error
    }
    return { id: orderId }
  })
  return orderId
}

const nockFailedEmail = async (clientUrl: string, careTeamReturn: any, expectedOrder: ConceptOrderDto, employeeId?: string) => {
  // Mock requester API returning invalid email
  authScope.post('/token', { ... }).reply(200, { ... })
  puurApiScope.get(`${clientUrl}/careteam`).reply(200, careTeamReturn)
  puurApiScope.get(clientUrl).reply(200, clientResponse)
  if (employeeId) {
    scopeEmployee.get(`/employees/${employeeId}/teams`).reply(200, multiEmployeeTeamResponse)
    scopeEmployee.get(`/HealthProfessionals/${employeeId}`).reply(200, requesterResponseWrongPhoneAndEmail)
  }
  const minimalOrder = createValidSubscribeMinimal(integrationIdB, relationB.relationId)
  minimalOrder.expectedOrder.clientId = '123456'
  const orderId = await taasOrderSetup.createTaasOrder(minimalOrder.expectedOrder, integrationIdB, relationB.relationId)
  wonkersTaasOrderApiScope.post('/internal/v1/taas-orders/concepts/orders').reply(200, (uri, requestBody) => {
    try {
      expect(requestBody).to.deep.equal(expectedOrder)
    } catch (error) {
      postOrderError = error as Error
    }
    return { id: orderId }
  })
  return orderId
}

const nockFailIntegrationBNoContact = async (clientUrl: string, careTeamReturn: any, expectedOrder: ConceptOrderDto, employeeId?: string) => {
  // Mock requester API returning no contact info
  authScope.post('/token', { ... }).reply(200, { ... })
  puurApiScope.get(`${clientUrl}/careteam`).reply(200, careTeamReturn)
  puurApiScope.get(clientUrl).reply(200, clientResponse)
  if (employeeId) {
    scopeEmployee.get(`/employees/${employeeId}/teams`).reply(200, multiEmployeeTeamResponse)
    scopeEmployee.get(`/HealthProfessionals/${employeeId}`).reply(200, requesterResponseNoContact)
  }
  const minimalOrder = createValidSubscribeMinimal(integrationIdB, relationB.relationId)
  minimalOrder.expectedOrder.clientId = '123456'
  const orderId = await taasOrderSetup.createTaasOrder(minimalOrder.expectedOrder, integrationIdB, relationB.relationId)
  wonkersTaasOrderApiScope.post('/internal/v1/taas-orders/concepts/orders').reply(200, (uri, requestBody) => {
    try {
      expect(requestBody).to.deep.equal(expectedOrder)
    } catch (error) {
      postOrderError = error as Error
    }
    return { id: orderId }
  })
  return orderId
}

const nockClientApiFail = async (clientUrl: string, careTeamReturn: any, expectedOrder: ConceptOrderDto, employeeId?: string) => {
  // Mock client API failure (500 or 404)
  authScope.post('/token', { ... }).reply(200, { ... })
  puurApiScope.get(`${clientUrl}/careteam`).reply(200, careTeamReturn)
  puurApiScope.get(clientUrl).reply(500) // CLIENT API FAILS
  if (employeeId) {
    scopeEmployee.get(`/employees/${employeeId}/teams`).reply(200, multiEmployeeTeamResponse)
    scopeEmployee.get(`/HealthProfessionals/${employeeId}`).reply(200, requesterResponse)
  }
  // Expected order should have UNKNOWN client data
  const minimalOrder = createValidSubscribeMinimal(integrationId, relationA.relationId)
  const orderId = await taasOrderSetup.createTaasOrder(minimalOrder.expectedOrder, integrationId, relationA.relationId)
  wonkersTaasOrderApiScope.post('/internal/v1/taas-orders/concepts/orders').reply(200, (uri, requestBody) => {
    try {
      expect(requestBody).to.deep.equal(expectedOrder)
    } catch (error) {
      postOrderError = error as Error
    }
    return { id: orderId }
  })
  return orderId
}

const nockCareteamApiFail = async (clientUrl: string, expectedOrder: ConceptOrderDto, employeeId?: string) => {
  // Mock careteam API failure
  authScope.post('/token', { ... }).reply(200, { ... })
  puurApiScope.get(`${clientUrl}/careteam`).reply(500) // CARETEAM API FAILS
  puurApiScope.get(clientUrl).reply(200, clientResponse)
  if (employeeId) {
    scopeEmployee.get(`/employees/${employeeId}/teams`).reply(200, multiEmployeeTeamResponse)
    scopeEmployee.get(`/HealthProfessionals/${employeeId}`).reply(200, requesterResponse)
  }
  // Expected order should have teamId = 'UNKNOWN'
  const minimalOrder = createValidSubscribeMinimal(integrationId, relationA.relationId)
  const orderId = await taasOrderSetup.createTaasOrder(minimalOrder.expectedOrder, integrationId, relationA.relationId)
  wonkersTaasOrderApiScope.post('/internal/v1/taas-orders/concepts/orders').reply(200, (uri, requestBody) => {
    try {
      expect(requestBody).to.deep.equal(expectedOrder)
    } catch (error) {
      postOrderError = error as Error
    }
    return { id: orderId }
  })
  return orderId
}
```

### Required Imports:

```typescript
import requesterResponseWrongPhoneAndEmail from '../fixtures/requesterResponseWrongPhoneAndEmail.json'
import requesterResponseNoContact from '../fixtures/requesterResponseNoContact.json'
import multiCareteamResponse from '../fixtures/multiCareteamResponse.json'
import { createValidSubscribeMinimalNoRequester, createValidReturnMinimalNoReturner } from '../utils/testUtils'
```

---

## Expected Test Count After Implementation

**Current:** 5 tests
**After Phase 1 (P0):** 10 tests (+5 critical)
**After Phase 2 (P1):** 15 tests (+5 important)
**After Phase 3 (P2):** 16 tests (+1 edge case)
**After Phase 4 (P3):** 17 tests (+1 optional)

**Target:** Minimum 15 tests (Phase 1 + Phase 2)

---

## Acceptance Criteria

✅ **Phase 1 (P0) completed:**
- All API failure scenarios tested (client, careteam, requester, auth)
- Fallback defaults verified (operations@tinybots.nl, UNKNOWN)
- All tests passing with 200 responses (except auth failure → 500)

✅ **Phase 2 (P1) completed:**
- Multi-tenant scenarios tested (integrationB)
- API enrichment success paths tested
- Unsubscribe enrichment scenarios tested

✅ **Test coverage increased from 5 → 15+ tests**

✅ **All tests passing:** `just -f devtools/Justfile test-wonkers-ecd`

---

## Next Steps

1. **Review this plan** - Approve priorities and scope
2. **Implement Phase 1 (P0)** - Critical tests first
3. **Run tests and verify** - Ensure all passing
4. **Implement Phase 2 (P1)** - Important tests
5. **Update plan document** - Mark completed tests
6. **Final verification** - Full test suite passing

---

## Notes

- **Current test file is TOO SIMPLE** - Only 5 tests is insufficient for production
- **Best-effort enrichment pattern MUST be tested** - This is the core feature
- **Multi-tenant support is production requirement** - Cannot skip
- **API failure scenarios are most critical** - External APIs WILL fail in production
- **Old test file had ~30 tests** - But many were validation tests (no longer relevant)
- **Target should be 15-20 tests** - Balance between coverage and maintenance

---

## Risk Assessment

**Without Phase 1 (P0) tests:**
- 🔴 **HIGH RISK** - Best-effort enrichment not verified
- 🔴 **HIGH RISK** - API failures may crash system
- 🔴 **HIGH RISK** - Fallback defaults not tested

**Without Phase 2 (P1) tests:**
- 🟡 **MEDIUM RISK** - Multi-tenant issues may occur
- 🟡 **MEDIUM RISK** - Enrichment success paths unverified

**With Phase 1 + Phase 2:**
- 🟢 **LOW RISK** - Core functionality well tested
- 🟢 **LOW RISK** - Most production scenarios covered
