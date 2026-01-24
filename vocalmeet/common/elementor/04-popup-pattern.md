# Popup/Modal Pattern (Đúng constraint “không raw form trong preview”)

## 1) Vì sao constraint này tồn tại

Trong Elementor editor:
- Preview area là nơi “kết quả render” được hiển thị
- Nếu bạn nhúng raw form trong preview, bạn đang biến preview thành nơi thao tác cấu hình/CRUD

Điều này gây:
- UX lẫn lộn (preview vừa là output vừa là input)
- Khó maintain (form logic chạy trong iframe)
- Rủi ro security (form submit endpoints bị expose theo cách khó kiểm soát)

## 2) Pattern phù hợp

Một pattern đáp ứng yêu cầu assessment:
1. Widget render trong preview: chỉ hiển thị button “Create product” (hoặc trạng thái)
2. Khi click, JS (editor context) mở modal/popup
3. Modal chứa 2 fields: name, price
4. Submit modal:
   - call custom WP endpoint (AJAX/REST)
   - nhận `product_id`
   - update widget settings
5. Preview re-render và hiển thị product

Điểm mấu chốt:
- Form “sống” trong modal do editor quản lý, không phải trong preview markup.

## 3) Các quyết định thiết kế (để hiểu sâu)

Bạn sẽ phải chọn:
- Modal UI framework: dùng dialog/modal utilities sẵn trong ecosystem WP/Elementor (nếu có) hay tự viết minimal.
- Nơi đặt DOM của modal: trong editor document (không nằm trong preview iframe).
- Cách update widget settings: thông qua API của Elementor editor (không hack DOM).

## 4) State machine tối thiểu cho widget

Tư duy theo state giúp code sạch:
- `idle` (chưa tạo product)
- `creating` (đang gọi API)
- `created` (có product_id)
- `error` (call fail)

Widget render dựa trên state/settings, không dựa trên “DOM đã click”.

## 5) Checklist đánh giá “đúng yêu cầu”

- Form fields không nằm trong preview DOM
- Không có consumer secret trong client JS
- Tạo product xảy ra trước khi “render stable” (product_id được persist trong widget settings)
