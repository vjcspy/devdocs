# Learning Path (Assessment-Focused): WordPress / WooCommerce / Elementor

## Mục tiêu

Learning path này được thiết kế để bạn:
- Nắm **concept nền tảng** (không chỉ “làm cho xong bài”)
- Hiểu rõ **ranh giới** giữa WordPress Core / Plugin / Theme và lý do nên chọn từng cách
- Có đủ kiến thức để làm đúng scope assessment:
  - Tạo page cho user tạo WooCommerce product (không qua admin)
  - Tạo custom Elementor widget tích hợp WooCommerce, tránh đưa raw form vào preview

Assessment overview (source of truth): `devdocs/vocalmeet/assessment/OVERVIEW.md`

## Cách học (khuyến nghị)

Nguyên tắc:
- Học theo “**data flow**”: UI → permission → validation → API call → persistence → render
- Luôn phân biệt rõ:
  - Editor/admin context vs frontend context
  - Client-side JS vs server-side PHP
  - WooCommerce REST API vs WooCommerce internal CRUD (WC_Product)

Nhịp học gợi ý:
- 60% concept + đọc docs/source code
- 40% thực hành: tạo plugin nhỏ, thử endpoint, xem request/response

## Thứ tự học

### Phase A — WordPress nền tảng (bắt buộc)
1. WordPress execution model và request lifecycle  
2. Plugin fundamentals (hooks, shortcodes, enqueue assets)  
3. Security primitives (capabilities, nonces, sanitization/escaping)  
4. REST/AJAX patterns trong WordPress

➡️ Xem: `devdocs/vocalmeet/common/wordpress/index.md`

### Phase B — WooCommerce (đúng trọng tâm assessment)
1. Product data model (simple product tối thiểu)
2. WooCommerce REST API: endpoints + payload chuẩn
3. Authentication models (local vs production, HTTPS implications)
4. “Create product” flow và error handling

➡️ Xem: `devdocs/vocalmeet/common/woocomerce/index.md`

### Phase C — Elementor widget development + integration
1. Elementor architecture: widget lifecycle, controls vs render
2. Editor scripts: cách chạy JS trong Elementor editor đúng chỗ
3. Pattern popup/modal (không nhét raw form vào preview)
4. Tích hợp WooCommerce từ widget (server proxy vs direct call)

➡️ Xem: `devdocs/vocalmeet/common/elementor/index.md`

## Checklist “đã hiểu sâu”

Bạn coi như nắm vững khi có thể tự trả lời (và giải thích lý do) các câu hỏi:
- Tại sao không nên gọi WooCommerce REST API trực tiếp từ browser với consumer secret?
- Nonce trong WordPress dùng để chống loại tấn công nào? Nonce có thay cho auth không?
- Elementor “controls” chạy ở đâu, “render” chạy ở đâu, và vì sao constraint “không đặt raw form trong preview” lại quan trọng?
- Khi tạo product: validate/sanitize ở lớp nào để vừa an toàn vừa dễ debug?
- Khi bị lỗi 401/403 từ REST API: chẩn đoán theo tầng nào (HTTPS, auth, capability, nonce, route)?
