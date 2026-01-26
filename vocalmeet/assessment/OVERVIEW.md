# WordPress Developer Assessment - Overview

> **Last Updated:** 2026-01-24

## 1. Mục tiêu

Bài assessment cho vị trí **WordPress Developer**, yêu cầu demonstrate kỹ năng:

- WordPress plugin development
- WooCommerce REST API integration
- Elementor widget development
- JavaScript (Backbone.js optional)

### Original Assessment Description

```text
Your task:

1.) Install WordPress locally
2.) Install the Elementor plugin from wordpress.org https://wordpress.org/plugins/elementor/
3.) Install the WooCommerce plugin https://en-ca.wordpress.org/plugins/woocommerce/
4.) Here is what I want you to do:

I and II)

Understand the WooCommerce Rest API: Connect to the WooCommerce REST API using standard authentication methods recommended by WooCommerce.
Build a page within WordPress where a user can create a WooCommerce product on the website (not through the admin dashboard). Fields for product name and price are sufficient. Link to documentation is below.

III) The more complex part:
WooCommerce Integration into Elementor
Goal is to create a new additional custom drag and drop widget to Elementor that integrates WooCommerce functionalities within the website builder section of Elementor (the general concept of widgets is outlined here: https://developers.elementor.com/elementor-widgets/)

The widget should add a WooCommerce functionality to Elementor (utilizing the WooCommerce REST API for example, but other methods are accepted too, see below the links to backbone) so that a user can create a WooCommerce product within this Elementor widget.

Please note: Try not put any raw code directly into the preview page of Elementor (the large window on the right is considered the preview window).

You could build the widget in a way, that it shows a button inside the widget that triggers a popup when someone clicks. The pop up opens and contains 2 fields, one to enter product name and one for price. After pressing ok, it creates the product using a rest call (or other methods). Then the user can drag and drop your widget from the left into the preview page and it displays the product.

Please note, Elementor itself is based on Backbone.

a. Try to see and figure out how to add a custom widget to Elementor
This link may help https://github.com/pojome/elementor-hello-world
https://developers.elementor.com/creating-an-extension-for-elementor/

b. Build the WooCommerce functionality into your custom Elementor widget. Build the widget as outlined above. Please keep in mind to build it in a way that a user creates a product to the widget before dragging the widget to the page or interacting with the preview page.

In general
- You can use the WooCommerce rest API to accomplish this task https://docs.woocommerce.com/document/woocommerce-rest-api/

Please note: REST calls usually require an HTTPS connection. You should be able to use a self-signed certificate.

Specifically for task 3 and Backbone, you may want to review some of the links. below. However, this (Backbone) is optional and depends on your solution on how to integrate WooCommerce with Elementor.

https://developer.wordpress.org/rest-api/using-the-rest-api/backbone-javascript-client/

https://backbonejs.org/

Note:

You can use any 3rd party open source library that might help you.
However, please do not use the Elementor Premium plugin or other any other WordPress Plugins
Do not change WordPress Core, WooCommerce core or Elementor core code
To add code to a Wordpress page, you can use the theme’s functions.php or create a plugin / mu-plugin (must use plugin)
```

---

## 2. Assessment Requirements

### Task 1-3: Environment Setup

| Task | Mô tả | Status |
|------|-------|--------|
| 1 | Install WordPress locally | ✅ Done (Docker) |
| 2 | Install Elementor plugin | ⏳ Pending |
| 3 | Install WooCommerce plugin | ⏳ Pending |

### Task 4.I-II: WooCommerce REST API Page

**Yêu cầu:**

- Kết nối WooCommerce REST API với authentication chuẩn
- Tạo một page trong WordPress cho phép user tạo WooCommerce product
- **KHÔNG** thông qua admin dashboard
- Fields tối thiểu: Product Name, Price

**Key Resources:**

- [WooCommerce REST API Docs](https://woocommerce.github.io/woocommerce-rest-api-docs/)

### Task 4.III: Custom Elementor Widget (Phần khó nhất)

**Yêu cầu:**

- Tạo custom drag-and-drop widget cho Elementor
- Widget tích hợp WooCommerce functionality
- User có thể tạo WooCommerce product từ widget này
- **KHÔNG đặt raw code/form trong preview area** (panel bên phải của Elementor)

**Suggested UX Flow:**

1. User drag widget từ panel vào page
2. Widget hiển thị button trong preview
3. Click button → popup với form (product name, price)
4. Submit → Tạo product via REST API
5. Widget hiển thị product vừa tạo

**Key Resources:**

- [Elementor Widget Guide](https://developers.elementor.com/elementor-widgets/)
- [Elementor Hello World Plugin](https://github.com/pojome/elementor-hello-world)
- [Creating Elementor Extension](https://developers.elementor.com/creating-an-extension-for-elementor/)
- [Backbone.js](https://backbonejs.org/) (optional)

### Constraints

- ✅ Có thể dùng 3rd party open source libraries
- ❌ KHÔNG dùng Elementor Premium plugin
- ❌ KHÔNG dùng WordPress plugins khác (ngoài Elementor & WooCommerce)
- ❌ KHÔNG sửa WordPress Core, WooCommerce Core, hoặc Elementor Core
- ✅ Code trong theme's functions.php hoặc tạo plugin/mu-plugin

---

## 3. Expert Analysis: Hiểu ý nghĩa Assessment

> **Mindset:** Không chỉ làm cho "hoạt động" mà phải demonstrate sự hiểu biết sâu về WordPress ecosystem, best practices, và tư duy của một Senior Developer.

### 3.1 Task 4.I-II: WooCommerce REST API Integration

#### 🎯 Họ muốn đánh giá gì?

| Khía cạnh | Họ đang test | Expert-level expectation |
|-----------|--------------|--------------------------|
| **API Understanding** | Hiểu REST API authentication, endpoints | Sử dụng OAuth 1.0a hoặc API keys đúng cách, không hardcode credentials |
| **Security Awareness** | Có biết về security không? | Nonce verification, input sanitization, capability checks |
| **WordPress Integration** | Có làm đúng "WordPress way" không? | Dùng hooks, shortcodes, proper enqueueing, AJAX handling |
| **Error Handling** | Code có production-ready không? | Graceful error handling, user feedback, logging |
| **Code Quality** | Có viết code clean không? | PSR standards, separation of concerns, reusable |

#### 🔐 Security Focus Points

```text
1. CREDENTIAL PROTECTION
   ├─ KHÔNG hardcode Consumer Key/Secret trong frontend JS
   ├─ Sử dụng server-side proxy để gọi WooCommerce API
   └─ Store credentials trong wp_options (encrypted) hoặc .env

2. INPUT VALIDATION
   ├─ sanitize_text_field() cho product name
   ├─ floatval() hoặc wc_format_decimal() cho price
   └─ wp_verify_nonce() cho form submission

3. PERMISSION CHECK
   ├─ Ai được phép tạo product? (logged-in users? specific roles?)
   ├─ current_user_can('edit_products') nếu cần
   └─ Rate limiting để prevent abuse
```

#### 🏗️ Architecture Focus Points

```text
1. SEPARATION OF CONCERNS
   ├─ API Handler: Chỉ giao tiếp với WooCommerce REST API
   ├─ Form Handler: Xử lý request/response
   └─ Template: Chỉ render UI

2. REUSABILITY
   ├─ API wrapper có thể reuse cho Plugin 2
   ├─ Form component có thể extend thêm fields sau
   └─ Config centralized (API endpoint, credentials location)

3. TESTABILITY
   ├─ API calls isolated để có thể mock
   ├─ Business logic tách khỏi WordPress hooks
   └─ Proper dependency injection
```

#### ✅ What They Want to See (Plugin 1)

1. **Đúng "WordPress way":** Dùng `wp_ajax_*` hooks, `wp_localize_script`, nonce
2. **Không expose credentials:** Server-side API calls, không gọi WooCommerce API trực tiếp từ browser
3. **Good UX:** Loading state, success/error messages, form validation
4. **Clean code:** OOP structure, proper namespacing, PHPDoc comments

---

### 3.2 Task 4.III: Custom Elementor Widget

#### 🎯 Họ muốn đánh giá gì?

| Khía cạnh | Họ đang test | Expert-level expectation |
|-----------|--------------|--------------------------|
| **Elementor Architecture** | Hiểu widget lifecycle, hooks | Đúng event hooks, proper asset loading |
| **Editor vs Frontend** | Phân biệt được 2 contexts | Enqueue đúng scripts cho đúng context |
| **Complex Integration** | Kết hợp nhiều systems | WooCommerce + Elementor + Custom JS |
| **UX Thinking** | Có nghĩ về user experience không? | Không raw form trong preview, intuitive flow |
| **Advanced JS** | Backbone.js / modern JS | Event handling, state management |

#### ⚡ Key Insight: "KHÔNG đặt raw code trong preview"

```text
Requirement này QUAN TRỌNG vì:

1. SEPARATION OF CONCERNS
   ├─ Panel (left): Configuration, data input, controls
   ├─ Preview (right): Visual representation, render output
   └─ Frontend (live site): What visitors see

2. ELEMENTOR PHILOSOPHY
   ├─ Preview = "What You See Is What You Get"
   ├─ Preview should show RESULT, not INPUT FORM
   └─ Controls belong in the panel sidebar

3. PROFESSIONAL APPROACH
   ├─ Popup triggered from panel = Configuration action
   ├─ Widget preview = Shows created product
   └─ Frontend render = Display product to visitors
```

#### 🏗️ Architecture Focus Points

```text
1. WIDGET LIFECYCLE UNDERSTANDING
   ├─ Controls registration: Khi nào chạy?
   ├─ Render method: Editor context vs Frontend context
   ├─ Scripts: Khi nào load editor.js vs frontend.js?
   └─ Settings save: Như thế nào product ID được lưu?

2. JAVASCRIPT ARCHITECTURE
   ├─ Elementor editor hooks (panel events)
   ├─ AJAX communication with server
   ├─ State management (product created? which product?)
   └─ DOM manipulation for popup

3. INTEGRATION POINTS
   ├─ Widget registers với Elementor như thế nào?
   ├─ Widget category custom?
   ├─ Dependencies declaration (WooCommerce required)
   └─ Deactivation handling
```

#### 🔐 Security Focus Points (Plugin 2)

```text
1. EDITOR CONTEXT SECURITY
   ├─ Chỉ users có quyền edit page mới thấy widget controls
   ├─ API calls vẫn phải verify permissions
   └─ Nonce for AJAX calls từ editor

2. FRONTEND CONTEXT SECURITY
   ├─ Không expose admin capabilities
   ├─ Sanitize all output (esc_html, esc_attr)
   └─ Product data đã được validate khi save
```

#### ✅ What They Want to See (Plugin 2)

1. **Elementor expertise:** Proper widget registration, correct hooks usage
2. **Understanding of contexts:** Editor scripts vs frontend scripts
3. **Complex problem solving:** Popup flow, state management
4. **Integration skills:** Combine Elementor API + WooCommerce API + Custom JS
5. **UX awareness:** Intuitive user flow, not hacky solutions

---

### 3.3 General Expert Considerations

#### 🛡️ WordPress Security Checklist

| Concern | Implementation |
|---------|---------------|
| XSS Prevention | `esc_html()`, `esc_attr()`, `wp_kses()` cho output |
| SQL Injection | `$wpdb->prepare()` nếu có custom queries |
| CSRF Protection | `wp_nonce_field()`, `wp_verify_nonce()` |
| Data Validation | `sanitize_*()` functions cho input |
| Capability Checks | `current_user_can()` trước sensitive actions |

#### ⚡ Performance Considerations

| Concern | Implementation |
|---------|---------------|
| Asset Loading | Chỉ load scripts/styles khi cần |
| Conditional Enqueueing | Check shortcode exists, widget được sử dụng |
| API Calls | Minimize, cache when possible |
| Database Queries | Avoid N+1, use transients for caching |

#### 📦 Code Quality Standards

| Aspect | Standard |
|--------|----------|
| PHP | PSR-12, WordPress Coding Standards |
| JavaScript | ES6+, modular structure |
| CSS | BEM naming, scoped selectors |
| Documentation | PHPDoc, inline comments cho complex logic |
| i18n | All strings translatable `__()`, `_e()` |

---

## 4. Project Structure

```
vocalmeet/                              # PROJECT_ROOT
├── devdocs/
│   ├── vocalmeet/assessment/
│   │   └── OVERVIEW.md                 # This file (business context)
│   └── misc/devtools/vocalmeet/
│       └── OVERVIEW.md                 # DevTools usage guide
│
├── devtools/vocalmeet/local/           # Local development environment
│   ├── docker-compose-assessment.yaml
│   ├── Justfile                        # ⭐ All CLI commands
│   ├── nginx/
│   ├── ssl/
│   └── scripts/
│
└── vocalmeet/assessment/wordpress/     # ⭐ SOURCE CODE
    ├── plugins/
    │   ├── vocalmeet-woo-api/          # Plugin 1 (Task 4.I-II)
    │   └── vocalmeet-elementor-woo/    # Plugin 2 (Task 4.III)
    └── themes/                         # (Optional)
```

---

## 5. Local Environment

> **⚠️ IMPORTANT:** Tất cả CLI commands **CHỈ** được chạy qua `just`.
>
> Xem chi tiết tại: **[devdocs/misc/devtools/vocalmeet/OVERVIEW.md](../../misc/devtools/vocalmeet/OVERVIEW.md)**

**Quick Start:**

```bash
cd devtools/vocalmeet/local
just setup    # First time
just start    # Daily usage
```

**URLs:**

| Service | URL |
|---------|-----|
| WordPress | <https://localhost> |
| phpMyAdmin | <http://localhost:8081> |

---

## 6. Technical Approach

### 6.1 Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Assessment Solution                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────┐      ┌─────────────────────────────────┐  │
│  │   Plugin 1:         │      │   Plugin 2:                     │  │
│  │   vocalmeet-woo-api │      │   vocalmeet-elementor-woo       │  │
│  │   ──────────────────│      │   ─────────────────────────────│  │
│  │   Task 4.I-II       │      │   Task 4.III                    │  │
│  │                     │      │                                 │  │
│  │   • Shortcode Page  │      │   • Custom Elementor Widget     │  │
│  │   • AJAX Handler    │      │   • Panel Controls (left)       │  │
│  │   • Product Form    │      │   • Preview Render (right)      │  │
│  └─────────────────────┘      └─────────────────────────────────┘  │
│            │                              │                         │
│            └──────────────┬───────────────┘                         │
│                           ▼                                         │
│              ┌───────────────────────┐                              │
│              │  WooCommerce REST API │                              │
│              │  /wp-json/wc/v3/      │                              │
│              │  (Server-side calls)  │                              │
│              └───────────────────────┘                              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 6.2 Data Flow: Product Creation

> **Why this matters:** Hiểu rõ data flow giúp identify security boundaries và error handling points.

```
┌─────────────────────────────────────────────────────────────────────┐
│                     PLUGIN 1: Shortcode Page                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  [Browser]                    [WordPress Server]                    │
│                                                                     │
│  ┌──────────────┐             ┌──────────────────┐                  │
│  │ Product Form │             │ wp_ajax_* hook   │                  │
│  │ (HTML/JS)    │─── AJAX ───▶│ + nonce verify   │                  │
│  │              │   POST      │ + sanitize input │                  │
│  └──────────────┘             └────────┬─────────┘                  │
│        ▲                               │                            │
│        │                               ▼                            │
│        │                      ┌──────────────────┐                  │
│        │                      │ WooCommerce API  │                  │
│        │                      │ (server-to-server│                  │
│   JSON response               │  with API keys)  │                  │
│   {success, product_id}       └────────┬─────────┘                  │
│        │                               │                            │
│        └───────────────────────────────┘                            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                     PLUGIN 2: Elementor Widget                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  [Elementor Editor]           [WordPress Server]                    │
│                                                                     │
│  ┌──────────────┐             ┌──────────────────┐                  │
│  │ Panel Control│             │ wp_ajax_* hook   │                  │
│  │ (Button) ────┼── Click ───▶│ + capability chk │                  │
│  │              │             │ + nonce verify   │                  │
│  └──────────────┘             └────────┬─────────┘                  │
│        │                               │                            │
│        ▼                               ▼                            │
│  ┌──────────────┐             ┌──────────────────┐                  │
│  │ Popup Form   │── AJAX ────▶│ WooCommerce API  │                  │
│  │ (JS Modal)   │   POST      │ Create Product   │                  │
│  └──────────────┘             └────────┬─────────┘                  │
│        │                               │                            │
│        ▼                               │                            │
│  ┌──────────────┐                      │                            │
│  │ Widget       │◀─ Update settings ───┘                            │
│  │ Preview      │   (product_id saved)                              │
│  │ (show product│                                                   │
│  └──────────────┘                                                   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 6.3 Design Decisions

| Decision | Approach | Rationale |
|----------|----------|-----------|
| **2 separate plugins** | Không merge thành 1 | Separation of concerns; dễ test riêng từng task |
| **Server-side API calls** | PHP gọi WooCommerce, không JS direct | Security - không expose API credentials ra browser |
| **AJAX pattern** | `wp_ajax_*` hooks | Standard WordPress way, built-in nonce support |
| **Popup cho Elementor** | JS modal, không in-preview form | Tuân thủ requirement "no raw code in preview" |

### 6.4 Plugin 1: vocalmeet-woo-api (Task 4.I-II)

**Purpose:** Standalone page để user tạo WooCommerce products via REST API.

**Structure:**

```
vocalmeet-woo-api/
├── vocalmeet-woo-api.php           # Main plugin file, hooks registration
├── includes/
│   ├── class-api-handler.php       # WooCommerce REST API wrapper
│   ├── class-product-form.php      # Shortcode + form logic
│   └── class-ajax-handler.php      # AJAX endpoints
├── assets/
│   ├── js/product-form.js          # Form validation + AJAX submission
│   └── css/product-form.css        # Form styling
└── templates/
    └── product-form.php            # Form HTML template
```

**Key Implementation:**

| Component | Responsibility | Key Functions |
|-----------|---------------|---------------|
| `vocalmeet-woo-api.php` | Plugin bootstrap | `register_activation_hook`, dependency checks |
| `class-api-handler.php` | WooCommerce communication | `create_product()`, handles API auth |
| `class-product-form.php` | Shortcode rendering | `[vocalmeet_product_form]`, enqueue assets |
| `class-ajax-handler.php` | AJAX endpoints | `wp_ajax_create_product`, nonce verification |

**Shortcode Usage:**

```php
// Any page/post can use:
[vocalmeet_product_form]

// Or with attributes (optional enhancement):
[vocalmeet_product_form button_text="Create Product"]
```

### 6.5 Plugin 2: vocalmeet-elementor-woo (Task 4.III)

**Purpose:** Custom Elementor widget để tạo WooCommerce products.

**Structure:**

```
vocalmeet-elementor-woo/
├── vocalmeet-elementor-woo.php     # Main plugin file
├── includes/
│   ├── class-plugin.php            # Elementor integration bootstrap
│   ├── class-ajax-handler.php      # AJAX for product creation
│   └── widgets/
│       └── class-product-creator.php  # Widget class
├── assets/
│   ├── js/
│   │   ├── editor.js               # Editor-only: popup trigger
│   │   └── frontend.js             # Frontend: product display interactions
│   └── css/
│       ├── editor.css              # Editor styling
│       └── frontend.css            # Frontend widget styling
└── templates/
    └── widget-output.php           # Widget HTML template
```

**Widget Lifecycle (quan trọng để hiểu):**

```
┌─────────────────────────────────────────────────────────────────┐
│                    ELEMENTOR EDITOR                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. User drags widget vào page                                  │
│     └─▶ register_controls() được gọi                            │
│     └─▶ render() được gọi (hiện placeholder)                    │
│                                                                 │
│  2. User click "Create Product" button trong preview            │
│     └─▶ editor.js handle click event                            │
│     └─▶ Show popup form (JS modal)                              │
│     └─▶ User fill form, submit                                  │
│     └─▶ AJAX call đến server                                    │
│     └─▶ Server tạo product, return product_id                   │
│     └─▶ JS update widget settings (product_id)                  │
│     └─▶ Widget re-render với product data                       │
│                                                                 │
│  3. User saves page                                             │
│     └─▶ product_id được save vào post meta                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                    FRONTEND (Live Site)                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  render() được gọi với saved settings                           │
│  └─▶ Fetch product data từ WooCommerce                          │
│  └─▶ Display product (name, price)                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Widget Class Key Methods:**

```php
class Product_Creator_Widget extends \Elementor\Widget_Base {
    
    // Identity
    public function get_name() { return 'vocalmeet-product-creator'; }
    public function get_title() { return 'Product Creator'; }
    public function get_icon() { return 'eicon-products'; }
    public function get_categories() { return ['general']; }
    
    // Scripts chỉ load trong editor
    public function get_script_depends() { return ['vocalmeet-editor']; }
    
    // Controls: settings được save
    protected function register_controls() {
        $this->add_control('product_id', [...]);  // Hidden, set by JS
        $this->add_control('button_text', [...]);  // Customizable
    }
    
    // Render: output HTML
    protected function render() {
        $product_id = $this->get_settings('product_id');
        if ($product_id) {
            // Show product
        } else {
            // Show "Create Product" button
        }
    }
}
```

---

## 7. Implementation Roadmap

### Phase 1: Environment Setup ⏳

| Task | Command | Status |
|------|---------|--------|
| Docker + HTTPS | `just setup` | ✅ Done |
| Generate SSL cert | `just ssl-generate` | ⏳ |
| Start WordPress | `just start` | ⏳ |
| Verify WooCommerce active | Admin → Plugins | ⏳ |
| Verify Elementor active | Admin → Plugins | ⏳ |
| Create REST API credentials | WooCommerce → Settings → Advanced → REST API | ⏳ |

**✅ Success Criteria:**
- [ ] Access `https://localhost` without errors
- [ ] WooCommerce REST API responds: `curl https://localhost/wp-json/wc/v3/`
- [ ] API credentials saved in `.env` hoặc plugin settings

---

### Phase 2: Plugin 1 - vocalmeet-woo-api ⏳

| Step | Task | Verification |
|------|------|-------------|
| 2.1 | Plugin skeleton + activation | Plugin appears in WP Admin |
| 2.2 | API Handler class | Unit test: `create_product()` returns product ID |
| 2.3 | Shortcode registration | `[vocalmeet_product_form]` renders form |
| 2.4 | AJAX handler | Browser DevTools: POST returns success JSON |
| 2.5 | Form styling | Visual check: form looks professional |
| 2.6 | End-to-end test | Create product → appears in WooCommerce Products |

**✅ Success Criteria:**
- [ ] Non-admin user can access form page
- [ ] Product created successfully via form
- [ ] Error messages display correctly (empty name, invalid price)
- [ ] No JS console errors

---

### Phase 3: Plugin 2 - vocalmeet-elementor-woo ⏳

| Step | Task | Verification |
|------|------|-------------|
| 3.1 | Plugin skeleton + Elementor check | Plugin activates only if Elementor active |
| 3.2 | Widget registration | Widget appears in Elementor panel |
| 3.3 | Basic render | Drag widget → shows placeholder in preview |
| 3.4 | Editor JS + Popup | Click button → popup appears |
| 3.5 | AJAX product creation | Submit popup → product created |
| 3.6 | Widget re-render | After creation → widget shows product info |
| 3.7 | Frontend render | View page → product displays correctly |

**✅ Success Criteria:**
- [ ] Widget draggable from panel
- [ ] **No raw form in preview area** (popup only)
- [ ] Product creation works in editor
- [ ] Saved page shows product on frontend
- [ ] Multiple widgets on same page work independently

---

### Phase 4: Polish & QA ⏳

| Category | Tasks |
|----------|-------|
| **Error Handling** | API failures, network errors, validation |
| **Edge Cases** | Empty fields, duplicate submissions, special characters in product name |
| **Code Quality** | PHPDoc comments, consistent naming, remove debug code |
| **Security Review** | Nonce on all AJAX, sanitize all inputs, escape all outputs |

**✅ Final Checklist:**
- [ ] No PHP warnings/errors in debug.log
- [ ] No JS console errors
- [ ] Works with fresh WordPress install
- [ ] Code follows WordPress Coding Standards

---

## 8. Technical Notes

### 8.1 WooCommerce REST API Authentication

WooCommerce REST API yêu cầu:

- **HTTPS** cho production (self-signed OK cho local)
- **Consumer Key & Secret** tạo từ WooCommerce → Settings → Advanced → REST API

**Endpoint để tạo product:**

```
POST /wp-json/wc/v3/products
Authorization: Basic base64(consumer_key:consumer_secret)
Content-Type: application/json

{
  "name": "Product Name",
  "type": "simple",
  "regular_price": "19.99"
}
```

### 8.2 Elementor Widget Key Concepts

1. **Widget Base Class:** Extend `\Elementor\Widget_Base`
2. **Controls:** UI elements trong panel (left side)
3. **Render:** HTML output trong preview (right side)
4. **Scripts/Styles:** Enqueue cho editor và frontend riêng biệt

**Important:** Widget controls được xử lý trong Elementor editor context (JavaScript). Để tạo product từ controls, cần:

- Hook vào Elementor JS events
- Make AJAX call đến custom endpoint hoặc WooCommerce REST API
- Update widget settings với product ID

### 8.3 HTTPS Requirement

WooCommerce REST API mặc định yêu cầu HTTPS. Local development sử dụng self-signed SSL certificate.

```php
// In plugin, for development only:
add_filter('https_ssl_verify', '__return_false');
```

---

## 9. References

### Official Documentation

- [WordPress Plugin Handbook](https://developer.wordpress.org/plugins/)
- [WooCommerce REST API](https://woocommerce.github.io/woocommerce-rest-api-docs/)
- [Elementor Developers](https://developers.elementor.com/)

### Useful Tutorials

- [Creating Elementor Widget](https://developers.elementor.com/creating-an-extension-for-elementor/)
- [Elementor Hello World](https://github.com/pojome/elementor-hello-world)

---

## 10. Related Files

| Type | Path |
|------|------|
| DevTools Guide | `devdocs/misc/devtools/vocalmeet/OVERVIEW.md` |
| Source Code | `vocalmeet/assessment/wordpress/` |
| Docker Compose | `devtools/vocalmeet/local/docker-compose-assessment.yaml` |
| Justfile | `devtools/vocalmeet/local/Justfile` |
