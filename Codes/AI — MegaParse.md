## What & When

**MegaParse** (Quivr) turns **PDF, DOCX, PPTX**, and images into **LLM-ready Markdown** with layout-aware parsing — tables, headings, and OCR when needed. It targets **high-recall** text extraction for RAG, often alongside or instead of [[AI — Docling]].

Use MegaParse when:

- PDFs need **vision or unstructured** parsers for complex layouts
- You want Markdown output tuned for **chunking and embedding**
- Building a **parse → chunk → index** pipeline in Python 3.11+
- Running the optional **HTTP API** (`make dev`) for a dedicated parse service
- Combining with [[AI — LangChain]] models for `MegaParseVision`

```bash
pip install megaparse
# System deps: poppler, tesseract; macOS: brew install libmagic
```

Requires **Python ≥ 3.11**. Cross-check [[AI — Docling]] for local IBM layout models without vision API costs.

---

## MegaParse vs Docling

| Aspect | MegaParse | Docling |
| --- | --- | --- |
| Primary output | Markdown string / dict | `DoclingDocument` + exports |
| Vision / LLM parse | `MegaParseVision` | OCR via pipeline options |
| Dependencies | poppler, tesseract, heavy stack | docling models |
| Best for | Quivr/RAG stacks, vision PDFs | Enterprise PDF structure, tables |
| Integration | LangChain chat models | LangChain via `langchain-docling` |

Use **Docling** for batch IBM pipeline and structured JSON; use **MegaParse** when Quivr-style Markdown fidelity or vision parsing is required.

---

## Basic Parsing

```python
from megaparse import MegaParse

parser = MegaParse()
result = parser.load("./documents/handbook.pdf")
print(result)   # markdown str or structured dict depending on version
```

```python
# Save last processed document
parser.save("./documents/handbook.md")
```

```python
from pathlib import Path

def parse_to_markdown(path: Path) -> str:
    mp = MegaParse()
    content = mp.load(str(path))
    if isinstance(content, dict):
        return content.get("markdown", str(content))
    return str(content)
```

---

## MegaParseVision — Layout + Vision Model

Uses a multimodal chat model (e.g. GPT-4o) to interpret pages — higher cost, better on scans and slides.

```python
import os
from langchain_openai import ChatOpenAI
from megaparse.parser.megaparse_vision import MegaParseVision

model = ChatOpenAI(model="gpt-4o", api_key=os.environ["OPENAI_API_KEY"])
vision = MegaParseVision(model=model)

markdown = vision.convert("./documents/scanned_deck.pdf")
print(markdown[:1000])
```

```python
# Wire vision parser into MegaParse core
from megaparse import MegaParse

mp = MegaParse(parser=vision)   # pass configured MegaParseVision instance
text = mp.load("./documents/scanned_deck.pdf")
```

> [!tip] Gate vision parsing behind a flag Run cheap unstructured parse first; call `MegaParseVision` only when extracted text is below a length threshold.

---

## Unstructured / Llama Parsers

```python
import os
from megaparse import MegaParse
from megaparse.core.parser.unstructured_parser import UnstructuredParser
from langchain_openai import ChatOpenAI

model = ChatOpenAI(model="gpt-4o-mini", api_key=os.environ["OPENAI_API_KEY"])
parser = UnstructuredParser(model=model)
mp = MegaParse(parser=parser)

result = mp.load("./documents/contract.docx")
mp.save("./documents/contract.md")
```

```python
# Optional LlamaCloud parser (pip install llama-parse; API key required)
from megaparse.core.parser.llama import LlamaParser

llama = LlamaParser(api_key=os.environ["LLAMA_CLOUD_API_KEY"])
mp = MegaParse(parser=llama)
result = mp.load("./documents/noisy.pdf")
```

---

## Parse → Chunk → Embed

```python
import tiktoken
from megaparse import MegaParse
from sentence_transformers import SentenceTransformer

mp = MegaParse()
parsed = mp.load("./documents/policy.pdf")
markdown = parsed["markdown"] if isinstance(parsed, dict) else parsed

enc = tiktoken.get_encoding("cl100k_base")

def chunk_markdown(text: str, max_tokens: int = 400):
    buf, count = [], 0
    for para in text.split("\n\n"):
        n = len(enc.encode(para))
        if count + n > max_tokens and buf:
            yield "\n\n".join(buf)
            buf, count = [], 0
        buf.append(para)
        count += n
    if buf:
        yield "\n\n".join(buf)

chunks = list(chunk_markdown(markdown))
model = SentenceTransformer("all-MiniLM-L6-v2")
embeddings = model.encode(chunks, convert_to_numpy=True)
print(len(chunks), embeddings.shape)
```

Index chunks into [[AI — Chroma]] or [[AI — Haystack]] document stores.

---

## Common Backend / RAG Patterns

### Worker: parse folder to markdown cache

```python
from pathlib import Path
from megaparse import MegaParse

SUPPORTED = {".pdf", ".docx", ".pptx", ".png", ".jpg"}

def cache_markdown(in_dir: Path, out_dir: Path) -> list[Path]:
    mp = MegaParse()
    written = []
    for path in in_dir.rglob("*"):
        if path.suffix.lower() not in SUPPORTED:
            continue
        parsed = mp.load(str(path))
        md = parsed["markdown"] if isinstance(parsed, dict) else str(parsed)
        target = out_dir / path.relative_to(in_dir).with_suffix(".md")
        target.parent.mkdir(parents=True, exist_ok=True)
        target.write_text(md, encoding="utf-8")
        written.append(target)
    return written
```

### Router: Docling first, MegaParse fallback

```python
from docling.document_converter import DocumentConverter

def parse_pdf(path: str, min_chars: int = 300) -> str:
    docling = DocumentConverter().convert(path)
    if docling.document:
        md = docling.document.export_to_markdown()
        if len(md.strip()) >= min_chars:
            return md
    from megaparse import MegaParse
    return str(MegaParse().load(path))
```

See [[AI — Docling]] for the primary path; MegaParse for low-quality extractions.

### FastAPI parse microservice

```python
import tempfile
from fastapi import FastAPI, UploadFile
from megaparse import MegaParse

app = FastAPI()
parser = MegaParse()

@app.post("/parse")
async def parse(file: UploadFile):
    suffix = "." + (file.filename or "upload").split(".")[-1]
    with tempfile.NamedTemporaryFile(suffix=suffix, delete=False) as tmp:
        tmp.write(await file.read())
        tmp.flush()
        content = parser.load(tmp.name)
    md = content["markdown"] if isinstance(content, dict) else content
    return {"markdown": md}
```

### LangChain ingestion

```python
from langchain_core.documents import Document
from langchain_text_splitters import RecursiveCharacterTextSplitter

md = parse_to_markdown(Path("deck.pdf"))
splitter = RecursiveCharacterTextSplitter(chunk_size=800, chunk_overlap=100)
docs = splitter.create_documents([md], metadatas=[{"source": "deck.pdf", "parser": "megaparse"}])
# vectorstore.add_documents(docs)
```

---

## Run as API (optional)

```bash
# Clone QuivrHQ/MegaParse — use Makefile targets
cp .env.example .env    # OPENAI_API_KEY / ANTHROPIC_API_KEY
make dev                # local parse API
```

Offload heavy parsing from your main API workers when CPU or vision models would block request threads.

---

## Environment & Ops Notes

| Requirement | Purpose |
| --- | --- |
| `poppler` | PDF rendering |
| `tesseract` | OCR on images / scans |
| `libmagic` (macOS) | MIME detection |
| Python 3.11+ | Package constraint |
| API keys | Vision / Llama parsers only |

Keep parsers **stateless** in workers; write Markdown to object storage and let the index job be idempotent.

---

## Quick Reference

| Task | Code |
| --- | --- |
| Install | `pip install megaparse` |
| Default parse | `MegaParse().load(path)` |
| Save markdown | `mp.save("out.md")` |
| Vision parse | `MegaParseVision(model=ChatOpenAI(...)).convert(path)` |
| Custom parser | `MegaParse(parser=UnstructuredParser(...))` |
| LlamaCloud | `LlamaParser(api_key=...)` |
| Chunk for RAG | Split `\n\n` + tiktoken token budget |
| Compare | [[AI — Docling]] for structured local conversion |
| Index | [[AI — LangChain]], [[AI — Haystack]], [[AI — Chroma]] |

---

## Tags

#python #megaparse #pdf #parsing #markdown #ocr #rag #ingestion #quivr #backend #llm
