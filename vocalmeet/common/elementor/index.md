# Elementor Learning Path (Custom Widget + WooCommerce Integration)

## Mục tiêu

Nắm vững Elementor theo hướng “editor platform” để:
- Tạo custom drag-and-drop widget đúng chuẩn (không sửa Elementor core)
- Hiểu rõ widget lifecycle: controls (panel) vs render (preview)
- Thiết kế UX flow: tạo product qua popup/modal, tránh raw form trong preview

## Làm việc với Elementor trên UI (quickstart)

### Mở editor đúng cách

1. Vào WP Admin → Pages (hoặc Posts/Templates) → chọn item cần chỉnh.
2. Bấm “Edit with Elementor”.
3. Nhìn nhanh 3 vùng chính:
   - Panel trái: tìm widget + chỉnh controls (Content / Style / Advanced).
   - Canvas giữa: preview realtime của trang.
   - Thanh dưới: Navigator, History, Responsive mode, Preview changes, Publish/Update.

### Mental model UI: “panel controls” vs “preview render”

- Thao tác trong panel (vd: đổi text, màu, spacing) chủ yếu cập nhật state/setting của widget.
- Canvas chỉ là preview render; đừng coi đây là “một form ứng dụng đầy đủ” để nhập dữ liệu phức tạp.
- Khi làm custom widget: bạn thường debug theo 2 hướng:
  - Controls không hiện/không lưu: vấn đề ở register_controls (PHP) hoặc context editor.
  - Preview không update/khác frontend: vấn đề ở render (PHP) + assets + điều kiện context.

### Workflow cơ bản trong editor

1. Tạo layout:
   - Elementor mới: ưu tiên Container (Flexbox). Nếu site đang dùng Section/Column legacy thì giữ đúng pattern hiện hữu.
2. Thêm widget:
   - Tìm widget ở panel (search) → kéo thả vào container/column.
3. Chỉnh Content:
   - Text, image, link, dynamic tags (nếu có).
4. Chỉnh Style:
   - Typography, color, background, border, spacing.
5. Chỉnh Advanced:
   - Margin/padding, motion effects, responsive visibility, custom CSS (nếu bật).
6. Dùng Navigator để quản lý cây layout:
   - Rename element để đỡ rối khi page lớn.
7. Kiểm tra responsive:
   - Chuyển Mobile/Tablet/Desktop → rà lại spacing và breakpoints.
8. Preview changes:
   - Xem như frontend thực (nếu có khác biệt so với canvas).
9. Publish/Update:
   - Chỉ publish sau khi đã kiểm tra responsive + preview.

### Các công cụ UI hay dùng (khi page bắt đầu phức tạp)

- Navigator: “outline tree” để chọn đúng element nhanh, rename, kéo thả vị trí.
- History: undo/redo theo action; hữu ích khi thử nghiệm layout.
- Finder (Ctrl/Cmd + E): tìm nhanh page/template/popup trong Elementor.
- Copy/Paste Style: copy style giữa các widget để giữ consistency.
- Save as Template / Global Widget: tái sử dụng component (tùy version/plugin).

## Ví dụ minh họa (đi theo thao tác UI)

### Ví dụ 1: Tạo hero đơn giản (heading + text + button)

1. Tạo Page mới → “Edit with Elementor”.
2. Add Container (hoặc Section) → set width và padding.
3. Kéo Heading vào → đặt title.
4. Kéo Text Editor vào → viết mô tả ngắn.
5. Kéo Button vào → set label + link.
6. Style nhanh:
   - Heading: typography + color.
   - Button: background + hover.
   - Container: background + padding.
7. Responsive:
   - Mobile: giảm padding, giảm font size, căn giữa nếu cần.
8. Preview changes → Update.

### Ví dụ 2: Test một custom widget trong editor (flow thực tế khi develop)

Mục tiêu: đảm bảo widget “kéo thả được”, controls hiện đúng, và preview thay đổi theo controls.

1. Mở page bất kỳ bằng Elementor.
2. Search widget theo tên (hoặc theo category bạn đăng ký).
3. Kéo widget vào canvas.
4. Trong tab Content:
   - Thử đổi 2–3 controls quan trọng (text, select, switcher…).
   - Quan sát preview cập nhật.
5. Trong tab Advanced:
   - Set margin/padding khác nhau để chắc widget wrapper hoạt động.
6. Bấm Preview changes để so sánh canvas vs frontend preview.

Checklist nhanh khi có lỗi:
- Widget không thấy trong panel: plugin chưa load đúng, register widget chưa chạy, hoặc điều kiện context sai.
- Control không hiện: lỗi register_controls hoặc condition.
- Control đổi nhưng preview không đổi: render không dùng settings, hoặc JS preview hook chưa gắn đúng (nếu bạn rely vào JS).

### Ví dụ 3: Popup/modal pattern (Elementor Pro)

Mục tiêu: không nhét raw form CRUD trong preview; thay bằng popup/modal để làm “flow”.

1. WP Admin → Templates → Popups → Add New.
2. Thiết kế popup content (form UI, CTA, step wizard… tùy case).
3. Set Display Conditions + Triggers (on click, on page load…).
4. Trong page chính: dùng Button/Link để trigger popup.
5. Preview changes để chắc trigger hoạt động và popup responsive.

## Thứ tự học

1. Elementor architecture & mental model  
   - `01-architecture.md`
2. Widget fundamentals (PHP side)  
   - `02-widget-fundamentals.md`
3. Editor scripts & runtime contexts (JS side)  
   - `03-editor-scripts-and-context.md`
4. Popup/modal pattern (không đặt raw form trong preview)  
   - `04-popup-pattern.md`
5. Integration strategies với WooCommerce  
   - `05-woocommerce-integration-strategies.md`
