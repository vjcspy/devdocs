# WordPress REST API & AJAX Patterns

## 1) Khi nào dùng REST, khi nào dùng AJAX

Trong WordPress, “AJAX” thường có nghĩa là gọi tới:
- `POST /wp-admin/admin-ajax.php?action=...`

Còn REST API thường là:
- `GET/POST /wp-json/<namespace>/<version>/<route>`

So sánh nhanh:
- **REST API**:
  - URL rõ ràng, semantics tốt
  - Dễ versioning (`v1`, `v2`)
  - Dễ dùng cho cả frontend và tích hợp hệ thống khác
- **admin-ajax.php**:
  - Legacy nhưng phổ biến
  - Có sẵn cơ chế nonce/capability patterns
  - Có thể đơn giản hơn cho bài nhỏ

Với assessment, REST API thường “đúng concept” hơn vì WooCommerce cũng là REST.

## 2) REST route custom (tư duy thiết kế)

Một route tốt cần trả lời 4 câu hỏi:
1. Namespace/version là gì? (ví dụ `vocalmeet/v1`)
2. Resource là gì? (ví dụ `products`)
3. Method là gì? (POST để tạo)
4. Authz policy là gì? Ai được tạo product?

Thay vì cho phép ai cũng tạo product, bạn thường muốn:
- chỉ user đã login, hoặc
- chỉ user có capability nhất định, hoặc
- tối thiểu có rate-limit/throttle (assessment có thể không yêu cầu, nhưng nên hiểu)

## 3) Server-side handler: validate → authorize → execute

Pattern “đúng chuẩn” trong WP:
1. **Validate**: dữ liệu có đúng shape không (name không rỗng, price là số dương).
2. **Sanitize**: làm sạch dữ liệu trước khi dùng/ghi DB.
3. **Authorize**: kiểm tra capability/nonce.
4. **Execute**: gọi WooCommerce REST API hoặc WooCommerce internal API.
5. **Respond**: trả JSON có cấu trúc ổn định (kể cả khi error).

## 4) Client-side calls

Trong JS, bạn thường gọi:
- `fetch(...)` tới REST endpoint
- Gắn `X-WP-Nonce` (nếu dùng cookie auth cho user login)

Điểm sâu cần hiểu:
- Browser **không nên** giữ secrets kiểu WooCommerce consumer secret.
- Vì vậy, “best practice” là:
  - Browser gọi custom WP endpoint (được bảo vệ bằng cookie auth + nonce)
  - Server (PHP) gọi WooCommerce REST API bằng credentials (hoặc gọi trực tiếp WooCommerce internal CRUD)

## 5) Debug: các lỗi thường gặp

Khi bị lỗi, phân lớp chẩn đoán:
- 401: auth không hợp lệ (headers sai, consumer key/secret sai, cookie auth thiếu nonce)
- 403: có auth nhưng không có quyền (capability / permission_callback)
- 404: route sai namespace/version/path
- 400/422: payload không hợp lệ (validate thất bại)

## 6) Gợi ý thực hành

1. Tạo REST endpoint custom `POST /wp-json/vocalmeet/v1/products` nhận `{ name, price }`.
2. Ở handler:
   - Validate + sanitize
   - Trả về JSON `{ product_id, name, price }`
3. Ở JS:
   - Submit form bằng fetch
   - Hiển thị lỗi theo status code
