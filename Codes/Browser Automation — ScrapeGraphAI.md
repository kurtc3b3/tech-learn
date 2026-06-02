## What & When

**ScrapeGraphAI** is a Python library that builds **LLM-powered scraping graphs**: you give a natural-language **prompt** and a **source** (URL, HTML, or local file), and a pipeline fetches the page (via Playwright), parses content, and returns **structured JSON** — without maintaining fragile CSS selectors.

Use ScrapeGraphAI when:

- Page layout changes often and selectors break
- You need **schema-like extraction** from one or many URLs via prompts
- Integrating with OpenAI, Anthropic, Ollama, or Google models through a single `graph_config`
- You want a higher-level alternative to hand-written [[Browser Automation — Playwright]] + [[AI — LangChain]] chains

```bash
pip install scrapegraphai

# Required — ScrapeGraphAI uses Playwright for live pages
playwright install
# or
playwright install chromium
```

Overview: [[Browser Automation]]. For Markdown-first RAG crawls, prefer [[Browser Automation — crawl4ai]].

---

## ScrapeGraphAI vs Related Tools

| Need | Use | Notes |
| --- | --- | --- |
| Prompt → JSON from URL | **SmartScraperGraph** | Default single-page graph |
| Markdown / chunk / cache crawl | [[Browser Automation — crawl4ai]] | `AsyncWebCrawler`, `result.markdown` |
| Full browser scripting | [[Browser Automation — Playwright]] | Login, custom flows |
| Composable LLM chains | [[AI — LangChain]] | RAG, agents, tools |
| Static HTML, no browser | [[Python — httpx Package]] + [[Python — BeautifulSoup4 (bs4)]] | Cheapest path |

ScrapeGraphAI **costs LLM tokens** per page; cache results and use cheaper models for exploration.

---

## Core API — SmartScraperGraph

The most common graph: **one prompt + one source → structured result**.

```python
import json
import os
from scrapegraphai.graphs import SmartScraperGraph

graph_config = {
    "llm": {
        "api_key": os.environ["OPENAI_API_KEY"],
        "model": "openai/gpt-4o-mini",
        "temperature": 0,
    },
    "verbose": True,
    "headless": True,
}

smart_scraper = SmartScraperGraph(
    prompt=(
        "Extract the company name, what they do, founders, "
        "and social media links as JSON."
    ),
    source="https://scrapegraphai.com/",
    config=graph_config,
)

result = smart_scraper.run()
print(json.dumps(result, indent=2, ensure_ascii=False))
```

| Argument | Role |
| --- | --- |
| `prompt` | Natural-language extraction instructions |
| `source` | URL string, or path to HTML/JSON/Markdown file |
| `config` | LLM + browser settings (`llm`, `headless`, `verbose`, …) |
| `schema` | Optional Pydantic model for enforced output shape |

---

## LLM Configuration

Models use a **`provider/model`** string. API keys usually come from env vars or `graph_config["llm"]`.

```python
# OpenAI
graph_config = {
    "llm": {
        "api_key": os.environ["OPENAI_API_KEY"],
        "model": "openai/gpt-4o-mini",
        "format": "json",
    },
    "headless": True,
}

# Ollama (local)
graph_config = {
    "llm": {
        "model": "ollama/llama3.2",
        "model_tokens": 8192,
        "format": "json",
    },
    "headless": False,
}

# Google Gemini
graph_config = {
    "llm": {
        "api_key": os.environ["GEMINI_API_KEY"],
        "model": "google_genai/gemini-2.0-flash",
    },
    "headless": True,
}
```

| Key | Purpose |
| --- | --- |
| `model` | Provider id, e.g. `openai/gpt-4o`, `ollama/mistral` |
| `api_key` | Provider token (omit for Ollama if local) |
| `temperature` | Lower (0–0.2) for deterministic extraction |
| `format` | `"json"` when you need parseable output |
| `model_tokens` | Context limit hint for local models |

Load secrets with `python-dotenv` in apps; never commit keys. Same discipline as [[AI — LangChain]] `ChatOpenAI(api_key=...)`.

---

## Writing Effective Prompts

Treat the prompt as your **extraction schema** in prose:

```python
PROMPT = """
You are extracting product data from an e-commerce product page.
Return a single JSON object with these fields:
- name (string)
- price (string, include currency)
- stock_availability (string)
- short_description (string)
- sku (string)
- category (string)

Rules:
- Only return valid JSON, no markdown fences or commentary.
- If a field is missing, use null.
"""
```

Tips:

- Name fields explicitly and specify types
- Say **JSON only** to reduce preamble text
- For nested data, describe object shapes in the prompt or use `schema=`
- Start with `headless=False` and `verbose=True` while debugging fetch issues

---

## Structured Output with Pydantic

```python
from pydantic import BaseModel, Field
from scrapegraphai.graphs import SmartScraperGraph

class Product(BaseModel):
    name: str = Field(description="Product title")
    price: str = Field(description="Price with currency")
    sku: str | None = None

graph = SmartScraperGraph(
    prompt="Extract product information from this page.",
    source="https://shop.example.com/product/123",
    schema=Product,
    config=graph_config,
)

data = graph.run()   # validated dict matching Product
```

Aligns with [[Python — Pydantic]] patterns in [[API - FastAPI]] and [[AI — LangChain]] tool schemas.

---

## Multi-Page — SmartScraperMultiGraph

Same prompt across several URLs; graph aggregates results.

```python
from scrapegraphai.graphs import SmartScraperMultiGraph

multi_graph = SmartScraperMultiGraph(
    prompt="Extract name, role, and email for each team member mentioned.",
    source=[
        "https://example.com/about",
        "https://example.com/team",
        "https://example.com/contact",
    ],
    schema=None,
    config=graph_config,
)

result = multi_graph.run()
```

For search-result crawling (top N Google hits), see `SearchGraph` in the upstream docs. For high-volume politeness and middleware, use [[Browser Automation — Scrapy]] and reserve ScrapeGraphAI for extract steps.

---

## Other Graphs (Overview)

| Graph | Purpose |
| --- | --- |
| `SmartScraperGraph` | Single URL → JSON via prompt |
| `SmartScraperMultiGraph` | List of URLs, one prompt |
| `SearchGraph` | Search engine results → extract |
| `ScriptCreatorGraph` | Generate reusable Python scrape script |
| `SpeechGraph` | Scrape + TTS audio output |

Most projects only need **SmartScraperGraph** or **SmartScraperMultiGraph**.

---

## End-to-End Script Pattern

```python
import json
import os
from pathlib import Path

from dotenv import load_dotenv
from scrapegraphai.graphs import SmartScraperGraph

load_dotenv()

graph_config = {
    "llm": {
        "api_key": os.environ["OPENAI_API_KEY"],
        "model": "openai/gpt-4o-mini",
        "format": "json",
    },
    "headless": True,
    "verbose": True,
}

TARGET = "https://www.scrapingcourse.com/ecommerce/product/aim-analog-watch/"
PROMPT = "Extract product name, price, availability, and description as JSON."

def main() -> None:
    graph = SmartScraperGraph(
        prompt=PROMPT,
        source=TARGET,
        config=graph_config,
    )
    data = graph.run()
    Path("product_output.json").write_text(
        json.dumps(data, indent=2, ensure_ascii=False),
        encoding="utf-8",
    )
    print("Wrote product_output.json")

if __name__ == "__main__":
    main()
```

---

## Feeding LangChain / RAG

ScrapeGraphAI returns **structured records**, not chunked Markdown. Typical flow:

```text
SmartScraperGraph.run() → JSON files / rows → LangChain Document → embed → vector store
```

```python
from langchain_core.documents import Document

# After graph.run()
docs = [
    Document(
        page_content=json.dumps(data, ensure_ascii=False),
        metadata={"source": TARGET, "type": "product"},
    )
]
# → text_splitter → embeddings → [[AI — Chroma]]  (see [[AI — LangChain]])
```

For **page-level Markdown chunks**, crawl with [[Browser Automation — crawl4ai]] first, then index in LangChain.

---

## Browser & Playwright Layer

ScrapeGraphAI renders pages through Playwright (`headless` in `graph_config`). It does not replace learning [[Browser Automation — Playwright]] for:

- Authenticated sessions (OAuth, 2FA)
- Complex multi-tab workflows
- Custom network interception

Set `headless=False` to watch the browser during development. Production CI: `playwright install --with-deps chromium`.

---

## Hosted API (Optional)

ScrapeGraphAI also offers a cloud **SmartScraper** API (`scrapegraph_py` SDK) with `website_url` + `user_prompt` + optional `output_schema`. Use the open-source graphs when you need full control, local models, or no external API dependency.

---

## Common Mistakes

| Mistake | Fix |
| --- | --- |
| Playwright browsers missing | `playwright install` after pip install |
| Model returns prose, not JSON | Add `format: "json"` and strict prompt |
| Wrong `model` string | Use `openai/...`, `ollama/...`, `google_genai/...` |
| Expensive re-runs on dev | Cache `run()` output to disk; use mini models |
| Using LLM scrape on static pages | httpx + bs4 first |
| Huge pages exceed context | Narrow prompt, split URLs, or pre-trim with crawl4ai |
| Secrets in repo | `os.environ` / `.env` only |

---

## Quick Reference

| Task | Code |
| --- | --- |
| Install | `pip install scrapegraphai && playwright install` |
| Single page | `SmartScraperGraph(prompt=..., source=url, config=...)` |
| Run | `result = graph.run()` |
| Multi URL | `SmartScraperMultiGraph(..., source=[url1, url2])` |
| LLM config | `config["llm"]["model"]`, `api_key`, `format` |
| Debug browser | `headless=False`, `verbose=True` |
| Typed output | `schema=MyPydanticModel` |

---

## Related Notes

- [[Browser Automation]]
- [[Browser Automation — Playwright]]
- [[Browser Automation — crawl4ai]]
- [[AI — LangChain]]
- [[AI]]
- [[Python — Pydantic]]
- [[Python — BeautifulSoup4 (bs4)]]
- [[Python — httpx Package]]
- [[Python Development]]

---

## Tags

#browser-automation #scrapegraphai #scraping #playwright #llm #langchain #python #ai #extraction
