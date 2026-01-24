# Elementor Architecture (Mental Model)

## 1) Elementor là “website builder” chạy trong WordPress

Elementor hoạt động như một ứng dụng UI trong WP admin:
- Panel bên trái: widget list + controls
- Preview bên phải: khu vực render page (thường là iframe)

Điểm quan trọng:
- Editor là một “runtime riêng” khác với frontend bình thường.
- Khi viết widget, bạn phải hiểu nơi nào chạy PHP, nơi nào chạy JS.

## 2) Widgets: “schema + state + render”

Một widget thường có 3 phần:
1. **Definition (PHP)**: metadata (name, title, icon, categories)
2. **Controls (editor UI)**: fields để user cấu hình (ví dụ product_id)
3. **Render (preview/frontend)**: HTML hiển thị dựa trên settings

Trong assessment, constraint “không đặt raw code/form trong preview” ngầm yêu cầu:
- Controls và/hoặc modal UI phải xử lý “create product”
- Preview chỉ hiển thị kết quả (product đã tạo), không nhúng form thô

## 3) Backbone.js và “editor state”

Elementor historically dựa trên Backbone (models/views/events) cho editor state.
Bạn không nhất thiết phải viết Backbone từ đầu, nhưng bạn nên hiểu:
- Có “model” đại diện settings của widget
- Có event khi settings thay đổi, khi widget được add/remove
- Có cơ chế enqueue “editor scripts” để can thiệp vào editor UX

Tư duy thực dụng:
- Bạn tận dụng API/hook mà Elementor cung cấp thay vì tự dựng framework.

## 4) Editor context vs frontend context

Bạn thường có 2 loại script:
- **Editor script**: chạy trong Elementor editor để:
  - mở modal/popup
  - gọi endpoint tạo product
  - update widget settings (ví dụ lưu product_id)
- **Frontend script** (optional): chạy khi user xem site bình thường

Trong assessment, trọng tâm là editor script vì thao tác “create product” diễn ra khi build page.

## 5) Workflow save/persist của Elementor (concept)

Khi user chỉnh controls:
- settings được lưu vào cấu hình page (Elementor data)
- Khi publish/update, dữ liệu được persist vào WP (thường post meta)

Điểm sâu:
- Widget settings là “source of truth” để render.
- Nếu bạn tạo product, bạn nên lưu `product_id` vào widget settings để render lại ổn định.
