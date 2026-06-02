## What & When

**markdownify** is a Python library that converts HTML to Markdown. It parses HTML with BeautifulSoup and walks the DOM tree, producing clean Markdown output. It is the standard tool for turning scraped web pages, email HTML, or CMS content into Markdown for LLM pipelines, documentation, or storage.

Use `markdownify` when:

- Converting scraped HTML to Markdown for RAG or LLM ingestion
- Normalizing CMS or email HTML into readable Markdown
- Building a scrape → clean → markdown pipeline (with `bs4`, `httpx`)
- Preserving structure (headings, lists, links, tables) instead of plain text
- Feeding web content into vector databases or chat context

For HTML parsing and extraction, see [[Python — BeautifulSoup4 (bs4)]]. For fetching pages, see [[Python — httpx Package]] or [[Python — requests Package]].

```bash
pip install markdownify
pip install lxml          # optional — faster bs4 parser (recommended)
```

---

## Basic Usage

```python
from markdownify import markdownify

html = """
<h1>Hello World</h1>
<p>Visit <a href="https://example.com">Example</a> for more.</p>
<ul>
  <li>First item</li>
  <li>Second item</li>
</ul>
"""

md = markdownify(html)
print(md)
```

```markdown
Hello World
===========

Visit [Example](https://example.com) for more.

* First item
* Second item
```

```python
# ATX headings (# style) — common for LLM / GitHub-flavored Markdown
md = markdownify(html, heading_style="ATX")

# From a BeautifulSoup object (skip re-parsing)
from bs4 import BeautifulSoup

soup = BeautifulSoup(html, "lxml")
md = markdownify(str(soup.find("article")), heading_style="ATX")
```

---

## markdownify vs bs4 `.get_text()`

|                    | markdownify              | bs4 `.get_text()`        |
| ------------------ | ------------------------ | ------------------------- |
| Output format      | Structured Markdown      | Plain text only           |
| Headings           | `# Title` or underlined  | Flat text                 |
| Links              | `[text](url)`            | Link text only            |
| Lists              | `- item` / `1. item`     | Concatenated text         |
| Code blocks        | ` ``` ` fenced blocks    | Raw code text             |
| Tables             | Markdown tables          | Cell text mashed together |
| Best for           | LLM context, docs, RAG   | Quick text extraction     |

> [!tip] Pre-clean HTML before converting Strip `<script>`, `<style>`, nav, and footer elements with bs4 first — markdownify removes script/style content but cannot guess which layout sections you want to keep.

---

## Configuration Options

All options are passed as keyword arguments to `markdownify()` or set on a `MarkdownConverter` subclass.

```python
from markdownify import markdownify

md = markdownify(
    html,
    heading_style="ATX",           # ATX | ATX_CLOSED | UNDERLINED (default)
    bullets="-",                   # bullet chars for nested ul lists
    autolinks=True,                # <https://url> when text == href
    default_title=False,           # use href as link title when missing
    code_language="python",        # language tag on all ``` blocks
    escape_asterisks=True,         # escape * in plain text
    escape_underscores=True,       # escape _ in plain text
    escape_misc=False,             # escape \, &, <, `, etc.
    newline_style="spaces",        # spaces (two spaces + newline) | backslash
    wrap=False,                    # wrap paragraph text
    wrap_width=80,
    table_infer_header=False,      # treat first row as header if no <th>
    keep_inline_images_in=[],      # e.g. ['td'] — keep ![alt](src) in cells
    bs4_options="lxml",            # bs4 parser or dict of bs4 kwargs
    strip_document="strip",        # lstrip | rstrip | strip | None
    strip_pre="strip",             # lstrip | rstrip | strip | strip_one | None
)
```

| Option | Default | Description |
| --- | --- | --- |
| `heading_style` | `UNDERLINED` | `ATX` → `# H1`, `SETEXT`/`UNDERLINED` → `===` / `---` |
| `bullets` | `'*+-'` | Bullet chars; cycles by nesting depth |
| `autolinks` | `True` | `<url>` when link text equals href |
| `code_language` | `''` | Default fence language for `<pre>` blocks |
| `code_language_callback` | `None` | Callable `(el) -> str` per `<pre>` element |
| `strip` | `None` | Tags to remove entirely (mutually exclusive with `convert`) |
| `convert` | `None` | Tags to convert; all others stripped |
| `newline_style` | `spaces` | `spaces` or `backslash` for `<br>` |
| `strong_em_symbol` | `*` | `*` or `_` for bold/italic |
| `sub_symbol` / `sup_symbol` | `''` | Markdown for `<sub>` / `<sup>` (e.g. `~` / `^`) |
| `wrap` / `wrap_width` | `False` / `80` | Hard-wrap paragraph text |
| `bs4_options` | `html.parser` | Parser name or dict for `BeautifulSoup()` |

---

## Heading Styles

```python
html = "<h1>Title</h1><h2>Section</h2><h3>Sub</h3>"

# UNDERLINED (default) — Setext-style for h1/h2, ATX for h3+
markdownify(html, heading_style="UNDERLINED")
# Title
# =====
# Section
# -------
# ### Sub

# ATX — GitHub / LLM friendly
markdownify(html, heading_style="ATX")
# # Title
# ## Section
# ### Sub

# ATX_CLOSED
markdownify(html, heading_style="ATX_CLOSED")
# # Title #
# ## Section ##
```

> [!tip] Use `heading_style="ATX"` for LLM pipelines Most models and Markdown renderers handle `#` headings more reliably than Setext underlines.

---

## Links & Images

```python
html = """
<a href="https://example.com">Example</a>
<a href="https://example.com" title="Homepage">Example</a>
<a href="https://example.com">https://example.com</a>
<img src="/logo.png" alt="Logo" title="Site logo">
"""

# Default
markdownify(html)
# [Example](https://example.com)
# [Example](https://example.com "Homepage")
# <https://example.com>          ← autolink shortcut
# Logo                           ← alt text only inside headings/cells

# Keep images as markdown inside table cells
markdownify(html, keep_inline_images_in=["td"])
# ![Logo](/logo.png "Site logo")
```

---

## Lists

```python
html = """
<ul>
  <li>Item one</li>
  <li>Item two
    <ul>
      <li>Nested</li>
    </ul>
  </li>
</ul>
<ol start="3">
  <li>Third</li>
  <li>Fourth</li>
</ol>
"""

# Default bullets cycle: * → + → -
markdownify(html, heading_style="ATX")

# Single bullet style at all depths
markdownify(html, bullets="-", heading_style="ATX")
```

---

## Code Blocks

```python
html = """
<p>Inline <code>print(x)</code> code.</p>
<pre><code>def hello():
    print("world")
</code></pre>
"""

# All fenced blocks tagged as python
markdownify(html, code_language="python", heading_style="ATX")

# Per-block language via callback
def detect_language(el):
    classes = el.get("class", [])
    for cls in classes:
        if cls.startswith("language-"):
            return cls.replace("language-", "")
    return ""

markdownify(html, code_language_callback=detect_language, heading_style="ATX")
```

---

## Tables

```python
html = """
<table>
  <thead>
    <tr><th>Name</th><th>Price</th></tr>
  </thead>
  <tbody>
    <tr><td>Widget</td><td>$9.99</td></tr>
    <tr><td>Gadget</td><td>$14.99</td></tr>
  </tbody>
</table>
"""

markdownify(html, heading_style="ATX")
# | Name   | Price  |
# | --- | --- |
# | Widget | $9.99  |
# | Gadget | $14.99 |

# Infer header row when <th> is missing
markdownify(html, table_infer_header=True, heading_style="ATX")
```

---

## Strip vs Convert Tags

Control which HTML elements are processed. **Use one or the other, not both.**

```python
html = """
<nav><a href="/">Home</a></nav>
<article>
  <h1>Title</h1>
  <p>Content here.</p>
  <aside>Sidebar noise</aside>
</article>
"""

# strip — remove these tags and their content
markdownify(html, strip=["nav", "aside"], heading_style="ATX")

# convert — only convert these tags; everything else is stripped
markdownify(html, convert=["h1", "p", "a"], heading_style="ATX")
```

---

## Custom `MarkdownConverter`

Subclass to override conversion for specific tags.

```python
from markdownify import MarkdownConverter

class CustomConverter(MarkdownConverter):
    class Options(MarkdownConverter.Options):
        heading_style = "ATX"

    def convert_a(self, el, text, parent_tags):
        """Render links as plain text with URL in parentheses."""
        href = el.get("href", "")
        text = text.strip()
        if href and text:
            return f"{text} ({href})"
        return text

    def convert_img(self, el, text, parent_tags):
        """Skip images entirely."""
        return ""

converter = CustomConverter()
md = converter.convert(html)
```

```python
# Reuse converter instance for batch processing
converter = CustomConverter(heading_style="ATX", bullets="-")

for page_html in pages:
    md = converter.convert(page_html)
    process(md)
```

---

## Pre-Clean HTML with bs4

Best practice: parse and clean with bs4, then convert the relevant fragment.

```python
from bs4 import BeautifulSoup
from markdownify import markdownify

def html_to_markdown(html: str, selector: str = "article") -> str:
    soup = BeautifulSoup(html, "lxml")

    # Remove noise
    for tag in soup(["script", "style", "noscript", "iframe", "nav", "footer"]):
        tag.decompose()

    # Extract main content
    main = soup.select_one(selector) or soup.body or soup

    return markdownify(
        str(main),
        heading_style="ATX",
        bullets="-",
        autolinks=True,
    )
```

---

## Scraping Pipeline — httpx + bs4 + markdownify

```python
import httpx
from bs4 import BeautifulSoup
from markdownify import markdownify

async def fetch_as_markdown(url: str) -> str:
    async with httpx.AsyncClient(timeout=15.0, follow_redirects=True) as client:
        response = await client.get(url)
        response.raise_for_status()

    soup = BeautifulSoup(response.text, "lxml")

    for tag in soup(["script", "style", "nav", "footer", "header"]):
        tag.decompose()

    main = soup.select_one("article, main, [role='main']") or soup.body

    return markdownify(
        str(main),
        heading_style="ATX",
        bullets="-",
        bs4_options="lxml",
    )
```

---

## Batch Processing with tqdm

```python
import httpx
from bs4 import BeautifulSoup
from markdownify import markdownify
from tqdm import tqdm

def scrape_pages(urls: list[str]) -> dict[str, str]:
    results = {}

    with httpx.Client(timeout=15.0, follow_redirects=True) as client:
        for url in tqdm(urls, desc="Converting to Markdown"):
            response = client.get(url)
            response.raise_for_status()

            soup = BeautifulSoup(response.text, "lxml")
            for tag in soup(["script", "style"]):
                tag.decompose()

            main = soup.select_one("article") or soup.body
            results[url] = markdownify(
                str(main),
                heading_style="ATX",
                bullets="-",
            )

    return results
```

---

## LLM / RAG Preparation

```python
from markdownify import markdownify

def prepare_for_llm(html: str, max_chars: int = 12_000) -> str:
    md = markdownify(
        html,
        heading_style="ATX",
        bullets="-",
        escape_asterisks=False,    # less noise for models
        escape_underscores=False,
        strip=["nav", "footer", "aside", "form"],
    )

    # Collapse excessive blank lines
    import re
    md = re.sub(r"\n{3,}", "\n\n", md).strip()

    if len(md) > max_chars:
        md = md[:max_chars] + "\n\n...[truncated]"

    return md
```

---

## Command Line

```bash
# Convert a file
markdownify page.html > page.md

# From stdin
cat page.html | markdownify > page.md

# Options mirror the Python API
markdownify page.html --heading-style ATX --bullets - --autolinks

markdownify -h
```

---

## Common Patterns

### HTML File to Markdown File

```python
from pathlib import Path
from bs4 import BeautifulSoup
from markdownify import markdownify

def convert_file(html_path: Path, md_path: Path) -> None:
    html = html_path.read_text(encoding="utf-8")
    soup = BeautifulSoup(html, "lxml")

    for tag in soup(["script", "style"]):
        tag.decompose()

    md = markdownify(str(soup.body or soup), heading_style="ATX", bullets="-")
    md_path.write_text(md, encoding="utf-8")
```

### Extract Metadata + Markdown Body

```python
from bs4 import BeautifulSoup
from markdownify import markdownify

def extract_page(html: str) -> dict:
    soup = BeautifulSoup(html, "lxml")

    title = soup.title.string.strip() if soup.title and soup.title.string else ""
    description = (soup.find("meta", attrs={"name": "description"}) or {}).get("content", "")

    for tag in soup(["script", "style", "nav", "footer"]):
        tag.decompose()

    main = soup.select_one("article, main") or soup.body

    return {
        "title": title,
        "description": description,
        "markdown": markdownify(str(main), heading_style="ATX", bullets="-"),
    }
```

### Compare Plain Text vs Markdown Output

```python
from bs4 import BeautifulSoup
from markdownify import markdownify

def compare_outputs(html: str) -> tuple[str, str]:
    soup = BeautifulSoup(html, "lxml")
    plain = soup.get_text(separator="\n", strip=True)
    markdown = markdownify(html, heading_style="ATX", bullets="-")
    return plain, markdown
```

---

## Quick Reference

| Task | Code |
| --- | --- |
| Convert HTML string | `markdownify(html)` |
| ATX headings | `markdownify(html, heading_style="ATX")` |
| Custom bullets | `markdownify(html, bullets="-")` |
| Strip tags | `markdownify(html, strip=["nav", "script"])` |
| Convert only specific tags | `markdownify(html, convert=["p", "h1", "a"])` |
| Code block language | `markdownify(html, code_language="python")` |
| Fast parser | `markdownify(html, bs4_options="lxml")` |
| Custom converter | `CustomConverter().convert(html)` |
| Pre-clean with bs4 | `tag.decompose()` then `markdownify(str(main))` |
| CLI | `markdownify page.html > page.md` |
| LLM-friendly output | `heading_style="ATX"`, pre-strip nav/footer |

---

## Tags

#python #markdownify #html #markdown #scraping #llm #rag #bs4 #web #backend
