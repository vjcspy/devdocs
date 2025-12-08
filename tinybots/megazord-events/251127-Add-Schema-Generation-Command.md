# 📋 [251127] - Add Schema Generation Command & Existence Check

## References

- `megazord-events/schemas/gen.ts` - Current schema generator script
- `megazord-events/package.json` - Package scripts configuration
- `megazord-events/devdocs/tinybots/megazord-events/OVERVIEW.md` - Repository documentation

## User Requirements

Hiện tại trong `megazord-events`:

1. Chưa có command để generate schema từ `schemas/gen.ts`
2. Generator chưa có check nếu schema đã tồn tại thì không generate nữa (tránh ghi đè)

## 🎯 Objective

Thêm npm script command để chạy schema generator và cải thiện logic generation để tránh ghi đè các schema file đã tồn tại.

### ⚠️ Key Considerations

- **Backward Compatibility**: Không được làm thay đổi cấu trúc hiện tại của schema files
- **Selective Generation**: Chỉ generate những schema chưa tồn tại để tránh ghi đè customization
- **Force Option**: Cần có cách để force regenerate tất cả nếu cần (qua CLI flag hoặc environment variable)
- **Developer Experience**: Command phải dễ nhớ và consistent với các npm scripts khác trong project

## 🔄 Implementation Plan

### Phase 1: Analysis & Preparation

- [ ] Xem xét current schema generation logic trong `gen.ts`
  - **Outcome**: Hiểu rõ flow generation hiện tại: loop qua TinybotsEvent enum → apply custom config → write JSON file
- [ ] Kiểm tra structure của generated schema files
  - **Outcome**: Xác nhận format: `eventName`, `level`, `hasTrigger`, `isActive`, `description`
- [ ] Review package.json scripts hiện có
  - **Outcome**: Xác định naming convention (dùng `:` separator) và position phù hợp cho script mới

### Phase 2: Implementation (File/Code Structure)

**Các files sẽ được modify:**

```text
megazord-events/
├── schemas/
│   └── gen.ts                  # 🔄 IN PROGRESS - Add existence check logic
└── package.json                # 🔄 IN PROGRESS - Add npm script
```

**Không tạo file mới** - chỉ modify 2 files trên.

### Phase 3: Detailed Implementation Steps

#### Step 1: Add npm Script Command

**File:** `package.json`

**Action:** Thêm script `generate:schemas` vào section `scripts`

```json
"generate:schemas": "ts-node schemas/gen.ts"
```

**Rationale:**

- Naming convention theo pattern `generate:*` (tương tự `test:only`)
- Sử dụng `ts-node` để chạy TypeScript trực tiếp (đã có trong devDependencies)
- Đơn giản, dễ nhớ: `yarn generate:schemas`

#### Step 2: Add File Existence Check to Generator

**File:** `schemas/gen.ts`

**Changes Required:**

1. Import `access` from `node:fs/promises` (hoặc `existsSync` từ `node:fs`)
2. Add helper function để check file existence
3. Modify `genSchema()` để skip nếu file đã tồn tại
4. Add optional force flag qua environment variable `FORCE_GENERATE=true`

**Proposed Logic:**

```typescript
import { writeFile, access } from 'node:fs/promises'
import { constants } from 'node:fs'
import path from 'node:path'

// ... existing code ...

const fileExists = async (filePath: string): Promise<boolean> => {
  try {
    await access(filePath, constants.F_OK)
    return true
  } catch {
    return false
  }
}

const genSchema = async (
  name: keyof typeof TinybotsEvent
): Promise<void> => {
  const filePath = path.join(__dirname, `./events/${name.toLowerCase()}.json`)
  
  // Check if file exists and FORCE_GENERATE is not set
  const forceGenerate = process.env.FORCE_GENERATE === 'true'
  
  if (!forceGenerate && await fileExists(filePath)) {
    console.log(`Skipping ${name} - file already exists`)
    return
  }

  const schema: EventSchema = {
    eventName: name,
    level: DefaultConfig.level,
    hasTrigger: DefaultConfig.hasTrigger,
    isActive: true,
    description: 'Auto generated schema definition by megazord-events'
  }

  const custom = CustomConfigs[name]

  if (custom) {
    schema.level = custom.level
    schema.hasTrigger = custom.hasTrigger
  }

  await writeFile(
    filePath,
    JSON.stringify(schema, null, 4)
  )
  
  console.log(`Generated ${name}`)
}
```

#### Step 3: Update Console Output

**File:** `schemas/gen.ts`

**Changes:**

- Cải thiện logging để thấy rõ files nào được generated vs skipped
- Show summary cuối cùng

**Proposed:**

```typescript
async function run() {
  console.log('🔧 Generating event schemas...')
  console.log(`Force mode: ${process.env.FORCE_GENERATE === 'true' ? 'ON' : 'OFF'}`)
  console.log('─'.repeat(50))
  
  await Promise.all(Object.values(TinybotsEvent).map(genSchema))
  
  console.log('─'.repeat(50))
  console.log('✅ Schema generation completed')
  console.log('💡 Tip: Use FORCE_GENERATE=true to regenerate all schemas')
}
```

#### Step 4: Update Documentation (Optional but Recommended)

**File:** `devdocs/tinybots/megazord-events/OVERVIEW.md`

**Action:** Update reference to generator command

Replace:

```text
rerun `yarn ts-node schemas/gen.ts` when altering `schemas/gen.ts`
```

With:

```text
run `yarn generate:schemas` to create new schemas or `FORCE_GENERATE=true yarn generate:schemas` to regenerate all when altering schema definitions
```

### Phase 4: Testing & Verification

#### Manual Testing Steps

1. **Test new schemas generation:**

   ```bash
   # Xóa 1-2 schema files để test
   rm schemas/events/inactivity.json
   rm schemas/events/activity.json
   
   # Run generator
   yarn generate:schemas
   
   # Verify: files được tạo lại
   ```

2. **Test existence check:**

   ```bash
   # Run lại generator
   yarn generate:schemas
   
   # Expected: "Skipping ... - file already exists" cho tất cả files
   ```

3. **Test force mode:**

   ```bash
   FORCE_GENERATE=true yarn generate:schemas
   
   # Expected: Tất cả files được regenerate
   ```

4. **Verify generated content:**

   ```bash
   # Check format và values đúng
   cat schemas/events/suspicious_inactivity.json
   # Should have level: 5, hasTrigger: true
   
   cat schemas/events/inactivity.json  
   # Should have level: 10, hasTrigger: false
   ```

## 📊 Summary of Results

> Sẽ update sau khi implementation hoàn thành

### ✅ Completed Achievements

- TBD

## 🚧 Outstanding Issues & Follow-up

### ⚠️ Issues/Clarifications

- [ ] **Question**: Có cần thêm flag `--force` qua CLI args thay vì environment variable không?
  - **Impact**: Tốt hơn cho DX nhưng cần thêm arg parser (có thể dùng `process.argv`)
  
- [ ] **Question**: Có cần validate existing schema structure trước khi skip không?
  - **Impact**: Đảm bảo file tồn tại là valid JSON với đúng structure
  
- [ ] **Enhancement**: Xem xét thêm `generate:schemas:watch` mode để auto-generate khi thay đổi enum?
  - **Impact**: Low priority - chỉ cần nếu thường xuyên thêm events mới
