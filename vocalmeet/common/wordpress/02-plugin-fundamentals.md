# Plugin Fundamentals (Hooks, Shortcode, Assets)

## 1) Plugin là gì trong WordPress

Plugin là package PHP được WordPress load để:
- Hook vào lifecycle (actions/filters)
- Register shortcode, REST routes, AJAX handlers
- Enqueue JS/CSS
- Tạo custom admin pages (nếu cần)

Trong assessment, plugin là “container” chính để bạn:
- Tạo page/form tạo product ngoài admin
- Tạo Elementor widget (cũng là plugin)

## 2) Plugin header & entrypoint

Một plugin tối thiểu cần:
- File PHP chính chứa plugin header (metadata)
- Code bootstrap để register hooks

Tư duy entrypoint:
- Giữ entrypoint mỏng: chỉ load class/functions cần thiết
- Không chạy logic nặng ngay khi file được include; dùng hooks để chạy “đúng thời điểm”

## 3) Shortcode vs Page template vs Block

Để “nhúng UI” vào một WordPress Page, có 3 cách phổ biến:
- **Shortcode**: đơn giản, phù hợp assessment.  
  Bạn viết `[vocalmeet_product_form]` và render HTML.
- **Block (Gutenberg)**: chuẩn mới, mạnh nhưng setup phức tạp hơn.
- **Page template** (theme): dễ bị phụ thuộc theme; không phù hợp khi muốn plugin độc lập.

## 4) Enqueue assets (JS/CSS) đúng cách

Nguyên tắc:
- Chỉ enqueue khi cần (đúng page, đúng shortcode, đúng editor)
- Tách editor assets và frontend assets nếu UI chạy ở Elementor editor

Concept quan trọng:
- WordPress có dependency management cho scripts (handle, deps, version).
- Bạn sẽ thường cần “bridge” dữ liệu từ PHP sang JS:
  - REST endpoint URL
  - nonce
  - settings

## 5) Activation/Deactivation hooks (khi nào cần)

Bạn dùng activation hook khi cần:
- Tạo options mặc định
- Tạo custom tables (assessment này thường không cần)

Nếu chỉ làm REST/AJAX và tạo product qua WooCommerce thì thường không cần activation logic phức tạp.

## 6) Gợi ý “mẫu” cấu trúc plugin dễ maintain

Không có một chuẩn duy nhất, nhưng nên có:
- `includes/` (PHP classes)
- `assets/` (js/css)
- `templates/` (php render templates, nếu muốn tách view)

Điểm quan trọng là “boundary”:
- File render template không chứa logic gọi API.
- Logic gọi API/validate/authorize nằm trong class handler.
