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

## 4.1) Authentication model A: Browser → Custom WP endpoint (cookie auth + nonce)

### A1) Cookie auth là gì?

Khi user đăng nhập WordPress, browser sẽ giữ các auth cookies (vd `wordpress_logged_in_*`).
Nếu frontend JS gọi về cùng origin (vd `https://vocalmeet.local`) thì browser sẽ tự gửi cookies theo request.

Nhưng WordPress REST API cần thêm một lớp chống CSRF cho cookie-auth requests: **nonce**.

### A2) Nonce (REST) là gì, dùng để làm gì?

Nonce ở WordPress là một token ngắn hạn, dùng để chứng minh request:
- được phát sinh từ session hợp lệ (user đang login)
- xuất phát từ site của bạn (giảm rủi ro CSRF)

Với REST API, client gửi nonce qua header:
- `X-WP-Nonce: <nonce>`

WordPress core sẽ validate nonce cho cookie-auth REST requests trước khi vào callback của route.

### A3) Tạo nonce và “đưa” sang JS như thế nào?

Pattern thường dùng:
1) PHP enqueue script
2) Inject config (REST base URL + nonce) vào JS qua `wp_localize_script` (hoặc `wp_add_inline_script`)

Ví dụ:

```php
add_action('wp_enqueue_scripts', function () {
    wp_enqueue_script(
        'vocalmeet-frontend',
        plugins_url('assets/frontend.js', __FILE__),
        [],
        '1.0.0',
        true
    );

    wp_localize_script('vocalmeet-frontend', 'VOCALMEET_REST', [
        'restUrl' => esc_url_raw(rest_url('vocalmeet/v1')),
        'nonce' => wp_create_nonce('wp_rest'),
    ]);
});
```

JS gọi endpoint:

```js
await fetch(`${VOCALMEET_REST.restUrl}/products`, {
  method: 'POST',
  credentials: 'same-origin',
  headers: {
    'Content-Type': 'application/json',
    'X-WP-Nonce': VOCALMEET_REST.nonce,
  },
  body: JSON.stringify({ name: 'Test', price: '10' }),
});
```

### A4) Server-side authorize như thế nào?

Trong `register_rest_route`, bạn đặt `permission_callback` để enforce quyền:

```php
register_rest_route('vocalmeet/v1', '/products', [
    'methods' => WP_REST_Server::CREATABLE,
    'permission_callback' => function () {
        return current_user_can('manage_woocommerce');
    },
    'callback' => 'vocalmeet_handle_create_product',
]);
```

Điểm quan trọng:
- Cookie + nonce chỉ chứng minh “request hợp lệ” cho session, không thay thế authorization.
- `permission_callback` mới là “ai được làm gì”.

### A5) Khi nào nên dùng model này?

Dùng khi:
- UI chạy trong browser, user login bằng WordPress
- bạn muốn gọi endpoint custom của chính bạn (không lộ secrets ra frontend)

Không phù hợp khi:
- client là hệ thống bên ngoài (mobile app, server khác) không có WP cookies
- user chưa login và bạn muốn public writes (rất hiếm khi nên làm)

## 4.2) Authentication model B: PHP server → WooCommerce REST API (credentials)

### B1) Vì sao server phải gọi WooCommerce bằng credentials?

WooCommerce REST API là API “business-critical” (tạo order/product).
Nếu đưa credentials xuống browser, ai cũng có thể lấy và gọi API trực tiếp.

Nên pattern đúng là:
- Browser gọi custom WP endpoint (cookie + nonce)
- PHP handler (đã authorize) mới gọi Woo (bằng credentials)

### B2) Credentials cho WooCommerce REST API là gì? Tạo ở đâu?

WooCommerce hỗ trợ API keys theo cặp:
- Consumer Key
- Consumer Secret

Tạo trong admin:
- WooCommerce → Settings → Advanced → REST API → Add key
- Chọn user owner của key
- Permissions: chọn tối thiểu cần thiết (thường `Read/Write` cho create product)

### B3) Server gọi WooCommerce REST API như thế nào?

Nếu gọi Woo từ chính WordPress server, bạn có 2 lựa chọn:

1) Prefer: gọi WooCommerce internal APIs (CRUD) thay vì gọi HTTP loopback
2) Nếu buộc phải gọi REST API: dùng `wp_remote_request` và Basic Auth

Ví dụ (REST call):

```php
$consumer_key = 'ck_...';
$consumer_secret = 'cs_...';

$auth = base64_encode($consumer_key . ':' . $consumer_secret);

$response = wp_remote_post(
    rest_url('wc/v3/products'),
    [
        'headers' => [
            'Authorization' => 'Basic ' . $auth,
            'Content-Type' => 'application/json',
        ],
        'body' => wp_json_encode([
            'name' => 'Test product',
            'type' => 'simple',
            'regular_price' => '10',
        ]),
        'timeout' => 15,
    ]
);
```

### B4) Best practices

- Không bao giờ expose Woo consumer secret ra browser (HTML/JS/network).
- Dùng `permission_callback` để enforce capabilities (vd `manage_woocommerce`).
- Key permissions tối thiểu, rotate khi nghi ngờ lộ.
- Không log secrets/Authorization headers.
- Prefer gọi internal Woo APIs nếu code chạy trong WP (ít network, ít failure mode).
- Nếu environment của bạn không route được `/wp-json/...`, fallback `?rest_route=...` vẫn OK, nhưng production nên dùng pretty REST URLs.

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
