# hermes-wordpress-mcp-setup

A Hermes Agent skill to connect Hermes to a WordPress site via MCP (Model Context Protocol).

## What it does

- Connects Hermes Agent to WordPress via the official [WordPress/mcp-adapter](https://github.com/WordPress/mcp-adapter) plugin
- Supports the MCP 2025-06-18 spec with StreamableHTTP (session-based) transport
- Covers both the general WordPress MCP server and the Elementor MCP server
- Includes pitfalls for ServBay (macOS local dev), www vs non-www redirects, and App Password auth

## Usage

Load this skill inside Hermes Agent:

```
hermes skill view wordpress-mcp-setup
```

Or follow the step-by-step guide in [SKILL.md](./SKILL.md).

## Requirements

- WordPress site with the following plugins installed:
  - WordPress MCP
  - MCP Adapter
  - (optional) MCP Tools for Elementor
- A WordPress Application Password
- Hermes Agent with MCP support (`mcp` Python package >= 2025-06-18 spec)

## License

MIT
