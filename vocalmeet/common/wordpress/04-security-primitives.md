# WordPress Security Primitives (Capabilities, Nonce, Sanitization)

## 1) Threat model tối thiểu cho assessment

Khi bạn cho phép “tạo product ngoài admin”, bạn đang mở một “write surface”.
Những rủi ro phổ biến:
- **CSRF**: attacker khiến user đã login thực hiện thao tác tạo product ngoài ý muốn.
- **Privilege escalation**: user không đủ quyền nhưng vẫn tạo được product.
- **XSS**: dữ liệu người dùng nhập bị render lại mà không escape.
- **Secret leakage**: lộ consumer secret (nếu gọi WooCommerce REST API trực tiếp từ browser).

## 2) Capabilities: ai được làm gì

WordPress quản lý quyền bằng:
- Roles (admin/editor/…)
- Capabilities (primitive permissions)

Rule of thumb:
- Authorization nên check ở server-side, dựa trên capability.
- UI chỉ “ẩn” nút không đủ; attacker vẫn có thể gọi endpoint trực tiếp.

Trong bài assessment, bạn cần quyết định:
- Cho phép tất cả user login tạo product? hay chỉ admin/editor?
- Nếu assessment không quy định, bạn vẫn nên hiểu vì sao check capability là chuẩn.

## 3) Nonce: dùng để chống CSRF (không phải auth)

Nonce trong WordPress:
- Là token ngắn hạn để chứng minh request “được tạo từ UI hợp lệ”.
- Chủ yếu dùng để chống **CSRF** khi dùng cookie-based auth.

Hiểu sâu:
- Nonce không thay thế authentication.
- Nonce không mã hoá payload; chỉ giúp server reject request “không có nonce hợp lệ”.

## 4) Validate vs Sanitize vs Escape

Đây là 3 khái niệm rất hay bị lẫn:
- **Validate**: dữ liệu có hợp lệ không? (ví dụ price phải là số >= 0)
- **Sanitize**: làm sạch dữ liệu để an toàn khi lưu/xử lý (ví dụ strip tags)
- **Escape**: encode đúng context khi output ra HTML/attribute/JS

Best practice:
- Validate + sanitize ở “entry point” (endpoint handler).
- Escape ở “output boundary” (khi render HTML).

## 5) “Không gọi WooCommerce REST API trực tiếp từ browser” vì sao?

WooCommerce REST API thường dùng consumer key/secret.
Nếu bạn đưa secret xuống JS:
- user có thể view source và lấy secret
- secret có thể bị leak qua network logs

Do đó, pattern tốt cho assessment:
- Browser gọi WP endpoint custom (cookie auth + nonce)
- Server-side gọi WooCommerce REST API bằng secret (hoặc dùng internal API)

## 6) Gợi ý thực hành

1. Viết 1 endpoint `POST /wp-json/vocalmeet/v1/products`.
2. Chỉ cho phép:
   - user đã login, và
   - có capability tạo product (tự chọn policy).
3. Thử gọi endpoint:
   - không nonce → thấy server reject
   - có nonce nhưng không đủ quyền → thấy 403
