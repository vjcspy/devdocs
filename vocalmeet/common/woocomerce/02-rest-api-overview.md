# WooCommerce REST API Overview

## 1) REST API trong WooCommerce dùng để làm gì

WooCommerce REST API là cách chuẩn để:
- Tạo/đọc/cập nhật/xoá resources (products, orders, customers…)
- Tích hợp hệ thống ngoài (ERP, CRM, mobile app)

Với assessment, bạn tập trung vào:
- Products endpoint
- Authentication
- Error handling

## 2) Base URL và versioning

WooCommerce REST API (v3) thường có dạng:
- `/wp-json/wc/v3/...`

Tư duy versioning:
- Version là contract; khi đổi version có thể đổi payload/behavior.
- Khi build plugin, bạn nên “pin” version để predictable.

## 3) Endpoint quan trọng cho product (conceptual)

Các thao tác cơ bản:
- Create: `POST /products`
- Read list: `GET /products`
- Read single: `GET /products/{id}`
- Update: `PUT /products/{id}`
- Delete: `DELETE /products/{id}`

## 4) Request/Response shape

Những thứ bạn luôn nên có trong tư duy khi gọi REST:
- Request headers (auth, content-type)
- Payload schema (name, regular_price, type…)
- Response schema (id, name, status, price…)
- Error schema (message, code, data/status)

## 5) Pagination, filtering, and idempotency (hiểu sâu hơn)

Ngay cả khi assessment không yêu cầu:
- Pagination là phần “thực tế” của API list.
- Filtering giúp bạn fetch đúng data.
- Idempotency quan trọng khi retry request (network flaky).

Ví dụ “tư duy idempotency” cho create:
- Nếu client retry do timeout, có thể tạo duplicate product.
- Trong production, thường cần request key hoặc check logic chống duplicate.

## 6) Debug mindset

Khi gặp lỗi, tách bạch:
- Lỗi transport (HTTPS, cert, DNS)
- Lỗi auth (401)
- Lỗi permission (403)
- Lỗi validation/business (400/422)
