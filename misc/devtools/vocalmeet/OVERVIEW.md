# DevTools - Vocalmeet Local Environment

> **Purpose:** Hướng dẫn chạy local development environment cho WordPress
> **Last Updated:** 2026-01-24 (Fixed Xdebug path mappings)

## Quick Start

```bash
cd devtools/vocalmeet/local

# First time setup (build image + SSL + start + install plugins)
just setup

# Daily usage
just start
```

**URLs:**
| Service | URL |
|---------|-----|
| WordPress | https://vocalmeet.local |
| phpMyAdmin | http://localhost:8081 |

---

## Available Commands

Run `just --list` để xem tất cả commands.

### Environment

| Command | Description |
|---------|-------------|
| `just setup` | First time: Build + SSL + start + install Elementor & WooCommerce |
| `just start` | Start all services |
| `just stop` | Stop (keeps data) |
| `just down` | Stop + remove containers (keeps volumes) |
| `just clean` | Remove everything including data |
| `just restart` | Restart all |
| `just status` | Show container status |
| `just build` | Rebuild WordPress image (after Dockerfile changes) |

### Logs

| Command | Description |
|---------|-------------|
| `just logs` | All services |
| `just log-wordpress` | WordPress only |
| `just log-mysql` | MySQL only |
| `just log-nginx` | Nginx only |
| `just debug-log` | PHP error log (`wp-content/debug.log`) |
| `just debug-on` | Bật Xdebug (không restart container) |
| `just debug-off` | Tắt Xdebug (giảm tải, không restart container) |

### WP-CLI

| Command | Description |
|---------|-------------|
| `just wp <cmd>` | Run any WP-CLI command |
| `just wp-version` | WordPress version |
| `just plugin-list` | List plugins |
| `just plugin-install <name>` | Install + activate plugin |

**Examples:**
```bash
just wp core version
just wp plugin list
just wp plugin install hello-dolly --activate
just wp user list
just wp wc product list
```

### Database

| Command | Description |
|---------|-------------|
| `just db-shell` | MySQL shell (vocalmeet user) |
| `just db-root` | MySQL shell (root) |
| `just db-export` | Export to `./backup.sql` |
| `just db-import <file>` | Import from file |
| `just db-reset` | Drop + recreate database |

### Helpers

| Command | Description |
|---------|-------------|
| `just open` | Open WordPress in browser |
| `just open-db` | Open phpMyAdmin |
| `just shell` | Bash into WordPress container |
| `just ssl-generate` | Regenerate SSL certificate |

### Assessment

| Command | Description |
|---------|-------------|
| `just assessment-start` | Start + ensure plugins installed |
| `just assessment-verify` | Verify setup |
| `just wc-create-api-keys` | Guide tạo WooCommerce API keys |

---

## Service Architecture

```
┌────────────────────────────────────────────────────────────┐
│                   Docker Network                            │
├────────────────────────────────────────────────────────────┤
│  ┌─────────┐    ┌───────────────────┐    ┌──────────────┐ │
│  │  Nginx  │───▶│ WordPress + Xdebug│───▶│    MySQL     │ │
│  │:443/:80 │    │       :80         │    │    :3306     │ │
│  └─────────┘    └───────────────────┘    └──────────────┘ │
│                          │                                 │
│                          ▼                                 │
│                  Bind Mount:                               │
│                  vocalmeet/assessment/wordpress            │
│                  → /var/www/html                           │
│                                                            │
│  ┌─────────────┐    ┌───────────┐                         │
│  │ phpMyAdmin  │    │  WP-CLI   │                         │
│  │   :8081     │    │(on-demand)│                         │
│  └─────────────┘    └───────────┘                         │
└────────────────────────────────────────────────────────────┘
```

**Key points:**
- WordPress source mounted từ local → IDE có thể đọc/index toàn bộ code
- Xdebug enabled → Debug trực tiếp từ PhpStorm
- Không cần install PHP locally

---

## Database Credentials

| Property | Value |
|----------|-------|
| Host (internal) | `mysql` |
| Host (external) | `localhost:3306` |
| Database | `vocalmeet` |
| Username | `vocalmeet` |
| Password | `vocalmeet_pass` |
| Root Password | `vocalmeet_root_pass` |

---

## WordPress Source Code

**Local path:** `vocalmeet/assessment/wordpress/`

**Container path:** `/var/www/html/`

**Structure sau khi WordPress install:**
```
vocalmeet/assessment/wordpress/
├── wp-admin/           # WordPress admin
├── wp-content/
│   ├── plugins/        # All plugins (WooCommerce, Elementor, custom...)
│   ├── themes/         # All themes
│   ├── uploads/        # Media files
│   └── debug.log       # PHP debug log
├── wp-includes/        # WordPress core
├── wp-config.php       # Generated config
└── index.php
```

**Workflow:**
1. Mở folder `vocalmeet/assessment/wordpress/` trong PhpStorm
2. Edit code → changes reflect ngay (no restart)
3. Debug với Xdebug + breakpoints
4. View logs: `just debug-log`

---

## Xdebug & PhpStorm Setup

### Xdebug Configuration

Xdebug đã được cấu hình sẵn trong container:
- Mode: `debug`
- Port: `9003`
- IDE Key: `PHPSTORM`
- Auto-start: `yes` (mọi request đều trigger debug)

Config file: `devtools/vocalmeet/local/php/xdebug.ini`

### PhpStorm Configuration

#### 1. Add PHP Interpreter (Docker)

1. **Settings** → **PHP** → **CLI Interpreter** → **+** → **From Docker...**
2. Chọn **Docker Compose**
3. Configuration file: `devtools/vocalmeet/local/docker-compose-assessment.yaml`
4. Service: `wordpress`
5. OK → Apply

#### 2. Configure Debug Server

1. **Settings** → **PHP** → **Servers** → **+**
2. Name: `vocalmeet.local`
3. Host: `vocalmeet.local`
4. Port: `80` (Hoặc `443` tùy thuộc vào request bạn gửi tới)
5. Debugger: `Xdebug`
6. ✅ **Use path mappings**
7. Map paths:

| Local Path | Server Path |
|------------|-------------|
| `vocalmeet/assessment/wordpress` | `/var/www/html` |

#### 3. Cấu hình biến môi trường (Mặc định đã có)

Đảm bảo `docker-compose-assessment.yaml` có biến `PHP_IDE_CONFIG` để PhpStorm tự động nhận diện server name:

```yaml
services:
  wordpress:
    environment:
      PHP_IDE_CONFIG: "serverName=vocalmeet.local"
```

#### 4. Start Debugging

1. Click **Start Listening for PHP Debug Connections** (phone icon in toolbar)
2. Set breakpoint trong code
3. Open WordPress trong browser → PhpStorm sẽ dừng tại breakpoint

### Verify Xdebug

```bash
# Check Xdebug loaded
just shell
php -v
# Should show: with Xdebug v3.x.x

# Check Xdebug config
php -i | grep xdebug
```

### Troubleshooting Xdebug

| Issue | Solution |
|-------|----------|
| PhpStorm không dừng tại breakpoint | Check path mapping đúng chưa |
| Connection refused | Kiểm tra firewall, port 9003 |
| Xdebug không load | `just build` để rebuild image |

**Enable Xdebug log:**
```bash
# Edit php/xdebug.ini, uncomment:
# xdebug.log=/tmp/xdebug.log
# xdebug.log_level=7

# Rebuild
just build
just restart

# View log
just shell
cat /tmp/xdebug.log
```

---

## HTTPS & SSL

WooCommerce REST API requires HTTPS. Setup uses self-signed certificate.

**Generate certificate:**
```bash
just ssl-generate
```

**Add to /etc/hosts:**
```bash
echo "127.0.0.1 vocalmeet.local" | sudo tee -a /etc/hosts
```

**Browser warning:** Accept self-signed certificate hoặc enable `chrome://flags/#allow-insecure-localhost`

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Services won't start | `just status` → `just logs` → `just restart` |
| SSL issues | `just ssl-generate` |
| PHP errors | `just debug-log` |
| DB connection failed | `just log-mysql` → `just db-shell` |
| Plugin not showing | `just wp plugin list` → `just debug-log` |
| Port conflict | Check `.env`, change `HTTPS_PORT`, `MYSQL_PORT` |
| Xdebug not working | Check PhpStorm server config, path mappings |

---

## Environment Variables

Override in `.env`:

```bash
MYSQL_PORT=3306
HTTPS_PORT=443
HTTP_PORT=80
PHPMYADMIN_PORT=8081
WORDPRESS_DEBUG=1
```

---

## File Structure

```
devtools/vocalmeet/local/
├── docker-compose-assessment.yaml
├── Dockerfile.wordpress            # Custom WP image with Xdebug
├── Justfile                        # ⭐ All commands here
├── justfiles/                      # Modular just files
├── nginx/nginx.conf
├── php/
│   └── xdebug.ini                  # Xdebug configuration
├── ssl/                            # Generated certificates
├── scripts/
│   ├── generate-ssl.sh
│   └── fix-htaccess.sh
├── .env
└── .env.example

vocalmeet/assessment/wordpress/     # WordPress source (mounted)
├── wp-admin/
├── wp-content/
│   ├── plugins/                    # Your plugins here
│   └── themes/                     # Your themes here
├── wp-includes/
└── ...
```

---

## Migration from Previous Setup

Nếu đã có data từ setup cũ (dùng Docker volume):

```bash
cd devtools/vocalmeet/local

# 1. Stop services
just stop

# 2. Remove old WordPress volume (keeps MySQL data)
docker volume rm vocalmeet_assessment_wordpress_data

# 3. Rebuild image (with Xdebug)
just build

# 4. Start fresh
just start

# WordPress sẽ tự install vào vocalmeet/assessment/wordpress/
# Database giữ nguyên nếu không reset
```

**Note:** Nếu muốn reset hoàn toàn:
```bash
just clean
just setup
```
