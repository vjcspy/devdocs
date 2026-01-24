# WooCommerce REST API Authentication (Hiểu cho đúng)

## 1) “Recommended auth methods” nghĩa là gì

WooCommerce REST API dùng “API keys” (Consumer Key / Consumer Secret).  
Từ đó có vài cách gửi credentials, tuỳ môi trường:
- **HTTPS**: thường dùng **HTTP Basic Auth** với `consumer_key:consumer_secret`.
- **Không có HTTPS**: có thể dùng **OAuth 1.0a signature** (phức tạp hơn).

Trong local assessment, bạn có thể dùng self-signed certificate để đảm bảo HTTPS.

## 2) Basic Auth trên HTTPS: vì sao “OK”

Basic Auth chỉ base64 encode, không mã hoá.
Nó chỉ an toàn khi:
- đường truyền được mã hoá bởi HTTPS

Do đó, “HTTPS requirement” không phải hình thức, mà là điều kiện để bảo vệ secret.

## 3) Không nên gửi consumer secret từ browser

Nếu bạn gọi WooCommerce REST API trực tiếp từ JS chạy trên browser, bạn sẽ phải đưa secret xuống client.
Đây là anti-pattern vì:
- secret bị lộ ngay lập tức (devtools/network)
- người dùng có thể dùng secret để thao tác store ngoài ý muốn

Pattern an toàn (và phù hợp plugin architecture):
- Browser → gọi custom WP endpoint (cookie auth + nonce)
- WP server → gọi WooCommerce REST API (giữ secret ở server)

## 4) Where to store credentials (concept)

Các lựa chọn thường gặp:
- WP options (plugin settings) lưu consumer key/secret
- Environment variables (đẹp hơn, nhưng trong WP local không phải lúc nào cũng tiện)

Nguyên tắc:
- Không commit secret vào git.
- Không render secret ra HTML/JS.

## 5) Debug auth issues

Nếu gặp 401/403, checklist:
- Endpoint đúng `/wp-json/wc/v3/...`?
- Basic auth header đúng format?
- Key/secret tạo đúng trong WooCommerce settings?
- Key có permissions đủ (read/write)?
- HTTPS có đang dùng? Self-signed có bị client reject không?

## 6) Gợi ý thực hành

1. Tạo key/secret trong WooCommerce → Settings → Advanced → REST API.
2. Thử gọi create product từ server-side PHP trước (để tránh CORS/secret).
3. Sau đó mới nối từ frontend qua custom endpoint.

## 7) Tip: Local Development & Self-Signed Certs

Khi chạy local với self-signed SSL, các request từ PHP (`wp_remote_post` hoặc WooCommerce Client Library) thường sẽ fail do lỗi verify SSL.

Để fix nhanh cho môi trường dev, bạn cần disable SSL verification:

```php
// Thêm vào plugin main file (chỉ cho local dev!)
add_filter('https_ssl_verify', '__return_false');
```

Nếu dùng thư viện `automattic/woocommerce` client:
```php
$woocommerce = new Client(
    $url, $key, $secret,
    [
        'version' => 'wc/v3',
        'verify_ssl' => false // Quan trọng cho local
    ]
);
```
