# 📋 [REFACTOR: 2026-01-20] - Schema Folder Restructure

## References

- `wonkers-graphql/src/graphql/schema/` - Current schema folder
- `wonkers-graphql/src/graphql/schema/index.ts` - Main schema entry
- `wonkers-graphql/src/schema/typeDefs.ts` - Legacy GraphQL type definitions
- `wonkers-graphql/src/types.ts` - Generated TypeScript types

## User Requirements

> Tổ chức lại folder structure trong `wonkers-graphql/src/graphql/schema` để:
> 1. Nhìn nhanh biết có bao nhiêu loại report đã được implement
> 2. Folder structure mapping 1:1 với GraphQL hierarchy
> 3. `kpi`, `organisationReports`, `allReports` đều nằm dưới `reports` (theo legacy schema)

## 🎯 Objective

Restructure `wonkers-graphql/src/graphql/schema` từ flat structure sang feature-based structure, đảm bảo folder hierarchy phản ánh đúng GraphQL schema hierarchy.

### ⚠️ Key Considerations

1. **GraphQL Hierarchy phải đúng**: 
   - `Query.rawData` → `rawData/`
   - `Query.reports` → `reports/`
   - `Query.reports.kpi` → `reports/kpi/`
   - `Query.reports.organisationReports` → `reports/organisationReports/`
   - `Query.reports.allReports` → `reports/allReports/`

2. **Không thay đổi logic**: Chỉ di chuyển files và update imports, không thay đổi business logic.

3. **Test vẫn phải pass**: Update test imports tương ứng.

## 🔄 Implementation Plan

### Phase 1: Analysis & Preparation

- [x] Analyze current folder structure
  - **Outcome**: Current structure có `kpi/`, `rawData/`, `reports/` ở cùng level. Nhưng theo GraphQL schema, `kpi` nằm dưới `Report` type.
- [x] Define scope and edge cases
  - **Outcome**: 
    - Move `kpi/` vào `reports/kpi/`
    - Tổ chức `reports/` thành sub-folders theo report type
    - Update tất cả imports
- [x] Evaluate existing test structures
  - **Outcome**: 3 test files cần update imports:
    - `test/graphql/schema/reports/tessaOrderStatusReportService.test.ts`
    - `test/graphql/schema/reports/sensaraEventReportService.test.ts`
    - `test/graphql/schema/rawData/robotProfile.test.ts` (không cần thay đổi)

### Phase 2: Implementation (File/Code/Test Structure)

**Current Structure:**

```
src/graphql/schema/
├── index.ts
├── kpi/                                    # ❌ Sai vị trí - phải nằm trong reports/
│   ├── index.ts
│   ├── config.ts
│   ├── kpiExtension.ts
│   ├── operationKpi.ts
│   ├── retentionKpi.ts
│   └── retentionKpiService.ts
├── rawData/                                # ✅ Đúng vị trí
│   ├── index.ts
│   ├── rawData.ts
│   ├── rawDataQuery.ts
│   └── robotProfile.ts
└── reports/                                # ❌ Flat structure - khó nhìn
    ├── index.ts
    ├── allReports.ts
    ├── organisationReportsExtension.ts
    ├── reportExtension.ts
    ├── salesOrderShipmentInformationReport.ts
    ├── salesOrderShipmentInformationReportService.ts
    ├── sensaraEventReport.ts
    ├── sensaraEventReportService.ts
    ├── tessaOrderStatusReport.ts
    └── tessaOrderStatusReportService.ts
```

**Target Structure:**

```
src/graphql/schema/
├── index.ts                                # 🔄 UPDATE imports
├── rawData/                                # ✅ UNCHANGED
│   ├── index.ts
│   ├── rawData.ts
│   ├── rawDataQuery.ts
│   └── robotProfile.ts
│
└── reports/                                # 🔄 RESTRUCTURE
    ├── index.ts                            # 🔄 UPDATE re-exports
    ├── report.type.ts                      # 🔄 RENAME from reportExtension.ts
    │
    ├── kpi/                                # 🔄 MOVE from schema/kpi/
    │   ├── index.ts
    │   ├── kpi.type.ts                     # RENAME from kpiExtension.ts
    │   ├── config.ts
    │   ├── operationKpi/
    │   │   ├── index.ts                    # 🚧 NEW
    │   │   └── operationKpi.type.ts        # 🔄 MOVE
    │   └── retentionKpi/
    │       ├── index.ts                    # 🚧 NEW
    │       ├── retentionKpi.type.ts        # 🔄 MOVE
    │       └── retentionKpi.service.ts     # 🔄 MOVE
    │
    ├── organisationReports/                # 🚧 NEW folder
    │   ├── index.ts                        # 🚧 NEW
    │   ├── organisationReports.type.ts     # 🔄 RENAME from organisationReportsExtension.ts
    │   ├── salesOrderShipment/
    │   │   ├── index.ts                    # 🚧 NEW
    │   │   ├── salesOrderShipment.type.ts  # 🔄 MOVE + RENAME
    │   │   └── salesOrderShipment.service.ts # 🔄 MOVE + RENAME
    │   └── tessaOrderStatus/
    │       ├── index.ts                    # 🚧 NEW
    │       ├── tessaOrderStatus.type.ts    # 🔄 MOVE + RENAME
    │       └── tessaOrderStatus.service.ts # 🔄 MOVE + RENAME
    │
    └── allReports/                         # 🚧 NEW folder
        ├── index.ts                        # 🚧 NEW
        ├── allReports.type.ts              # 🔄 MOVE + RENAME
        └── sensaraEvent/
            ├── index.ts                    # 🚧 NEW
            ├── sensaraEvent.type.ts        # 🔄 MOVE + RENAME
            └── sensaraEvent.service.ts     # 🔄 MOVE + RENAME
```

**Test Structure (mirror source):**

```
test/graphql/schema/
├── rawData/
│   └── robotProfile.test.ts                # ✅ UNCHANGED
└── reports/
    ├── organisationReports/
    │   └── tessaOrderStatus/
    │       └── tessaOrderStatus.service.test.ts  # 🔄 MOVE + UPDATE imports
    └── allReports/
        └── sensaraEvent/
            └── sensaraEvent.service.test.ts      # 🔄 MOVE + UPDATE imports
```

### Phase 3: Detailed Implementation Steps

#### Step 1: Create new folder structure

```bash
# Create organisationReports folders
mkdir -p src/graphql/schema/reports/organisationReports/salesOrderShipment
mkdir -p src/graphql/schema/reports/organisationReports/tessaOrderStatus

# Create allReports folders
mkdir -p src/graphql/schema/reports/allReports/sensaraEvent

# Create kpi sub-folders
mkdir -p src/graphql/schema/reports/kpi/operationKpi
mkdir -p src/graphql/schema/reports/kpi/retentionKpi
```

#### Step 2: Move and rename files

**2.1 Move kpi/ into reports/kpi/**

| From | To |
|------|-----|
| `schema/kpi/config.ts` | `schema/reports/kpi/config.ts` |
| `schema/kpi/kpiExtension.ts` | `schema/reports/kpi/kpi.type.ts` |
| `schema/kpi/operationKpi.ts` | `schema/reports/kpi/operationKpi/operationKpi.type.ts` |
| `schema/kpi/retentionKpi.ts` | `schema/reports/kpi/retentionKpi/retentionKpi.type.ts` |
| `schema/kpi/retentionKpiService.ts` | `schema/reports/kpi/retentionKpi/retentionKpi.service.ts` |

**2.2 Reorganize organisationReports/**

| From | To |
|------|-----|
| `reports/organisationReportsExtension.ts` | `reports/organisationReports/organisationReports.type.ts` |
| `reports/salesOrderShipmentInformationReport.ts` | `reports/organisationReports/salesOrderShipment/salesOrderShipment.type.ts` |
| `reports/salesOrderShipmentInformationReportService.ts` | `reports/organisationReports/salesOrderShipment/salesOrderShipment.service.ts` |
| `reports/tessaOrderStatusReport.ts` | `reports/organisationReports/tessaOrderStatus/tessaOrderStatus.type.ts` |
| `reports/tessaOrderStatusReportService.ts` | `reports/organisationReports/tessaOrderStatus/tessaOrderStatus.service.ts` |

**2.3 Reorganize allReports/**

| From | To |
|------|-----|
| `reports/allReports.ts` | `reports/allReports/allReports.type.ts` |
| `reports/sensaraEventReport.ts` | `reports/allReports/sensaraEvent/sensaraEvent.type.ts` |
| `reports/sensaraEventReportService.ts` | `reports/allReports/sensaraEvent/sensaraEvent.service.ts` |

**2.4 Rename report root file**

| From | To |
|------|-----|
| `reports/reportExtension.ts` | `reports/report.type.ts` |

#### Step 3: Create index.ts files

Create barrel exports for each new folder:

- `reports/kpi/operationKpi/index.ts`
- `reports/kpi/retentionKpi/index.ts`
- `reports/kpi/index.ts` (update)
- `reports/organisationReports/salesOrderShipment/index.ts`
- `reports/organisationReports/tessaOrderStatus/index.ts`
- `reports/organisationReports/index.ts`
- `reports/allReports/sensaraEvent/index.ts`
- `reports/allReports/index.ts`
- `reports/index.ts` (update)

#### Step 4: Update imports in source files

- `schema/index.ts` - Update imports from `./kpi` to `./reports/kpi`
- `reports/report.type.ts` - Update import path for `OrganisationReports`
- `reports/organisationReports/organisationReports.type.ts` - Update imports
- `reports/allReports/allReports.type.ts` - Update imports
- All moved files - Update relative imports

#### Step 5: Move and update test files

**5.1 Create test folder structure**

```bash
mkdir -p test/graphql/schema/reports/organisationReports/tessaOrderStatus
mkdir -p test/graphql/schema/reports/allReports/sensaraEvent
```

**5.2 Move test files**

| From | To |
|------|-----|
| `test/.../reports/tessaOrderStatusReportService.test.ts` | `test/.../reports/organisationReports/tessaOrderStatus/tessaOrderStatus.service.test.ts` |
| `test/.../reports/sensaraEventReportService.test.ts` | `test/.../reports/allReports/sensaraEvent/sensaraEvent.service.test.ts` |

**5.3 Update test imports**

```typescript
// tessaOrderStatus.service.test.ts
// FROM:
import { TessaOrderStatusReportService } from '../../../../src/graphql/schema/reports/tessaOrderStatusReportService'
// TO:
import { TessaOrderStatusReportService } from '../../../../../../src/graphql/schema/reports/organisationReports/tessaOrderStatus/tessaOrderStatus.service'

// sensaraEvent.service.test.ts
// FROM:
import { SensaraEventReportService } from '../../../../src/graphql/schema/reports/sensaraEventReportService'
// TO:
import { SensaraEventReportService } from '../../../../../../src/graphql/schema/reports/allReports/sensaraEvent/sensaraEvent.service'
```

#### Step 6: Cleanup old files/folders

- Delete `src/graphql/schema/kpi/` folder
- Delete old files in `reports/`:
  - `organisationReportsExtension.ts`
  - `salesOrderShipmentInformationReport.ts`
  - `salesOrderShipmentInformationReportService.ts`
  - `tessaOrderStatusReport.ts`
  - `tessaOrderStatusReportService.ts`
  - `allReports.ts`
  - `sensaraEventReport.ts`
  - `sensaraEventReportService.ts`
  - `reportExtension.ts`
- Delete old test files

#### Step 7: Verify

```bash
# Build check
cd wonkers-graphql && yarn build

# Run tests
just -f devtools/Justfile test-wonkers-graphql
```

## 📊 Summary of Results

> Do not summarize until implementation is done

### ✅ Completed Achievements

- [ ] Pending implementation

## 🚧 Outstanding Issues & Follow-up

### ⚠️ Issues/Clarifications (Optional)

- None at this time
