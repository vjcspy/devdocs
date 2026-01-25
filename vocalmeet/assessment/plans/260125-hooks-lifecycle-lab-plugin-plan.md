# 📋 [ASSESSMENT: 2026-01-25] - Hooks & Lifecycle Lab Plugin (5.1)

## References

- `vocalmeet/assessment/wordpress`
- `devdocs/vocalmeet/assessment/OVERVIEW.md`
- `devdocs/vocalmeet/common/wordpress/01-execution-model.md`

## User Requirements

- Tôi sẽ cần tạo 1 new plugin để làm 5.1 đi
- source wordpress ở đây `vocalmeet/assessment/wordpress`
- trước tiên, tạo file plan chi tiết đã
- nhớ follow `/Users/kai/work/k/vocalmeet/AGENTS.md`

## 🎯 Objective

Tạo một plugin “lab” tối giản để học sâu WordPress lifecycle + hooks bằng cách:
- Trace timeline hooks theo từng request type (frontend/admin/rest/ajax)
- Demonstrate thứ tự chạy theo `priority`
- Demonstrate filter biến đổi output đúng boundary
- Có “tín hiệu quan sát được” (observable) để verify: option/state + log + UI output

### ⚠️ Key Considerations

- Plugin chỉ dùng WordPress core APIs, không phụ thuộc thêm plugin khác.
- Không để lộ secrets, không hardcode credentials.
- Không tạo side effects ngoài ý muốn: chỉ log/ghi option khi cần, có cơ chế reset.
- Giữ code đơn giản, tập trung vào “thấy được lifecycle chạy ra sao”.

## 🔄 Implementation Plan

### Phase 1: Analysis & Preparation

- [x] Confirm target plugin location in source tree
  - **Outcome**: Plugin nằm dưới `vocalmeet/assessment/wordpress/wp-content/plugins/`.
- [x] Define “observable outputs” cho từng concept (priority, filter, lifecycle)
  - **Outcome**:
    - Option `vocalmeet_hooks_lab_init_order` lưu thứ tự callback chạy
    - `error_log` in timeline hooks để thấy request đi qua các bước
    - Filter `the_content` append marker để thấy filter chạy đúng context
- [x] Decide which request types to support in lab (minimum viable)
  - **Outcome**: Frontend + REST ping là bắt buộc; admin trace là optional nếu cần.

### Phase 2: Implementation (File/Code/Test Structure)

Proposed plugin structure:

```
vocalmeet/assessment/wordpress/
└── wp-content/
    └── plugins/
        └── vocalmeet-hooks-lab/                  # ✅ Implemented - plugin lab
            ├── vocalmeet-hooks-lab.php           # ✅ Implemented - plugin bootstrap
            └── includes/
                ├── class-hooks-lab.php           # ✅ Implemented - hook registrations
                └── class-hooks-lab-rest.php      # ✅ Implemented - REST route for verification
```

Key behaviors:
- On `init`:
  - Register 2 callbacks với priority khác nhau để update option “thứ tự chạy”
  - Register a shortcode `[vocalmeet_hooks_lab_status]` hiển thị trạng thái option (để verify không cần đọc DB)
- On `the_content`:
  - Append marker (chỉ khi `is_singular()`), để verify filter chạy đúng boundary
- On lifecycle hooks:
  - `plugins_loaded`, `init`, `wp_loaded`, `wp`, `template_redirect`, `wp_head`, `wp_footer`: ghi `error_log` (chỉ khi bật “trace mode”)
- REST API:
  - `GET /wp-json/vocalmeet/v1/hooks-lab/ping`: trả về JSON gồm `ok`, `init_order`, timestamp

### Phase 3: Detailed Implementation Steps

1) Create plugin bootstrap
- ✅ Tạo `vocalmeet-hooks-lab.php` với plugin header.
- ✅ Load classes và gọi `->register()` qua hook `plugins_loaded` (tránh chạy logic nặng ngay khi include).

2) Implement init priority demonstration
- Tạo 2 callbacks cho hook `init`:
  - callback A priority `5`: set option `['first']`
  - callback B priority `20`: append `'second'`
- Add a reset mechanism (để học/demonstrate nhiều lần):
  - Expose reset via REST `POST /hooks-lab/reset` hoặc query param guard (ưu tiên REST để sạch).

3) Implement filter demonstration (`the_content`)
- Add filter `the_content`:
  - Chỉ chạy khi `is_singular()`
  - Append `<p>Rendered by vocalmeet hooks lab.</p>`
- Ensure không phá layout: chỉ append, không replace toàn bộ.

4) Implement lifecycle tracing (error_log)
- Register actions cho timeline hooks:
  - `plugins_loaded`, `init`, `wp_loaded`, `wp`, `template_redirect`, `wp_head`, `wp_footer`
- Add a “trace enabled” switch:
  - `define('VOCALMEET_HOOKS_LAB_TRACE', true)` trong wp-config (preferred), hoặc
  - option `vocalmeet_hooks_lab_trace_enabled`
- Khi trace tắt: không log để tránh noise.

5) Implement REST verification endpoint
- Hook `rest_api_init` để register route:
  - `GET /vocalmeet/v1/hooks-lab/ping` (permission: public)
  - Response JSON:
    - `ok: true`
    - `init_order: [...]`
    - `context`: (optional) `is_user_logged_in`, `request_uri`

Status:
- ✅ Init priority demo implemented
- ✅ Shortcode `[vocalmeet_hooks_lab_status]` implemented
- ✅ Content filter demo implemented
- ✅ Trace hooks + trace toggle implemented (REST endpoint)
- ✅ REST endpoints implemented:
  - `GET /wp-json/vocalmeet/v1/hooks-lab/ping`
  - `POST /wp-json/vocalmeet/v1/hooks-lab/reset` (admin)
  - `POST /wp-json/vocalmeet/v1/hooks-lab/trace` (admin)

6) Manual verification steps (không cần test framework)
- Activate plugin trong WP admin.
- Truy cập 1 page/post bất kỳ:
  - Xác nhận marker được append (filter hoạt động).
- Gọi endpoint:
  - `GET /wp-json/vocalmeet/v1/hooks-lab/ping`:
    - Xác nhận `init_order` là `["first","second"]`
- Check logs:
  - Xác nhận timeline hook log xuất hiện đúng thứ tự khi trace mode bật.

7) Optional extensions (nếu còn thời gian)
- Add admin page “Hooks Lab” để:
  - Toggle trace on/off
  - Reset init order
- Add extra hook coverage:
  - `admin_init`, `admin_enqueue_scripts`
  - `wp_enqueue_scripts` để học enqueue timing

## 🚧 Outstanding Issues & Follow-up
- (None)
