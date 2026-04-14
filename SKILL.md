     1|---
     2|name: wordpress-mcp-setup
     3|description: Connect Hermes to a WordPress site via MCP (WordPress/mcp-adapter). Covers discovery, auth, config, and pitfalls for local (ServBay) and remote WP installs.
     4|version: 1.0.0
     5|author: Hermes Agent
     6|metadata:
     7|  hermes:
     8|    tags: [WordPress, MCP, Elementor, ServBay]
     9|    related_skills: [native-mcp]
    10|---
    11|
    12|# WordPress MCP Setup
    13|
    14|Connect Hermes to a WordPress site so that `mcp_wordpress_*` and `mcp_elementor_*` tools are available natively.
    15|
    16|## Key Facts
    17|
    18|- The canonical plugin is **WordPress/mcp-adapter** (official). The Automattic/wordpress-mcp repo is deprecated and redirects to it.
    19|- It uses **MCP 2025-06-18 spec** with session-based HTTP transport (StreamableHTTP).
    20|- Raw curl to the endpoint will always fail with `{"error": "Missing Mcp-Session-Id"}` — that's expected. The Python `mcp` client handles sessions automatically.
    21|- Auth is **WordPress Application Passwords** (not your WP login password). Format: `username:xxxx xxxx xxxx xxxx xxxx xxxx`.
    22|
    23|## Step 1 — Verify MCP plugins are installed on WP
    24|
    25|```bash
    26|curl -s "https://<wp-site>/wp-json/wp/v2/plugins" \
    27|  -u "<user>:<app-password>" | python3 -c "
    28|import sys, json
    29|for p in json.load(sys.stdin):
    30|    print(p['status'], p['name'])
    31|"
    32|```
    33|
    34|Look for: `WordPress MCP`, `MCP Adapter`, and optionally `MCP Tools for Elementor`.
    35|
    36|## Step 2 — Discover MCP endpoint routes
    37|
    38|```bash
    39|curl -s "https://<wp-site>/wp-json/" -u "<user>:<app-password>" | python3 -c "
    40|import sys, json
    41|for ns in sorted(json.load(sys.stdin).get('namespaces', [])):
    42|    print(ns)
    43|" | grep mcp
    44|```
    45|
    46|Expected output:
    47|```
    48|mcp
    49|```
    50|
    51|Then get exact routes:
    52|```bash
    53|curl -s "https://<wp-site>/wp-json/mcp" -u "<user>:<app-password>"
    54|```
    55|
    56|Typical routes:
    57|- `/wp-json/mcp/mcp-adapter-default-server` — general WordPress MCP
    58|- `/wp-json/mcp/elementor-mcp-server` — Elementor MCP (if plugin installed)
    59|
    60|## Step 3 — Generate Basic Auth header
    61|
    62|```bash
    63|python3 -c "
    64|import base64
    65|print(base64.b64encode('<user>:<app-password>'.encode()).decode())
    66|"
    67|```
    68|
    69|## Step 4 — Add to ~/.hermes/config.yaml
    70|
    71|```yaml
    72|mcp_servers:
    73|  wordpress:
    74|    url: "https://<wp-site>/wp-json/mcp/mcp-adapter-default-server"
    75|    headers:
    76|      Authorization: "Basic <base64-encoded-credentials>"
    77|    timeout: 120
    78|    connect_timeout: 60
    79|
    80|  elementor:
    81|    url: "https://<wp-site>/wp-json/mcp/elementor-mcp-server"
    82|    headers:
    83|      Authorization: "Basic <base64-encoded-credentials>"
    84|    timeout: 120
    85|    connect_timeout: 60
    86|```
    87|
    88|## Step 5 — Verify StreamableHTTP support in Hermes venv
    89|
    90|```bash
    91|~/.hermes/hermes-agent/venv/bin/python -c \
    92|  "from mcp.client.streamable_http import streamablehttp_client; print('OK')"
    93|```
    94|
    95|If it fails, the mcp package needs upgrading:
    96|```bash
    97|~/.hermes/hermes-agent/venv/bin/python -m pip install --upgrade mcp
    98|```
    99|
   100|Note: On ServBay (macOS), `pip` and `python3` in PATH point to ServBay's Python, not Hermes's. Always use the full venv path above.
   101|
   102|## Step 6 — Restart Hermes
   103|
   104|Tools appear after restart. Names will be:
   105|- `mcp_wordpress_*` — general WP operations
   106|- `mcp_elementor_*` — Elementor page/template operations
   107|
   108|## Pitfalls
   109|
   110|- **www vs non-www redirect**: `REDACTED` redirects to `www.REDACTED` but the WP REST API may return 404 on www. Always use the exact URL from `siteurl` in WP settings.
   111|- **Old static site vs WP**: The production domain may serve a legacy static site while WordPress lives on a staging/dev subdomain (e.g. `REDACTED`).
   112|- **ServBay Python**: `python3` in PATH is ServBay's alias wrapper and prints `local: can only be used in a function` warnings. Use `REDACTED` or the Hermes venv path directly.
   113|- **App password spaces**: WP application passwords contain spaces (e.g. `REDACTED`). The spaces are valid — include them in the auth string before base64-encoding.
   114|- **REST API disabled**: Some security plugins disable the WP REST API for unauthenticated users. Always pass `-u` to curl when testing.
   115|