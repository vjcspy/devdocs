# 📋 [260121] - Implement Hardware Type Parsing from ONS Survey

## References

- Global standard: `devdocs/tinybots/OVERVIEW.md`
- Repo-specific standard: `devdocs/tinybots/wonkers-nedap/OVERVIEW.md`
- Related repos: `wonkers-taas-orders`, `wonkers-db`, `tiny-internal-services`

## User Requirements

> From stakeholder:
> 
> **Relevance:**
> People can order either a voice assistant or a robot. This question needs to be added to the forms where people order Tessa and then parsed by us.
> 
> **Contact for forms:** Evan has created new ONS ECD form question
> 
> **Tasks:**
> 1. Get the parameter `hardwareType` - values will be provided by Evan later
> 2. Implement parsing the new question in `wonkers-nedap`
> 3. Update concept orders to accept new field (db changes, concept library, wonkers-taas-orders)
> 4. Map concept order field to taas-order field for type (currently default ROBOT)

---

## 🔍 Pre-Implementation Analysis

### Existing Infrastructure (Already Available ✅)

| Component | Status | Details |
|-----------|--------|---------|
| **Database** | ✅ Ready | `hardware_type` table exists with values: `1=ROBOT`, `2=VOICE_ASSISTANT` |
| **DB Column** | ✅ Ready | `taas_concept_order.hardware_type_id` column exists (migration V52_5), DEFAULT=1 |
| **wonkers-taas-orders V6 API** | ✅ Ready | `POST /internal/v6/taas-orders/concepts/orders` accepts `hardwareType` field |
| **tiny-internal-services** | ✅ Ready | `HardwareTypeInputDataType` class handles string→id transformation |

### Required Changes (This Task)

| Component | Status | Details |
|-----------|--------|---------|
| **wonkers-nedap** | 🚧 TODO | Parse `hardwareType` from survey and call V6 endpoint |

---

## ⚠️ Assumptions & Questions for Stakeholder Verification

> **IMPORTANT:** The following assumptions need to be confirmed before implementation. Update this section with confirmed answers.

### Assumption 1: Survey Question Key

| Item | Details |
|------|---------|
| **Question** | What is the exact key/identifier for the hardware type question in the ONS form? |
| **My Assumption** | The key will be `hardwareType` (matching the DTO field name) |
| **Expected Format** | e.g., `hardwareType`, `hwType`, `productType` |
| **Status** | ⬜ PENDING CONFIRMATION |
| **Stakeholder Response** | _[To be filled by Evan]_ |

---

### Assumption 2: Hardware Type Values in Form

| Item | Details |
|------|---------|
| **Question** | What values will the form send for hardware type? |
| **Database Values** | `ROBOT` (id=1), `VOICE_ASSISTANT` (id=2) |
| **My Assumption** | Form will send exact strings: `ROBOT` or `VOICE_ASSISTANT` |
| **Alternative Options** | Could be lowercase (`robot`/`voice_assistant`) or Dutch labels |
| **Status** | ⬜ PENDING CONFIRMATION |
| **Stakeholder Response** | _[To be filled by Evan]_ |

---

### Assumption 3: Default Value Behavior

| Item | Details |
|------|---------|
| **Question** | If hardware type question is not answered or missing, what should the default be? |
| **Option A** | Default to `ROBOT` (backward compatible) |
| **Option B** | Leave as `null` (require manual selection in backoffice) |
| **My Assumption** | Default to `ROBOT` for backward compatibility |
| **Status** | ⬜ PENDING CONFIRMATION |
| **Stakeholder Response** | _[To be filled]_ |

---

### Assumption 4: Scope Confirmation

| Item | Details |
|------|---------|
| **Question** | Confirm that only `wonkers-nedap` needs changes? |
| **My Analysis** | DB and `wonkers-taas-orders` V6 API are already prepared |
| **Status** | ⬜ PENDING CONFIRMATION |
| **Stakeholder Response** | _[To be filled]_ |

---

## 🎯 Objective

Implement parsing of the new `hardwareType` survey question in `wonkers-nedap` and send it to `wonkers-taas-orders` V6 API, enabling the system to distinguish between ROBOT and VOICE_ASSISTANT orders.

### ⚠️ Key Considerations

1. **Backward Compatibility**: Existing orders without `hardwareType` should default to `ROBOT`
2. **V6 Migration**: Switch from V1 to V6 endpoint for concept orders
3. **Value Mapping**: May need to map form values to exact `HardwareTypeInputDataType` values

---

## 🔄 Implementation Plan

### Phase 1: Analysis & Preparation

- [ ] Confirm survey question key from Evan (Assumption 1)
- [ ] Confirm form values from Evan (Assumption 2)
- [ ] Confirm default behavior (Assumption 3)
- [ ] Confirm scope (Assumption 4)
- [ ] Review existing `ConceptOrderMapper` implementation
  - **File**: `wonkers-nedap/src/mappers/ConceptOrderMapper.ts`
- [ ] Review existing `WonkersTaasOrderService` implementation
  - **File**: `wonkers-nedap/src/service/WonkersTaasOrderService.ts`

---

### Phase 2: Implementation Structure

```
wonkers-nedap/src/
├── mappers/
│   └── ConceptOrderMapper.ts        # 🔄 UPDATE - Add hardwareType parsing
├── model/
│   └── ConceptOrderDto.ts           # 🔄 UPDATE - Add hardwareType field
├── service/
│   └── WonkersTaasOrderService.ts   # 🔄 UPDATE - Call V6 endpoint
```

---

### Phase 3: Detailed Implementation Steps

#### Task 3.1: Update `ConceptOrderDto` to Include `hardwareType`

**File:** `wonkers-nedap/src/model/ConceptOrderDto.ts`

**Current:** No `hardwareType` field

**Proposed Change:**
```typescript
// Add new field
@IsString()
@IsOptional()
@IsIn(['ROBOT', 'VOICE_ASSISTANT'])
hardwareType?: 'ROBOT' | 'VOICE_ASSISTANT' | null
```

**Status:** ⬜ TODO

---

#### Task 3.2: Update `ConceptOrderMapper.map()` to Parse `hardwareType`

**File:** `wonkers-nedap/src/mappers/ConceptOrderMapper.ts`

**Current:** Does not parse `hardwareType`

**Proposed Change:**
```typescript
// Add after existing field mappings (around line 125)
dto.hardwareType = this.getHardwareType(resultProperties, 'hardwareType') // Key TBD from Evan
```

**New Method:**
```typescript
static getHardwareType(
  answeredQuestions: SurveyAnsweredQuestion[],
  key: string
): 'ROBOT' | 'VOICE_ASSISTANT' {
  const data = answeredQuestions.filter((question) =>
    question.additionalInfo?.toLowerCase()?.includes(key.toLowerCase())
  )
  
  if (!data || data.length === 0 || data.length > 1) {
    return 'ROBOT' // Default value (TBD from stakeholder)
  }
  
  const surveyValue = data[0].answer?.text?.toUpperCase()
  
  // Value mapping logic (TBD based on form values from Evan)
  if (surveyValue?.includes('VOICE') || surveyValue?.includes('ASSISTANT')) {
    return 'VOICE_ASSISTANT'
  }
  
  return 'ROBOT'
}
```

**Status:** ⬜ TODO (Blocked by Assumption 1 & 2)

---

#### Task 3.3: Update `WonkersTaasOrderService` to Call V6 Endpoint

**File:** `wonkers-nedap/src/service/WonkersTaasOrderService.ts`

**Current:**
```typescript
public async addConceptOrder (orderDto: ConceptOrderDto): Promise<ConceptOrderDto> {
  const url = `${this.wonkersTaasOrderAddress}/internal/v1/taas-orders/concepts/orders`
  // ...
}
```

**Proposed Change:**
```typescript
public async addConceptOrder (orderDto: ConceptOrderDto): Promise<ConceptOrderDto> {
  // Change V1 → V6
  const url = `${this.wonkersTaasOrderAddress}/internal/v6/taas-orders/concepts/orders`
  // ...
}
```

**Status:** ⬜ TODO

---

### Phase 4: Update Tests

**Files:**
- `wonkers-nedap/test/mappers/ConceptOrderMapperTest.ts` (if exists)
- `wonkers-nedap/test/service/WonkersTaasOrderServiceTest.ts` (if exists)

**Test Cases to Add:**

1. **Parse `hardwareType` = ROBOT**
   - Survey has `hardwareType` question with value indicating robot
   - Expected: `dto.hardwareType = 'ROBOT'`

2. **Parse `hardwareType` = VOICE_ASSISTANT**
   - Survey has `hardwareType` question with value indicating voice assistant
   - Expected: `dto.hardwareType = 'VOICE_ASSISTANT'`

3. **Default when missing**
   - Survey does NOT have `hardwareType` question
   - Expected: `dto.hardwareType = 'ROBOT'` (default)

4. **V6 Endpoint called**
   - Verify `WonkersTaasOrderService` calls `/internal/v6/...` instead of `/internal/v1/...`

**Status:** ⬜ TODO

---

### Phase 5: Integration Testing

**Command:** `just -f devtools/tinybots/local/Justfile test-wonkers-nedap`

- [ ] All existing tests pass
- [ ] New tests for `hardwareType` parsing pass
- [ ] Manual test with sample survey data (if available)

**Status:** ⬜ TODO

---

## 📊 Task Breakdown Summary

| Task | File | Description | Depends On | Status |
|------|------|-------------|------------|--------|
| 1.0 | - | Confirm assumptions with stakeholder | - | ⬜ BLOCKED |
| 3.1 | ConceptOrderDto.ts | Add `hardwareType` field | - | ⬜ TODO |
| 3.2 | ConceptOrderMapper.ts | Add `getHardwareType()` method | Assumption 1, 2 | ⬜ BLOCKED |
| 3.3 | WonkersTaasOrderService.ts | Change to V6 endpoint | - | ⬜ TODO |
| 4.0 | Tests | Add/update test cases | 3.1, 3.2, 3.3 | ⬜ TODO |
| 5.0 | - | Integration testing | 4.0 | ⬜ TODO |

---

## 📝 Verification Checklist

After implementation, verify:

- [ ] `hardwareType` is parsed from ONS survey correctly
- [ ] Default value is applied when question is missing
- [ ] V6 endpoint is called (not V1)
- [ ] Concept order is created with correct `hardware_type_id` in database
- [ ] All existing tests pass
- [ ] New tests cover hardware type scenarios
- [ ] Backward compatibility maintained (existing flows still work)

---

## 📅 Timeline & Notes

| Date | Update |
|------|--------|
| 2026-01-21 | Plan created, awaiting stakeholder confirmation on assumptions |
| _TBD_ | Assumptions confirmed by Evan |
| _TBD_ | Implementation started |
| _TBD_ | Implementation completed |
| _TBD_ | PR submitted |

---

## 🚧 Blockers

1. **Survey Question Key** - Need exact key from Evan (Assumption 1)
2. **Form Values** - Need exact values from Evan (Assumption 2)
3. **Default Behavior Confirmation** - Need confirmation (Assumption 3)

---

## 📎 Appendix: Reference Code

### Current `ConceptOrderMapper.map()` (relevant section)

```112:127:wonkers-nedap/src/mappers/ConceptOrderMapper.ts
    dto.tessaExpertNeeded = this.getTessaExpertNeeded(
      resultProperties,
      'tessaExpertNeeded'
    )
    dto.teamId = setStringProperty(resultProperties, 'teamId', 256)

    if (!dto.teamId && result.employeeObjectId > 0) {
      dto.addLazyPatch<OnsNedapApi, 'getTeamsByEmployeeId'>(
        getTeamsByEmployeeAlias,
        (val) => {
          dto.teamId = this.mapTeamNameFromGetTeamsResult(val)
        }
      )
    }

    dto.notes = setStringProperty(resultProperties, 'notes', 1024)
    return dto
```

### Current `WonkersTaasOrderService.addConceptOrder()` 

```18:27:wonkers-nedap/src/service/WonkersTaasOrderService.ts
  public async addConceptOrder (orderDto: ConceptOrderDto): Promise<ConceptOrderDto> {
    try {
      const url = `${this.wonkersTaasOrderAddress}/internal/v1/taas-orders/concepts/orders`
      const response = await axios.post(url, orderDto)
      return response.data
    } catch (error) {
      console.error(error)
      throw new InternalServerError('Issue with sending request to tinybots')
    }
  }
```

### `ConceptOrderV6Dto` in wonkers-taas-orders

```1:12:wonkers-taas-orders/src/model/dto/ConceptOrderV6Dto.ts
import 'reflect-metadata'
import { Expose } from 'class-transformer'
import { IsDefined } from 'class-validator'
import { ConceptOrderDto } from './ConceptOrderDto'
import { AsHardwareTypeInputDataType, HardwareTypeInputDataType } from 'tiny-internal-services'

export class ConceptOrderV6Dto extends ConceptOrderDto {
  @Expose()
  @IsDefined()
  @AsHardwareTypeInputDataType
  hardwareType: HardwareTypeInputDataType
}
```
