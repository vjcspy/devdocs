# Editor Scripts & Runtime Context (JS Side)

## 1) Vì sao JS là “phần khó”

Phần PHP giúp bạn định nghĩa widget và render output.
Nhưng thao tác “create product khi đang build page” là workflow trong editor, nên thường cần JS để:
- bắt event click từ button/control
- mở popup/modal
- gọi endpoint tạo product
- cập nhật widget settings để render lại

## 2) Ba nơi JS có thể chạy

Khi debug, bạn luôn hỏi: “script này đang chạy ở đâu?”
- WordPress admin (global)
- Elementor editor panel (controls UI)
- Preview iframe (rendered page)

Trong assessment, mục tiêu là:
- chạy UI tạo product ở editor context
- preview chỉ render output

## 3) Enqueue “editor-only”

Best practice:
- Chỉ load editor script khi đang ở Elementor editor.
- Tránh load editor script ở frontend để giảm rủi ro và tránh side effects.

## 4) Event-driven mindset

Elementor editor là hệ thống event-driven:
- Widget được add vào page → event
- Settings đổi → event
- Widget re-render → event

Bạn tận dụng cơ chế này để:
- mở modal khi user bấm “Create”
- sau khi tạo product thành công, set `product_id` vào settings
- trigger refresh để preview render product mới

## 5) Data flow “không lộ secret”

Luồng data an toàn:
- JS gọi custom WP endpoint (cùng origin) + nonce
- Endpoint server-side gọi WooCommerce REST API
- Response trả về `product_id` + fields cần hiển thị

Tránh:
- JS gọi trực tiếp `/wc/v3/...` với consumer secret

## 6) Gợi ý thực hành

1. Trong editor, bấm button → console log “clicked”.
2. Bước tiếp: thay console log bằng gọi `fetch` tới endpoint `ping`.
3. Bước cuối: gọi endpoint create product và set `product_id` vào settings.

## 7) Tip: JS Hook quan trọng

Để chạy code khi widget của bạn được render trong preview (hoặc khi editor load), hook quan trọng nhất là:

```javascript
jQuery(window).on('elementor/frontend/init', function() {
   elementorFrontend.hooks.addAction('frontend/element_ready/vocalmeet-product-creator.default', function($scope) {
       // $scope là container của widget
       // Code logic của bạn ở đây (bind click event, open popup, etc.)
   });
});
```

Lưu ý: Hook này chạy cả ở Editor preview và Frontend thực tế.
