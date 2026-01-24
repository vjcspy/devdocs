# Create Product Flow (UI → API → Persist → Display)

## 1) End-to-end flow (conceptual)

Một “create product” flow tốt thường có các bước:
1. User nhập `name`, `price`
2. Client-side validate cơ bản (UX): rỗng, không phải số…
3. Submit tới server endpoint (WP custom endpoint)
4. Server-side:
   - authorize (capability, nonce)
   - validate + sanitize
   - call WooCommerce REST API (hoặc internal CRUD)
5. Trả response JSON (success/error)
6. Client render trạng thái:
   - success: hiển thị product info + link
   - error: hiển thị message có hướng xử lý

## 2) Payload tối thiểu cho WooCommerce REST API

Khi tạo simple product, concept payload:
- `name`
- `type`: `simple`
- `regular_price`: string

## 3) Error handling “đúng tầng”

Tách 3 loại lỗi:
- Client input error (UX): báo ngay, không gửi request
- Server validation/authz error: trả 400/403 với message rõ
- Upstream WooCommerce REST error: wrap lại để client hiểu

Điểm sâu: giữ “error contract” ổn định
- Client chỉ cần biết: `code`, `message`, `details` (optional)
- Không leak secret/stacktrace

## 4) Observability tối thiểu (để debug nhanh)

Trong local assessment, bạn vẫn nên có chiến lược debug:
- Log hoặc surface lỗi trong WP debug log (nếu bật)
- Trả message ngắn gọn cho client
- Nếu call upstream fail, giữ nguyên status code hợp lý

## 5) Gợi ý thực hành

Tạo checklist test cases:
- name rỗng
- price rỗng
- price không phải số
- price âm
- request không có nonce
- user không đủ quyền
