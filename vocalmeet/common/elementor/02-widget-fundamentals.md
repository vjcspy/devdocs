# Widget Fundamentals (PHP Side)

## 1) Elementor widget là một class PHP

Về bản chất, bạn viết một class kế thừa base widget class của Elementor và đăng ký nó qua plugin của bạn.

Một widget thường định nghĩa:
- Identity: `get_name()`, `get_title()`, `get_icon()`, `get_categories()`
- Controls: `register_controls()`
- Output: `render()` (và/hoặc template method khác tuỳ version)

## 2) Controls: nơi đặt “inputs” cho user

Controls là UI trong panel bên trái.
Trong assessment, controls có thể chứa:
- product_id (hidden/readonly)
- nút “Create product” (hoặc một control custom trigger)
- thông tin trạng thái (created/failed)

Điểm quan trọng:
- Controls nên là nơi user “cấu hình” widget, chứ không phải preview.
- Nếu cần UX phức tạp (popup), bạn vẫn có thể trigger từ editor script dựa trên control.

## 3) Render: nơi chỉ hiển thị kết quả

Preview area (bên phải) nên:
- hiển thị product (nếu đã tạo)
- hiển thị placeholder + hướng dẫn (nếu chưa tạo)

Nếu bạn nhét raw form vào preview:
- bạn đang trộn UI creation (editor) với UI display (preview)
- dễ vi phạm constraint của assessment

## 4) Widget settings là contract giữa editor và render

Để “render được product đã tạo”, widget cần dữ liệu bền:
- `product_id`

Flow đúng:
- Editor JS tạo product → nhận `product_id` → set vào widget settings → trigger re-render
- Render (PHP) đọc settings → fetch product info → render HTML

## 5) Plugin packaging cho widget

Widget không đứng một mình; nó sống trong plugin:
- Plugin bootstrap: đảm bảo Elementor active, version compatible
- Hook để register widget
- Enqueue editor scripts/styles

Gợi ý: tham khảo structure “hello world” plugin của Elementor để hiểu bootstrap pattern.

## 6) Gợi ý thực hành

Bạn tự làm 2 bước:
1. Tạo widget “Hello” render static text.
2. Thêm 1 control (text) và render text đó trong preview.
Khi làm được 2 bước này, bạn đã nắm 80% cơ chế widget.
