# WooCommerce Integration Strategies trong Elementor Widget

## 1) Vấn đề cốt lõi cần giải

Widget cần tạo product “ngay trong editor workflow”, nhưng:
- WooCommerce REST API cần credentials (consumer secret)
- Client-side JS không nên giữ secret
- Editor có preview iframe, dễ khiến form logic chạy sai context

Do đó, integration strategy là phần quan trọng nhất.

## 2) Strategy A — Client → WooCommerce REST API trực tiếp

Không khuyến nghị vì:
- bạn phải đưa consumer secret xuống client
- rủi ro leak và bị lạm dụng

Chỉ hợp lý khi:
- bạn có auth model khác (ví dụ OAuth/short-lived token) và backend kiểm soát, nhưng assessment không yêu cầu.

## 3) Strategy B — Client → Custom WP endpoint → WooCommerce REST API

Đây là strategy “đẹp” cho assessment:
- Client gọi endpoint của plugin bạn (same-origin)
- Endpoint được bảo vệ bằng:
  - cookie auth (user login)
  - nonce (CSRF protection)
  - capability check (authorization)
- Server-side dùng consumer key/secret để gọi WooCommerce REST API

Ưu điểm:
- Demonstrate “WooCommerce REST API understanding”
- Không leak secret
- Dễ debug (log/trace ở server)

## 4) Strategy C — Client → Custom WP endpoint → WooCommerce internal CRUD

Strategy này rất thực dụng trong production plugin:
- Không cần call REST nội bộ
- Không cần store consumer secret chỉ để call local REST

Nhưng với assessment, nếu chọn strategy này bạn cần:
- giải thích rõ vì sao vẫn đáp ứng yêu cầu “REST API understanding” (ví dụ bạn vẫn implement REST wrapper và biết auth, nhưng chọn internal để tránh secret management trong local).

## 5) HTTPS và self-signed cert

Vì consumer secret cần bảo vệ bởi HTTPS:
- Local environment nên chạy HTTPS (self-signed OK)
- Client-side fetch có thể gặp warning/cert errors nếu environment cấu hình sai

Với strategy B/C (client gọi custom endpoint), bạn vẫn nên chạy HTTPS để nhất quán với requirement.

## 6) Lựa chọn nên dùng cho assessment

Nếu mục tiêu là “đúng rubric”:
- Ưu tiên Strategy B cho phần “create product” từ widget
- Lưu `product_id` trong widget settings để render ổn định
