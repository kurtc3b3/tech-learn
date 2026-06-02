## What & When

**Scrapling** is an adaptive Python scraping framework: **Selector** parser (Scrapy/Parsel-style), tiered **fetchers** (HTTP → browser → stealth), Spiders, CLI, and **`scrapling[ai]` MCP**. It bridges static [[Python — httpx Package]] + [[Python — BeautifulSoup4 (bs4)]] and full [[Browser Automation — Playwright]] scripts you rewrite on every redesign.

Use Scrapling when:

- Selectors break after layout changes — **adaptive relocation** (fingerprints + similarity)
- One API to swap HTTP / Playwright / stealth without new parsers
- Cloudflare Turnstile or TLS fingerprinting — `StealthyFetcher`, MCP `stealthy_fetch`
- Token-efficient agent scraping — MCP extracts via `css_selector` before the model sees HTML
- Multi-URL crawls with sessions and proxy rotation (Spider API; see docs)

Use httpx + bs4 for static HTML. Use Playwright for E2E tests and fine-grained browser control. See [[Browser Automation]] for the stack decision flow.

**Python 3.10+** · [scrapling.readthedocs.io](https://scrapling.readthedocs.io/en/latest/)

```bash
pip install scrapling                    # parser only
pip install "scrapling[fetchers]"
scrapling install                        # Chromium + deps
scrapling install --force
pip install "scrapling[ai]"              # MCP (needs fetchers + scrapling install for browser tools)
pip install "scrapling[all]"             # fetchers + shell + MCP
```

> [!tip] Browser fetchers and MCP `fetch` / `stealthy_fetch` require `[fetchers]` or `[ai]`/`[all]` plus `scrapling install` once per environment.

---

## Scrapling vs httpx vs Playwright

| Need | Prefer | Why |
| --- | --- | --- |
| Static HTML, APIs | [[Python — httpx Package]] | No browser weight |
| HTML already downloaded | bs4 or `Selector(html)` | Scrapling wins on heavy CSS/XPath loops |
| JS SPA | Playwright or `DynamicFetcher` | Playwright = general automation; Scrapling = scrape-oriented wrapper |
| Anti-bot / Cloudflare | `StealthyFetcher`, MCP stealth tools | Patched Chromium; comply with site law/ToS |
| Layout drift | `auto_save` + `adaptive=True` | Fingerprint relocation |
| AI agent in IDE | `scrapling[ai]` + [[AI — MCP]] | Narrow content before tokens |
| Mass crawl | [[Browser Automation — Scrapy]] | Different scale/middleware story |

**Mental model:** httpx → bytes; Playwright → drive browser; Scrapling → **fetch tier + parse + adaptive tracking** (+ optional MCP).

---

## Selector Parser

`from scrapling.parser import Selector` — same API on fetcher responses and raw HTML:

```python
from scrapling.parser import Selector

page = Selector("<html><body><p class='x'>Hi</p></body></html>")
page.css("p.x::text").get()
page.xpath("//p[@class='x']/text()").get()
page.find_all("p", class_="x")
page.find_by_text("Hi", tag="p")
```

Supports Scrapy pseudo-elements (`::text`, `::attr(href)`), XPath, regex/text filters, chained selectors, navigation (`parent`, `next_sibling`, `below_elements`), `find_similar()`, auto selector generation.

---

## Fetchers (three tiers)

`from scrapling.fetchers import Fetcher, FetcherSession, DynamicFetcher, DynamicSession, StealthyFetcher, StealthySession` (+ `Async*` variants).

| Class | Backend | Use when |
| --- | --- | --- |
| `Fetcher` / `FetcherSession` | curl_cffi | Static pages; TLS impersonation; HTTP/3 |
| `DynamicFetcher` / `DynamicSession` | Playwright Chromium | JS, `network_idle`, `load_dom` |
| `StealthyFetcher` / `StealthySession` | Stealth Chromium | Cloudflare, `solve_cloudflare=True` |

```python
from scrapling.fetchers import Fetcher, FetcherSession, DynamicFetcher, StealthyFetcher

page = Fetcher.get("https://quotes.toscrape.com/")
quotes = page.css(".quote .text::text").getall()

with FetcherSession(impersonate="chrome") as session:
    page = session.get("https://quotes.toscrape.com/", stealthy_headers=True)

page = DynamicFetcher.fetch("https://quotes.toscrape.com/", headless=True, network_idle=True)

with StealthySession(headless=True, solve_cloudflare=True) as session:
    page = session.fetch("https://nopecha.com/demo/cloudflare")
    page.css("#padded_content a").getall()
```

Extras: `ProxyRotator`, ad blocking (~3.5k domains) on browser fetchers, `AsyncStealthySession` tab pools (`get_pool_stats()`). Overlaps [[Browser Automation — Playwright]] for engine; Scrapling adds scrape defaults and parser integration.

---

## Adaptive Relocation

Fingerprints on first match; similarity search when DOM changes.

```python
from scrapling.fetchers import StealthyFetcher

StealthyFetcher.adaptive = True
page = StealthyFetcher.fetch("https://example.com/shop", headless=True, network_idle=True)

products = page.css(".product", auto_save=True)   # baseline — save fingerprints
products = page.css(".product", adaptive=True)  # later runs — relocate elements
```

| Flag | Role |
| --- | --- |
| `auto_save=True` | Persist fingerprints on first successful match |
| `adaptive=True` | Relocate when selector no longer hits |
| `StealthyFetcher.adaptive = True` | Class default for that fetcher family |

> [!warning] Major redesigns still need new selectors or LLM extractors ([[Browser Automation — ScrapeGraphAI]], [[AI — crawl4ai]]).

---

## Examples

**httpx + parser (no Scrapling fetcher):**

```python
import httpx
from scrapling.parser import Selector

page = Selector(httpx.get("https://example.com").text)
page.css("h1::text").get()
```

**Official adaptive + stealth one-liner:**

```python
from scrapling.fetchers import StealthyFetcher
StealthyFetcher.adaptive = True
p = StealthyFetcher.fetch("https://example.com", headless=True, network_idle=True)
p.css(".product", auto_save=True)
p.css(".product", adaptive=True)
```

**CLI** (`scrapling[shell]`):

```bash
scrapling extract get 'https://example.com' page.md
scrapling extract stealthy-fetch 'https://nopecha.com/demo/cloudflare' out.html \
  --css-selector '#padded_content a' --solve-cloudflare
```

---

## Patterns

**Pick tier:** static → `Fetcher.get` · JS → `DynamicFetcher.fetch` · blocked → `StealthyFetcher` · have HTML → `Selector(html)` only.

**Reuse sessions** — avoid `StealthyFetcher.fetch(url)` per URL in a loop; use `StealthySession` / `DynamicSession` or MCP `open_session` → `session_id` → `close_session` ([[AI — MCP]]).

**Vault pipelines:**

```text
httpx → Selector → [[Python — markdownify]] → [[AI — Chroma]]
StealthyFetcher + adaptive → JSON → [[AI — LangChain]]
scrapling mcp + css_selector → Cursor (fewer tokens)
```

Pair [[Python — tenacity]] for retries; [[Python — asyncio]] + `AsyncStealthySession` for parallel tabs.

| Mistake | Fix |
| --- | --- |
| Parser-only install, then `DynamicFetcher` | `pip install "scrapling[fetchers]"` + `scrapling install` |
| `adaptive=True` without baseline | `auto_save=True` first |
| Playwright test APIs expected | [[Browser Automation — Playwright]] for E2E |

---

## MCP — `scrapling[ai]`

Pre-extracts with CSS before content reaches the model — see [[AI — MCP]] for stdio/HTTP transport concepts.

```bash
pip install "scrapling[ai]"
scrapling install
```

```json
{
  "mcpServers": {
    "ScraplingServer": {
      "command": "scrapling",
      "args": ["mcp"]
    }
  }
}
```

Set `command` to `which scrapling` if PATH fails. Streamable HTTP: `scrapling mcp --http`.

| Tool | Role |
| --- | --- |
| `get` / `bulk_get` | HTTP + optional `css_selector` |
| `fetch` / `bulk_fetch` | JS browser |
| `stealthy_fetch` / `bulk_stealthy_fetch` | Anti-bot |
| `open_session` / `close_session` / `list_sessions` | Persistent browser |
| `screenshot` | PNG/JPEG from session |

Prompt example: *Use `bulk_get` on [urls] with `css_selector='.product'` for title and price.*

`from scrapling.core.ai import ScraplingMCPServer` for programmatic serving.

---

## Quick Reference

| Task | Command / code |
| --- | --- |
| Parser | `pip install scrapling` · `Selector(html)` |
| Fetchers | `pip install "scrapling[fetchers]"` · `scrapling install` |
| MCP | `pip install "scrapling[ai]"` · `scrapling mcp` |
| HTTP | `Fetcher.get(url)` |
| JS | `DynamicFetcher.fetch(url, network_idle=True)` |
| Stealth | `StealthyFetcher.fetch(url, solve_cloudflare=True)` |
| Adaptive | `css(sel, auto_save=True)` then `css(sel, adaptive=True)` |
| Text | `page.css('.x::text').getall()` |

**Related:** [[Browser Automation]] · [[Browser Automation — Playwright]] · [[AI — MCP]] · [[Python — httpx Package]] · [[Python — BeautifulSoup4 (bs4)]] · [[Python — tenacity]] · [[Browser Automation — Scrapy]] · [[Browser Automation — crawl4ai]]

---

## Tags

#browser-automation #scraping #scrapling #playwright #stealth #adaptive #mcp #python #web #backend
