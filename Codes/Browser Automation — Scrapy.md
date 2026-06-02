## What & When

**Scrapy** is a Python framework for **large-scale web crawling** — request scheduling, concurrency, retries, middleware, pipelines, and feed exports built in.

Use Scrapy when:

- Crawling **hundreds or thousands** of URLs with link following
- You need **pipelines** (DB, validation) and **middleware** (headers, proxies)
- **Politeness** — delays, concurrency caps, `robots.txt`
- Output to **JSON/CSV/XML** via feed exports

Use [[Browser Automation — Playwright]] or **scrapy-playwright** for heavy JS. Use [[Python — httpx Package]] for small API jobs. See [[Browser Automation]].

```bash
pip install scrapy
scrapy startproject myproject
```

---

## Scrapy vs Playwright vs httpx

| | Scrapy | Playwright | httpx + bs4 |
| --- | --- | --- | --- |
| Scale | ✅ Thousands | Small–medium | Small |
| Crawl engine | ✅ Built-in | ❌ Manual | ❌ Manual |
| JS rendering | ⚠️ scrapy-playwright | ✅ | ❌ |
| Pipelines | ✅ First-class | DIY | DIY |
| Best for | Site-wide ETL | Login, SPA | Static pages |

---

## Project Layout

```text
myproject/
├── scrapy.cfg
└── myproject/
    ├── items.py
    ├── middlewares.py
    ├── pipelines.py
    ├── settings.py       # ROBOTSTXT, FEEDS, concurrency
    └── spiders/books.py
```

```bash
scrapy crawl books -o books.json
scrapy shell "https://books.toscrape.com"
```

---

## Spider

```python
# spiders/books.py
import scrapy

class BooksSpider(scrapy.Spider):
    name = "books"
    allowed_domains = ["books.toscrape.com"]
    start_urls = ["https://books.toscrape.com/"]
    custom_settings = {"DOWNLOAD_DELAY": 0.5}

    def parse(self, response):
        for book in response.css("article.product_pod"):
            yield {
                "title": book.css("h3 a::attr(title)").get(),
                "price": book.css("p.price_color::text").get(),
                "url":   response.urljoin(book.css("h3 a::attr(href)").get()),
            }
        next_page = response.css("li.next a::attr(href)").get()
        if next_page:
            yield response.follow(next_page, callback=self.parse)
```

### `CrawlSpider` — `Rule(LinkExtractor(...), callback=..., follow=True)` for declarative link following.

---

## Item

```python
# items.py
import scrapy

class BookItem(scrapy.Item):
    title = scrapy.Field()
    price = scrapy.Field()
    url   = scrapy.Field()

# In spider:
item = BookItem()
item["title"] = response.css("h1::text").get()
yield item
```

---

## Pipeline

Process items in order (`ITEM_PIPELINES` — lower number runs first).

```python
# pipelines.py
import scrapy
from itemadapter import ItemAdapter

class PricePipeline:
    def process_item(self, item, spider):
        adapter = ItemAdapter(item)
        price = adapter.get("price", "")
        adapter["price"] = float(price.replace("£", "")) if price else 0.0
        return item

class DuplicatesPipeline:
    def __init__(self):
        self.seen: set[str] = set()

    def process_item(self, item, spider):
        url = ItemAdapter(item)["url"]
        if url in self.seen:
            raise scrapy.exceptions.DropItem(f"Duplicate: {url}")
        self.seen.add(url)
        return item
```

```python
# settings.py
ITEM_PIPELINES = {
    "myproject.pipelines.PricePipeline": 100,
    "myproject.pipelines.DuplicatesPipeline": 200,
}
```

---

## Middleware

Downloader: `process_request` / `process_response` — headers, proxies, retries. Spider: `process_spider_output` — filter responses.

```python
class RandomUserAgentMiddleware:
    def process_request(self, request, spider):
        request.headers["User-Agent"] = "MyCrawler/1.0"
        return None

# settings.py
DOWNLOADER_MIDDLEWARES = {
    "myproject.middlewares.RandomUserAgentMiddleware": 400,
    "scrapy.downloadermiddlewares.useragent.UserAgentMiddleware": None,
}
```

---

## settings.py — ROBOTSTXT & Throttle

```python
ROBOTSTXT_OBEY = True

CONCURRENT_REQUESTS = 16
CONCURRENT_REQUESTS_PER_DOMAIN = 8
DOWNLOAD_DELAY = 0.25
AUTOTHROTTLE_ENABLED = True
AUTOTHROTTLE_START_DELAY = 0.5
AUTOTHROTTLE_MAX_DELAY = 10.0

DOWNLOAD_TIMEOUT = 30
RETRY_TIMES = 3
RETRY_HTTP_CODES = [500, 502, 503, 504, 408, 429]

HTTPCACHE_ENABLED = True          # dev re-runs
LOG_LEVEL = "INFO"
```

> [!warning] Set `ROBOTSTXT_OBEY = False` only when you own the site or have permission.

---

## Feed Exports

```bash
scrapy crawl books -o books.json
scrapy crawl books -o books.csv
scrapy crawl books -o books.jl      # JSON Lines
```

```python
FEEDS = {
    "output/books-%(time)s.json": {
        "format": "json",
        "encoding": "utf8",
        "indent": 2,
    },
    "s3://bucket/books.csv": {
        "format": "csv",
        "fields": ["title", "price", "url"],
    },
}
```

| Format | Extension |
| --- | --- |
| JSON | `.json` |
| JSON Lines | `.jl`, `.jsonl` |
| CSV | `.csv` |
| XML | `.xml` |

---

## Selectors

`response.css("h3 a::text").get()` · `response.css("a::attr(href)").get()` · `response.xpath("//h1/text()").get()` · `yield response.follow(href, callback=self.parse_detail, meta={...})`

---

## scrapy-playwright (Optional)

For JS-heavy pages without leaving Scrapy's scheduler.

```bash
pip install scrapy-playwright
playwright install
```

```python
# settings.py
DOWNLOAD_HANDLERS = {
    "http": "scrapy_playwright.handler.ScrapyPlaywrightDownloadHandler",
    "https": "scrapy_playwright.handler.ScrapyPlaywrightDownloadHandler",
}
TWISTED_REACTOR = "twisted.internet.asyncioreactor.AsyncioSelectorReactor"
PLAYWRIGHT_BROWSER_TYPE = "chromium"
```

```python
class SpaSpider(scrapy.Spider):
    name = "spa"
    start_urls = ["https://spa.example.com/"]

    def start_requests(self):
        yield scrapy.Request(
            self.start_urls[0],
            meta={"playwright": True, "playwright_include_page": True},
            callback=self.parse,
        )

    async def parse(self, response):
        page = response.meta["playwright_page"]
        await page.wait_for_selector(".loaded")
        title = await page.title()
        await page.close()
        yield {"title": title, "url": response.url}
```

See [[Browser Automation — Playwright]]. Scrapy owns scheduling; Playwright renders.

---

## Async Note

Scrapy uses **Twisted**, not [[Python — asyncio]] — scrapy-playwright bridges an asyncio reactor. From code: `CrawlerProcess(settings={...}).crawl(Spider); process.start()`.

---

## Backend Patterns

### Celery Scheduled Crawl

```python
from celery import shared_task
import subprocess

@shared_task
def run_books_crawl():
    subprocess.run(
        ["scrapy", "crawl", "books", "-o", "/data/books.json"],
        cwd="/app/myproject", check=True,
    )
```

### FastAPI Trigger

```python
from fastapi import BackgroundTasks
import subprocess

@app.post("/admin/crawl")
async def trigger(background_tasks: BackgroundTasks):
    background_tasks.add_task(
        lambda: subprocess.Popen(
            ["scrapy", "crawl", "books", "-o", "/tmp/books.json"],
            cwd="/app/myproject",
        )
    )
    return {"status": "started"}
```

`SitemapSpider` — `sitemap_urls` + `sitemap_rules` for sitemap-driven crawls.

---

## Common Mistakes

| Mistake | Fix |
| --- | --- |
| Empty JS page | scrapy-playwright or pre-render |
| Blocked | Check `ROBOTSTXT_OBEY` and rate limits |
| DB in spider | Move writes to pipeline |
| Memory spike | Lower `CONCURRENT_REQUESTS`; stream feeds |

---

## Quick Reference

| Task | Code |
| --- | --- |
| Install | `pip install scrapy` |
| New project | `scrapy startproject myproject` |
| Run + export | `scrapy crawl books -o out.json` |
| Shell | `scrapy shell URL` |
| CSS text | `response.css("h1::text").get()` |
| Follow | `yield response.follow(href, callback=self.parse)` |
| Robots | `ROBOTSTXT_OBEY = True` |
| Delay | `DOWNLOAD_DELAY = 0.5` |
| Feeds | `FEEDS = {"out.json": {"format": "json"}}` |
| JS | scrapy-playwright + `meta={"playwright": True}` |

---

## Related Notes

- [[Browser Automation]]
- [[Browser Automation — Playwright]]
- [[Python — asyncio]]
- [[Python — BeautifulSoup4 (bs4)]]
- [[Python — httpx Package]]

---

## Tags

#browser-automation #scrapy #scraping #crawl #python #pipeline #middleware #backend #web
