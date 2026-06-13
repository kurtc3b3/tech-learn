## What & When

**Camoufox** is an open-source **anti-detect Firefox fork** with a Python wrapper around Playwright. Fingerprints are injected at the **C++ engine level** (via BrowserForge) — not via detectable JavaScript patches — and Playwright's page agent runs in an **isolated scope** so pages cannot see automation bindings.

Use Camoufox when:

- Sites block vanilla [[Browser Automation — Playwright]] / Chromium (fingerprinting, CDP leaks)
- You need **rotating device fingerprints** (OS, WebGL, screen, fonts, locale) per session
- **AI agent clusters** need lightweight headless Firefox (~200 MB vs Chrome 800 MB+)
- You already have Playwright scripts and want a **drop-in browser swap**

Prefer [[Browser Automation — Scrapling]] for adaptive selectors + tiered fetchers in one Python API. Prefer [[Browser Automation — Obscura]] for Rust CDP server + parallel scrape CLI. See [[Browser Automation]].

```bash
pip install camoufox
python -m camoufox fetch   # download pinned Firefox build
```

Docs: [camoufox.com](https://camoufox.com) · PyPI: `camoufox`

---

## Camoufox vs Playwright vs Scrapling

| Need | Prefer | Why |
| --- | --- | --- |
| Static HTML | [[Python — httpx Package]] | No browser |
| General JS scrape / E2E | [[Browser Automation — Playwright]] | Mature locators, tracing |
| Heavy anti-bot / fingerprint | **Camoufox** | Engine-level stealth, Firefox base |
| Cloudflare Turnstile (one API) | [[Browser Automation — Scrapling]] | `StealthyFetcher`, `solve_cloudflare` |
| Low-RAM CDP server | [[Browser Automation — Obscura]] | ~30 MB, Rust V8 engine |
| Agent CLI (snapshot refs) | [[Commands/CLI — agent-browser]] | Token-efficient a11y tree |

**Mental model:** Playwright API stays the same — swap `sync_playwright()` → `Camoufox()`.

---

## Sync API

Drop-in replacement for Playwright Firefox/Chromium launch:

```python
from camoufox.sync_api import Camoufox

with Camoufox(headless=False, os="windows", geoip=True) as browser:
    page = browser.new_page()
    page.goto("https://example.com", wait_until="domcontentloaded")
    print(page.title())
```

---

## Async API

```python
import asyncio
from camoufox.async_api import AsyncCamoufox

async def scrape_title(url: str) -> str:
    async with AsyncCamoufox(os=["windows", "macos"], humanize=True) as browser:
        page = await browser.new_page()
        await page.goto(url, wait_until="domcontentloaded")
        return await page.title()

asyncio.run(scrape_title("https://books.toscrape.com"))
```

---

## Fingerprint & Proxy

Camoufox generates realistic fingerprints with **BrowserForge** and can align geolocation with proxy IP:

```python
from camoufox.sync_api import Camoufox

proxy = {"server": "http://user:pass@proxy.example.com:8080"}

with Camoufox(
    os="windows",
    geoip=True,           # match timezone/locale to proxy exit IP
    proxy=proxy,
    humanize=True,        # natural cursor movement
    locale="en-US",
) as browser:
    page = browser.new_page()
    page.goto("https://example.com")
```

| Param | Role |
| --- | --- |
| `os` | `"windows"`, `"macos"`, `"linux"`, or list to randomize |
| `geoip` | `True` or target IP — sets lat/long, timezone, country |
| `humanize` | `True` or max cursor move seconds |
| `screen` | `browserforge.fingerprints.Screen` constraints |
| `fonts` | Extra font families installed on host |
| `block_images` | Save proxy bandwidth |
| `disable_coop` | Click cross-origin iframe widgets (e.g. Turnstile) |

> [!warning] Headless on macOS/Windows is easier to fingerprint than headed or Linux `headless="virtual"`. Prefer headed for hard targets; pair with rotating residential proxies.

---

## Persistent Sessions

Reuse cookies and profile data across runs:

```python
with Camoufox(
    persistent_context=True,
    user_data_dir="./camoufox-profile",
    headless=False,
) as context:
    page = context.new_page()
    page.goto("https://example.com/login")
    # complete login manually or via locators once
    page.context.storage_state(path="auth.json")
```

Load saved state on later runs via Playwright-compatible `storage_state` on new contexts.

---

## Cloudflare Turnstile Pattern

```python
from camoufox.sync_api import Camoufox

with Camoufox(disable_coop=True, window=(1280, 720), headless=False) as browser:
    page = browser.new_page()
    page.goto("https://nopecha.com/demo/cloudflare")
    page.wait_for_load_state("networkidle")
    page.wait_for_timeout(5000)
    page.mouse.click(210, 290)   # adjust from headed debug session
    page.wait_for_load_state("networkidle")
    print(page.content())
```

Pair with [[Python — tenacity]] for retries; respect site ToS and robots.txt.

---

## Playwright + bs4 Pipeline

Same pattern as [[Browser Automation — Playwright]] — render in Camoufox, parse with [[Python — BeautifulSoup4 (bs4)]]:

```python
from bs4 import BeautifulSoup
from camoufox.sync_api import Camoufox

def fetch_products(url: str) -> list[dict]:
    with Camoufox(os="macos", humanize=True) as browser:
        page = browser.new_page()
        page.goto(url, wait_until="domcontentloaded")
        html = page.content()
    soup = BeautifulSoup(html, "lxml")
    return [
        {"title": c.select_one("h3 a").get_text(strip=True)}
        for c in soup.select("article.product_pod")
    ]
```

---

## AI Agent Context

Camoufox targets **AI agent clusters** — debloated Firefox, cleaner DOM (less animation/telemetry noise), parallel sessions at lower RAM than Chromium. For agent **tooling** (snapshot refs, MCP), see [[Commands/CLI — agent-browser]] and [[Browser Automation — Obscura]] MCP mode.

---

## Quick Reference

| Task | Code |
| --- | --- |
| Install | `pip install camoufox && python -m camoufox fetch` |
| Sync | `from camoufox.sync_api import Camoufox` |
| Async | `from camoufox.async_api import AsyncCamoufox` |
| Random OS | `Camoufox(os=["windows", "macos", "linux"])` |
| Proxy + geo | `Camoufox(geoip=True, proxy={...})` |
| Human cursor | `Camoufox(humanize=True)` |
| Persist profile | `persistent_context=True, user_data_dir="..."` |
| Navigate | `page.goto(url, wait_until="domcontentloaded")` |

---

## Related Notes

- [[Browser Automation]]
- [[Browser Automation — Playwright]]
- [[Browser Automation — Scrapling]]
- [[Browser Automation — Obscura]]
- [[Commands/CLI — agent-browser]]
- [[Python — BeautifulSoup4 (bs4)]]
- [[Python — tenacity]]
- [[AI]]

---

## Tags

#browser-automation #camoufox #stealth #playwright #scraping #firefox #anti-bot #python #ai-agents #web
