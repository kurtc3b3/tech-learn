## What & When

**crawl4ai** is an async Python crawler built for **LLM-ready output**: it drives a real browser (Playwright under the hood), converts pages to cleaned **Markdown**, and supports extraction, chunking, and caching in one pipeline. It targets **RAG ingest** and research crawls more than brittle CSS scrapers.

Use crawl4ai when:

- You need **JS-rendered** pages as Markdown for [[AI]] / vector stores
- Building **crawl → chunk → embed** pipelines without hand-rolling [[Python — markdownify]]
- Batch async crawls with `arun_many()` and per-URL config
- You want built-in content filters, cache, and optional LLM extraction — not raw [[Browser Automation — Playwright]] scripts

```bash
pip install crawl4ai

# Installs browser deps and runs first-time setup (preferred)
crawl4ai-setup

# Or manually (same engine as Playwright)
playwright install chromium
```

Overview: [[Browser Automation]]. For prompt/schema graphs instead of Markdown-first crawl, see [[Browser Automation — ScrapeGraphAI]].

---

## crawl4ai vs Related Tools

| Need | Use | Notes |
| --- | --- | --- |
| LLM-ready Markdown + crawl | **crawl4ai** | `result.markdown`, chunking, cache |
| Manual browser control | [[Browser Automation — Playwright]] | Full E2E, custom flows |
| httpx + HTML → MD | [[Python — httpx Package]] + [[Python — markdownify]] | Static pages only |
| Prompt-driven JSON extract | [[Browser Automation — ScrapeGraphAI]] | `SmartScraperGraph` |
| Many URLs, pipelines | [[Browser Automation — Scrapy]] | Distributed crawl |

crawl4ai **includes** HTML→Markdown; you only need markdownify when you already have HTML from another source.

---

## Core API — AsyncWebCrawler & arun

Configuration splits into two objects (v0.7+):

| Config | Scope | Examples |
| --- | --- | --- |
| `BrowserConfig` | Session / browser | `headless`, `browser_type`, proxy |
| `CrawlerRunConfig` | Single `arun()` | cache, filters, wait, extraction |

```python
import asyncio
from crawl4ai import AsyncWebCrawler, BrowserConfig, CrawlerRunConfig, CacheMode

async def main() -> None:
    browser_cfg = BrowserConfig(headless=True)
    run_cfg = CrawlerRunConfig(
        cache_mode=CacheMode.BYPASS,
        excluded_tags=["nav", "footer", "script"],
        word_count_threshold=10,
    )

    async with AsyncWebCrawler(config=browser_cfg) as crawler:
        result = await crawler.arun(
            url="https://example.com",
            config=run_cfg,
        )

    if result.success:
        print(result.markdown[:500])
        print("Links:", len(result.links.get("internal", [])))
    else:
        print(result.error_message)

asyncio.run(main())
```

> [!tip] Reuse one `AsyncWebCrawler` Create the crawler once per process; pass a fresh or reused `CrawlerRunConfig` to each `arun()` call.

---

## Markdown Output

`arun()` returns a `CrawlResult`. Primary fields for downstream [[AI]] work:

| Field | Use |
| --- | --- |
| `result.markdown` | Cleaned Markdown (default generator) |
| `result.markdown_v2` | Richer generator output when configured |
| `result.cleaned_html` | Filtered HTML before conversion |
| `result.extracted_content` | JSON from `extraction_strategy` |
| `result.screenshot` | Base64 PNG if `screenshot=True` |

```python
from crawl4ai import (
    AsyncWebCrawler,
    CrawlerRunConfig,
    DefaultMarkdownGenerator,
    PruningContentFilter,
)

md_generator = DefaultMarkdownGenerator(
    content_filter=PruningContentFilter(threshold=0.48),
    options={"ignore_links": False, "body_width": 0},
)

run_cfg = CrawlerRunConfig(
    markdown_generator=md_generator,
    target_elements=["article", "main"],   # focus MD on these
    css_selector="article.post",           # or narrow whole page
)

async with AsyncWebCrawler() as crawler:
    result = await crawler.arun("https://docs.example.com/guide", config=run_cfg)
    md = result.markdown or ""
```

Compared to [[Python — markdownify]]: crawl4ai fetches, waits for JS, prunes boilerplate, then emits Markdown. Use markdownify when you already control the HTML string.

---

## Dynamic Pages & Waiting

```python
run_cfg = CrawlerRunConfig(
    wait_for="css:.article-loaded",      # Playwright wait selector
    page_timeout=60_000,
    delay_before_return_html=2.0,        # extra settle time
    js_code=[
        "window.scrollTo(0, document.body.scrollHeight);",
    ],
    scan_full_page=True,                 # lazy-load by scrolling
)
```

For login or multi-step flows, combine session options in `CrawlerRunConfig` (`session_id`) or drop to [[Browser Automation — Playwright]] for full control.

---

## Chunking for RAG

Chunk before embedding so retrievers get right-sized passages. crawl4ai can chunk via `chunking_strategy` on `CrawlerRunConfig` or via markdown generator options.

```python
from crawl4ai import AsyncWebCrawler, CrawlerRunConfig
from crawl4ai.chunking_strategy import RegexChunking

run_cfg = CrawlerRunConfig(
    chunking_strategy=RegexChunking(
        patterns=[r"\n#{1,3}\s", r"\n\n"],   # split on headings / paragraphs
        max_chunk_size=1500,
        overlap=100,
    ),
)

async with AsyncWebCrawler() as crawler:
    result = await crawler.arun("https://example.com/docs", config=run_cfg)

chunks = result.markdown_chunks or []   # list of chunk strings + metadata
for i, chunk in enumerate(chunks):
    print(i, chunk[:80], "...")
```

Typical RAG pipeline:

```text
arun() → markdown / chunks → embed ([[AI — LangChain]], [[AI — Chroma]]) → store
```

```python
# Minimal handoff to LangChain-style ingest (pseudo)
documents = [
    {"page_content": c, "metadata": {"url": result.url, "chunk": i}}
    for i, c in enumerate(chunks)
]
```

Pair with [[AI]] embedding models; tune `max_chunk_size` to your context window and retriever `k`.

---

## Batch Crawls — arun_many

```python
urls = [
    "https://example.com/page1",
    "https://example.com/page2",
]

run_cfg = CrawlerRunConfig(
    cache_mode=CacheMode.ENABLED,
    stream=True,              # process results as they finish
)

async with AsyncWebCrawler() as crawler:
    async for result in await crawler.arun_many(urls, config=run_cfg):
        if result.success:
            print(result.url, len(result.markdown or ""))
```

| Parameter | Purpose |
| --- | --- |
| `semaphore_count` | Max parallel tabs (default ~5) |
| `mean_delay` / `max_range` | Jitter between requests |
| `check_robots_txt` | Respect robots when enabled |

For site-wide rules and pipelines at scale, prefer [[Browser Automation — Scrapy]]; use crawl4ai when each page must render JS and become Markdown.

---

## LLM Extraction (Optional)

When you need structured fields, not full-page Markdown:

```python
from crawl4ai import LLMConfig, LLMExtractionStrategy

llm_cfg = LLMConfig(
    provider="openai/gpt-4o-mini",
    api_token="sk-...",   # or env var supported by provider
)

strategy = LLMExtractionStrategy(
    llm_config=llm_cfg,
    instruction="Extract product name, price, and SKU as JSON.",
)

run_cfg = CrawlerRunConfig(extraction_strategy=strategy)

async with AsyncWebCrawler() as crawler:
    result = await crawler.arun("https://shop.example.com/item/1", config=run_cfg)
    print(result.extracted_content)
```

For **prompt-only** graphs without crawl4ai's Markdown pipeline, [[Browser Automation — ScrapeGraphAI]] is often simpler.

---

## Caching & Politeness

```python
from crawl4ai import CacheMode

# Read/write disk cache (faster re-runs)
CrawlerRunConfig(cache_mode=CacheMode.ENABLED)

# Always refetch
CrawlerRunConfig(cache_mode=CacheMode.BYPASS)

# Read only, never write
CrawlerRunConfig(cache_mode=CacheMode.READ_ONLY)
```

Enable `check_robots_txt=True` for production crawls. Combine with [[Python — tenacity]] at your orchestration layer for retries on transient failures.

---

## Docker / CI Setup

```bash
# After pip install in CI images
crawl4ai-setup
# or
playwright install --with-deps chromium
```

```dockerfile
RUN pip install crawl4ai && crawl4ai-setup
```

Match local and CI browser installs — missing Chromium is the most common deploy failure.

---

## Common Mistakes

| Mistake | Fix |
| --- | --- |
| `playwright` / Chromium not installed | Run `crawl4ai-setup` or `playwright install` |
| Empty `result.markdown` | Set `wait_for`, increase `page_timeout`, check `target_elements` |
| Huge nav/footer in MD | `excluded_tags`, `PruningContentFilter`, or `css_selector` |
| Passing old `arun(url, headless=True)` kwargs | Move options into `BrowserConfig` / `CrawlerRunConfig` |
| Embedding whole page without chunks | Use `chunking_strategy` or split in [[AI — LangChain]] |
| Using crawl4ai for static blogs only | Start with httpx + bs4 — faster and cheaper |

---

## Quick Reference

| Task | API / command |
| --- | --- |
| Install | `pip install crawl4ai && crawl4ai-setup` |
| Crawler | `async with AsyncWebCrawler(config=...) as crawler` |
| Single URL | `await crawler.arun(url, config=run_cfg)` |
| Many URLs | `await crawler.arun_many(urls, config=run_cfg)` |
| Markdown | `result.markdown` |
| Chunks | `CrawlerRunConfig(chunking_strategy=...)` |
| Browser session | `BrowserConfig(headless=True, ...)` |
| Per-crawl behavior | `CrawlerRunConfig(...)` |

---

## Related Notes

- [[Browser Automation]]
- [[Browser Automation — Playwright]]
- [[Browser Automation — ScrapeGraphAI]]
- [[Python — markdownify]]
- [[Python — BeautifulSoup4 (bs4)]]
- [[Python — httpx Package]]
- [[Python — asyncio]]
- [[AI]]
- [[AI — LangChain]]
- [[AI — Chroma]]

---

## Tags

#browser-automation #crawl4ai #scraping #playwright #rag #markdown #python #async #ai
