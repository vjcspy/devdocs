# 📋 [260121] - Fix Concept Return Mapping Issues for Ecare Puur

## References

- Global standard: `devdocs/tinybots/OVERVIEW.md`
- Repo-specific standard: `devdocs/tinybots/wonkers-ecd/OVERVIEW.md`
- Previous implementation: `devdocs/tinybots/wonkers-ecd/251126-PROD-736-implement-concept-order.md`
- Related repos: `wonkers-taas-orders`, `tb-concept-taas-orders`

## User Requirements

Following the production release of concept orders and returns (PROD-736), there are issues with parsing concept returns from Ecare Puur that lead to manual actions:

### Issue 1: Order Association Not Working
> When accepting the concept return it should automatically find the order associated with the return. From my head this uses a combination of the relation and the clientId. This is however not working from concept returns from wonkers-ecd.

### Issue 2: Pickup Address Not Mapped
> Pickup address not mapped correctly in the return. It does exist in the application.
> 
> Parameters in form:
> - recipient (naam van de bewoner/organisatie van het ophaaladres)
> - street (straatnaam)
> - homeNumberExtension (huisnummer + toevoeging)
> - zipcode (postcode)
> - city (plaatsnaam)

### Issue 3: Return Reason Possibly Not Available
> I got some sounds from the Tessa experts that return reason is not available - check if it's correctly mapped. Did not get confirmation from Rosa Veenbaas on this.

**Context:** Concept returns are created from "Afmeldbericht" in the same flow.

---

## 🔍 Root Cause Analysis

### Issue 1: Order Association - Root Cause CONFIRMED ✅

**Problem Chain:**

1. **`EcarePuurMappingService.mapUnsubscribe()` does NOT pass `clientId` or `clientUuid`:**

```typescript
// Current implementation (BROKEN)
const returnDto: ConceptReturnDto = {
  returner,
  returnReason,
  notes,
  integrationId: integration.id,
  relationId,
  tessaCode: unsubscribeFields.tessaCode,
  // ❌ MISSING: clientId, clientUuid
}
```

2. **`wonkers-taas-orders` uses `clientId` for order matching:**

```typescript
// ConceptOrderService.addReturnConcept()
const orders = await this.adminTaasOrderRepository.findMatchingTaasOrders(
  conceptReturnDto.clientId,  // Always NULL!
  conceptReturnDto.tessaCode
)
```

3. **SQL Query expects `clientId`:**

```sql
WHERE tor.client_id=? AND drt.serial LIKE ?
```

**Result:** Query runs with `clientId=NULL` → No orders found.

---

### Issue 2: Pickup Address - Root Cause CONFIRMED ✅

**Problem Chain:**

1. **`mapUnsubscribePuurNotification()` DOES extract retrieval address:**

```typescript
let retrievalAddress: RetrievalAddress = { ... }
if (this.retrievalServiceRelationIds.includes(relationId)) {
  retrievalAddress = this.getRetrievalServiceAddress(notification)
}
const additionalFields: UnsubscribeFields = {
  ...retrievalAddress,  // ✅ Address IS extracted
  // ...
}
```

2. **BUT `mapUnsubscribe()` does NOT pass it to `ConceptReturnDto`:**

```typescript
const returnDto: ConceptReturnDto = {
  returner,
  returnReason,
  notes,
  integrationId,
  relationId,
  tessaCode,
  // ❌ MISSING: pickupAddress
}
```

**Result:** Pickup address is extracted but never sent to the API.

---

### Issue 3: Return Reason - NEEDS VERIFICATION ⚠️

**Current Implementation:**

```typescript
private RETURN_REASONS: string[] = [
  'returnReason',
  'returnReasonMain',
  'returnReasonA',
  'returnReasonB',
  'returnReasonC',
  'returnReasonD',
  'returnReasonE',
  'returnReasonF',
]

private getReturnReason(notification: PuurNotificationDto): string[] {
  const returnReasons: string[] = []
  for (const reason of this.RETURN_REASONS) {
    const param = notification.AdditionalFields.find((field) => field.Key === reason)
    if (param && param.Value) {
      returnReasons.push(param.Value)
    }
  }
  return returnReasons
}
```

**Mapping to DTO:**
```typescript
returnReason: returnReasons.length > 0 ? returnReasons.join(', ') : null
```

**Potential Issues:**
- Field keys may not match what the form sends
- May need to use `Display` instead of `Value`
- May need different field keys entirely

**Status:** Awaiting sample Afmeldbericht payload to verify.

---

## ⚠️ Assumptions & Questions for Stakeholder Verification

> **IMPORTANT:** The following assumptions need to be confirmed before implementation. Update this section with confirmed answers.

### Assumption 1: Client Identification for Order Matching

| Item | Details |
|------|---------|
| **Question** | How should the system identify the client when matching a concept return to an existing order? |
| **My Assumption** | Use Ecare API enrichment to get client info, then derive `clientId` using either `ecareNumber` or `name` based on relation config (same as subscribe flow). Use `notification.PatientId` as `clientUuid`. |
| **Rationale** | Matches the subscribe flow pattern where client is enriched before creating concept order. |
| **Status** | ⬜ PENDING CONFIRMATION |
| **Stakeholder Response** | _[To be filled]_ |

---

### Assumption 2: Pickup Address Scope

| Item | Details |
|------|---------|
| **Question** | Should pickup address be captured for ALL organizations, or only for organizations using the retrieval service (`retrievalServiceRelationIds`)? |
| **My Assumption** | Capture pickup address for ALL returns where address fields are provided in the form, regardless of `retrievalServiceRelationIds`. The config should only affect validation strictness, not data capture. |
| **Rationale** | Concept orders follow "accept all input" pattern. Back-office can review incomplete data. |
| **Status** | ⬜ PENDING CONFIRMATION |
| **Stakeholder Response** | _[To be filled]_ |

---

### Assumption 3: Return Reason Field Keys

| Item | Details |
|------|---------|
| **Question** | What are the exact field keys used in the Afmeldbericht form for return reason? |
| **My Assumption** | Form uses one of: `returnReason`, `returnReasonMain`, `returnReasonA-F`. Current join logic should work. |
| **Expected Keys** | `returnReason`, `returnReasonMain`, `returnReasonA`, `returnReasonB`, `returnReasonC`, `returnReasonD`, `returnReasonE`, `returnReasonF` |
| **Status** | ⬜ PENDING - Need sample payload |
| **Stakeholder Response** | _[To be filled]_ |

---

### Assumption 4: Afmeldbericht Creates Concept Returns

| Item | Details |
|------|---------|
| **Question** | Confirm that Afmeldbericht notifications should create concept returns (not production returns)? |
| **My Assumption** | YES - per PROD-736 implementation, all Ecare Puur notifications go through concept flow. |
| **Status** | ⬜ PENDING CONFIRMATION |
| **Stakeholder Response** | _[To be filled]_ |

---

### Assumption 5: Client Enrichment in Unsubscribe Flow

| Item | Details |
|------|---------|
| **Question** | Should the unsubscribe flow fetch client info from Ecare API (like subscribe does)? |
| **My Assumption** | YES - and it's already implemented, but the enriched client object is only used for `mapNotificationToForm()`, not for `mapUnsubscribe()`. This is the bug. |
| **Status** | ⬜ PENDING CONFIRMATION |
| **Stakeholder Response** | _[To be filled]_ |

---

## 📦 Sample Payload (Afmeldbericht)

> **TO BE PROVIDED:** Please paste a sample Afmeldbericht notification payload here for field key verification.

```json
// TODO: Paste sample Afmeldbericht payload here
{
  "Type": "Afmeldbericht",
  "PatientId": "...",
  "SenderId": "...",
  "AdditionalFields": [
    // ... need actual fields
  ]
}
```

---

## 🎯 Implementation Plan

### Phase 1: Fix `mapUnsubscribe()` to Include Missing Fields

**File:** `wonkers-ecd/src/service/ecare/EcarePuurMappingService.ts`

#### Task 1.1: Add `clientId` and `clientUuid` Parameters

**Current Signature:**
```typescript
public mapUnsubscribe(
  notification: PuurNotificationDto,
  unsubscribeFields: UnsubscribeFields,
  integration: IntegrationUser,
): ConceptReturnDto
```

**New Signature:**
```typescript
public mapUnsubscribe(
  notification: PuurNotificationDto,
  unsubscribeFields: UnsubscribeFields,
  integration: IntegrationUser,
  client: Client,  // NEW: For clientId derivation
): ConceptReturnDto
```

#### Task 1.2: Add `clientId`, `clientUuid` to `ConceptReturnDto`

```typescript
const returnDto: ConceptReturnDto = {
  returner,
  returnReason: `${unsubscribeFields.returnReason ? unsubscribeFields.returnReason : ''}`,
  notes: `${notes} Items retour: ${unsubscribeFields.returnItems ?? '-'}. Tessa Code opgegeven: ${unsubscribeFields.tessaCode ?? '-'}.`,
  integrationId: integration.id,
  relationId,
  tessaCode: unsubscribeFields.tessaCode,
  // NEW FIELDS:
  clientId: this.getClientId(client, relationId),
  clientUuid: notification.PatientId,
}
```

#### Task 1.3: Add `pickupAddress` to `ConceptReturnDto`

```typescript
// Build pickup address from unsubscribeFields (RetrievalAddress)
const pickupAddress: OptionalAddressV2 | null = unsubscribeFields.zipcode ? {
  recipient: unsubscribeFields.recipient,
  street: unsubscribeFields.street,
  homeNumber: unsubscribeFields.homeNumber,
  homeNumberExtension: unsubscribeFields.homeNumberExtension,
  city: unsubscribeFields.city,
  zipcode: unsubscribeFields.zipcode,
  locationDescription: unsubscribeFields.locationDescription,
  country: 'Netherlands',
} : null

const returnDto: ConceptReturnDto = {
  // ... existing fields
  pickupAddress,  // NEW
}
```

---

### Phase 2: Update `EcarePuurService.unsubscribe()` to Pass Client

**File:** `wonkers-ecd/src/service/ecare/EcarePuurService.ts`

**Current Call:**
```typescript
const returnDto = this.ecarePuurMappingService.mapUnsubscribe(
  notification,
  unsubscribeFields,
  integrationUser,
)
```

**Updated Call:**
```typescript
const returnDto = this.ecarePuurMappingService.mapUnsubscribe(
  notification,
  unsubscribeFields,
  integrationUser,
  client,  // Pass enriched client
)
```

---

### Phase 3: Relax Pickup Address Validation (If Assumption 2 Confirmed)

**File:** `wonkers-ecd/src/service/ecare/EcarePuurMappingService.ts`

**Current:** Only retrieval service organizations get address extraction with strict validation.

**Proposed:** Extract address for ALL organizations, but make it optional (no throwing).

```typescript
public async mapUnsubscribePuurNotification(
  notification: PuurNotificationDto,
  relationId: number,
): Promise<UnsubscribeFields> {
  // ...
  
  let retrievalAddress: RetrievalAddress = {
    homeNumber: null,
    homeNumberExtension: null,
    street: null,
    city: null,
    locationDescription: null,
    zipcode: null,
    recipient: null,
  }

  // TRY to get address for ALL organizations (best-effort)
  try {
    if (this.retrievalServiceRelationIds.includes(relationId)) {
      // Strict extraction for retrieval service orgs
      retrievalAddress = this.getRetrievalServiceAddress(notification)
    } else {
      // Optional extraction for other orgs
      retrievalAddress = this.getOptionalRetrievalAddress(notification)
    }
  } catch (error) {
    console.warn('Retrieval address extraction failed, using empty address', error)
  }
  
  // ...
}

// NEW METHOD: Optional address extraction (no throwing)
private getOptionalRetrievalAddress(notification: PuurNotificationDto): RetrievalAddress {
  let homeNumber = null
  let extension = null
  
  const homeNumberExtension = EcarePuurMappingTools.getOptionalField(notification, 'homeNumberExtension')
  if (homeNumberExtension) {
    try {
      const details = this.getHomeNumberAndExtension(homeNumberExtension)
      homeNumber = parseInt(details.homeNumber)
      extension = details.extension
    } catch (error) {
      console.warn('Invalid homeNumberExtension in retrieval address', error)
    }
  }
  
  return {
    homeNumber,
    homeNumberExtension: extension,
    street: EcarePuurMappingTools.getOptionalField(notification, 'street'),
    city: EcarePuurMappingTools.getOptionalField(notification, 'city'),
    locationDescription: EcarePuurMappingTools.getOptionalField(notification, 'locationDescription'),
    zipcode: EcarePuurMappingTools.getOptionalField(notification, 'zipcode'),
    recipient: EcarePuurMappingTools.getOptionalField(notification, 'recipient'),
  }
}
```

---

### Phase 4: Verify Return Reason Mapping (After Payload Received)

**File:** `wonkers-ecd/src/service/ecare/EcarePuurMappingService.ts`

**Action:** After receiving sample Afmeldbericht payload:
1. Check if field keys match `RETURN_REASONS` array
2. Check if values are in `Value` or `Display` property
3. Update mapping if needed

**Placeholder for changes:**
```typescript
// TODO: Update after payload verification
private RETURN_REASONS: string[] = [
  // Verify these keys match form
  'returnReason',
  'returnReasonMain',
  'returnReasonA',
  // ...
]

private getReturnReason(notification: PuurNotificationDto): string[] {
  const returnReasons: string[] = []
  for (const reason of this.RETURN_REASONS) {
    const param = notification.AdditionalFields.find((field) => field.Key === reason)
    if (param && param.Value) {
      // TODO: Verify if we should use param.Value or param.Display
      returnReasons.push(param.Value)
    }
  }
  return returnReasons
}
```

---

### Phase 5: Update Tests

**Files:**
- `wonkers-ecd/test/ecareService/EcarePuurMappingServiceTest.ts`
- `wonkers-ecd/test/ecareService/EcarePuurServiceTest.ts`
- `wonkers-ecd/test/controller/EcarePuurNotificationControllerIT.ts`

**Test Cases to Add/Update:**

1. **`mapUnsubscribe()` returns `clientId` and `clientUuid`**
2. **`mapUnsubscribe()` returns `pickupAddress` when fields provided**
3. **`mapUnsubscribe()` returns `pickupAddress: null` when fields missing**
4. **Unsubscribe flow passes client to mapping service**
5. **Return reason mapping with various field combinations**

---

## 📊 Task Breakdown Summary

| Task | File | Description | Depends On | Status |
|------|------|-------------|------------|--------|
| 1.1 | EcarePuurMappingService | Add `client` param to `mapUnsubscribe()` | - | ⬜ TODO |
| 1.2 | EcarePuurMappingService | Add `clientId`, `clientUuid` to DTO | 1.1 | ⬜ TODO |
| 1.3 | EcarePuurMappingService | Add `pickupAddress` to DTO | - | ⬜ TODO |
| 2.0 | EcarePuurService | Pass `client` to `mapUnsubscribe()` | 1.1 | ⬜ TODO |
| 3.0 | EcarePuurMappingService | Relax pickup address extraction | Assumption 2 | ⬜ BLOCKED |
| 4.0 | EcarePuurMappingService | Verify return reason mapping | Payload | ⬜ BLOCKED |
| 5.0 | Tests | Update/add test cases | 1-4 | ⬜ TODO |

---

## 🔄 Optional: Improve Order Matching in wonkers-taas-orders

**File:** `wonkers-taas-orders/src/repository/AdminTaasOrderRepository.ts`

**Current Query:**
```sql
WHERE tor.client_id=? AND drt.serial LIKE ?
```

**Proposed Enhancement (add `relationId`):**
```sql
WHERE tor.client_id=? AND tor.relation_id=? AND drt.serial LIKE ?
```

**Rationale:** Adding `relationId` to the query increases accuracy and prevents false matches across different organizations with the same `clientId`.

**Priority:** P2 (optional improvement, main fix is in wonkers-ecd)

---

## 📝 Verification Checklist

After implementation, verify:

- [ ] Concept return has `clientId` populated
- [ ] Concept return has `clientUuid` populated  
- [ ] Concept return has `pickupAddress` populated (when form has address)
- [ ] Concept return has `returnReason` populated (when form has reason)
- [ ] Order is automatically linked when accepting concept return
- [ ] Back-office shows pickup address correctly
- [ ] All existing tests pass
- [ ] New tests cover the fixed scenarios

---

## 📅 Timeline & Notes

| Date | Update |
|------|--------|
| 2026-01-21 | Plan created, awaiting stakeholder confirmation on assumptions |
| _TBD_ | Sample payload received |
| _TBD_ | Assumptions confirmed |
| _TBD_ | Implementation started |
| _TBD_ | Implementation completed |
| _TBD_ | PR submitted |

---

## 🚧 Blockers

1. **Sample Afmeldbericht Payload** - Needed to verify return reason field mapping
2. **Assumption Confirmations** - Needed before Phase 3 (pickup address scope)
