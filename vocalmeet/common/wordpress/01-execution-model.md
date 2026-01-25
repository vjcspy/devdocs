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

## 1.1) Mental model: request pipeline và “điểm chèn” (hook points)

Bạn nên hình dung WordPress như một pipeline:
1. Bootstrap (load config, core, plugin, theme)
2. Determine context (frontend/admin/rest/ajax)
3. Setup environment (current user, query vars, rewrite)
4. Dispatch handler (template loader hoặc REST route handler hoặc ajax action)
5. Render + output (theme template, wp_head/wp_footer, enqueue assets)

Hooks là “điểm chèn” vào pipeline đó, để plugin của bạn:
- chặn/điều hướng request
- đăng ký route, shortcode, widget
- enqueue assets đúng chỗ
- biến đổi output (filters)

## 2) Hooks: Action vs Filter

Tư duy đúng:

- **Action**: “đến thời điểm này, hãy làm gì đó” (side effects).
- **Filter**: “trước khi trả dữ liệu, hãy biến đổi nó” (pure-ish transformation).

Vì sao hooks là concept số 1:

- WordPress là hệ thống event-driven; plugin gần như luôn “gắn” vào lifecycle.
- Elementor và WooCommerce cũng dùng hooks (PHP side), nên hiểu hooks giúp bạn đọc code nhanh hơn.

## 2.1) Cơ chế hooks (để dùng đúng, debug nhanh)

### Action
- Đăng ký bằng `add_action($hook_name, $callback, $priority, $accepted_args)`
- Kích hoạt bằng `do_action($hook_name, ...$args)`

### Filter
- Đăng ký bằng `add_filter($hook_name, $callback, $priority, $accepted_args)`
- Kích hoạt bằng `apply_filters($hook_name, $value, ...$args)`

### Priority
- Số nhỏ chạy trước (priority mặc định là `10`)
- Dùng priority để “đi trước/đi sau” logic của plugin khác (Elementor/WooCommerce)

### Arguments / accepted_args
- Nếu hook truyền nhiều tham số, `accepted_args` phải đủ để nhận.

## 2.2) Code mẫu: action cơ bản + priority

```php
<?php

add_action('init', function () {
    update_option('vocalmeet_init_order', ['first']);
}, 5);

add_action('init', function () {
    $order = get_option('vocalmeet_init_order', []);
    $order[] = 'second';
    update_option('vocalmeet_init_order', $order);
}, 20);
```

## 2.3) Code mẫu: filter biến đổi output

```php
<?php

add_filter('the_content', function ($content) {
    if (!is_singular()) {
        return $content;
    }
    return $content . '<p>Rendered by vocalmeet.</p>';
}, 10, 1);
```

## 2.4) Debug hook timeline (mẫu “trace” thứ tự hook)

```php
<?php

add_action('plugins_loaded', function () { error_log('vocalmeet: plugins_loaded'); });
add_action('init', function () { error_log('vocalmeet: init'); });
add_action('wp_loaded', function () { error_log('vocalmeet: wp_loaded'); });
add_action('wp', function () { error_log('vocalmeet: wp'); });
add_action('template_redirect', function () { error_log('vocalmeet: template_redirect'); });
add_action('wp_head', function () { error_log('vocalmeet: wp_head'); });
add_action('wp_footer', function () { error_log('vocalmeet: wp_footer'); });
```

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

## 3.1) Hook timeline thường gặp cho frontend request

Đây là chuỗi hook bạn sẽ gặp thường xuyên nhất khi user xem một trang bất kỳ:
- `muplugins_loaded` → `plugins_loaded`
- `setup_theme` → `after_setup_theme`
- `init`
- `wp_loaded`
- `wp`
- `template_redirect`
- `wp_head`
- `wp_footer`

Ý nghĩa thực dụng (gắn với assessment):
- `plugins_loaded`: kiểm tra plugin dependency (Elementor/WooCommerce active) và load bootstrap.
- `init`: đăng ký shortcode, custom post types, rewrite rules, hoặc chuẩn bị route vars.
- `wp_loaded`: WP đã “sẵn sàng”, thường dùng để xử lý logic phụ thuộc query vars.
- `template_redirect`: hook tốt để “intercept” trước khi template render (redirect, guard, set headers).
- `wp_head`/`wp_footer`: nơi theme output, thường dùng để enqueue assets và inject small markup (nhưng tránh làm logic nặng).

## 3.2) Admin lifecycle (để biết khi nào enqueue admin/editor assets)

Khi vào `/wp-admin/...`:
- `admin_init`: xử lý logic admin-level (settings, capability checks)
- `admin_menu`: đăng ký menu pages
- `admin_enqueue_scripts`: enqueue assets cho admin screens

Trong assessment, Elementor editor chạy trong admin, nên bạn hay phải:
- enqueue “editor scripts” trong đúng screen/context

## 3.3) REST API lifecycle (custom route)

REST API requests đi qua:
- `rest_api_init`: nơi bạn register REST routes
- `permission_callback`: nơi enforce authorization
- callback handler: validate/sanitize và trả response

Code mẫu: đăng ký REST route tối thiểu

```php
<?php

add_action('rest_api_init', function () {
    register_rest_route('vocalmeet/v1', '/ping', [
        'methods' => 'GET',
        'permission_callback' => '__return_true',
        'callback' => function () {
            return rest_ensure_response(['ok' => true]);
        },
    ]);
});
```

## 3.4) AJAX lifecycle (admin-ajax.php)

AJAX requests đi qua `admin-ajax.php` và dispatch theo `action=...`:
- `wp_ajax_{action}`: chỉ cho user đã login
- `wp_ajax_nopriv_{action}`: cho user chưa login

Code mẫu: đăng ký AJAX action tối thiểu

```php
<?php

add_action('wp_ajax_vocalmeet_ping', function () {
    wp_send_json_success(['ok' => true]);
});
```

## 4) Template hierarchy (biết để không bị “mù đường”)

Template hierarchy là cơ chế WP chọn file theme để render.
Trong assessment, bạn được phép dùng theme’s `functions.php`, nhưng vì constraint “must use plugin”, hướng đi an toàn là:

- Render UI qua shortcode hoặc block (plugin cung cấp)
- Tránh phụ thuộc theme để dễ review, dễ migrate

## 4.1) Tại sao `template_redirect` hay được dùng trong plugin

`template_redirect` chạy sau khi WP đã quyết định loại request, trước khi theme template output.
Do đó nó phù hợp để:
- redirect nếu user không đủ quyền
- handle download/response đặc biệt
- set headers (cẩn thận)

Code mẫu: chặn truy cập một page theo slug nếu chưa login

```php
<?php

add_action('template_redirect', function () {
    if (is_page('create-product') && !is_user_logged_in()) {
        wp_safe_redirect(wp_login_url(get_permalink()));
        exit;
    }
});
```

## 5) Gợi ý thực hành

Bạn tự làm 2 bài nhỏ để kiểm chứng hiểu biết:

1. Viết plugin tối thiểu add một shortcode `[hello_world]` và enqueue 1 file JS chỉ khi shortcode xuất hiện.
2. Tạo 1 endpoint REST custom `GET /wp-json/vocalmeet/v1/ping` trả về JSON `{ ok: true }`.

## 5.1) Bài tập “hiểu hooks sâu” (khuyến nghị)

1. Tạo 1 plugin có:
   - 2 callbacks cùng hook `init` nhưng khác priority
   - 1 filter `the_content`
2. Dùng `error_log` để trace xem callback chạy khi nào.
3. Tự trả lời:
   - Vì sao `plugins_loaded` khác `init`?
   - Vì sao enqueue assets đúng hook lại quan trọng (nhất là Elementor editor)?
