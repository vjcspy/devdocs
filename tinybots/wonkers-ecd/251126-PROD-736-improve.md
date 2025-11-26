# 📋 [PROD-736: 2025-11-26] - Implement Concept Order Flow for Ecare Puur

## References

- Global standard: `/Users/kai/work/tinybots/devdocs/tinybots/OVERVIEW.md`
- Repo-specific standard: `/Users/kai/work/tinybots/devdocs/tinybots/wonkers-ecd/OVERVIEW.md`
- Template: `/Users/kai/work/tinybots/devdocs/agent/TEMPLATE.md`
- Existing ZSP implementation: `src/service/zsp/ZspService.ts` (reference pattern)
- Ecare Puur service: `src/service/ecare/EcarePuurService.ts` (to be refactored)
- Ecare Puur mapping: `src/service/ecare/EcarePuurMappingService.ts` (validation logic)
- Controller: `src/controller/EcarePuurNotificationController.ts`

## User Requirements

In wonkers-ecd, we have a flow for ecare, where they have a healthcare dossier of clients. In that dossier they can apply for Tessa with a form.

**Current Behavior:**
We accept the input, validate the input and return an error if fields are incorrect. This is not wanted, because they have to fill in the whole form again, we want to make the approach easier.

**Desired Behavior:**
Use concept orders. This flow is also in wonkers-ecd under the zsp flow. We accept all input and instead create a concept order even if the fields are wrong!

**Task:** Implement this concept order flow for standard ecd ecare (Ecare Puur).

## 🎯 Objective

Refactor the Ecare Puur notification flow in `wonkers-ecd` to use concept orders instead of production orders, accepting all form submissions regardless of validation errors and allowing back-office staff to review and correct data before order fulfillment.

### ⚠️ Key Considerations

1. **Breaking Change Risk**: This changes the behavior from immediate order placement to concept order creation - ensure backward compatibility or coordinate deployment
2. **Data Quality**: Accepting invalid data means back-office staff must review all orders - need proper concept order management workflow
3. **Existing Pattern**: ZSP already implements this pattern via `ConceptService` - reuse the same approach for consistency
4. **Validation Strategy**: Need to relax strict validation while still capturing all submitted data
5. **Error Handling**: Remove strict validation errors but maintain structural DTO validation
6. **Client/Order Tracking**: Concept orders may need different tracking in `ecd_order` table
7. **Email/Slack Notifications**: Verify concept order notifications work correctly for Ecare Puur

## 🔄 Implementation Plan

### Phase 1: Analysis & Preparation

- [ ] Analyze current Ecare Puur flow
  - **Outcome**: Document current `EcarePuurService.notify()` logic:
    - Maps to `SubscribeFields` / `UnsubscribeFields`
    - Validates required fields strictly
    - Fetches requester/client from Ecare APIs
    - Creates production order via `WonkersTaasOrderService`
    - Stores in `ecd_order` table

- [ ] Analyze ZSP concept order pattern
  - **Outcome**: Document ZSP implementation:
    - `ZspService.subscribe()` uses `ConceptService.createConceptOrder()`
    - `ZspService.unsubscribe()` uses `ConceptService.createConceptReturn()`
    - Relaxed validation - accepts all input
    - Builds concept order form with all submitted data
    - Uses `tb-concept-taas-orders` client

- [ ] Identify validation points to relax
  - **Outcome**: List validation rules in `EcarePuurMappingService` that need to change:
    - Phone number format validation
    - Required field validation
    - Address completeness checks
    - Date format validation
    - Email format validation

- [ ] Define concept order mapping strategy
  - **Outcome**: Design how to map Ecare Puur form to concept order DTO:
    - Map all `AdditionalFields` even if invalid
    - Preserve original values for review
    - Flag validation errors as notes/comments
    - Maintain requester/client enrichment from APIs

### Phase 2: Implementation (File/Code Structure)

Proposed changes to existing files:

```text
src/service/ecare/
├── EcarePuurService.ts              # 🔄 IN PROGRESS - Refactor to use ConceptService
│   ├── notify()                     # Main entry - switch to concept flow
│   ├── _subscribe()                 # Change from WonkersTaasOrderService to ConceptService
│   └── _unsubscribe()               # Change to concept return flow
├── EcarePuurMappingService.ts       # 🔄 IN PROGRESS - Relax validation
│   ├── mapToSubscribeFields()       # Accept invalid data
│   ├── mapToUnsubscribeFields()     # Accept invalid data
│   └── _validateXYZ()               # Make validations non-blocking
└── EcarePuurApiService.ts           # ✅ NO CHANGE - Keep API enrichment

src/controller/
└── EcarePuurNotificationController.ts # ✅ NO CHANGE - Webhook entry point stays same

src/model/
├── EcarePuurNotification.ts         # ✅ NO CHANGE - DTO validation at entry
└── EcarePuurFields.ts               # 🔄 EVALUATE - May need optional fields

src/repository/
└── ClientIdRepository.ts            # 🚧 EVALUATE - Check if concept orders need different tracking

test/
├── ecareService/                    # 🔄 UPDATE - Update tests for concept flow
│   ├── EcarePuurServiceTest.ts     # Update expectations
│   └── EcarePuurMappingServiceTest.ts # Test relaxed validation
└── controller/
    └── EcarePuurNotificationControllerTest.ts # Update integration tests
```

**New Dependencies:**

- Already have `tb-concept-taas-orders` (used by ZSP)
- `ConceptService` already registered in DI container

### Phase 3: Detailed Implementation Steps

#### Step 1: Refactor EcarePuurMappingService Validation

**File**: `src/service/ecare/EcarePuurMappingService.ts`

**Current Behavior:**

```typescript
// Strict validation throws errors
_validateRequiredField(value, fieldName) {
  if (!value) throw new BadRequestError(`${fieldName} is required`)
}
```

**New Behavior:**

```typescript
// Capture validation errors but don't throw
_validateRequiredField(value, fieldName): ValidationWarning | null {
  if (!value) return { field: fieldName, issue: 'missing_required' }
  return null
}
```

**Changes:**

- Make all validation methods return warnings instead of throwing
- Collect all validation warnings
- Pass warnings to concept order as notes/comments
- Keep structural DTO validation (class-validator) for webhook payload

#### Step 2: Integrate ConceptService in EcarePuurService

**File**: `src/service/ecare/EcarePuurService.ts`

**Current Subscribe Flow:**

```typescript
private async _subscribe(notification: PuurNotificationDto, ...) {
  // 1. Map and validate (throws on error)
  const fields = this._mappingService.mapToSubscribeFields(...)
  
  // 2. Enrich from Ecare APIs
  const requester = await this._getRequester(...)
  const client = await this._apiService.getClient(...)
  
  // 3. Build production order
  const orderDto = this._buildOrderV2Dto(fields, requester, client, ...)
  
  // 4. Place production order
  const result = await this._wonkersTaasOrderService.placeOrder(orderDto)
  
  // 5. Store tracking
  await this._clientIdRepository.store(...)
  
  return result.orderId
}
```

**New Concept Flow:**

```typescript
private async _subscribe(notification: PuurNotificationDto, ...) {
  // 1. Map without throwing (collect warnings)
  const { fields, warnings } = this._mappingService.mapToSubscribeFields(...)
  
  // 2. Enrich from Ecare APIs (best effort)
  const requester = await this._getRequesterOrDefault(...)
  const client = await this._getClientOrDefault(...)
  
  // 3. Build concept order form
  const conceptForm = this._buildConceptOrderForm(
    fields, 
    requester, 
    client,
    warnings // Include validation warnings
  )
  
  // 4. Create concept order
  const result = await this._conceptService.createConceptOrder(
    conceptForm,
    notification.IntegrationId
  )
  
  // 5. Store tracking (concept order ID)
  await this._clientIdRepository.storeConceptOrder(...)
  
  return result.conceptOrderId
}
```

**New Helper Methods:**

- `_buildConceptOrderForm()` - Map to concept order DTO
- `_getRequesterOrDefault()` - Don't fail if API enrichment fails
- `_getClientOrDefault()` - Return partial data if API fails
- `_formatValidationWarnings()` - Format warnings as notes

#### Step 3: Update Unsubscribe Flow for Concept Returns

**File**: `src/service/ecare/EcarePuurService.ts`

**Current Unsubscribe:**

- Looks up order IDs from `ecd_order`
- Validates order status
- Creates production return via `WonkersTaasOrderService`

**New Concept Return:**

```typescript
private async _unsubscribe(notification: PuurNotificationDto, ...) {
  // 1. Map return fields (relaxed validation)
  const { fields, warnings } = this._mappingService.mapToUnsubscribeFields(...)
  
  // 2. Enrich returner data (best effort)
  const returner = await this._getReturnerOrDefault(...)
  
  // 3. Build concept return form
  const conceptReturnForm = this._buildConceptReturnForm(
    fields,
    returner,
    warnings
  )
  
  // 4. Create concept return
  await this._conceptService.createConceptReturn(
    conceptReturnForm,
    notification.IntegrationId
  )
}
```

#### Step 4: Update ClientIdRepository for Concept Orders

**File**: `src/repository/ClientIdRepository.ts`

**Evaluate if changes needed:**

- Current: Stores `ecd_client_id` → `taas_order_id` mapping
- Concept: May need to store `ecd_client_id` → `concept_order_id` mapping
- Or use different approach since concepts are temporary

**Potential Change:**

```typescript
// Add method for concept order tracking
async storeConceptOrder(
  ecd_client_id: string,
  concept_order_id: string,
  integration_id: number
): Promise<void>

// Update query methods to handle both production and concept orders
async findOrderIds(ecd_client_id: string): Promise<{
  productionOrderIds: number[],
  conceptOrderIds: string[]
}>
```

#### Step 5: Update Controller Response

**File**: `src/controller/EcarePuurNotificationController.ts`

**Current:**

```typescript
const orderId = await this._ecarePuurService.notify(...)
return res.status(200).json({ orderId })
```

**Evaluate if response needs to change:**

- Option 1: Keep same response format (return concept order ID)
- Option 2: Add flag to indicate concept vs production order
- Option 3: Return both concept ID and status

**Recommended:**

```typescript
const result = await this._ecarePuurService.notify(...)
return res.status(200).json({
  conceptOrderId: result.conceptOrderId,
  type: 'concept', // Indicate it's a concept order
  message: 'Order created for review'
})
```

#### Step 6: Email and Slack Notifications

**Verification:**

- Ensure `ConceptService` triggers appropriate notifications
- Check if Ecare Puur needs custom email templates for concept orders
- Verify Slack notifications go to correct channels

**Files to check:**

- Concept email templates in `tb-concept-taas-orders` library
- Slack config in `config/default.json` → `conceptSlackConfig`
- Email config per environment

### Phase 4: Testing Strategy

#### Unit Tests

**File**: `test/ecareService/EcarePuurMappingServiceTest.ts`

- Test relaxed validation returns warnings instead of throwing
- Test all validation scenarios collect multiple warnings
- Test invalid data is preserved in mapped output

**File**: `test/ecareService/EcarePuurServiceTest.ts`

- Test subscribe creates concept order via ConceptService
- Test unsubscribe creates concept return
- Mock ConceptService calls
- Test API enrichment failures don't block concept creation
- Test validation warnings are included in concept form

#### Integration Tests

**File**: `test/controller/EcarePuurNotificationControllerTest.ts`

- Test webhook with valid data creates concept order
- Test webhook with invalid data still creates concept order
- Test response format includes concept order ID
- Test multiple validation errors handled gracefully

#### Manual Testing Checklist

- [ ] Submit valid Ecare Puur form → creates concept order
- [ ] Submit form with missing required fields → creates concept order with warnings
- [ ] Submit form with invalid phone number → creates concept order, preserves data
- [ ] Submit form with invalid address → creates concept order
- [ ] Verify concept order appears in back-office for review
- [ ] Verify email notifications sent correctly
- [ ] Verify Slack notifications go to correct channel
- [ ] Submit unsubscribe request → creates concept return
- [ ] Check `ecd_order` table tracking works

## 📊 Summary of Results

> Do not summarize the results until the implementation is done and I request it

### ✅ Completed Achievements

- [ ] Analyzed current Ecare Puur vs ZSP flows
- [ ] Refactored validation to accept all input
- [ ] Integrated ConceptService for subscribe/unsubscribe
- [ ] Updated tests for concept order flow
- [ ] Verified email/Slack notifications
- [ ] Manual testing completed
- [ ] Deployment coordinated with back-office team

## 🚧 Outstanding Issues & Follow-up

### ⚠️ Issues/Clarifications

1. **Back-office workflow** - Verify back-office team has process to review/approve concept orders from Ecare Puur
   - Impact: Orders won't be fulfilled until manually reviewed
   - Action needed: Coordinate with operations team

2. **Data retention** - How long should concept orders be retained before cleanup?
   - Impact: Database growth if concepts not cleaned up
   - Action needed: Define retention policy

3. **Migration strategy** - Should we support both flows during transition?
   - Impact: May need feature flag to switch between production and concept flows
   - Action needed: Discuss deployment approach

4. **Error handling** - What happens if ConceptService is unavailable?
   - Impact: Webhooks would fail entirely
   - Action needed: Define fallback behavior or retry logic

5. **Ecare API failures** - If enrichment APIs fail, concept orders missing data
   - Impact: Back-office may need to manually lookup client/requester info
   - Action needed: Document enrichment failure scenarios

6. **Order tracking** - Clarify if `ecd_order` table should track concept orders differently
   - Impact: May affect unsubscribe/return lookups
   - Action needed: Review repository changes with team

### 📝 Next Steps

1. Review plan with team and product owner
2. Confirm concept order workflow with back-office
3. Implement Phase 3 changes step by step
4. Create feature branch and submit PR
5. Coordinate deployment with operations team
6. Monitor concept order creation after deployment
7. Document new behavior in OVERVIEW.md

---

## 📋 Task Breakdown

### Task 1: Relax Validation Logic to Accept All Input

**Description:**
Refactor the `EcarePuurMappingService` to change validation from strict (throwing errors) to relaxed (collecting warnings). This allows the service to accept all form submissions regardless of validation errors, capturing issues as warnings instead of rejecting the request.

**Why this task:**
This is the foundation for accepting invalid data. Without relaxed validation, the service will continue to throw errors and reject invalid forms before they can be converted to concept orders.

**Files to change:**

- `src/service/ecare/EcarePuurMappingService.ts` - Refactor all validation methods
  - Change `_validateRequiredField()` to return `ValidationWarning | null` instead of throwing
  - Change `_validatePhoneNumber()` to return warnings instead of throwing
  - Change `_validateAddress()` to return warnings instead of throwing
  - Change `_validateEmail()` to return warnings instead of throwing
  - Update `mapToSubscribeFields()` to collect all warnings and return `{ fields, warnings }`
  - Update `mapToUnsubscribeFields()` to collect all warnings and return `{ fields, warnings }`
- `src/model/EcarePuurFields.ts` - Evaluate if fields need to be made optional (currently may have required constraints)
- `test/ecareService/EcarePuurMappingServiceTest.ts` - Update all tests
  - Test that invalid data returns warnings instead of throwing
  - Test that multiple validation errors are collected
  - Test that original data is preserved even when invalid

**Expected outcome:**
All validation errors become non-blocking warnings that can be passed along to the concept order system.

---

### Task 2: Integrate ConceptService for Order Creation

**Description:**
Refactor `EcarePuurService` to use `ConceptService` instead of `WonkersTaasOrderService` for creating orders and returns. This changes the flow from creating immediate production orders to creating concept orders that require back-office review.

**Why this task:**
This is the core business logic change - switching from production order flow to concept order flow (matching the existing ZSP pattern).

**Files to change:**

- `src/service/ecare/EcarePuurService.ts` - Main refactoring
  - Update `_subscribe()` method to call `ConceptService.createConceptOrder()` instead of `WonkersTaasOrderService.placeOrder()`
  - Add `_buildConceptOrderForm()` helper to map Ecare fields + warnings to concept order DTO
  - Update `_unsubscribe()` method to call `ConceptService.createConceptReturn()`
  - Add `_buildConceptReturnForm()` helper for return mapping
  - Update `_getRequester()` / `_getClient()` to handle API failures gracefully (return partial data instead of throwing)
  - Pass validation warnings from Task 1 into concept order form as notes/comments
- `src/repository/ClientIdRepository.ts` - Evaluate tracking changes
  - Add `storeConceptOrder()` method if concept orders need different tracking
  - Update `findOrderIds()` to distinguish between production and concept orders (if needed)
- `test/ecareService/EcarePuurServiceTest.ts` - Update all tests
  - Mock `ConceptService` instead of `WonkersTaasOrderService`
  - Test concept order creation with validation warnings
  - Test API enrichment failures don't block concept creation
  - Test concept return creation

**Expected outcome:**
Ecare Puur notifications create concept orders instead of production orders, with all validation warnings included.

---

### Task 3: Update Controller Response and Integration Tests

**Description:**
Update the `EcarePuurNotificationController` response format to indicate concept order creation, and update all integration tests to verify the new concept order flow works end-to-end.

**Why this task:**
The webhook API contract may need to change to inform clients that concept orders are being created. Integration tests ensure the full flow (webhook → service → concept order) works correctly.

**Files to change:**

- `src/controller/EcarePuurNotificationController.ts` - Update response format
  - Change response to include `conceptOrderId` instead of `orderId`
  - Add `type: 'concept'` field to indicate concept order
  - Add descriptive message: `'Order created for review'`
  - Handle any response format changes from Task 2
- `test/controller/EcarePuurNotificationControllerTest.ts` - Integration tests
  - Test webhook with valid data creates concept order
  - Test webhook with invalid data (missing fields, bad phone, etc.) still creates concept order
  - Test response format includes `conceptOrderId` and `type: 'concept'`
  - Test multiple validation errors handled gracefully
  - Verify no errors thrown for invalid input
- Email/Slack notification verification (manual testing checklist)
  - Document where to verify concept email templates (`tb-concept-taas-orders` library)
  - Document Slack config location (`config/default.json` → `conceptSlackConfig`)
  - Add manual test cases for notification verification

**Expected outcome:**
API clients receive clear indication that concept orders are created, and all integration tests pass with the new flow.

---

### Task Dependencies

```text
Task 1 (Validation) → Task 2 (ConceptService) → Task 3 (Controller/Tests)
```

**Sequence:**

1. Complete Task 1 first - validation must be relaxed before concept orders can accept invalid data
2. Complete Task 2 second - concept service integration depends on relaxed validation output
3. Complete Task 3 last - controller and tests depend on both previous tasks being complete

**Estimated Effort:**

- Task 1: ~4-6 hours (validation refactoring + tests)
- Task 2: ~6-8 hours (service refactoring + concept mapping + tests)
- Task 3: ~3-4 hours (controller updates + integration tests + manual verification)
