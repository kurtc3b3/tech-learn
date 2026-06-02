## What & When

**Docling** (IBM Research) converts **PDF, DOCX, PPTX, HTML, images**, and more into a structured **`DoclingDocument`** — then exports **Markdown, JSON, or HTML** for RAG ingestion. Layout models handle tables, reading order, and OCR when enabled.

Use Docling when:

- Ingesting **enterprise PDFs** with tables and multi-column layout
- You want **open-source**, local conversion (no cloud parser required)
- Exporting **clean Markdown** for chunking and embedding
- Batch-converting document folders before indexing
- Integrating with [[AI — LangChain]], [[AI — Haystack]], or [[AI — LlamaIndex]] loaders

```bash
pip install docling
# Optional GPU / extras — see official install guide for your platform
```

For Quivr-style layout parsing with vision LLMs, compare [[AI — MegaParse]]. See [[Concepts/AI]] for the full RAG pipeline map.

---

## Docling vs MegaParse vs Plain Extractors

| Tool | Strength | Trade-off |
| --- | --- | --- |
| **Docling** | Layout + tables, local AI models | Heavier install; model download |
| [[AI — MegaParse]] | LLM/vision parsers, Quivr ecosystem | Extra system deps (poppler, tesseract) |
| `pypdf` / `pdfplumber` | Fast text dump | Loses structure, poor on scans |
| [[Python — markdownify]] | HTML → Markdown | Not for native PDF layout |

> [!tip] Parse once, index many times Write normalized `.md` to object storage; re-chunk when embedding models change without re-parsing PDFs.

---

## Basic Conversion

```python
from docling.document_converter import DocumentConverter

converter = DocumentConverter()
result = converter.convert("reports/annual.pdf")

if result.document:
    markdown = result.document.export_to_markdown()
    print(markdown[:500])
```

```python
# URL or local path
result = converter.convert("https://arxiv.org/pdf/2408.09869")
print(result.status)
print(result.document.export_to_markdown())
```

```python
# Save directly
result.document.save_as_markdown("output/annual.md")
```

---

## Batch Conversion

```python
from pathlib import Path
from docling.document_converter import DocumentConverter

converter = DocumentConverter()
sources = list(Path("inbox").glob("**/*.{pdf,docx,pptx}"))

for path in sources:
    result = converter.convert(str(path))
    if not result.document:
        print(f"SKIP {path}: {result.status}")
        continue
    out = Path("parsed") / f"{path.stem}.md"
    out.parent.mkdir(parents=True, exist_ok=True)
    out.write_text(result.document.export_to_markdown(), encoding="utf-8")
    print(f"OK {path} -> {out}")
```

```python
# convert_all generator
for result in converter.convert_all([str(p) for p in sources]):
    name = result.input.file if result.input else "unknown"
    print(name, result.status)
```

---

## Pipeline Options (PDF / OCR / Tables)

```python
from docling.document_converter import DocumentConverter, PdfFormatOption
from docling.datamodel.base_models import InputFormat
from docling.datamodel.pipeline_options import PdfPipelineOptions

pipeline_options = PdfPipelineOptions()
pipeline_options.do_ocr = True              # scanned pages
pipeline_options.do_table_structure = True    # TableFormer

converter = DocumentConverter(
    format_options={
        InputFormat.PDF: PdfFormatOption(pipeline_options=pipeline_options),
    }
)

result = converter.convert("scanned_invoice.pdf")
text = result.document.export_to_markdown()
```

```python
# Plain text export (strip markdown formatting)
plain = result.document.export_to_markdown(strict_text=True)
```

---

## Export Formats

```python
doc = result.document

md = doc.export_to_markdown()
html = doc.export_to_html()
data = doc.export_to_dict()          # lossless JSON-like structure

# DocTags (compact model-friendly format)
# doc.export_to_doctags()  — when enabled in your docling version
```

```python
# Image handling in markdown
from docling_core.types.doc import ImageRefMode

md_with_placeholders = doc.export_to_markdown(
    image_mode=ImageRefMode.PLACEHOLDER
)
```

---

## LangChain Integration

```python
from langchain_docling import DoclingLoader
from langchain_text_splitters import RecursiveCharacterTextSplitter

loader = DoclingLoader(
    file_path="reports/annual.pdf",
    export_type="markdown",    # or "doc_chunks"
)
documents = loader.load()

splitter = RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=150)
chunks = splitter.split_documents(documents)

# vectorstore.add_documents(chunks)  — see [[AI — LangChain]], [[AI — Chroma]]
```

```bash
pip install langchain-docling
```

---

## Common Backend / RAG Patterns

### Parse → chunk → embed job

```python
from pathlib import Path
from docling.document_converter import DocumentConverter
from langchain_text_splitters import RecursiveCharacterTextSplitter
from langchain_core.documents import Document

converter = DocumentConverter()
splitter = RecursiveCharacterTextSplitter(chunk_size=900, chunk_overlap=120)

def parse_and_chunk(pdf_path: Path) -> list[Document]:
    result = converter.convert(str(pdf_path))
    if not result.document:
        raise RuntimeError(f"conversion failed: {pdf_path}")
    md = result.document.export_to_markdown()
    return splitter.create_documents(
        [md],
        metadatas=[{"source": str(pdf_path), "parser": "docling"}],
    )
```

### FastAPI upload endpoint

```python
import tempfile
from fastapi import FastAPI, UploadFile
from docling.document_converter import DocumentConverter

app = FastAPI()
converter = DocumentConverter()

@app.post("/parse")
async def parse_pdf(file: UploadFile):
    with tempfile.NamedTemporaryFile(suffix=".pdf") as tmp:
        tmp.write(await file.read())
        tmp.flush()
        result = converter.convert(tmp.name)
    md = result.document.export_to_markdown() if result.document else ""
    return {"markdown": md, "status": str(result.status)}
```

### Quality check before indexing

```python
def is_usable(markdown: str, min_chars: int = 200) -> bool:
    stripped = markdown.strip()
    if len(stripped) < min_chars:
        return False
    if "\ufffd" in stripped:     # replacement char — encoding garbage
        return False
    return True
```

Fall back to [[AI — MegaParse]] vision parsing for stubborn scans.

### Metadata for citations

```python
def extract_title(markdown: str) -> str:
    for line in markdown.splitlines():
        if line.startswith("# "):
            return line[2:].strip()
    return "untitled"

meta = {
    "source": "reports/annual.pdf",
    "title": extract_title(md),
    "parser": "docling",
}
```

---

## CLI (quick one-offs)

```bash
# Installed with docling package
docling ./report.pdf --output ./parsed
docling --help
```

Use CLI for ad-hoc conversion; embed `DocumentConverter` in workers for production.

---

## Quick Reference

| Task | Code |
| --- | --- |
| Convert file | `DocumentConverter().convert(path)` |
| To Markdown | `result.document.export_to_markdown()` |
| Save file | `result.document.save_as_markdown("out.md")` |
| Batch | `converter.convert_all(sources)` |
| OCR + tables | `PdfPipelineOptions(do_ocr=True, do_table_structure=True)` |
| Plain text | `export_to_markdown(strict_text=True)` |
| HTML export | `result.document.export_to_html()` |
| LangChain loader | `DoclingLoader(file_path=..., export_type="markdown")` |
| Compare parser | [[AI — MegaParse]] for vision-heavy docs |

---

## Tags

#python #docling #pdf #parsing #markdown #rag #ingestion #ibm #backend #documents
