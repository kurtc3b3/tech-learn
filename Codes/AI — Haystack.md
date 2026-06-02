## What & When

**Haystack** (deepset, 2.x) is a **pipeline-oriented** framework for building **search and RAG** systems. You wire **components** (embedders, retrievers, generators) into a `Pipeline` with explicit inputs/outputs — strong fit for production backends that want clear observability per stage.

Use Haystack when:

- You prefer **explicit pipelines** over LangChain's LCEL style
- Building **hybrid retrieval** (dense + BM25) with first-class components
- Running **batch indexing** jobs separate from query serving
- Integrating **OpenAI**, local models, or sentence-transformers as components
- Connecting to vector DBs ([[AI — Chroma]], [[AI — Qdrant]], [[AI — Milvus]], pgvector)

```bash
pip install haystack-ai
pip install sentence-transformers    # local embeddings
pip install chromadb                 # optional — Chroma document store
```

For graph-based agents and HITL, use [[AI — LangGraph]]. For chain composition with tools, see [[AI — LangChain]].

---

## Haystack vs LangChain / LlamaIndex

| Aspect | Haystack 2.x | LangChain | LlamaIndex |
| --- | --- | --- | --- |
| Composition | `Pipeline` + components | LCEL `\|` runnables | Query engines / agents |
| Strength | Search/RAG pipelines | Tools, agents, ecosystem | Data connectors, indexing |
| Observability | Per-component I/O | LangSmith | LlamaCloud / callbacks |
| Agent loops | Via custom components | LangGraph | Agent workflows |

---

## Document Model

```python
from haystack import Document

doc = Document(
    content="Haystack pipelines connect components with typed inputs.",
    meta={"source": "internal/wiki", "category": "rag"},
)

docs = [
    Document(content="Chroma works well for local dev.", meta={"source": "notes"}),
    Document(content="Qdrant scales for production vector search.", meta={"source": "notes"}),
]
```

```python
# From dicts / files
from haystack.components.converters import TextFileToDocument

converter = TextFileToDocument()
result = converter.run(sources=["./docs/readme.txt"])
file_docs = result["documents"]
```

---

## Indexing Pipeline — Embed & Store

```python
from haystack import Pipeline
from haystack.components.embedders import SentenceTransformersDocumentEmbedder
from haystack.components.writers import DocumentWriter
from haystack.document_stores.in_memory import InMemoryDocumentStore

document_store = InMemoryDocumentStore()
indexing = Pipeline()
indexing.add_component("embedder", SentenceTransformersDocumentEmbedder(model="sentence-transformers/all-MiniLM-L6-v2"))
indexing.add_component("writer", DocumentWriter(document_store=document_store))

indexing.connect("embedder.documents", "writer.documents")

indexing.run({"embedder": {"documents": docs}})
print(document_store.count_documents())
```

```python
# Chroma document store (pip install chromadb)
from haystack_integrations.document_stores.chroma import ChromaDocumentStore

chroma_store = ChromaDocumentStore(collection_name="backend_kb")
writer = DocumentWriter(document_store=chroma_store)
```

See [[AI — Chroma]] for local persistence; [[AI — Qdrant]] for managed vector search.

---

## Query Pipeline — Retrieve & Generate

```python
from haystack import Pipeline
from haystack.components.embedders import SentenceTransformersTextEmbedder
from haystack.components.retrievers.in_memory import InMemoryEmbeddingRetriever
from haystack.components.builders import PromptBuilder
from haystack.components.generators import OpenAIGenerator

prompt_template = """
Answer using only the context. If unsure, say you don't know.

Context:
{% for doc in documents %}
- {{ doc.content }}
{% endfor %}

Question: {{ query }}
"""

query_pipeline = Pipeline()
query_pipeline.add_component("text_embedder", SentenceTransformersTextEmbedder(model="sentence-transformers/all-MiniLM-L6-v2"))
query_pipeline.add_component("retriever", InMemoryEmbeddingRetriever(document_store=document_store, top_k=3))
query_pipeline.add_component("prompt_builder", PromptBuilder(template=prompt_template))
query_pipeline.add_component("llm", OpenAIGenerator(model="gpt-4o-mini"))

query_pipeline.connect("text_embedder.embedding", "retriever.query_embedding")
query_pipeline.connect("retriever.documents", "prompt_builder.documents")
query_pipeline.connect("prompt_builder.prompt", "llm.prompt")

result = query_pipeline.run({
    "text_embedder": {"text": "Which vector DB is good for production?"},
    "prompt_builder": {"query": "Which vector DB is good for production?"},
})

print(result["llm"]["replies"][0])
```

---

## OpenAI Chat Generator (2.x)

```python
from haystack.components.generators.chat import OpenAIChatGenerator
from haystack.dataclasses import ChatMessage

chat = OpenAIChatGenerator(model="gpt-4o-mini")
response = chat.run(messages=[ChatMessage.from_user("Hello Haystack")])
print(response["replies"][0].text)
```

Use `OpenAIChatGenerator` for multi-turn chat; `OpenAIGenerator` for single-string prompts from `PromptBuilder`.

---

## Hybrid Retrieval (BM25 + Dense)

```python
from haystack.components.retrievers import InMemoryBM25Retriever
from haystack.components.joiners import DocumentJoiner

# Index same document_store with BM25 + embeddings (both supported on InMemory)
hybrid = Pipeline()
hybrid.add_component("dense_retriever", InMemoryEmbeddingRetriever(document_store=document_store, top_k=5))
hybrid.add_component("sparse_retriever", InMemoryBM25Retriever(document_store=document_store, top_k=5))
hybrid.add_component("joiner", DocumentJoiner(join_mode="merge"))

hybrid.connect("dense_retriever.documents", "joiner.documents")
hybrid.connect("sparse_retriever.documents", "joiner.documents")
```

> [!tip] Start in-memory, swap document store Swap `InMemoryDocumentStore` for Chroma/Qdrant/Postgres integrations without rewriting component logic.

---

## Custom Component

```python
from haystack import component
from typing import List
from haystack import Document

@component
class MetadataFilter:
    @component.output_types(documents=List[Document])
    def run(self, documents: List[Document], category: str):
        filtered = [d for d in documents if d.meta.get("category") == category]
        return {"documents": filtered}
```

Register in a pipeline between retriever and prompt builder to enforce tenant or ACL filters.

---

## Common Backend / RAG Patterns

### Separate index and query services

```python
# indexer.py — cron / worker
def rebuild_index(paths: list[str]) -> int:
  raw = TextFileToDocument().run(sources=paths)["documents"]
  indexing.run({"embedder": {"documents": raw}})
  return document_store.count_documents()

# api.py — FastAPI
def ask(question: str) -> str:
  out = query_pipeline.run({
      "text_embedder": {"text": question},
      "prompt_builder": {"query": question},
  })
  return out["llm"]["replies"][0]
```

### Ingest from parsed PDFs

```python
def docs_from_markdown(md: str, source: str) -> list[Document]:
    from haystack.components.preprocessors import DocumentSplitter
    splitter = DocumentSplitter(split_by="word", split_length=200, split_overlap=30)
    base = [Document(content=md, meta={"source": source})]
    return splitter.run(documents=base)["documents"]
```

Pair with [[AI — Docling]] or [[AI — MegaParse]] for PDF → markdown.

### Evaluation hook

```python
def retrieve_only(question: str) -> list[Document]:
    out = query_pipeline.run({
        "text_embedder": {"text": question},
        "prompt_builder": {"query": question},
    })
    return out["retriever"]["documents"]
```

Feed outputs to [[AI — RAGAS]] for retrieval metrics.

---

## Quick Reference

| Task | Code |
| --- | --- |
| Create document | `Document(content="...", meta={...})` |
| Document store | `InMemoryDocumentStore()` / Chroma integration |
| Build pipeline | `Pipeline()` + `add_component` + `connect` |
| Run pipeline | `pipeline.run({component: {inputs}})` |
| Embed documents | `SentenceTransformersDocumentEmbedder` |
| Embed query | `SentenceTransformersTextEmbedder` |
| Retrieve | `InMemoryEmbeddingRetriever(document_store=..., top_k=3)` |
| Prompt | `PromptBuilder(template="...")` |
| Generate | `OpenAIGenerator` / `OpenAIChatGenerator` |
| BM25 | `InMemoryBM25Retriever` |
| Custom step | `@component` class with `run()` |

---

## Tags

#python #haystack #rag #pipeline #retrieval #embeddings #search #backend #deepset
