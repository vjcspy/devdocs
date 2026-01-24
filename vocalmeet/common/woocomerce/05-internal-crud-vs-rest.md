# WooCommerce Internal CRUD (WC_Product) vs REST API

## 1) Hai cách tạo product

Trong WordPress plugin, bạn có thể tạo product theo 2 hướng:

### A) Gọi WooCommerce REST API
- Ưu:
  - Đúng “API integration” mindset
  - Giống production scenario khi tích hợp hệ thống ngoài
  - Contract rõ ràng (request/response)
- Nhược:
  - Phải quản lý auth + secret
  - Thêm overhead HTTP call (dù call nội bộ cùng server)

### B) Dùng WooCommerce internal CRUD (PHP)
- Ý tưởng:
  - Tạo `WC_Product_Simple`, set fields, rồi `save()`
- Ưu:
  - Không cần HTTP, nhanh và đơn giản trong cùng WP instance
  - Không cần giữ consumer secret để call REST
- Nhược:
  - Ít thể hiện “REST API understanding” nếu bài yêu cầu rõ
  - Coupling vào WooCommerce PHP API (version changes có thể ảnh hưởng)

## 2) Nên chọn cái nào cho assessment

Assessment có yêu cầu “Understand the WooCommerce REST API” và “Connect… using standard authentication methods”.
Vì vậy:
- Hướng REST API là phù hợp nhất để “demonstrate” đúng yêu cầu.

Tuy nhiên, hiểu internal CRUD vẫn quan trọng vì:
- Giúp bạn debug: REST API cuối cùng cũng map vào internal logic
- Giúp bạn chọn design tốt hơn trong dự án thực tế

## 3) Hybrid pattern (thường gặp trong plugin thực tế)

Một pattern thực dụng:
- Frontend/Elementor widget gọi custom WP endpoint
- Server-side endpoint dùng internal CRUD để tạo product (không lộ secret)
- Bạn vẫn có thể “demo REST API understanding” bằng cách:
  - có module wrapper cho REST (và dùng ở chỗ phù hợp), hoặc
  - giải thích rõ trade-offs trong review

Trong assessment này, nếu dùng hybrid, cần giải thích vì sao vẫn đáp ứng yêu cầu.
