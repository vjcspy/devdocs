# Data Modeling trong WordPress (CPT, Meta, Options)

## 1) WordPress lưu dữ liệu ở đâu (tổng quan)

WordPress core cung cấp các “data buckets” chính:
- Posts/Pages/Attachments… trong bảng `wp_posts`
- Taxonomies (categories/tags/custom taxonomy)
- Post meta trong `wp_postmeta`
- Options trong `wp_options`
- Users + user meta

WooCommerce sẽ mở rộng mô hình này, nhưng concept gốc vẫn giống:
- Product là một dạng “post-like entity” + meta
- Có thêm tables riêng cho hiệu năng/analytics tuỳ phiên bản

## 2) CPT (Custom Post Type): khi nào cần

CPT phù hợp khi bạn có “content type” riêng:
ví dụ “Project”, “Event”, “Job”.

Trong assessment, bạn không cần tạo CPT mới vì WooCommerce đã định nghĩa product.
Nhưng hiểu CPT giúp bạn:
- hiểu “product” là một content type được quản lý bởi WooCommerce
- đọc được cách WooCommerce dùng hooks/meta để cấu hình product

## 3) Meta: sức mạnh và rủi ro

Meta giúp bạn thêm fields tuỳ ý (key/value).
Nhưng nếu bạn lạm dụng:
- Query chậm (meta queries nặng)
- Data không có schema rõ ràng

Với assessment, meta xuất hiện khi:
- bạn muốn lưu `product_id` trong widget settings (Elementor)
- bạn cần gắn thêm thông tin vào product

## 4) Options: config của plugin

Options phù hợp để lưu:
- setting/plugin config
- feature flags

Trong assessment, bạn có thể cần lưu:
- WooCommerce consumer key/secret (nếu plugin dùng server-side call)

Nguyên tắc an toàn:
- Không commit secrets vào git.
- Tránh render secret ra UI.

## 5) Mapping cho assessment

Các data entities liên quan:
- WooCommerce product (entity chính)
- Elementor widget settings (lưu cấu hình widget + product_id)
- Plugin settings (nếu có) để lưu credentials cho local

Nếu bạn hiểu “data modeling” tốt, bạn sẽ quyết định đúng:
- cái gì lưu trong widget settings,
- cái gì lưu trong WP options,
- và cái gì nên được tính toán on-the-fly.
