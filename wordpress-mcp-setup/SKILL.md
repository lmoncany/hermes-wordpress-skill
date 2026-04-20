---
name: wordpress-mcp-setup
description: Connect Hermes to a WordPress site via MCP (WordPress/mcp-adapter). Covers discovery, auth, config, pitfalls for local (ServBay) and remote WP installs, and full Elementor MCP tool reference.
version: 2.1.0
author: Loic Moncany
metadata:
  hermes:
    tags: [WordPress, MCP, Elementor, ServBay, Elementor Pro]
    related_skills: [native-mcp]
---

# WordPress MCP Setup

Connect Hermes to a WordPress site so that `mcp_wordpress_*` and `mcp_elementor_*` tools are available natively.

## Key Facts

- The canonical plugin is **WordPress/mcp-adapter** (official). The Automattic/wordpress-mcp repo is deprecated and redirects to it.
- It uses **MCP 2025-06-18 spec** with session-based HTTP transport (StreamableHTTP).
- Raw curl to the endpoint will always fail with `{"error": "Missing Mcp-Session-Id"}` — that's expected. The Python `mcp` client handles sessions automatically.
- Auth is **WordPress Application Passwords** (not your WP login password). Format: `username:xxxx xxxx xxxx xxxx xxxx xxxx`.

---

## Step 1 — Verify MCP plugins are installed on WP

```bash
curl -s "https://<wp-site>/wp-json/wp/v2/plugins" \
  -u "<user>:<app-password>" | python3 -c "
import sys, json
for p in json.load(sys.stdin):
    print(p['status'], p['name'])
"
```

Look for: `WordPress MCP`, `MCP Adapter`, and optionally `MCP Tools for Elementor`.

## Step 2 — Discover MCP endpoint routes

```bash
curl -s "https://<wp-site>/wp-json/" -u "<user>:<app-password>" | python3 -c "
import sys, json
for ns in sorted(json.load(sys.stdin).get('namespaces', [])):
    print(ns)
" | grep mcp
```

Expected output:
```
mcp
```

Then get exact routes:
```bash
curl -s "https://<wp-site>/wp-json/mcp" -u "<user>:<app-password>"
```

Typical routes:
- `/wp-json/mcp/mcp-adapter-default-server` — general WordPress MCP
- `/wp-json/mcp/elementor-mcp-server` — Elementor MCP (if plugin installed)

## Step 3 — Generate Basic Auth header

```bash
python3 -c "
import base64
print(base64.b64encode('<user>:<app-password>'.encode()).decode())
"
```

## Step 4 — Add to ~/.hermes/config.yaml

```yaml
mcp_servers:
  wordpress:
    url: "https://<wp-site>/wp-json/mcp/mcp-adapter-default-server"
    headers:
      Authorization: "Basic <base64-encoded-credentials>"
    timeout: 120
    connect_timeout: 60

  elementor:
    url: "https://<wp-site>/wp-json/mcp/elementor-mcp-server"
    headers:
      Authorization: "Basic <base64-encoded-credentials>"
    timeout: 120
    connect_timeout: 60
```

For local dev with a self-signed SSL certificate (e.g. ServBay), add:
```yaml
    ssl_verify: false
```
to each server block.

## Step 5 — Verify StreamableHTTP support in Hermes venv

```bash
~/.hermes/hermes-agent/venv/bin/python -c \
  "from mcp.client.streamable_http import streamablehttp_client; print('OK')"
```

If it fails, the mcp package needs upgrading:
```bash
~/.hermes/hermes-agent/venv/bin/python -m pip install --upgrade mcp
```

Note: On ServBay (macOS), `pip` and `python3` in PATH point to ServBay's Python, not Hermes's. Always use the full venv path above.

## Step 6 — Restart Hermes

Tools appear after restart. Names will be:
- `mcp_wordpress_*` — general WP operations
- `mcp_elementor_*` — Elementor page/template operations

---

## WordPress MCP Tools

The WordPress MCP server exposes an **ability-based** API — all operations go through 3 meta-tools:

| Tool | What it does |
|------|-------------|
| `mcp-adapter-discover-abilities` | List all available WordPress abilities (CRUD for posts, pages, users, media, settings, etc.) |
| `mcp-adapter-get-ability-info` | Get the full input/output schema for a specific ability |
| `mcp-adapter-execute-ability` | Execute any ability with the provided parameters |

### Workflow

Always start by discovering what abilities are available:
```
mcp_wordpress_mcp-adapter-discover-abilities
```

Then inspect a specific ability before using it:
```
mcp_wordpress_mcp-adapter-get-ability-info  { "ability": "create_post" }
```

Then execute it:
```
mcp_wordpress_mcp-adapter-execute-ability  { "ability": "create_post", "params": { ... } }
```

---

## Elementor MCP Tools (100 tools)

The Elementor MCP server exposes 100 direct tools organized into these groups:

### Discovery & Introspection

| Tool | What it does |
|------|-------------|
| `elementor-mcp-list-widgets` | All registered widget types with names, icons, categories |
| `elementor-mcp-get-widget-schema` | Full JSON schema for any widget's available settings |
| `elementor-mcp-get-container-schema` | Schema for container controls (flex/grid, direction, justify, align) |
| `elementor-mcp-list-pages` | All pages/posts built with Elementor |
| `elementor-mcp-list-templates` | All saved Elementor templates |
| `elementor-mcp-list-dynamic-tags` | All available Pro dynamic tags |
| `elementor-mcp-list-code-snippets` | All existing Pro Custom Code snippets |

### Reading Page Structure

| Tool | What it does |
|------|-------------|
| `elementor-mcp-get-page-structure` | Full element tree for a page (containers, widgets, IDs) |
| `elementor-mcp-get-element-settings` | Current settings of a specific element |
| `elementor-mcp-find-element` | Search elements by type, widget type, or settings content |
| `elementor-mcp-get-global-settings` | Active kit/global settings (colors, typography, spacing) |

### Page Management

| Tool | What it does |
|------|-------------|
| `elementor-mcp-create-page` | Create a new WP page with Elementor enabled |
| `elementor-mcp-update-page-settings` | Update page-level settings (background, padding, custom CSS) |
| `elementor-mcp-delete-page-content` | Clear all Elementor content from a page |
| `elementor-mcp-import-template` | Import a JSON template into a page |
| `elementor-mcp-export-page` | Export a page's full Elementor data as JSON |
| `elementor-mcp-build-page` | **Power tool** — build a complete page from a declarative structure in one call |

### Container & Layout Management

| Tool | What it does |
|------|-------------|
| `elementor-mcp-add-container` | Add a flex or grid container to a page |
| `elementor-mcp-update-container` | Update container settings (partial merge) |
| `elementor-mcp-reorder-elements` | Reorder children of a container by element ID array |
| `elementor-mcp-move-element` | Move an element to a new parent/position |
| `elementor-mcp-remove-element` | Remove an element and all its children |
| `elementor-mcp-duplicate-element` | Duplicate an element with fresh IDs |

### Element & Widget Updates

| Tool | What it does |
|------|-------------|
| `elementor-mcp-update-element` | Update any element's settings (partial merge) |
| `elementor-mcp-update-widget` | Update a specific widget's settings |
| `elementor-mcp-batch-update` | Update multiple elements in a single save operation |
| `elementor-mcp-add-widget` | Add any widget by type to a container |

### Widget Shortcuts (Free)

Quick-add tools for the most common free widgets:

`add-heading` · `add-text-editor` · `add-image` · `add-button` · `add-video` · `add-icon`
`add-spacer` · `add-divider` · `add-icon-box` · `add-accordion` · `add-alert` · `add-counter`
`add-google-maps` · `add-icon-list` · `add-image-box` · `add-image-carousel` · `add-progress`
`add-social-icons` · `add-star-rating` · `add-tabs` · `add-testimonial` · `add-toggle`
`add-html` · `add-menu-anchor` · `add-shortcode` · `add-rating` · `add-text-path`

### Widget Shortcuts (Pro)

`add-form` · `add-posts-grid` · `add-countdown` · `add-price-table` · `add-flip-box`
`add-animated-headline` · `add-call-to-action` · `add-slides` · `add-testimonial-carousel`
`add-price-list` · `add-gallery` · `add-share-buttons` · `add-table-of-contents`
`add-blockquote` · `add-lottie` · `add-hotspot` · `add-nav-menu` · `add-loop-grid`
`add-loop-carousel` · `add-media-carousel` · `add-nested-tabs` · `add-nested-accordion`
`add-portfolio` · `add-author-box` · `add-login` · `add-code-highlight` · `add-reviews`
`add-off-canvas` · `add-progress-tracker` · `add-search`

### Templates & Theme Builder (Pro)

| Tool | What it does |
|------|-------------|
| `elementor-mcp-save-as-template` | Save a page or element as a reusable template |
| `elementor-mcp-apply-template` | Apply a saved template to a page at a given position |
| `elementor-mcp-create-theme-template` | Create a Pro theme builder template (header, footer, single, archive, 404, etc.) |
| `elementor-mcp-set-template-conditions` | Set display conditions (entire site, specific pages, post types, etc.) |

### Popups (Pro)

| Tool | What it does |
|------|-------------|
| `elementor-mcp-create-popup` | Create a new Elementor Pro popup template |
| `elementor-mcp-set-popup-settings` | Configure popup triggers, timing, and display conditions |

### Global Styles

| Tool | What it does |
|------|-------------|
| `elementor-mcp-update-global-colors` | Update the site-wide color palette in the Elementor kit |
| `elementor-mcp-update-global-typography` | Update site-wide typography settings |

### Dynamic Tags (Pro)

| Tool | What it does |
|------|-------------|
| `elementor-mcp-list-dynamic-tags` | List all available dynamic tags with groups/categories |
| `elementor-mcp-set-dynamic-tag` | Bind a dynamic tag to an element setting |

### Media

| Tool | What it does |
|------|-------------|
| `elementor-mcp-search-images` | Search Openverse (CC-licensed) images by keyword |
| `elementor-mcp-sideload-image` | Download an external image URL into the WP Media Library |
| `elementor-mcp-add-stock-image` | Search + sideload + add image to page in one call |
| `elementor-mcp-upload-svg-icon` | Upload an SVG to the Media Library and get an Elementor icon object |

### Custom Code

| Tool | What it does |
|------|-------------|
| `elementor-mcp-add-custom-js` | Add a JS snippet to a page via an HTML widget |
| `elementor-mcp-add-custom-css` | Add custom CSS to a specific element or the entire page (Pro) |
| `elementor-mcp-add-code-snippet` | Create a site-wide Custom Code snippet (Pro) — inject CSS or JS globally |
| `elementor-mcp-list-code-snippets` | List all existing site-wide code snippets |

---

## Example Workflows

### Build a landing page from scratch

```
1. mcp_elementor_elementor-mcp-create-page        { "title": "My Landing Page" }
2. mcp_elementor_elementor-mcp-add-container       { "post_id": <id>, ... }
3. mcp_elementor_elementor-mcp-add-heading         { "post_id": <id>, "container_id": <cid>, "title": "Hello" }
4. mcp_elementor_elementor-mcp-add-button          { "post_id": <id>, "container_id": <cid>, "text": "Get Started" }
```

Or use the power tool for a complete page in one shot:
```
mcp_elementor_elementor-mcp-build-page  { "title": "My Page", "structure": [ ... ] }
```

### Inspect then edit an existing page

```
1. mcp_elementor_elementor-mcp-list-pages
2. mcp_elementor_elementor-mcp-get-page-structure  { "post_id": <id> }
3. mcp_elementor_elementor-mcp-find-element        { "post_id": <id>, "widget_type": "heading" }
4. mcp_elementor_elementor-mcp-update-widget       { "post_id": <id>, "element_id": <eid>, "settings": { "title": "New Title" } }
```

### Update global brand colors

```
mcp_elementor_elementor-mcp-update-global-colors  {
  "colors": [
    { "id": "primary", "title": "Primary", "color": "#FF6B35" },
    { "id": "secondary", "title": "Secondary", "color": "#1A1A2E" }
  ]
}
```

### Create a site-wide header (Pro)

```
1. mcp_elementor_elementor-mcp-create-theme-template  { "type": "header", "title": "Main Header" }
2. ... add widgets to it ...
3. mcp_elementor_elementor-mcp-set-template-conditions  { "post_id": <id>, "conditions": [{ "type": "general", "name": "general" }] }
```

---

## Onboarding Flow

When a user says "set up my WordPress site", "connect my WordPress", or "onboard WordPress MCP", run this interactive flow:

### Step 1 — Collect credentials

Ask the user for:
- Site URL (e.g. `https://mysite.com`)
- WordPress username
- Application Password (Users > Profile > Application Passwords in WP admin — NOT their login password)

### Step 2 — Detect installed plugins

```bash
curl -s "<site-url>/wp-json/wp/v2/plugins" \
  -u "<user>:<app-password>" | python3 -c "
import sys, json
try:
    plugins = json.load(sys.stdin)
    for p in plugins:
        print(p.get('plugin',''), '|', p.get('status',''), '|', p.get('name',''))
except Exception as e:
    print('ERROR:', e)
"
```

Map results against required plugins:

| Plugin slug | Name | Tier |
|-------------|------|------|
| `mcp-adapter/mcp-adapter` | MCP Adapter | Required |
| `wordpress-mcp/wordpress-mcp` | WordPress MCP | Required |
| `elementor-mcp/elementor-mcp` | MCP Tools for Elementor | Optional |
| `elementor/elementor` | Elementor | Optional |

### Step 3 — Report findings and offer to install

Tell the user what was found, for example:

> I found 2 issues:
> - ✅ MCP Adapter — active
> - ❌ WordPress MCP — not installed
> - ❌ MCP Tools for Elementor — not installed
> - ✅ Elementor — active
>
> Would you like me to install and activate the missing free plugins automatically?

### Step 4 — Install missing plugins (if user agrees)

Install from WordPress.org by POSTing the slug to the plugins endpoint:

```bash
# Install + activate in one call
curl -s -X POST "<site-url>/wp-json/wp/v2/plugins" \
  -u "<user>:<app-password>" \
  -H "Content-Type: application/json" \
  -d '{"slug": "<plugin-slug>", "status": "active"}'
```

Available free slugs:
- `wordpress-mcp` — WordPress MCP
- `elementor-mcp` — MCP Tools for Elementor
- `elementor` — Elementor

> Elementor Pro is a paid plugin and cannot be auto-installed. If the user wants Pro features, direct them to elementor.com to download it and install manually.

If an installed plugin is inactive, activate it:
```bash
curl -s -X POST "<site-url>/wp-json/wp/v2/plugins/<plugin>/<plugin>" \
  -u "<user>:<app-password>" \
  -H "Content-Type: application/json" \
  -d '{"status": "active"}'
```
Note: use the `plugin` field value from the list response as the path (e.g. `elementor-mcp/elementor-mcp`). No URL encoding needed in the path — use the literal slash.

### Step 5 — Discover MCP endpoints

```bash
curl -s "<site-url>/wp-json/mcp" -u "<user>:<app-password>"
```

Expected routes:
- `mcp-adapter-default-server` — WordPress MCP
- `elementor-mcp-server` — Elementor MCP (if plugin active)

### Step 6 — Generate Basic Auth header

```python
import base64
token = base64.b64encode(f"<user>:<app-password>".encode()).decode()
print(token)
```

### Step 7 — Write config.yaml entries

Append to `~/.hermes/config.yaml`:

```yaml
mcp_servers:
  wordpress:
    url: "<site-url>/wp-json/mcp/mcp-adapter-default-server"
    headers:
      Authorization: "Basic <base64-token>"
    timeout: 120
    connect_timeout: 60

  elementor:                          # only if elementor-mcp is active
    url: "<site-url>/wp-json/mcp/elementor-mcp-server"
    headers:
      Authorization: "Basic <base64-token>"
    timeout: 120
    connect_timeout: 60
```

For local dev with self-signed SSL, add `ssl_verify: false` to each block.

### Step 8 — Confirm and summarise

Tell the user:
> Setup complete! Restart Hermes to load the new MCP tools.
> Available after restart:
> - `mcp_wordpress_*` — X tools (posts, pages, users, media, settings...)
> - `mcp_elementor_*` — 100 tools (only if Elementor MCP was installed)

---

## Required WordPress Plugins

Install all of these from the WordPress plugin directory or uploads:

| Plugin | Source | Required? |
|--------|--------|-----------|
| **MCP Adapter** | [WordPress/mcp-adapter](https://github.com/WordPress/mcp-adapter) | ✅ Required — core MCP endpoint |
| **WordPress MCP** | WordPress.org (`wordpress-mcp`) | ✅ Required — posts, pages, users, media, settings |
| **MCP Tools for Elementor** | WordPress.org (`elementor-mcp`) | ⚙️ Optional — enables all 100 Elementor tools |
| **Elementor** | WordPress.org (`elementor`) | ⚙️ Optional — needed only if using Elementor tools |
| **Elementor Pro** | [elementor.com](https://elementor.com) | ⚙️ Optional — Pro widgets, theme builder, popups, dynamic tags |

The onboarding flow auto-installs and activates any missing free plugins (MCP Adapter, WordPress MCP, MCP Tools for Elementor, Elementor) directly via the WP REST API. Elementor Pro requires a manual install due to its paid license.

After installing, verify all are active:
```bash
curl -s "https://<your-wp-site>/wp-json/wp/v2/plugins" \
  -u "<user>:<app-password>" | python3 -c "
import sys, json
for p in json.load(sys.stdin):
    print(p['status'], p['name'])
"
```

---

## Pitfalls

- **www vs non-www**: The WP REST API may return 404 if you use the wrong variant. Always use the exact URL from `siteurl` in WP Settings > General.
- **Subdomain vs main domain**: On staging setups, WordPress may live on a subdomain (e.g. `dev.example.com`) while the main domain serves a different site. Always target the WP install URL directly.
- **Local dev SSL**: Local dev environments (ServBay, LocalWP, MAMP, etc.) use self-signed certificates. Add `ssl_verify: false` to each MCP server block in `config.yaml` to avoid SSL errors. If omitted, Hermes silently fails to connect at session start with `[SSL: CERTIFICATE_VERIFY_FAILED]` and the MCP tools never appear — verify by checking `~/.hermes/logs/agent.log` for the error. Fix: add `ssl_verify: false` then restart Hermes. **CRITICAL**: The `ssl_verify` config key is not wired in `mcp_tool.py` by default and gets wiped on every Hermes update. After any Hermes update, verify with `grep -n "ssl_verify" ~/.hermes/hermes-agent/tools/mcp_tool.py` — if it returns nothing, re-apply these 3 patches using the mcp_patch tool:
  1. After line reading `connect_timeout = config.get(...)`: add `ssl_verify = config.get("ssl_verify", True)`
  2. Inside `client_kwargs` dict (new API, mcp >= 1.24.0): add `"verify": ssl_verify,`
  3. Inside `_http_kwargs` dict (legacy API, mcp < 1.24.0): add `"verify": ssl_verify,`
- **macOS local Python**: On macOS local dev stacks, `python3` in PATH may point to the stack's own Python wrapper. Always use the Hermes venv path (`~/.hermes/hermes-agent/venv/bin/python`) for MCP-related commands.
- **App password spaces**: WP Application Passwords contain spaces (e.g. `XXXX XXXX XXXX XXXX XXXX XXXX`). Include the spaces in the auth string before base64-encoding — they are valid and required.
- **REST API disabled**: Some security plugins (Wordfence, iThemes Security, etc.) disable the WP REST API for unauthenticated users. Always pass `-u` to curl when testing. If you get a 401, check your security plugin settings.
- **Missing Mcp-Session-Id**: Raw curl to MCP endpoints always returns `{"error": "Missing Mcp-Session-Id"}`. This is expected — the Hermes MCP client handles sessions automatically. Don't test MCP endpoints with bare curl.
- **batch-update for efficiency**: When editing multiple elements, use `elementor-mcp-batch-update` instead of calling `update-element` repeatedly — it saves all changes in a single WP write operation.
- **get-widget-schema before add-widget**: When using `add-widget` for less common widgets, always call `get-widget-schema` first to see all available settings — widget schemas can be large and non-obvious.
