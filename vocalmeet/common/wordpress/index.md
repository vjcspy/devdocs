# WordPress Learning Path (Nền tảng)

## Mục tiêu

Nắm vững các concept WordPress cần thiết để:
- Viết plugin đúng chuẩn (không sửa core, không phụ thuộc plugin khác)
- Tạo page/frontend form an toàn để user tạo dữ liệu (product)
- Hiểu đúng REST/AJAX trong WordPress

## Thứ tự học

1. Execution model & request lifecycle  
   - `01-execution-model.md`
2. Plugin fundamentals (hooks, shortcodes, enqueue)  
   - `02-plugin-fundamentals.md`
3. REST API & AJAX patterns  
   - `03-rest-and-ajax.md`
4. Security primitives (capabilities, nonces, sanitization/escaping)  
   - `04-security-primitives.md`
5. Data layer & custom data modeling (CPT/meta)  
   - `05-data-modeling.md`

## Kết quả mong đợi

Sau khi học xong, bạn có thể:
- Tạo một plugin “đúng kiểu WordPress”: activation hooks, assets, shortcode, AJAX/REST endpoint
- Thiết kế flow submit form: validate → authorize → persist → respond JSON → render UI
- Giải thích rõ ràng khác nhau giữa sanitize/escape/validate và làm đúng chỗ
