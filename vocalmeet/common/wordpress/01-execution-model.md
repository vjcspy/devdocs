# WordPress Execution Model & Request Lifecycle

## 1) WordPress “chạy” như thế nào

Khi một request HTTP tới WordPress, WP sẽ:
1. Load `wp-config.php` (config), kết nối DB, load core.
2. Load plugins (mu-plugins trước, sau đó plugins thường).
3. Load theme.
4. Chạy hệ thống hooks (actions/filters) theo từng giai đoạn.
5. Resolve route:
   - Frontend request (template hierarchy)
   - Admin request (`/wp-admin/...`)
   - REST API request (`/wp-json/...`)
   - AJAX request (`/wp-admin/admin-ajax.php`)

Điểm quan trọng cho assessment:
- Bạn không sửa core, nên mọi thay đổi đều đi qua **plugin + hooks**.
- “Form ngoài frontend” thường được xử lý qua:
  - WordPress REST API endpoint custom, hoặc
  - AJAX endpoint, hoặc
  - Submit POST về 1 page và xử lý server-side (ít dùng hơn cho UX).

## 2) Hooks: Action vs Filter

Tư duy đúng:
- **Action**: “đến thời điểm này, hãy làm gì đó” (side effects).
- **Filter**: “trước khi trả dữ liệu, hãy biến đổi nó” (pure-ish transformation).

Vì sao hooks là concept số 1:
- WordPress là hệ thống event-driven; plugin gần như luôn “gắn” vào lifecycle.
- Elementor và WooCommerce cũng dùng hooks (PHP side), nên hiểu hooks giúp bạn đọc code nhanh hơn.

## 3) Context: Admin vs Frontend vs Editor

Với assessment, bạn sẽ làm việc ít nhất 3 “ngữ cảnh”:
- **Frontend**: user xem site bình thường.
- **Admin**: dashboard, settings, REST API keys.
- **Elementor Editor**: một “ứng dụng web” chạy trong WP admin, có preview iframe và panel controls.

Hệ quả thiết kế:
- Code server-side (PHP) phải kiểm tra đúng context để enqueue đúng assets.
- Code client-side (JS) phải biết mình đang chạy ở:
  - editor panel,
  - preview iframe,
  - hay frontend.

## 4) Template hierarchy (biết để không bị “mù đường”)

Template hierarchy là cơ chế WP chọn file theme để render.
Trong assessment, bạn được phép dùng theme’s `functions.php`, nhưng vì constraint “must use plugin”, hướng đi an toàn là:
- Render UI qua shortcode hoặc block (plugin cung cấp)
- Tránh phụ thuộc theme để dễ review, dễ migrate

## 5) Gợi ý thực hành

Bạn tự làm 2 bài nhỏ để kiểm chứng hiểu biết:
1. Viết plugin tối thiểu add một shortcode `[hello_world]` và enqueue 1 file JS chỉ khi shortcode xuất hiện.
2. Tạo 1 endpoint REST custom `GET /wp-json/vocalmeet/v1/ping` trả về JSON `{ ok: true }`.
