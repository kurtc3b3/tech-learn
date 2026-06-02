## What & When

**Playwright** drives Chromium, Firefox, and WebKit with auto-waiting locators — ideal for JavaScript-heavy pages, login flows, screenshots, and reliable UI interaction.

Use Playwright when:

- The page requires **JavaScript rendering** (SPAs, infinite scroll)
- You need **login**, cookies, or multi-step flows
- You want **screenshots**, traces, or network interception
- A **small batch** of URLs — not a fleet-scale crawl

Prefer [[Python — httpx Package]] + [[Python — BeautifulSoup4 (bs4)]] for static HTML. Prefer [[Browser Automation — Scrapy]] for thousands of URLs. See [[Browser Automation]].

```bash
pip install playwright
playwright install              # browser binaries
playwright install chromium     # one browser only
```

---

## Playwright vs Selenium vs httpx

| | Playwright | Selenium | httpx + bs4 |
| --- | --- | --- | --- |
| JS rendering | ✅ | ✅ | ❌ |
| Auto-wait locators | ✅ | ⚠️ Manual | N/A |
| Async API | ✅ Native | Limited | ✅ |
| Overhead | Medium | Higher | Lowest |
| Best for | Modern scrape, E2E | Legacy stacks | Static HTML, APIs |

---

## Sync API

```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.launch(headless=True)
    page = browser.new_page()
    page.goto("https://books.toscrape.com", wait_until="domcontentloaded")
    titles = page.locator("article h3 a").all_inner_texts()
    browser.close()
```

---

## Async API

Works with [[Python — asyncio]] — reuse one browser, many contexts/pages per `async_playwright()` block.

```python
import asyncio
from playwright.async_api import async_playwright

async def scrape_titles(url: str) -> list[str]:
    async with async_playwright() as p:
        browser = await p.chromium.launch(headless=True)
        page = await browser.new_page()
        await page.goto(url, wait_until="networkidle")
        titles = await page.locator("article h3 a").all_inner_texts()
        await browser.close()
        return titles

# asyncio.run(asyncio.gather(*(scrape_titles(u) for u in urls)))
```

---

## `page.goto`

```python
page.goto(url)                                    # default wait_until="load"
page.goto(url, wait_until="domcontentloaded")     # DOM ready — common for scraping
page.goto(url, wait_until="networkidle")          # no network 500ms — SPAs
page.goto(url, timeout=60_000)                    # ms; raises TimeoutError
```

| `wait_until` | Use when |
| --- | --- |
| `domcontentloaded` | DOM parseable — most scraping |
| `networkidle` | Client fetches settled |
| `load` | Full page incl. images |

---

## Locators

Auto-retry until actionable. Prefer over raw `query_selector`.

```python
page.locator("article h3 a")
page.get_by_role("button", name="Sign in")
page.get_by_text("Add to basket", exact=True)
page.get_by_label("Email")
page.locator("article").filter(has_text="Science").locator("h3 a")

page.locator("h3 a").first.inner_text()
page.locator("h3 a").all_inner_texts()
page.get_by_role("button", name="Submit").click()
page.get_by_label("Password").fill("secret")
```

---

## Waits

```python
from playwright.sync_api import expect

page.locator(".results").wait_for(state="visible", timeout=10_000)
expect(page.locator("h1")).to_have_text("Books to Scrape")
expect(page.locator("article")).to_have_count(20)

with page.expect_navigation():
    page.get_by_role("link", name="Next").click()

with page.expect_response("**/api/products**") as resp_info:
    page.get_by_role("button", name="Load more").click()
data = resp_info.value.json()

page.wait_for_timeout(1000)   # last resort — prefer locator waits
```

---

## Screenshots & Tracing

```python
page.screenshot(path="page.png", full_page=True)
page.locator(".product").first.screenshot(path="product.png")
page.pdf(path="report.pdf", format="A4")   # Chromium only

context.tracing.start(screenshots=True, snapshots=True)
page.goto(url)
context.tracing.stop(path="trace.zip")     # playwright show-trace trace.zip
```

---

## Context, Cookies, Storage

**Browser** = process; **context** = isolated session (cookies, storage).

```python
context = browser.new_context(
    user_agent="MyBot/1.0",
    viewport={"width": 1280, "height": 720},
    extra_http_headers={"Accept-Language": "en"},
)
page = context.new_page()

# Save / restore login session
context.storage_state(path="auth.json")
context2 = browser.new_context(storage_state="auth.json")

context.add_cookies([
    {"name": "session", "value": "abc", "domain": "example.com", "path": "/"},
])
```

---

## Scraping Patterns

### Playwright + bs4

Render in Playwright; parse with [[Python — BeautifulSoup4 (bs4)]].

```python
from bs4 import BeautifulSoup
from playwright.sync_api import sync_playwright

def scrape_products(url: str) -> list[dict]:
    with sync_playwright() as p:
        browser = p.chromium.launch(headless=True)
        page = browser.new_page()
        page.goto(url, wait_until="domcontentloaded")
        for _ in range(5):
            page.evaluate("window.scrollTo(0, document.body.scrollHeight)")
            page.locator(".product-card").last.wait_for(state="visible", timeout=5_000)
        html = page.content()
        browser.close()
    soup = BeautifulSoup(html, "lxml")
    return [
        {"title": c.select_one("h2").get_text(strip=True),
         "price": c.select_one(".price").get_text(strip=True)}
        for c in soup.select(".product-card")
    ]
```

### Intercept JSON API

```python
products: list[dict] = []
def on_response(response):
    if "/api/products" in response.url and response.ok:
        products.extend(response.json()["items"])
page.on("response", on_response)
page.goto("https://shop.example.com")
```

### Resilience — [[Python — tenacity]]

```python
from tenacity import retry, stop_after_attempt, wait_exponential
from playwright.sync_api import sync_playwright, TimeoutError as PWTimeout

@retry(stop=stop_after_attempt(3), wait=wait_exponential(min=2, max=30), reraise=True)
def fetch_html(url: str) -> str:
    with sync_playwright() as p:
        browser = p.chromium.launch(headless=True)
        page = browser.new_page()
        try:
            page.goto(url, wait_until="domcontentloaded", timeout=30_000)
            return page.content()
        finally:
            browser.close()
```

---

## Backend Patterns

### FastAPI — Screenshot Service (async, shared browser)

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI, HTTPException
from fastapi.responses import Response
from playwright.async_api import async_playwright, Browser

browser: Browser | None = None

@asynccontextmanager
async def lifespan(app: FastAPI):
    global browser
    pw = await async_playwright().start()
    browser = await pw.chromium.launch(headless=True)
    yield
    await browser.close()
    await pw.stop()

app = FastAPI(lifespan=lifespan)

@app.get("/screenshot")
async def screenshot(url: str):
    if not url.startswith("https://"):
        raise HTTPException(400, "HTTPS only")
    ctx = await browser.new_context()
    page = await ctx.new_page()
    try:
        await page.goto(url, wait_until="domcontentloaded", timeout=30_000)
        return Response(await page.screenshot(full_page=True), media_type="image/png")
    finally:
        await ctx.close()
```

### Cron Job — Reuse `storage_state`

```python
def job_scrape_dashboard():
    with sync_playwright() as p:
        browser = p.chromium.launch(headless=True)
        ctx = browser.new_context(storage_state="auth.json")
        page = ctx.new_page()
        page.goto("https://internal.example.com/reports")
        rows = page.locator("table tbody tr").all_inner_texts()
        browser.close()
    return rows
```

---

`playwright codegen URL` · `playwright show-trace trace.zip` · Docker: `args=["--disable-dev-shm-usage"]`

---

## Quick Reference

| Task | Code |
| --- | --- |
| Install | `pip install playwright && playwright install` |
| Sync | `with sync_playwright() as p:` |
| Async | `async with async_playwright() as p:` |
| Launch | `p.chromium.launch(headless=True)` |
| Navigate | `page.goto(url, wait_until="domcontentloaded")` |
| Locator | `page.locator("css")` / `get_by_role(...)` |
| Click / fill | `locator.click()` / `locator.fill("x")` |
| HTML | `page.content()` |
| Screenshot | `page.screenshot(path="x.png", full_page=True)` |
| Session | `context.storage_state(path="auth.json")` |
| Load session | `new_context(storage_state="auth.json")` |

---

## Related Notes

- [[Browser Automation]]
- [[Browser Automation — Scrapy]]
- [[Python — asyncio]]
- [[Python — BeautifulSoup4 (bs4)]]
- [[Python — tenacity]]
- [[Python — httpx Package]]

---

## Tags

#browser-automation #playwright #scraping #python #asyncio #headless #backend #web
