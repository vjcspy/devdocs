# DevTools - Vocalmeet Local Environment

> **Purpose:** Hướng dẫn chạy local development environment cho WordPress
> **Last Updated:** 2026-01-24

## Quick Start

```bash
cd devtools/vocalmeet/local

# First time setup (SSL + services + plugins)
just setup

# Daily usage
just start
```

**URLs:**
| Service | URL |
|---------|-----|
| WordPress | https://localhost |
| phpMyAdmin | http://localhost:8081 |

---

## Available Commands

Run `just --list` để xem tất cả commands.

### Environment

| Command | Description |
|---------|-------------|
| `just setup` | First time: SSL + start + install Elementor & WooCommerce |
| `just start` | Start all services |
| `just stop` | Stop (keeps data) |
| `just down` | Stop + remove containers (keeps volumes) |
| `just clean` | Remove everything including data |
| `just restart` | Restart all |
| `just status` | Show container status |

### Logs

| Command | Description |
|---------|-------------|
| `just logs` | All services |
| `just log-wordpress` | WordPress only |
| `just log-mysql` | MySQL only |
| `just log-nginx` | Nginx only |
| `just debug-log` | PHP error log (`wp-content/debug.log`) |

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
┌────────────────────────────────────────────────────────┐
│                   Docker Network                        │
├────────────────────────────────────────────────────────┤
│  ┌─────────┐    ┌───────────┐    ┌──────────────────┐ │
│  │  Nginx  │───▶│ WordPress │───▶│      MySQL       │ │
│  │:443/:80 │    │   :80     │    │      :3306       │ │
│  └─────────┘    └───────────┘    └──────────────────┘ │
│                       │                               │
│                       ▼                               │
│              Volume Mounts:                           │
│              • wordpress_data                         │
│              • mysql_data                             │
│              • plugins → wp-content/plugins/vocalmeet │
│                                                       │
│  ┌─────────────┐    ┌───────────┐                    │
│  │ phpMyAdmin  │    │  WP-CLI   │                    │
│  │   :8081     │    │(on-demand)│                    │
│  └─────────────┘    └───────────┘                    │
└────────────────────────────────────────────────────────┘
```

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

## Plugin Development

**Source code location:** `vocalmeet/assessment/wordpress/plugins/`

**Mounted to:** `/var/www/html/wp-content/plugins/vocalmeet/`

**Workflow:**
1. Edit files in `vocalmeet/assessment/wordpress/plugins/`
2. Changes are immediately visible (no restart needed)
3. Activate: `just wp plugin activate <name>`
4. Debug: `just debug-log`

---

## HTTPS & SSL

WooCommerce REST API requires HTTPS. Setup uses self-signed certificate.

**Generate certificate:**
```bash
just ssl-generate
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
├── Justfile                        # ⭐ All commands here
├── nginx/nginx.conf
├── ssl/                            # Generated certificates
├── scripts/generate-ssl.sh
├── .env
└── .env.example
```
