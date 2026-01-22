# Retention KPI (Daily) – Implementation Plan

## Mục tiêu
- Tính KPI retention theo ngày (daily), xuất mỗi điểm dữ liệu gồm `date` (UTC), `retentionKPI` (tỉ lệ 0..1), và `population`.
- Bỏ qua `archived_at` trong tính toán (theo logic trong notebook).
- Cho phép cấu hình chính sách phạm vi ngày qua `DateRangePolicy` với 2 chế độ:
  - `ignoreRetentionPeriod` (mặc định)
  - `respectRetentionPeriod`
- Tham số bắt buộc: `P` (kích thước cửa sổ dân số tính theo ngày) và `R` (ngưỡng retention theo ngày). `startDate`, `endDate` là tùy chọn.
- Giữ nguyên tích hợp GraphQL dựa trên Nexus tại `src/graphql/schema/kpi`, dùng `ctx.prisma.dashboard` để truy vấn bảng `taas_subscription`.

## Ngữ cảnh dự án
- GraphQL schema (Nexus) nằm trong `src/graphql/schema`, có nhóm KPI hiện tại trong `src/graphql/schema/kpi/`:
  - `index.ts` (export KPI types)
  - `kpiExtension.ts` (type `KPI` có field `operationKPIs`)
  - `operationKpi.ts` (field `retentionKpi` và args)
  - `retentionKpi.ts` (type/data point + resolver logic)
  - `retentionKpiService.ts` (dịch vụ tính toán hiện tại theo tháng)
- Prisma context:
  - `ctx.prisma.dashboard` và `ctx.prisma.tinybots` được cung cấp bởi `buildPrismaContext()` trong `src/graphql/context/prismaContext.ts`.
  - Prisma client khởi tạo/đóng kết nối trong `src/graphql/lib/prisma.ts`.
- Generated schema output: `src/generated/graphql/schema.graphql` đang chứa định nghĩa KPI cũ (theo tháng).

## Chính sách phạm vi ngày (DateRangePolicy)
Khai báo trong file mới `src/graphql/schema/kpi/config.ts`:
- `DateRangePolicy = 'ignoreRetentionPeriod' | 'respectRetentionPeriod'`.
- `DATE_RANGE_POLICY` mặc định: `'ignoreRetentionPeriod'`.

Cách áp dụng:
- Các ngày đều được chuẩn hóa về UTC midnight (`00:00:00Z`). Gọi ngày đang xét là `x`.
- `today` là UTC midnight của ngày hiện tại.
- Với mỗi policy, xác định `xStart`, `xEnd` (phạm vi ngày để xuất KPI) và phạm vi `start_at` để prefetch dữ liệu một lần.

### ignoreRetentionPeriod (mặc định)
- Khi không truyền `startDate`/`endDate`:
  - Tính `minStartAt` từ DB (`min(taas_subscription.start_at)`), UTC midnight.
  - `xStart = minStartAt + P`.
  - `xEnd = today`.
  - Prefetch subscriptions với `start_at` trong `[minStartAt, today]`.
- Khi có `startDate` và `endDate`:
  - `xStart = startDate` (UTC midnight), `xEnd = endDate` (UTC midnight).
  - Prefetch với `start_at` trong `[startDate - P, endDate]` (để đủ cửa sổ dân số P trước `xStart`).
- Ghi chú: Với `R` lớn, điểm KPI gần `today` có thể thấp/0 do nhiều subscription chưa đạt `R` ngày — chấp nhận.

### respectRetentionPeriod
- Khi không truyền `startDate`/`endDate`:
  - `minStartAt` (UTC midnight) như trên.
  - `xStart = minStartAt + P`.
  - `xEnd = today - R`.
  - Prefetch với `start_at` trong `[minStartAt, today - R]`.
- Khi có `startDate` và/hoặc `endDate`:
  - `xStart = startDate`, `xEnd = endDate`.
  - Prefetch subscriptions với `start_at` trong `[startDate - P, endDate]` và thêm điều kiện `start_at < today - R`.
- Phần xác nhận:
  - Giữ `effectiveEndDate = today - R` khi dùng mặc định (không truyền ngày).
  - Dùng điều kiện AND `start_at < today - R` kết hợp với khoảng `start_at` cho prefetch — đã được xác nhận.

## Định nghĩa KPI và thuật toán
- Mỗi ngày `x ∈ [xStart..xEnd]`:
  - `population`: số subscription có `start_at ∈ [x - P + 1 .. x]`.
  - `lengthDaysAtX` cho 1 subscription bắt đầu tại `s` và kết thúc (nếu có) tại `e`:
    - `effectiveEndAtForX = min(e ?? x, x)` — nếu còn hoạt động sau `x`, tính đến `x`; nếu đã kết thúc trước/đúng `x`, dùng `e`.
    - `lengthDaysAtX = daysBetweenUTC(s, effectiveEndAtForX)`.
  - `retained`: số subscription trong `population` có `lengthDaysAtX >= R`.
  - `retentionKPI = retained / population` (0..1). Nếu `population = 0`, đặt `retentionKPI = 0`.
- Bỏ qua `archived_at` trong mọi phép tính.
- `end_at = null` nghĩa là còn chạy, nhưng tại ngày `x` chỉ tính độ dài đến `x`.

## Chuẩn hóa ngày (UTC midnight)
- Luôn chuyển `startDate`, `endDate`, `minStartAt`, `today`, mọi `start_at`/`end_at` về dạng `YYYY-MM-DD` và thao tác ở mốc `00:00:00Z`.
- Hàm tiện ích `flattenDate(dateLike)` như mẫu trong `rawData/robotProfile.ts` có thể tái sử dụng hoặc viết riêng trong `kpi/utils`.

## Truy vấn prefetch (Prisma)
- Chỉ truy vấn 1 lần tất cả subscriptions trong phạm vi xác định theo policy để tối ưu hiệu năng.
- Bảng: `ctx.prisma.dashboard.taas_subscription`.
- Select các cột cần thiết: `id`, `start_at`, `end_at` (có thể thêm `shipped_at` nếu cần, mặc định không).
- Bỏ qua `archived_at` (không filter theo `archived_at`, hoặc filter `archived_at = null` nếu muốn loại bỏ bản ghi archive).
- Điều kiện `where` theo policy:
  - ignoreRetentionPeriod (no dates): `start_at ∈ [minStartAt, today]`.
  - ignoreRetentionPeriod (with dates): `start_at ∈ [startDate - P, endDate]`.
  - respectRetentionPeriod (no dates): `start_at ∈ [minStartAt, today - R]`.
  - respectRetentionPeriod (with dates): `start_at ∈ [startDate - P, endDate] AND start_at < today - R`.

## Tối ưu sliding-window
- Sau prefetch, sắp xếp subscriptions theo `start_at`.
- Duyệt từng ngày `x` tăng dần, dùng 2 con trỏ (head/tail) để duy trì cửa sổ `population` theo điều kiện `start_at ∈ [x - P + 1 .. x]`.
- Tính `retained` bằng cách kiểm tra `lengthDaysAtX >= R` cho các phần tử trong cửa sổ; có thể cache `lengthAtX` theo subscription nếu cần.
- Tránh filter toàn bộ mỗi ngày.

## Cập nhật GraphQL (Nexus)
- Đổi kiểu trả về từ monthly sang daily:
  - Tạo type mới `RetentionKpiPoint` với fields:
    - `date: String!` (ISO `YYYY-MM-DD`)
    - `retentionKPI: Float!` (0..1)
    - `population: Int!`
- Cập nhật `OperationKPI.retentionKpi`:
  - Args: `P: Int!`, `R: Int!`, `startDate: String`, `endDate: String`.
  - Loại bỏ `clientId`, `teamId`.
  - Return: `[RetentionKpiPoint!]!`.
- Thêm `config.ts` trong `src/graphql/schema/kpi/` để cấu hình `DATE_RANGE_POLICY`.

## Luồng xử lý Resolver và Service
- `retentionKpi.ts`:
  - Định nghĩa `RetentionKpiArgsDto` với validation: `P`, `R` bắt buộc và > 0; `startDate`/`endDate` tùy chọn, đúng định dạng ngày.
  - `resolveRetentionKpi(args, ctx)`:
    - Kiểm tra quyền (giữ như hiện tại hoặc thay bằng pattern `BaseResolver.Wrap` nếu cần).
    - Chuẩn hóa ngày đầu vào, lấy `today` (UTC midnight).
    - Tính `minStartAt` (khi cần) bằng Prisma aggregate `min(start_at)`.
    - Suy ra `xStart`, `xEnd` và phạm vi prefetch theo `DATE_RANGE_POLICY`.
    - Gọi `RetentionKpiService.calculateDailyKpi({ P, R, xStart, xEnd, prefetchRange, policy })` và trả về danh sách điểm.
- `retentionKpiService.ts`:
  - Chuyển từ monthly sang daily.
  - Constructor nhận `ctx.prisma.dashboard` (type: `ReturnType<typeof getDashboardPrisma>`), KHÔNG import `@prisma/client` trực tiếp.
  - Thêm hàm `prefetchSubscriptions(range)` và `calculateDailyKpi(params)` theo thuật toán trên.

## Xác thực tham số và lỗi
- `P`, `R`: số nguyên dương (`>= 1`). Nếu không hợp lệ, trả lỗi GraphQL `BAD_USER_INPUT`.
- `startDate`/`endDate`: đúng định dạng `YYYY-MM-DD`.
- Nếu `xStart > xEnd`: trả về mảng rỗng (không ném lỗi) — trường hợp người dùng chọn phạm vi sai.
- Khi `population = 0` tại một ngày: `retentionKPI = 0`.

## Các edge cases
- Subscription có `end_at = null`: coi là đang hoạt động, nhưng khi tính cho ngày `x`, chỉ tính đến `x`.
- Subscription kết thúc trước `x`: dùng `end_at` thực tế để tính `lengthDaysAtX`.
- Bản ghi có `start_at` ngoài phạm vi prefetch: không ảnh hưởng vì đã prefetch đúng phạm vi.
- Giá trị ngày trùng biên (VD: `x - P + 1`): dùng chuẩn hóa UTC để tránh lệch timezone.

## Hiệu năng
- Một lần prefetch theo phạm vi chính sách, giới hạn cột select.
- Duyệt theo sliding-window, `O(N + D)` với `N` số subscription prefetch và `D` số ngày.
- Bộ nhớ: danh sách subscription prefetch có thể lớn; cân nhắc stream nếu thực tế quá lớn (không triển khai ở bước này).

## Thay đổi file cụ thể
1. `src/graphql/schema/kpi/config.ts` (tạo mới)
   - Khai báo `export type DateRangePolicy = 'ignoreRetentionPeriod' | 'respectRetentionPeriod'`
   - `export const DATE_RANGE_POLICY: DateRangePolicy = 'ignoreRetentionPeriod'`
2. `src/graphql/schema/kpi/retentionKpi.ts`
   - Tạo `RetentionKpiPoint` thay cho `RetentionKpiDataPoint`.
   - Cập nhật `resolveRetentionKpi` để nhận args `{ P: Int!, R: Int!, startDate?, endDate? }`.
   - Tính `xStart`, `xEnd`, prefetch range theo `DATE_RANGE_POLICY`.
   - Trả về danh sách `RetentionKpiPoint` theo ngày.
3. `src/graphql/schema/kpi/operationKpi.ts`
   - Đổi args field `retentionKpi` thành: `P`, `R` (nonNull int), `startDate?`, `endDate?`.
   - Loại bỏ `clientId`, `teamId`.
   - Type trả về: danh sách `RetentionKpiPoint`.
4. `src/graphql/schema/kpi/index.ts`
   - Export `RetentionKpiPoint` thay cho `RetentionKpiDataPoint`.
5. `src/graphql/schema/index.ts`
   - Thêm `RetentionKpiPoint` vào `types` thay vì `RetentionKpiDataPoint`.
6. `src/graphql/schema/kpi/retentionKpiService.ts`
   - Viết lại từ đầu theo daily KPI, sliding-window, nhận `ctx.prisma.dashboard` trong constructor.
   - Truy vấn prefetch với `findMany` theo policy, bỏ qua `archived_at`.

## Kiểm thử
- Unit tests cho service (`retentionKpiService`):
  - `end_at = null` và `end_at` trước/sau ngày `x`.
  - Biên `lengthDaysAtX = R - 1`, `= R`, `> R`.
  - `population = 0` → `retentionKPI = 0`.
  - Phạm vi prefetch theo 2 policy (no dates / with dates).
  - `xStart > xEnd` → mảng rỗng.
- Resolver tests (`retentionKpi.ts`):
  - Validation `P`, `R` bắt buộc và > 0.
  - Chuẩn hóa ngày UTC.
  - Default theo `ignoreRetentionPeriod`.
  - `respectRetentionPeriod` cho trường hợp không truyền ngày và có truyền ngày.
- Có thể đặt test tại `test/graphql/schema/...` hoặc `test/graphqlIT/InternalIT.ts` nếu muốn E2E.

## Ví dụ GraphQL queries
- Mặc định (ignoreRetentionPeriod, không truyền ngày):
```graphql
query {
  kpi {
    operationKPIs {
      retentionKpi(P: 30, R: 30) {
        date
        retentionKPI
        population
      }
    }
  }
}
```
- Có truyền ngày:
```graphql
query {
  kpi {
    operationKPIs {
      retentionKpi(P: 30, R: 30, startDate: "2023-01-01", endDate: "2023-03-31") {
        date
        retentionKPI
        population
      }
    }
  }
}
```
- Tôn trọng kỳ hạn (respectRetentionPeriod, sau khi đổi config):
```ts
// src/graphql/schema/kpi/config.ts
export const DATE_RANGE_POLICY: DateRangePolicy = 'respectRetentionPeriod'
```
```graphql
query {
  kpi {
    operationKPIs {
      retentionKpi(P: 30, R: 30) {
        date         # từ minStartAt + P đến today - R
        retentionKPI
        population
      }
    }
  }
}
```

## Ghi chú tích hợp và tương thích
- Generated schema cũ có `RetentionKpiDataPoint` theo tháng; sau thay đổi cần chạy build để cập nhật artifacts (`nexus-typegen`, `schema.graphql`).
- Quyền truy cập: hiện tại kiểm tra `ctx.permissions?.includes('ROBOT_PROFILE_READ_ALL')`. Có thể chuyển sang `BaseResolver.Wrap` nếu muốn đồng nhất với các resolver khác, nhưng không bắt buộc trong bước này.
- Không thêm `clientId`, `teamId` vào filter nữa (theo yêu cầu mới). Nếu sau này cần, có thể tái bổ sung.

## Checklist triển khai
- [ ] Tạo `config.ts` và set `DATE_RANGE_POLICY` mặc định.
- [ ] Viết type `RetentionKpiPoint`.
- [ ] Đổi args `OperationKPI.retentionKpi` → `{ P!, R!, startDate?, endDate? }`.
- [ ] Viết lại `resolveRetentionKpi` theo 2 policy.
- [ ] Viết lại `retentionKpiService` theo daily + sliding-window.
- [ ] Cập nhật exports/index và `src/graphql/schema/index.ts`.
- [ ] Viết tests (service + resolver).
- [ ] Chạy build và verify generated schema.

## Rollback
- Giữ bản cũ `RetentionKpiDataPoint` và monthly service trong git history để dễ rollback.
- Nếu gặp vấn đề ở môi trường thực, đổi `DATE_RANGE_POLICY` về `'ignoreRetentionPeriod'` để giảm rủi ro đối với dữ liệu gần `today`.