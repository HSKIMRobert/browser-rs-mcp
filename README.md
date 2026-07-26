# browser-rs

An MCP-only browser server in Rust. It drives a locally installed Chrome or
Chromium through CDP and exposes browser control to any MCP client. It does not
bundle an agent, model, or chat UI.

The main design choices are:

- **Accessibility-first interaction:** snapshots produce a compact tree with
  `[ref]` values, and action tools return a settle diff for verification.
- **Raw CDP:** one multiplexed WebSocket handles browser and page sessions.
- **Real browser defaults:** headful mode, a persistent profile, and no page
  patching by default. Headless stealth patching is an explicit fallback.
- **Two transports:** stdio for local clients, or HTTP with streamable MCP and
  legacy SSE endpoints.

## Install

On macOS arm64 and Linux x64, the installer downloads a prebuilt binary. A
locally installed Google Chrome or Chromium is also required.

```bash
curl -fsSL https://raw.githubusercontent.com/maestrojeong/browser-rs-mcp/main/install.sh | sh
browser-rs --help
```

The installer uses `/usr/local/bin`, or `~/.local/bin` when the former is not
writable. Set `AB_BIN_DIR` to choose another directory and `AB_VERSION` to pin
a release:

```bash
curl -fsSL https://raw.githubusercontent.com/maestrojeong/browser-rs-mcp/main/install.sh \
  | AB_VERSION=v0.1.12 AB_BIN_DIR="$HOME/bin" sh
```

Other options are listed in **[INSTALL.md](INSTALL.md)**, including direct
downloads, SHA-256 files, source installation, and updating a running binary.
Build from source with:

```bash
cargo install --git https://github.com/maestrojeong/browser-rs-mcp ab-mcp
```

Set `AB_CHROME` when Chrome is not in a standard location if needed.

## Start and connect

Use stdio for a client that launches the server:

```bash
browser-rs
```

Use HTTP when several clients should share one browser process and profile:

```bash
browser-rs --port 9321
# streamable HTTP: http://127.0.0.1:9321/mcp
# legacy SSE:      http://127.0.0.1:9321/sse
```

Example stdio configuration:

```jsonc
{
  "mcpServers": {
    "browser-rs": {
      "command": "browser-rs"
    }
  }
}
```

For HTTP, configure the client with `http://127.0.0.1:9321/mcp` for streamable
HTTP, or `/sse` for clients that still use legacy SSE. See
**[INSTALL.md](INSTALL.md)** for client-specific examples.

Keep HTTP on loopback unless it is behind a trusted, authenticated proxy. A
non-loopback bind requires `AB_HTTP_CAPABILITY`; clients must then send the
same value in `X-Browser-Capability`. This header is a capability check, not
TLS or a replacement for authentication.

## Shared profiles and ownership

Each HTTP request can identify its topic, worker, or job with a stable owner:

```text
http://127.0.0.1:9321/mcp?owner=42%3A100%3Aresearch
X-Browser-Owner: 42:100:research
```

New tabs are assigned to the request owner. Page listing, switching, and page
operations resolve only pages owned by that connection; knowing another
page's ID is not sufficient to access it. An owner-scoped `browser_close`
closes that owner's tabs without stopping the shared browser.

When a topic or job is deleted, close its tabs and mappings explicitly:

```bash
curl -X DELETE \
  'http://127.0.0.1:9321/owners?owner=42%3A100%3Aresearch'
```

This does not delete the browser profile or affect other owners. Connections
without an owner are administrative and can access all tabs, so do not expose
an ownerless HTTP endpoint publicly.

## Tools

MCP exposes 62 `browser_*` tools:

**Navigation and inspection:** `browser_navigate` · `browser_new_page` · `browser_snapshot` · `browser_read` · `browser_get_visible_html` · `browser_get_visible_text` · `browser_find` · `browser_take_screenshot` · `browser_save_pdf` · `browser_pages` · `browser_tabs` · `browser_switch_page` · `browser_profile` · `browser_status`

**Interaction:** `browser_click` · `browser_type` · `browser_press_key` · `browser_hover` · `browser_select_option` · `browser_fill_form` · `browser_drag` · `browser_file_upload` · `browser_navigate_back` · `browser_wait_for` · `browser_resize` · `browser_evaluate` · `browser_run_code_unsafe` · `browser_iframe_click` · `browser_iframe_fill` · `browser_close_page` · `browser_close`

**Network and requests:** `browser_network_requests` · `browser_route_block` · `browser_route_mock` · `browser_route_clear` · `browser_network_state_set` · `browser_api_request`

**Cookies and storage:** `browser_cookie_list` · `browser_cookie_get` · `browser_cookie_set` · `browser_cookie_delete` · `browser_cookie_clear` · `browser_localstorage_list` · `browser_localstorage_get` · `browser_localstorage_set` · `browser_localstorage_delete` · `browser_localstorage_clear` · `browser_sessionstorage_list` · `browser_sessionstorage_get` · `browser_sessionstorage_set` · `browser_sessionstorage_delete` · `browser_sessionstorage_clear` · `browser_storage_save` · `browser_storage_load`

**Diagnostics and page utilities:** `browser_console_messages` · `browser_fingerprint_check` · `browser_handle_dialog` · `browser_highlight` · `browser_hide_highlight` · `browser_webauthn` · `browser_claim_page` · `browser_release_page`

Most interaction tools accept a snapshot `ref` or a CSS selector. The common
workflow is: snapshot, act, then inspect the returned accessibility diff.

## CLI and environment

```text
browser-rs                          # stdio MCP transport
browser-rs --port 9321 [options]    # HTTP MCP transport
  --host <host>            HTTP bind host (default 127.0.0.1)
  --user-data-dir <path>   Persistent browser profile directory
  --profile <path>         Alias for --user-data-dir
  --headless               Run headless
  --headed                 Run headful (default)
  --connect <port|url>     Attach to an existing Chrome
  --stealth                Enable the JS fallback layer
```

`--port` enables HTTP mode; without it, the server uses stdio. The equivalent
environment variables are `AB_HTTP`, `AB_PROFILE`, `AB_HEADLESS`, `AB_CONNECT`,
`AB_STEALTH`, and `AB_CHROME`. `AB_HTTP_CAPABILITY` protects HTTP/SSE requests
with `X-Browser-Capability` and is required for non-loopback binds.

For `--connect`, start Chrome with an explicit remote debugging port, then pass
that port or its URL:

```bash
google-chrome --remote-debugging-port=9222
browser-rs --connect 9222
```

## Development

Requirements: Rust 1.85 or newer, Chrome/Chromium, and Node.js for the optional
benchmark scripts.

```bash
cargo test --workspace
cargo build --release -p ab-mcp
```

The `bench/` directory contains local detector and browser comparison runners.
They are regression tools whose results depend on browser versions, sites, and
detectors; they are not compatibility or stealth guarantees:

```bash
node bench/run.mjs target/release/browser-rs
node bench/external.mjs target/release/browser-rs
node bench/rebrowser.mjs target/release/browser-rs
```

See [DESIGN.md](DESIGN.md) for architecture and tradeoffs. The source is
organized into `ab-cdp` (CDP transport), `ab-browser` (browser and page logic),
and `ab-mcp` (the MCP server).

## Release

Update `workspace.package.version` in `Cargo.toml`, then commit and tag:

```bash
git commit -am "Release vX.Y.Z"
git tag vX.Y.Z
git push origin main vX.Y.Z
```

The `v*` tag workflow builds macOS arm64 and Linux x64 binaries, publishes
SHA-256 files, and attaches them to the GitHub Release. `install.sh` fetches the
latest release by default; set `AB_VERSION=vX.Y.Z` to pin one.

## License

Apache-2.0. See [LICENSE](LICENSE).
