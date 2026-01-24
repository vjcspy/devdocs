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

## 3. Project Structure

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

## 4. Local Environment

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

## 5. Technical Approach

### 5.1 Architecture

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
│  │   • REST API Auth   │      │   • Custom Elementor Widget     │  │
│  │   • Frontend Form   │      │   • Panel Controls (left)       │  │
│  │   • Product CRUD    │      │   • Preview Render (right)      │  │
│  │                     │      │   • WooCommerce Integration     │  │
│  └─────────────────────┘      └─────────────────────────────────┘  │
│            │                              │                         │
│            └──────────────┬───────────────┘                         │
│                           ▼                                         │
│                 ┌─────────────────────┐                             │
│                 │  WooCommerce        │                             │
│                 │  REST API           │                             │
│                 │  /wp-json/wc/v3/    │                             │
│                 └─────────────────────┘                             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 5.2 Plugin 1: vocalmeet-woo-api (Task 4.I-II)

**Purpose:** Standalone page để user tạo WooCommerce products via REST API.

**Structure:**

```
vocalmeet-woo-api/
├── vocalmeet-woo-api.php           # Main plugin file
├── includes/
│   ├── class-api-handler.php       # WooCommerce REST API wrapper
│   └── class-product-form.php      # Frontend form handler
├── assets/
│   ├── js/product-form.js          # AJAX form submission
│   └── css/product-form.css        # Form styling
└── templates/
    └── product-form.php            # Form template
```

**Key Implementation Points:**

1. Tạo WooCommerce REST API credentials (Consumer Key/Secret)
2. Register shortcode `[vocalmeet_product_form]`
3. Handle form submission via AJAX
4. Call WooCommerce REST API để tạo product

### 5.3 Plugin 2: vocalmeet-elementor-woo (Task 4.III)

**Purpose:** Custom Elementor widget để tạo WooCommerce products.

**Structure:**

```
vocalmeet-elementor-woo/
├── vocalmeet-elementor-woo.php     # Main plugin file
├── includes/
│   ├── class-plugin.php            # Plugin bootstrap
│   └── widgets/
│       └── class-product-creator.php  # Widget class
├── assets/
│   ├── js/
│   │   ├── editor.js               # Elementor editor scripts
│   │   └── frontend.js             # Frontend scripts (popup)
│   └── css/
│       ├── editor.css
│       └── frontend.css
└── templates/
    └── popup-form.php              # Product creation popup
```

**Widget Class Structure:**

```php
class Product_Creator_Widget extends \Elementor\Widget_Base {
    
    public function get_name() { return 'vocalmeet-product-creator'; }
    public function get_title() { return 'WooCommerce Product Creator'; }
    public function get_icon() { return 'eicon-products'; }
    public function get_categories() { return ['vocalmeet']; }
    
    // Controls in Panel (LEFT side) - where product is created
    protected function register_controls() {
        // Product settings section
        // Button to trigger product creation
    }
    
    // Render in Preview (RIGHT side) - display only
    protected function render() {
        // Show created product or placeholder
        // Button that triggers popup (NOT raw form)
    }
}
```

**Key Implementation Points:**

1. Register widget với Elementor
2. Panel controls cho product configuration
3. JavaScript event handling cho product creation
4. REST API call từ editor context
5. Render product trong preview area

---

## 6. Implementation Roadmap

### Phase 1: Environment Setup ⏳

- [x] Create docker-compose with HTTPS support
- [ ] Generate SSL certificate (`just ssl-generate`)
- [ ] Start WordPress environment (`just start`)
- [ ] Install plugins (`just setup` does this automatically)
- [ ] Create WooCommerce REST API credentials

### Phase 2: Plugin 1 - vocalmeet-woo-api ⏳

- [ ] Create plugin skeleton
- [ ] Implement WooCommerce REST API authentication
- [ ] Create product creation form
- [ ] Handle AJAX submission
- [ ] Style the form
- [ ] Test product creation

### Phase 3: Plugin 2 - vocalmeet-elementor-woo ⏳

- [ ] Study Elementor widget architecture
- [ ] Create plugin skeleton
- [ ] Register custom widget
- [ ] Implement panel controls
- [ ] Implement frontend render
- [ ] Add popup for product creation
- [ ] Integrate with WooCommerce REST API
- [ ] Test in Elementor editor

### Phase 4: Polish & Documentation ⏳

- [ ] Code cleanup
- [ ] Add error handling
- [ ] Test edge cases

---

## 7. Technical Notes

### 7.1 WooCommerce REST API Authentication

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

### 7.2 Elementor Widget Key Concepts

1. **Widget Base Class:** Extend `\Elementor\Widget_Base`
2. **Controls:** UI elements trong panel (left side)
3. **Render:** HTML output trong preview (right side)
4. **Scripts/Styles:** Enqueue cho editor và frontend riêng biệt

**Important:** Widget controls được xử lý trong Elementor editor context (JavaScript). Để tạo product từ controls, cần:

- Hook vào Elementor JS events
- Make AJAX call đến custom endpoint hoặc WooCommerce REST API
- Update widget settings với product ID

### 7.3 HTTPS Requirement

WooCommerce REST API mặc định yêu cầu HTTPS. Local development sử dụng self-signed SSL certificate.

```php
// In plugin, for development only:
add_filter('https_ssl_verify', '__return_false');
```

---

## 8. References

### Official Documentation

- [WordPress Plugin Handbook](https://developer.wordpress.org/plugins/)
- [WooCommerce REST API](https://woocommerce.github.io/woocommerce-rest-api-docs/)
- [Elementor Developers](https://developers.elementor.com/)

### Useful Tutorials

- [Creating Elementor Widget](https://developers.elementor.com/creating-an-extension-for-elementor/)
- [Elementor Hello World](https://github.com/pojome/elementor-hello-world)

---

## 9. Related Files

| Type | Path |
|------|------|
| DevTools Guide | `devdocs/misc/devtools/vocalmeet/OVERVIEW.md` |
| Source Code | `vocalmeet/assessment/wordpress/` |
| Docker Compose | `devtools/vocalmeet/local/docker-compose-assessment.yaml` |
| Justfile | `devtools/vocalmeet/local/Justfile` |
