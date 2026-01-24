# WooCommerce Product Model (Nền tảng)

## 1) Product là gì trong WooCommerce

WooCommerce product là entity đại diện cho “thứ được bán”.  
Từ góc nhìn thực dụng (assessment):
- Bạn chỉ cần tạo **simple product** với:
  - `name`
  - `regular_price`

Nhưng để hiểu sâu, cần nắm:
- Product type (simple/variable/…)
- Giá (regular/sale), tồn kho, shipping, images, categories…
- Product lifecycle: draft → publish

## 2) Dữ liệu product được lưu thế nào

WooCommerce xây trên WordPress, nên product thường “post-like”:
- Có trạng thái, title, slug
- Có meta fields cho các thuộc tính WooCommerce

Hiểu điều này giúp bạn:
- Không nhầm lẫn “post” vs “product”
- Hiểu vì sao capability/permission có thể liên quan

## 3) Minimal payload để tạo simple product

Về concept, minimal product cần:
- tên (để định danh/hiển thị)
- giá (để tính toán)

Các fields khác có thể default.

Điểm sâu: “price là string”
- Trong REST API, `regular_price` thường là string (để tránh floating precision).
- Bạn vẫn validate trên server: có parse được số không, có >= 0 không.

## 4) Product ID và “display”

Khi tạo product thành công, bạn sẽ nhận:
- `id`

ID này là “key” cho:
- render lại thông tin product
- lưu vào Elementor widget settings
- gọi `GET /products/{id}` để hiển thị

## 5) Gợi ý thực hành

1. Tạo 3 products:
   - price hợp lệ
   - price = 0
   - price âm (kỳ vọng bị reject)
2. Quan sát response structure:
   - field nào luôn có, field nào optional
