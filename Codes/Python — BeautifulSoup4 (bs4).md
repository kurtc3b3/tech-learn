## What & When

**BeautifulSoup4** (imported as `bs4`) is a Python library for parsing HTML and XML documents. It creates a parse tree from page source that can be navigated, searched, and modified. It is the go-to tool for web scraping and HTML processing in Python.

Use bs4 when:

- Scraping data from web pages
- Parsing and extracting content from HTML responses
- Cleaning or transforming HTML documents
- Processing RSS/Atom feeds or XML data
- Extracting links, tables, metadata, or structured content

```bash
pip install beautifulsoup4
pip install lxml        # recommended fast parser
pip install html5lib    # most lenient parser — handles broken HTML
```

---

## Parsers

```python
from bs4 import BeautifulSoup

html = "<html><body><p>Hello</p></body></html>"

# html.parser — stdlib, no extra install, slower
soup = BeautifulSoup(html, "html.parser")

# lxml — fast, lenient, recommended for most use cases
soup = BeautifulSoup(html, "lxml")

# lxml-xml — for XML documents
soup = BeautifulSoup(xml_content, "lxml-xml")

# html5lib — slowest but most accurate, handles broken HTML perfectly
soup = BeautifulSoup(html, "html5lib")
```

|Parser|Speed|Leniency|Extra Install|
|---|---|---|---|
|`html.parser`|Medium|Medium|❌|
|`lxml`|Fast|High|`pip install lxml`|
|`lxml-xml`|Fast|Strict|`pip install lxml`|
|`html5lib`|Slow|Highest|`pip install html5lib`|

> [!tip] Use `lxml` for production scraping It is the fastest and handles most real-world HTML well. Use `html5lib` only when pages are heavily malformed.

---

## Creating a Soup Object

```python
from bs4 import BeautifulSoup
import requests

# From a string
soup = BeautifulSoup("<p>Hello <b>world</b></p>", "lxml")

# From an HTTP response
response = requests.get("https://example.com")
soup = BeautifulSoup(response.text, "lxml")

# From a file
with open("page.html", encoding="utf-8") as f:
    soup = BeautifulSoup(f, "lxml")

# From bytes (bs4 detects encoding)
soup = BeautifulSoup(response.content, "lxml")
```

---

## The Parse Tree

```python
from bs4 import BeautifulSoup

html = """
<html>
  <head><title>My Page</title></head>
  <body>
    <h1 class="title">Hello World</h1>
    <p id="intro">Welcome to <b>my</b> site.</p>
    <a href="https://example.com">Link</a>
  </body>
</html>
"""

soup = BeautifulSoup(html, "lxml")

# Navigate the tree
soup.html                       # <html> tag
soup.head                       # <head> tag
soup.title                      # <title>My Page</title>
soup.title.string               # "My Page"
soup.body.h1                    # <h1 class="title">Hello World</h1>
soup.body.h1.string             # "Hello World"
soup.body.p                     # first <p>
soup.body.p.b                   # <b>my</b>
soup.body.p.b.string            # "my"
```

---

## Tag Properties

```python
tag = soup.find("h1")

tag.name                        # "h1"
tag.string                      # "Hello World" (only if single text node)
tag.text                        # "Hello World" (concatenates all text)
tag.get_text()                  # "Hello World"
tag.get_text(separator=" ", strip=True)  # with separator and stripped

tag.attrs                       # {"class": ["title"]}
tag["class"]                    # ["title"]
tag.get("class")                # ["title"]
tag.get("id")                   # None — safe access
tag.get("href", "#")            # with default

tag.has_attr("class")           # True
```

---

## `find` — First Match

```python
soup = BeautifulSoup(html, "lxml")

# By tag name
soup.find("p")                      # first <p>
soup.find("h1")                     # first <h1>

# By attribute
soup.find("p", id="intro")
soup.find("a", href=True)           # any <a> with href
soup.find("div", class_="card")     # class_ — underscore avoids Python keyword conflict

# By multiple attributes
soup.find("input", {"type": "text", "name": "username"})

# By CSS class (string)
soup.find("p", class_="highlight")

# By CSS class (multiple — tag must have ALL)
soup.find("div", class_="card featured")

# By text content
soup.find("p", string="Hello")
soup.find("p", string=re.compile(r"Hello"))

# By function
soup.find(lambda tag: tag.name == "p" and tag.get("id"))
```

---

## `find_all` — All Matches

```python
# All tags with name
soup.find_all("a")                      # list of all <a> tags
soup.find_all(["a", "p"])               # all <a> and <p> tags

# With attributes
soup.find_all("div", class_="card")
soup.find_all("a", href=re.compile(r"^https"))

# Limit results
soup.find_all("p", limit=3)            # first 3 matches only

# Recursive=False — direct children only
soup.body.find_all("p", recursive=False)

# All tags (no filter)
soup.find_all(True)

# By function
soup.find_all(lambda tag: len(tag.attrs) > 2)
```

---

## CSS Selectors — `select` & `select_one`

Often more concise than `find`/`find_all` for complex queries.

```python
# select_one — returns first match (like find)
soup.select_one("h1")
soup.select_one("#intro")               # by id
soup.select_one(".card")                # by class
soup.select_one("div.card.featured")    # tag + multiple classes
soup.select_one("a[href]")              # has attribute
soup.select_one("a[href^='https']")     # href starts with https
soup.select_one("a[href$='.pdf']")      # href ends with .pdf
soup.select_one("a[href*='example']")   # href contains example

# select — returns list (like find_all)
soup.select("p")                        # all <p>
soup.select("div > p")                  # direct child
soup.select("div p")                    # descendant
soup.select("h1 + p")                   # adjacent sibling
soup.select("h1 ~ p")                   # general sibling
soup.select("li:nth-of-type(2)")        # nth element
soup.select("li:first-child")
soup.select("li:last-child")
soup.select("input:not([disabled])")    # negation

# Attribute selectors
soup.select("[data-id]")                # has data-id attribute
soup.select("[data-id='42']")           # exact value
soup.select("a[href], img[src]")        # multiple selectors (comma)
```

---

## Navigating the Tree

```python
tag = soup.find("p", id="intro")

# Parent
tag.parent                  # direct parent element
tag.parents                 # generator of all ancestors

# Children
tag.children                # generator of direct children
list(tag.children)          # list of direct children
tag.contents                # list of direct children (same as list(children))
tag.descendants             # generator of all descendants

# Siblings
tag.next_sibling            # next sibling (may be NavigableString whitespace)
tag.previous_sibling        # previous sibling
tag.next_siblings           # generator of all following siblings
tag.previous_siblings       # generator of all preceding siblings

# Skip whitespace — use find methods instead
tag.find_next_sibling("p")          # next sibling <p> tag
tag.find_previous_sibling("p")      # previous sibling <p> tag
tag.find_next_siblings("p")         # all following sibling <p> tags
tag.find_all_next("a")              # all following <a> anywhere in tree
tag.find_all_previous("a")          # all preceding <a> anywhere in tree
```

---

## Text Extraction

```python
# .string — only works if tag has a single text node
tag = soup.find("p")
tag.string          # "Hello World" or None if mixed content

# .text / .get_text() — concatenates all text (including nested)
tag.text
tag.get_text()
tag.get_text(separator="\n", strip=True)    # custom separator, strip whitespace

# All text in document
soup.get_text(separator=" ", strip=True)

# strings — generator of NavigableStrings (text nodes)
list(tag.strings)

# stripped_strings — strips whitespace from each
list(tag.stripped_strings)
```

---

## Working with Attributes

```python
tag = soup.find("a")

# Read
tag["href"]                     # raises KeyError if missing
tag.get("href")                 # None if missing
tag.get("href", "#")            # with default
tag.attrs                       # dict of all attributes

# class is always a list
tag["class"]                    # ["nav-link", "active"]
" ".join(tag["class"])          # "nav-link active"

# Modify
tag["href"] = "https://new.example.com"
tag["class"].append("bold")
tag.attrs["data-id"] = "42"

# Delete
del tag["href"]
tag.attrs.pop("class", None)    # safe delete
```

---

## Modifying the Tree

```python
from bs4 import BeautifulSoup, Tag, NavigableString

soup = BeautifulSoup("<div><p>Hello</p></div>", "lxml")

# Modify tag content
tag = soup.find("p")
tag.string = "Hello World"          # replace text content
tag.string.replace_with("Updated")  # replace NavigableString

# Insert new tag
new_tag = soup.new_tag("a", href="https://example.com")
new_tag.string = "Click here"
tag.append(new_tag)                 # add as last child
tag.insert(0, new_tag)              # insert at position
tag.insert_before(new_tag)          # insert before this tag
tag.insert_after(new_tag)           # insert after this tag

# Wrap / unwrap
tag.wrap(soup.new_tag("div"))       # wrap tag in new element
tag.unwrap()                        # replace tag with its contents

# Remove
tag.decompose()                     # remove from tree and destroy
tag.extract()                       # remove from tree and return it

# Replace
tag.replace_with(soup.new_tag("span"))
```

---

## Extracting Links

```python
import requests
from bs4 import BeautifulSoup
from urllib.parse import urljoin, urlparse

def extract_links(url: str) -> list[dict]:
    response = requests.get(url)
    soup = BeautifulSoup(response.text, "lxml")
    base_url = f"{urlparse(url).scheme}://{urlparse(url).netloc}"

    links = []
    for a in soup.find_all("a", href=True):
        href = a["href"].strip()
        if href.startswith("#") or href.startswith("mailto:"):
            continue
        absolute = urljoin(base_url, href)
        links.append({
            "text": a.get_text(strip=True),
            "href": absolute,
            "title": a.get("title", ""),
        })
    return links
```

---

## Extracting Tables

```python
def extract_table(soup: BeautifulSoup, table_index: int = 0) -> list[dict]:
    tables = soup.find_all("table")
    if not tables:
        return []

    table = tables[table_index]

    # Get headers
    headers = [
        th.get_text(strip=True)
        for th in table.find("tr").find_all(["th", "td"])
    ]

    # Get rows
    rows = []
    for tr in table.find_all("tr")[1:]:     # skip header row
        cells = [td.get_text(strip=True) for td in tr.find_all(["td", "th"])]
        if cells:
            rows.append(dict(zip(headers, cells)))

    return rows
```

---

## Extracting Metadata

```python
def extract_metadata(soup: BeautifulSoup) -> dict:
    return {
        "title":       soup.title.string if soup.title else None,
        "description": (soup.find("meta", attrs={"name": "description"}) or {}).get("content"),
        "keywords":    (soup.find("meta", attrs={"name": "keywords"}) or {}).get("content"),
        "og_title":    (soup.find("meta", property="og:title") or {}).get("content"),
        "og_image":    (soup.find("meta", property="og:image") or {}).get("content"),
        "canonical":   (soup.find("link", rel="canonical") or {}).get("href"),
        "robots":      (soup.find("meta", attrs={"name": "robots"}) or {}).get("content"),
    }
```

---

## Full Scraping Example — with requests + tqdm

```python
import requests
from bs4 import BeautifulSoup
from tqdm import tqdm
import time

BASE_URL = "https://books.toscrape.com"

def scrape_books(pages: int = 5) -> list[dict]:
    books = []

    for page in tqdm(range(1, pages + 1), desc="Scraping pages"):
        url = f"{BASE_URL}/catalogue/page-{page}.html"
        response = requests.get(url)
        soup = BeautifulSoup(response.text, "lxml")

        for article in soup.select("article.product_pod"):
            title  = article.find("h3").find("a")["title"]
            price  = article.select_one(".price_color").get_text(strip=True)
            rating = article.find("p", class_="star-rating")["class"][1]
            link   = BASE_URL + "/catalogue/" + article.find("h3").find("a")["href"]

            books.append({
                "title":  title,
                "price":  price,
                "rating": rating,
                "url":    link,
            })

        time.sleep(0.5)     # polite delay between requests

    return books
```

---

## Async Scraping — with httpx

```python
import asyncio
import httpx
from bs4 import BeautifulSoup
from tqdm.asyncio import tqdm_asyncio

async def fetch_page(client: httpx.AsyncClient, url: str) -> list[dict]:
    response = await client.get(url)
    soup = BeautifulSoup(response.text, "lxml")
    return [
        {
            "title": a.get_text(strip=True),
            "href":  a["href"],
        }
        for a in soup.select("article h3 a")
    ]

async def scrape_all(urls: list[str]) -> list[dict]:
    async with httpx.AsyncClient(
        base_url="https://books.toscrape.com",
        timeout=10.0,
    ) as client:
        tasks = [fetch_page(client, url) for url in urls]
        results = await tqdm_asyncio.gather(*tasks, desc="Fetching pages")
        return [item for page in results for item in page]
```

---

## Common Patterns

### Safe Extraction Helper

```python
from bs4 import BeautifulSoup, Tag
from typing import Optional

def safe_text(tag: Optional[Tag], selector: str, default: str = "") -> str:
    """Safely extract text from a CSS selector."""
    el = tag.select_one(selector) if tag else None
    return el.get_text(strip=True) if el else default

def safe_attr(tag: Optional[Tag], selector: str, attr: str, default: str = "") -> str:
    """Safely extract an attribute from a CSS selector."""
    el = tag.select_one(selector) if tag else None
    return el.get(attr, default) if el else default

# Usage
title   = safe_text(soup, "h1.product-title")
img_url = safe_attr(soup, "img.product-image", "src")
price   = safe_text(soup, "span.price", default="0.00")
```

### Strip HTML Tags — Plain Text Only

```python
from bs4 import BeautifulSoup

def strip_html(html: str) -> str:
    return BeautifulSoup(html, "lxml").get_text(separator=" ", strip=True)

strip_html("<p>Hello <b>World</b>!</p>")    # "Hello World !"
```

### Clean HTML — Remove Scripts & Styles

```python
from bs4 import BeautifulSoup

def clean_html(html: str) -> str:
    soup = BeautifulSoup(html, "lxml")

    for tag in soup(["script", "style", "noscript", "iframe"]):
        tag.decompose()

    for tag in soup.find_all(True):
        tag.attrs = {k: v for k, v in tag.attrs.items() if k in {"href", "src", "alt"}}

    return str(soup)
```

---

## Quick Reference

|Task|Code|
|---|---|
|Parse HTML|`BeautifulSoup(html, "lxml")`|
|Parse from URL|`BeautifulSoup(requests.get(url).text, "lxml")`|
|Find first|`soup.find("tag")`|
|Find all|`soup.find_all("tag")`|
|CSS selector one|`soup.select_one("div.card")`|
|CSS selectors all|`soup.select("div.card")`|
|By id|`soup.find(id="main")` or `soup.select_one("#main")`|
|By class|`soup.find("div", class_="card")`|
|By attribute|`soup.find("a", href=True)`|
|Get text|`tag.get_text(strip=True)`|
|Get attribute|`tag.get("href", "")`|
|All attributes|`tag.attrs`|
|Parent|`tag.parent`|
|Children|`list(tag.children)`|
|Next sibling|`tag.find_next_sibling("p")`|
|All links|`soup.find_all("a", href=True)`|
|Strip HTML|`BeautifulSoup(html, "lxml").get_text()`|
|New tag|`soup.new_tag("a", href="...")`|
|Remove tag|`tag.decompose()`|
|Replace tag|`tag.replace_with(new_tag)`|

---
## Tags

#python #bs4 #beautifulsoup #scraping #html #parsing #web #backend