# Review PROD-736: Concept Order Implementation for E-care Puur

## Git Comparison Summary

**Branches compared:**
- Base: `task/PROD-736-TASK1-fix-testing`
- Implementation: `task/PROD-736-TASK2-implementation`

**File changes:**
- 8 files changed: 561 insertions(+), 850 deletions(-)
- Net reduction: 289 lines (primarily from test simplification)

**Key commits:**
1. Replaced legacy order tracking with concept order integration
2. Removed dependencies: `ClientIdRepository`, `OrderStatusService`, `WonkersTaasOrderService`
3. Introduced `ConceptService` for order/return handling
4. Enhanced data enrichment with relaxed validation
5. Added comprehensive tests for EcarePuurService

---

## Câu hỏi 1: Old flow xử lý từng step để save vào order như thế nào?

### 1.1. Luồng Subscribe (Đặt order) - OLD FLOW

**Bước 1: STRICT Validation**
```typescript
// File: EcarePuurMappingService.ts (OLD)
// Yêu cầu TẤT CẢ fields bắt buộc dựa trên tessaExpertNeeded
if (tessaExpertNeeded === 'yes') {
  address = {
    homeNumber, extension,
    recipient: getOptionalField(),  // Optional nếu tessaExpertNeeded = 'yes'
    street: getOptionalField(),
    city: getOptionalField(),
    zipcode: getOptionalField()
  }
} else {
  // Yêu cầu ĐẦY ĐỦ address nếu tessaExpertNeeded = 'no'
  address = {
    homeNumber: getRequiredField(),      // THROW nếu thiếu
    extension: getRequiredField(),
    recipient: getRequiredField(),
    street: getRequiredField(),
    city: getRequiredField(),
    zipcode: getRequiredField()
  }
}

// Validate phone, email CHẶT CHẼ
phoneNumber = phoneNumberService.checkPhoneNumber(phoneNumberForm) // THROW nếu sai format
await validateOrReject(subscribeFields).catch(errors => {
  // XÂY DỰNG message lỗi chi tiết và THROW BadRequestError
  throw new BadRequestError(message)
})
```

**Bước 2: Check Duplicate Orders**
```typescript
// File: EcarePuurService.ts (OLD)
const orders = await this.clientIdRepository.getOrderIds(notification.PatientId, integrationUser.id)
if (orders.length > 0) {
  // Kiểm tra xem client đã có order chưa
  await this.orderStatusService.checkHasOrder(orders, integrationUser.organisations[0].id)
  // THROW error nếu đã có order tồn tại
}
```

**Bước 3: Enrich Data - REQUIRED**
```typescript
// API calls KHÔNG ĐƯỢC PHÉP fail
try {
  const eCareHeaders = await this.ecarePuurApiService.getHeaders(integrationUser.id)
  if (subscribeFields.requesterEmail == null) {
    const requester = await this.ecarePuurApiService.getRequester(...)
    if (requester.email == null) {
      // THROW error nếu không có email
      throw new BadRequestError('Er is geen contact informatie bekend...')
    }
    subscribeFields.requesterEmail = requester.email
  }
} catch (error) {
  // FALLBACK: Set default data nhưng LOG là ERROR
  console.error('found an error while retrieving requester...')
  subscribeFields.requesterEmail = 'operations@tinybots.nl'
  subscribeFields.requesterFirstname = notification.SenderId
  subscribeFields.requesterLastname = 'ERROR RETRIEVING EMPLOYEE'
}

const client = await this.ecarePuurApiService.getClient(...) // Bắt buộc success
const careteam = await this.ecarePuurTeamService.getTeamsString(...) // Bắt buộc success
```

**Bước 4: Map to OrderV2Dto (Production DTO)**
```typescript
// File: EcarePuurMappingService.ts (OLD)
const orderDto = this.ecarePuurMappingService.mapSubscribe(
  notification, subscribeFields, client, careteam, integrationUser
)
// Returns: OrderV2Dto - structured DTO cho production order
```

**Bước 5: Place Order in Production System**
```typescript
// File: EcarePuurService.ts (OLD)
const orderId = await this.wonkersTaasOrderService.placeOrder(orderDto)
// POST /internal/v4/taas-orders
// Response: { id: 12345 }
```

**Bước 6: Save Client Mapping**
```typescript
await this.clientIdRepository.addClient(orderId, notification.PatientId, integrationUser.id)
// Lưu mapping: orderId <-> PatientId vào database để tracking
```

**Kết quả:**
- Order được tạo trong production system
- Order có trạng thái (NEW, PROCESSING, DELIVERED, ...)
- Có order tracking qua ClientIdRepository
- Có validation CHẶT CHẼ - từ chối input không hợp lệ

---

### 1.2. Luồng Unsubscribe (Trả order) - OLD FLOW

**Bước 1: Validate & Enrich Returner**
```typescript
// File: EcarePuurService.ts (OLD)
try {
  if (unsubscribeFields.returnerEmail == null) {
    if (notification.SenderId == null) {
      throw new BadRequestError('Er is geen contact informatie bekend...')
    }
    const returner = await this.ecarePuurApiService.getRequester(...)
    if (returner.email == null) {
      throw new BadRequestError('Er is geen contact informatie bekend...')
    }
    unsubscribeFields.returnerEmail = returner.email
  }
} catch (error) {
  console.error('found an error while retrieving returner...')
  // Fallback with defaults
}
```

**Bước 2: LOOKUP Existing Order**
```typescript
// File: EcarePuurService.ts (OLD)
const orders = await this.clientIdRepository.getOrderIds(notification.PatientId, integrationUser.id)
const order = await this.orderStatusService.getOrderId(orders, integrationUser.organisations[0].id)
// GET order từ database dựa trên PatientId
```

**Bước 3: Check Order Status & Decide Action**
```typescript
// File: EcarePuurService.ts (OLD)
if (this.orderStatusService.canBeDeleted(order)) {
  // Nếu order vẫn ở trạng thái NEW/PENDING → DELETE
  await this.wonkersTaasOrderService.deleteOrder(order.id, relationId)
  return order.id
} else {
  // Nếu order đã PROCESSING/DELIVERED → CREATE RETURN
  const returnDto = this.ecarePuurMappingService.mapUnsubscribe(...)
  return this.wonkersTaasOrderService.returnOrder(returnDto, order.id)
  // POST /internal/v4/taas-orders/{orderId}/return
}
```

**Kết quả:**
- Phải TÌM order hiện tại
- Phải KIỂM TRA trạng thái order
- Quyết định DELETE hoặc CREATE RETURN dựa trên status
- Return được link với orderId cụ thể

---

## Câu hỏi 2: New flow xử lý từng bước như thế nào?

### 2.1. Luồng Subscribe (Đặt order) - NEW FLOW

**Bước 1: RELAXED Validation (Accept All Input)**
```typescript
// File: EcarePuurMappingService.ts (NEW)
// KHÔNG còn phân biệt tessaExpertNeeded
// TẤT CẢ fields đều optional
address = {
  homeNumber,  // getOptionalField() - không throw
  homeNumberExtension: extension,
  recipient: getOptionalField(),  // Không throw nếu thiếu
  street: getOptionalField(),
  city: getOptionalField(),
  zipcode: getOptionalField(),
  locationDescription: getOptionalField(),
  addressType: getOptionalField()
}

// Wrap validation trong try-catch, chỉ LOG warning, KHÔNG THROW
try {
  await validateOrReject(subscribeFields)
} catch (errors) {
  console.warn('Validation warnings for concept order (not blocking):', errors)
  // KHÔNG throw BadRequestError - accept tất cả input
}
```

**Bước 2: REMOVED - No Duplicate Check**
```typescript
// File: EcarePuurService.ts (NEW)
// REMOVED: Order lookup và duplicate checking
// Concept orders ACCEPT duplicates - không block
```

**Bước 3: Enrich Data - BEST EFFORT**
```typescript
// File: EcarePuurService.ts (NEW)
let client: Client
try {
  const eCareHeaders = await this.ecarePuurApiService.getHeaders(integrationUser.id)
  client = await this.ecarePuurApiService.getClient(eCareHeaders, notification.PatientId, integrationUser.id)
  
  // Chỉ enrich requester nếu CHƯA có trong form
  if (subscribeFields.requesterEmail == null) {
    const requester = await this.ecarePuurApiService.getRequester(...)
    subscribeFields.requesterEmail = requester.email
    // KHÔNG check requester.email == null - accept bất kỳ giá trị nào
  }
} catch (error) {
  console.warn('Ecare API enrichment failed, using defaults', error)
  // Fallback: Sử dụng minimal client data
  client = {
    ecareNumber: 'UNKNOWN',
    clientUuid: notification.PatientId,
    name: 'UNKNOWN',
    system: 'ECARE PUUR'
  }
  // Fallback: Set default requester nếu API fail
  if (subscribeFields.requesterEmail == null) {
    subscribeFields.requesterEmail = 'operations@tinybots.nl'
    subscribeFields.requesterFirstname = notification.SenderId ?? 'UNKNOWN'
    subscribeFields.requesterLastname = 'ERROR RETRIEVING EMPLOYEE'
    subscribeFields.requesterPhoneNumber = '0612345678'
  }
}

// Careteam lookup - cũng best effort
let careteam = 'UNKNOWN'
try {
  const eCareHeaders = await this.ecarePuurApiService.getHeaders(integrationUser.id)
  careteam = await this.ecarePuurTeamService.getTeamsString(...)
  if (!careteam || careteam.length === 0) {
    careteam = 'UNKNOWN'
  }
} catch (error) {
  console.warn('Careteam lookup failed, using UNKNOWN', error)
}
```

**Bước 4: Build DUAL Mapping**
```typescript
// File: EcarePuurService.ts (NEW)
// 4a. Build ConceptOrderDto (structured data)
const orderDto = this.ecarePuurMappingService.mapSubscribe(
  notification, subscribeFields, client, careteam, integrationUser
)
// Returns: ConceptOrderDto - structured DTO cho processing engine

// 4b. Build ConceptForm (raw data - preserve ALL fields)
const form = this.ecarePuurMappingService.mapNotificationToForm(
  notification, client, integrationUser
)
// Returns: ConceptForm với FormAnswer[] - preserve TẤT CẢ AdditionalFields
```

**Bước 5: Create Concept Order (Pass BOTH Form + DTO)**
```typescript
// File: EcarePuurService.ts (NEW)
const conceptOrderId = await this.conceptService.createConceptOrder(form, orderDto)
// POST /internal/v1/taas-orders/concepts/orders
// Body: { form: ConceptForm, orderDto: ConceptOrderDto }
// Response: { id: "concept-12345" }
```

**Bước 6: REMOVED - No Client Mapping**
```typescript
// KHÔNG lưu client mapping vào database
// Concept orders không cần tracking qua ClientIdRepository
```

**Kết quả:**
- Concept order được lưu với FULL raw data (form) + structured data (orderDto)
- KHÔNG có trạng thái (status) - chỉ là "concept"
- KHÔNG có order tracking
- ACCEPT tất cả input - không validation chặt chẽ

---

### 2.2. Luồng Unsubscribe (Trả order) - NEW FLOW

**Bước 1: Relaxed Validation & Enrich Returner**
```typescript
// File: EcarePuurService.ts (NEW)
// 1. Map without throwing
const unsubscribeFields = await this.ecarePuurMappingService.mapUnsubscribePuurNotification(notification, relationId)

// 2. Enrich returner - best effort
if (unsubscribeFields.returnerEmail == null) {
  try {
    const eCareHeaders = await this.ecarePuurApiService.getHeaders(integrationUser.id)
    const returner = await this.ecarePuurApiService.getRequester(...)
    unsubscribeFields.returnerEmail = returner.email
    // KHÔNG check null - accept bất kỳ giá trị nào
  } catch (error) {
    console.warn('Returner lookup failed, using defaults', error)
    unsubscribeFields.returnerEmail = 'operations@tinybots.nl'
    unsubscribeFields.returnerFirstname = notification.SenderId ?? 'UNKNOWN'
    unsubscribeFields.returnerLastname = 'ERROR RETRIEVING EMPLOYEE'
    unsubscribeFields.returnerPhoneNumber = '0612345678'
  }
}
```

**Bước 2: REMOVED - No Order Lookup**
```typescript
// File: EcarePuurService.ts (NEW)
// REMOVED: Order lookup
// REMOVED: Order status checking
// REMOVED: Delete/Return decision
```

**Bước 3: Build ConceptReturnDto**
```typescript
// File: EcarePuurService.ts (NEW)
const returnDto = this.ecarePuurMappingService.mapUnsubscribe(notification, unsubscribeFields, integrationUser)
// Returns: ConceptReturnDto
```

**Bước 4: Get Client Data - Best Effort**
```typescript
// File: EcarePuurService.ts (NEW)
let client: Client
try {
  const eCareHeaders = await this.ecarePuurApiService.getHeaders(integrationUser.id)
  client = await this.ecarePuurApiService.getClient(eCareHeaders, notification.PatientId, integrationUser.id)
} catch (error) {
  console.warn('Client lookup failed, using defaults', error)
  client = {
    ecareNumber: 'UNKNOWN',
    clientUuid: notification.PatientId,
    name: 'UNKNOWN',
    system: 'ECARE PUUR'
  }
}
```

**Bước 5: Build ConceptForm (Raw Data)**
```typescript
// File: EcarePuurService.ts (NEW)
const form = this.ecarePuurMappingService.mapNotificationToForm(notification, client, integrationUser)
// Preserve ALL AdditionalFields from notification
```

**Bước 6: Create Concept Return (Pass BOTH)**
```typescript
// File: EcarePuurService.ts (NEW)
const conceptReturnId = await this.conceptService.createConceptReturn(form, returnDto)
// POST /internal/v1/taas-orders/concepts/returns
// Body: { form: ConceptForm, returnDto: ConceptReturnDto }
// Response: { id: "concept-return-12345" }
```

**Kết quả:**
- Concept return được tạo MÀ KHÔNG CẦN tìm order hiện tại
- KHÔNG kiểm tra trạng thái
- LUÔN tạo concept return (không có delete option)
- Return KHÔNG link với orderId cụ thể

---

## Câu hỏi 3: So sánh chi tiết các điểm khác nhau

### 3.1. Data và cách build khác nhau thế nào?

| Aspect | Old Flow (Production Order) | New Flow (Concept Order) |
|--------|----------------------------|--------------------------|
| **DTO Type** | `OrderV2Dto` / `ReturnDto` | `ConceptOrderDto` / `ConceptReturnDto` |
| **Raw Data Preservation** | KHÔNG lưu - chỉ lưu structured DTO | CÓ lưu - `ConceptForm` với ALL AdditionalFields |
| **Address Validation** | CHẶT CHẼ - required dựa trên `tessaExpertNeeded` | RELAXED - tất cả fields optional |
| **Phone Validation** | THROW error nếu sai format | LOG warning, accept null |
| **Email Validation** | THROW error nếu thiếu | Accept null, use defaults |
| **Client Tracking** | Lưu mapping `orderId <-> PatientId` vào DB | KHÔNG lưu mapping |
| **Order Status** | Có trạng thái (NEW, PROCESSING, DELIVERED, ...) | KHÔNG có trạng thái - chỉ "concept" |
| **Duplicate Check** | BLOCK nếu client đã có order | ACCEPT duplicates - không check |
| **Data Structure** | Single DTO (OrderV2Dto) | Dual mapping: Form (raw) + DTO (structured) |

#### External API Calls Comparison:

| API Call | Old Flow | New Flow |
|----------|---------|----------|
| **Headers API** | `/token` - REQUIRED success | `/token` - best effort, fallback on fail |
| **Client API** | `GET /clients/{id}` - REQUIRED success | `GET /clients/{id}` - best effort, use UNKNOWN on fail |
| **Requester API** | `GET /HealthProfessionals/{id}` - THROW if email null | `GET /HealthProfessionals/{id}` - accept null, use defaults |
| **Careteam API** | `GET /clients/{id}/careteam` - REQUIRED success | `GET /clients/{id}/careteam` - use UNKNOWN on fail |
| **Order Lookup** | `ClientIdRepository.getOrderIds()` - REQUIRED | REMOVED - không lookup |
| **Order Status Check** | `OrderStatusService.getOrderId()` - REQUIRED | REMOVED - không check status |
| **Place Order** | `POST /internal/v4/taas-orders` | `POST /internal/v1/taas-orders/concepts/orders` |
| **Return Order** | `POST /internal/v4/taas-orders/{id}/return` (nếu status cho phép) | `POST /internal/v1/taas-orders/concepts/returns` (LUÔN tạo) |
| **Delete Order** | `DELETE /internal/v4/taas-orders/{id}` (nếu status = NEW) | REMOVED - không delete |

**Error Handling:**
- **Old Flow**: API failures → THROW BadRequestError → 400 response
- **New Flow**: API failures → LOG warning → Use defaults → 200 response (concept created)

---

### 3.2. Test case changes trong `EcarePuurNotificationControllerIT.ts`

#### **CRITICAL: Test cases bị XÓA NHẦ VÀ CẦN PHỤC HỒI**

Sau khi review chi tiết, tôi phát hiện **5 test cases quan trọng bị xóa nhầm**. Đây là những test KHÔNG liên quan đến validation, mà test các scenarios quan trọng cho concept order flow:

**❌ Test cases bị XÓA NHẦM (CẦN RESTORE):**

1. **`should subscribe for a Taas Tessa minimal with the ecareId`**
   - **Mục đích:** Test với `integrationIdB` (different integration tenant)
   - **Tại sao quan trọng:** Multi-tenant scenario - verify concept orders work cho nhiều integrations khác nhau
   - **Status:** **CẦN PHỤC HỒI** - đây KHÔNG phải validation test

2. **`should subscribe for a Taas Tessa minimal with the ecareId and no requester`**
   - **Mục đích:** Test khi requester KHÔNG có trong form → phải enrich từ API
   - **Tại sao quan trọng:** Verify best-effort enrichment pattern - core feature của concept order flow
   - **Status:** **CẦN PHỤC HỒI** - test API enrichment logic

3. **`should register for a Taas Tessa full with senderId`**
   - **Mục đích:** Test với `senderId` và `multiCareteamResponse` (multiple careteams)
   - **Tại sao quan trọng:** Edge case khi client thuộc nhiều careteams
   - **Status:** **CẦN PHỤC HỒI** - test careteam aggregation logic

4. **`should be ok to have no email`**
   - **Mục đích:** Test khi requester API trả về `requesterResponseWrongPhoneAndEmail` (email invalid/null)
   - **Tại sao quan trọng:** **CRITICAL TEST** - verify fallback defaults khi API enrichment fails
   - **Lý do XÓA sai:** Test này GIỮ expectation `expect(200)` - ĐÚNG với concept order flow (best-effort enrichment)
   - **Status:** **CẦN PHỤC HỒI NGAY** - core test cho best-effort pattern

5. **`should fail auth for a Taas Tessa`**
   - **Mục đích:** Test khi Ecare auth fails (`authScope.post('/token').reply(401)`)
   - **Tại sao quan trọng:** Verify graceful degradation - system vẫn hoạt động khi Ecare API down
   - **Expectation:** `expect(500)` with Slack notification
   - **Status:** **CẦN PHỤC HỒI** - test error handling cho external service failures

**Tóm tắt:**
- **5 tests bị xóa nhầm** (KHÔNG phải validation tests)
- Helper functions vẫn còn: `nockSuccessIntegrationB`, `nockFailedEmail`, `nockFailIntegrationBNoContact`
- **Impact:** Missing coverage for:
  - Multi-tenant scenarios
  - API enrichment failures (best-effort pattern)
  - Multiple careteams edge case
  - Ecare auth failure handling

---

#### **Test cases đã XÓA ĐÚNG (627 lines removed):**

**1. Validation Tests (~15 test cases) - XÓA ĐÚNG**
- ❌ `should return 400 when requesterEmail is not a valid email`
- ❌ `should return 400 when requesterPhoneNumber has invalid format`
- ❌ `should return 400 when receiverEmail is not a valid email`
- ❌ `should return 400 when receiverPhoneNumber has invalid format`
- ❌ `should return 400 when address is incomplete (tessaExpertNeeded = no)`
- ❌ `should return 400 when homeNumberExtension has invalid format`
- ❌ `should return 400 when retrieval address missing for retrieval service`
- ❌ `should return 400 when non-retrieval service provides retrieval address`

**Lý do XÓA ĐÚNG:** Relaxed validation → ACCEPT all input → Validation tests không còn relevant

---

**2. Integration Duplicate Check Tests (~3 test cases) - XÓA ĐÚNG**
- ❌ `should return 400 when client already has order in relationA`
- ❌ `should return 400 when client already has order in relationB`
- ❌ `should place order successfully when order exists in different relation`

**Lý do XÓA ĐÚNG:** No order lookup → ACCEPT duplicates → Duplicate tests không còn relevant

---

**3. API Failure Tests OLD STYLE (~3 test cases) - XÓA ĐÚNG (nhưng note: test #4 ở trên cần restore)**
- ❌ `should return 400 when requester API returns no contact info` (expect 400)
- ❌ `should return 400 when returner API returns no email` (expect 400)

**Lý do XÓA:** OLD tests expect 400 errors → NEW flow uses defaults and continues

**NHƯNG:** Test `should be ok to have no email` expects 200 → CẦN RESTORE (đây là NEW style test)

---

**4. Ophaalservice Specific Tests (~3 test cases) - XÓA ĐÚNG**
- ❌ `should handle ophaalservice retrieval address correctly`
- ❌ `should reject retrieval address for non-ophaalservice relations`
- ❌ `should validate retrieval address fields`

**Lý do XÓA ĐÚNG:** Relaxed validation → Accept all address data → Ophaalservice-specific validation không còn

---

**5. Order Status & Delete Tests (~4 test cases) - XÓA ĐÚNG**
- ❌ `should delete order when status = NEW`
- ❌ `should create return when status = PROCESSING`
- ❌ `should create return when status = DELIVERED`
- ❌ `should fail when order not found for return`

**Lý do XÓA ĐÚNG:** No order lookup → Always create concept return → Order status logic không còn

---

**Tổng cộng test cases XÓA ĐÚNG: ~25 test cases** (validation + duplicate + old-style API failure + ophaalservice + order status)

---

#### **Test cases đã GIỮ LẠI (5 essential integration tests):**

✅ **Security Test**
- `should return 401 when no authorization header`

✅ **500 Error Handling**
- `should return 500 when concept order API fails`

✅ **Subscribe Minimal**
- `should create concept order with minimal notification (no requester in form)`
- Mocks: headers, client, careteam, requester APIs
- Verifies: Fallback defaults used when APIs fail

✅ **Subscribe Full**
- `should create concept order with full notification (requester already in form)`
- Mocks: headers, client, careteam APIs
- Verifies: Form data preserved, no API enrichment needed

✅ **Unsubscribe Minimal**
- `should create concept return without order lookup`
- Mocks: headers, client, returner APIs
- Verifies: No order lookup, direct concept return creation

---

#### **Test cases đã THÊM (93 lines - EcarePuurServiceTest.ts - NEW file):**

✅ **Subscribe with API Failures** (unit tests)
- `should create concept order even when client API fails`
- `should create concept order even when careteam API fails`
- `should create concept order even when requester API fails`
- **Verifies:** Best-effort enrichment với fallback defaults

✅ **Unsubscribe without Order Lookup** (unit tests)
- `should create concept return without looking up existing order`
- `should use default returner when API fails`
- **Verifies:** No order lookup, best-effort returner enrichment

✅ **Form Preservation** (unit tests)
- `should call mapNotificationToForm() to preserve raw data`
- `should pass both form and orderDto to ConceptService`
- **Verifies:** Dual mapping pattern (form + DTO)

---

#### **Summary: Test Philosophy Change**

**Old Philosophy:**
- Strict validation → Many test cases verify validation rules
- Order tracking → Tests verify duplicate checks, order lookup, status logic
- Complex integration tests → 870 lines với 30+ scenarios

**New Philosophy:**
- Relaxed validation → Remove validation test cases
- No order tracking → Remove duplicate/status test cases
- Simplified integration tests → 260 lines với 5 essential scenarios
- Added unit tests → 93 lines verify service layer logic (API failures, fallback defaults, dual mapping)

**Net Result:**
- Integration tests: -627 lines (simplified)
- Unit tests: +93 lines (new service tests)
- Total: -534 lines (cleaner, more focused)
- Test coverage: 97 → 98 passing tests