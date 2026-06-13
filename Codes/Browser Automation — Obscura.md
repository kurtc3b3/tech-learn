## What & When

**Obscura** is a Rust headless browser for **AI agents and web scraping**. It embeds **V8**, speaks **Chrome DevTools Protocol (CDP)**, and ships a CLI for fetch/scrape plus an **MCP server** for LLM tool use. Stealth mode adds fingerprint randomization and a compiled-in **3,520-domain tracker blocklist**.

Use Obscura when:

- You need **low memory** (~30 MB) and fast startup vs headless Chrome
- Agents or scrapers connect via **CDP** (`connectOverCDP`) — drop-in for Playwright/Puppeteer
- You want **parallel URL scraping** from the shell (`obscura scrape`)
- You expose browser tools to Cursor/Claude via **MCP** without running full Chromium

Prefer [[Browser Automation — Camoufox]] for Playwright-native Python stealth (Firefox). Prefer [[Commands/CLI — agent-browser]] for snapshot-ref CLI tuned to coding agents. See [[Browser Automation]] and [[AI — MCP]].

```bash
# macOS Apple Silicon — from GitHub Releases
curl -LO https://github.com/h4ckf0r0day/obscura/releases/latest/download/obscura-aarch64-macos.tar.gz
tar xzf obscura-aarch64-macos.tar.gz

# Or Docker
docker run -d --name obscura -p 127.0.0.1:9222:9222 h4ckf0r0day/obscura
```

Repo: [github.com/h4ckf0r0day/obscura](https://github.com/h4ckf0r0day/obscura)

---

## Obscura vs Playwright vs Camoufox

| Need | Prefer | Why |
| --- | --- | --- |
| Python locators + tracing | [[Browser Automation — Playwright]] | Full automation API |
| Firefox anti-detect Python | [[Browser Automation — Camoufox]] | BrowserForge fingerprints |
| Lightweight CDP server | **Obscura** | ~30 MB, instant boot, scrape CLI |
| Agent snapshot CLI | [[Commands/CLI — agent-browser]] | Ref-based a11y tree |
| Screenshots / PDF | Playwright + Chrome | Obscura has no pixel renderer |

> [!note] Use `connectOverCDP`, not Playwright `connect`. Obscura does not implement Playwright's wire protocol.

---

## CLI — Fetch One Page

```bash
# Title via JS eval
obscura fetch https://example.com --eval "document.title"

# Rendered HTML
obscura fetch https://news.ycombinator.com --dump html

# Markdown (custom CDP method)
obscura fetch https://example.com --dump markdown

# Wait for SPA network settle
obscura fetch https://example.com --wait-until networkidle0 --stealth

# Proxy
obscura --proxy socks5://127.0.0.1:1080 fetch https://example.com --dump text
```

| `--dump` | Output |
| --- | --- |
| `html` | Rendered DOM |
| `text` | Visible text |
| `markdown` | DOM → Markdown |
| `links` | Anchor hrefs |
| `assets` | Sub-resource URLs (NDJSON) |
| `original` | Raw response body (images, JSON) |

---

## CLI — Parallel Scrape

```bash
obscura scrape \
  https://quotes.toscrape.com/page/1/ \
  https://quotes.toscrape.com/page/2/ \
  --concurrency 10 \
  --eval "Array.from(document.querySelectorAll('.quote .text')).map(e => e.textContent)" \
  --format json \
  --stealth \
  --quiet
```

Build with stealth: `cargo build --release --features stealth`.

---

## CDP Server + Playwright (Python)

Start server, connect from Python — same pattern as remote Chrome:

```bash
obscura serve --port 9222 --stealth
```

```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.connect_over_cdp("http://127.0.0.1:9222")
    context = browser.contexts[0] if browser.contexts else browser.new_context()
    page = context.new_page()
    page.goto("https://quotes.toscrape.com/", wait_until="domcontentloaded")
    quotes = page.eval_on_selector_all(
        ".quote .text",
        "els => els.map(e => e.textContent)",
    )
    print(quotes)
    browser.close()
```

Use `playwright-core` / `puppeteer-core` in Node — do not bundle Chrome when Obscura is the engine.

---

## CDP Server + Playwright (TypeScript)

```typescript
import { chromium } from "playwright-core";

const browser = await chromium.connectOverCDP({
  endpointURL: "ws://127.0.0.1:9222",
});
const page = await browser.newContext().then((ctx) => ctx.newPage());
await page.goto("https://example.com");
console.log(await page.title());
await browser.close();
```

### Supported (common)

`page.goto`, `page.evaluate`, `page.click`, `page.fill`, `page.waitForSelector`, cookies, request interception, `page.content`, `page.title`.

### Not supported

- `page.screenshot` — no pixel renderer
- Enterprise WAF bypass guarantees — combine `--stealth`, proxies, and headed debugging

---

## MCP Server

Expose browser tools to [[AI — MCP]] hosts (Cursor, Claude Desktop):

```bash
obscura mcp                    # stdio
obscura mcp --http --port 8080 # HTTP transport
```

Claude Desktop / Cursor config:

```json
{
  "mcpServers": {
    "obscura": {
      "command": "/path/to/obscura",
      "args": ["mcp", "--stealth"]
    }
  }
}
```

| Tool | Role |
| --- | --- |
| `browser_navigate` | Go to URL |
| `browser_snapshot` | URL, title, body text |
| `browser_click` / `browser_fill` / `browser_type` | Interact by CSS selector |
| `browser_evaluate` | Run JS expression |
| `browser_wait_for` | Wait for selector |
| `browser_network_requests` | List requests |
| `browser_close` | Reset session |

Optional flags: `--proxy`, `--user-agent`, `--stealth`.

---

## Stealth Mode

Enable with `--stealth` (CLI) or `--features stealth` (build).

- Per-session fingerprint randomization (GPU, canvas, audio, battery)
- `navigator.webdriver` masked; native function `toString()` patched
- 3,520 tracker/analytics domains blocked at request layer
- Realistic `userAgentData` (Chrome-like high-entropy values)

Pair with rotating residential proxies for protected sites; comply with ToS.

---

## Patterns

**Agent loop:** MCP host calls `browser_navigate` → `browser_snapshot` → model plans → `browser_click` / `browser_fill`.

**Scrape fleet:** `obscura scrape` with `--concurrency` and `--eval`; post-process JSON in Python.

**Playwright reuse:** Keep existing Playwright scripts; point CDP at Obscura instead of `chromium.launch()`.

**Vault pipeline:**

```text
obscura fetch --dump markdown → chunk → [[AI — Chroma]]
obscura mcp + Cursor → agent tools alongside [[AI — LangChain]]
```

---

## Quick Reference

| Task | Command |
| --- | --- |
| One-shot title | `obscura fetch URL --eval "document.title"` |
| HTML dump | `obscura fetch URL --dump html` |
| Parallel scrape | `obscura scrape url1 url2 --concurrency 10 --format json` |
| CDP server | `obscura serve --port 9222 --stealth` |
| Playwright | `chromium.connect_over_cdp("http://127.0.0.1:9222")` |
| MCP stdio | `obscura mcp --stealth` |
| Docker | `docker run -p 9222:9222 h4ckf0r0day/obscura` |

---

## Related Notes

- [[Browser Automation]]
- [[Browser Automation — Playwright]]
- [[Browser Automation — Camoufox]]
- [[Commands/CLI — agent-browser]]
- [[AI — MCP]]
- [[AI]]
- [[Python — asyncio]]

---

## Tags

#browser-automation #obscura #cdp #mcp #scraping #rust #stealth #ai-agents #playwright #web
