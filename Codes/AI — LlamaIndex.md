## What & When

**LlamaIndex** is a Python framework for **connecting LLMs to your data**: ingestion, indexing, retrieval, and **query engines** over documents, SQL, graphs, and vector stores. It sits at the RAG layer alongside [[AI — LangChain]] with a document-centric API.

Use LlamaIndex when:

- You want a **fast path from files → index → Q&A** without hand-wiring every RAG step
- Building **query engines** (synthesis, routing, sub-question) on top of [[AI — Chroma]], [[AI — Qdrant]], or other stores
- You need **data connectors** (PDF, Notion, APIs) and composable **retrievers** in one stack
- Prototyping RAG before committing to LangGraph-style agent graphs in [[AI — LangGraph]]

```python
from llama_index.core import VectorStoreIndex, SimpleDirectoryReader, Settings
from llama_index.core import StorageContext
```

```bash
pip install llama-index
# Integrations (install as needed):
# pip install llama-index-vector-stores-chroma
# pip install llama-index-vector-stores-qdrant
# pip install llama-index-llms-openai
# pip install llama-index-embeddings-openai
```

Vector backends: [[AI — Chroma]], [[AI — Qdrant]], [[AI — Milvus]], [[AI — FAISS]] (via community). Concept map: [[AI]].

---

## LlamaIndex vs Related Tools

| Need | Use | Notes |
| --- | --- | --- |
| Document-first RAG, query engines | **LlamaIndex** | `VectorStoreIndex`, `QueryEngine` |
| General chains, tools, agents | [[AI — LangChain]] | Often paired or chosen instead |
| Stateful multi-step agents | [[AI — LangGraph]] | Graph over LangChain primitives |
| Pipeline / evaluation focus | [[AI — Haystack]], [[AI — RAGAS]] | Different sweet spots |
| Raw vector CRUD | Vector store notes | LlamaIndex wraps them |

---

## Global Settings — LLM and Embeddings

```python
from llama_index.core import Settings
from llama_index.llms.openai import OpenAI
from llama_index.embeddings.openai import OpenAIEmbedding

Settings.llm = OpenAI(model="gpt-4o-mini")
Settings.embed_model = OpenAIEmbedding(model="text-embedding-3-small")
Settings.chunk_size = 512
Settings.chunk_overlap = 50
```

Set once per process; all indexes and engines inherit unless overridden.

---

## Ingest — Documents from Disk

```python
from llama_index.core import SimpleDirectoryReader

documents = SimpleDirectoryReader("./data").load_data()
# each Document has .text and .metadata (file path, etc.)
```

---

## VectorStoreIndex — Default RAG Index

Builds embeddings and stores them (in-memory by default, or attached vector store).

```python
from llama_index.core import VectorStoreIndex

index = VectorStoreIndex.from_documents(documents)   # embed + index in one step
# or from pre-chunked nodes:
# index = VectorStoreIndex(nodes)
```

```python
query_engine = index.as_query_engine(similarity_top_k=4)
response = query_engine.query("What is the main topic of these files?")
print(response)
```

```python
# Retriever only (no LLM synthesis) — compose your own chain
retriever = index.as_retriever(similarity_top_k=6)
nodes_with_scores = retriever.retrieve("search phrase")
```

---

## Persist and Reload

```python
index.storage_context.persist(persist_dir="./storage")

from llama_index.core import load_index_from_storage, StorageContext

storage_context = StorageContext.from_defaults(persist_dir="./storage")
index = load_index_from_storage(storage_context)
query_engine = index.as_query_engine()
```

For production, persist vectors in [[AI — Qdrant]] or [[AI — Chroma]] instead of default local JSON.

---

## Chroma Integration

```python
import chromadb
from llama_index.vector_stores.chroma import ChromaVectorStore
from llama_index.core import StorageContext, VectorStoreIndex

db = chromadb.PersistentClient(path="./chroma_llama")
chroma_collection = db.get_or_create_collection("llama_docs")

vector_store = ChromaVectorStore(chroma_collection=chroma_collection)
storage_context = StorageContext.from_defaults(vector_store=vector_store)

index = VectorStoreIndex.from_documents(
    documents,
    storage_context=storage_context,
)
```

See [[AI — Chroma]] for low-level collection operations.

---

## Qdrant Integration

```python
from qdrant_client import QdrantClient
from llama_index.vector_stores.qdrant import QdrantVectorStore
from llama_index.core import StorageContext, VectorStoreIndex

client = QdrantClient(url="http://localhost:6333")
vector_store = QdrantVectorStore(client=client, collection_name="llama_docs")
storage_context = StorageContext.from_defaults(vector_store=vector_store)

index = VectorStoreIndex.from_documents(documents, storage_context=storage_context)
```

See [[AI — Qdrant]] for filters and upsert patterns.

---

## Query Engines — Modes

```python
# Streaming
streaming_engine = index.as_query_engine(streaming=True)
response = streaming_engine.query("Summarize section 2.")
for token in response.response_gen:
    print(token, end="")
```

```python
# Chat engine — multi-turn with memory
chat_engine = index.as_chat_engine(chat_mode="context")
reply = chat_engine.chat("What did we just discuss?")
```

Combine with [[AI — LangChain]] when you need tool-calling agents in the same app.

---

## Composable Retrievers

```python
from llama_index.core.retrievers import VectorIndexRetriever
from llama_index.core.query_engine import RetrieverQueryEngine
from llama_index.core.response_synthesizers import get_response_synthesizer

retriever = VectorIndexRetriever(index=index, similarity_top_k=8)
synthesizer = get_response_synthesizer(response_mode="compact")

query_engine = RetrieverQueryEngine(
    retriever=retriever,
    response_synthesizer=synthesizer,
)
```

Useful when mixing **multiple indexes** or custom post-filtering before synthesis.

---

## Patterns

### Ingest → index → query script

```python
def build_rag(data_dir: str, persist_dir: str):
    docs = SimpleDirectoryReader(data_dir).load_data()
    index = VectorStoreIndex.from_documents(docs)
    index.storage_context.persist(persist_dir=persist_dir)
    return index
```

### Swap vector store without changing query code

Keep `StorageContext.from_defaults(vector_store=...)` as the single integration point; swap Chroma vs Qdrant via config.

### Evaluate retrieval before generation

Retrieve with `as_retriever()`, inspect `node.get_content()` and scores, then enable `as_query_engine()` once chunks look correct ([[AI — RAGAS]] for metrics).

---

## Quick Reference

| Task | Code |
| --- | --- |
| Load files | `SimpleDirectoryReader(path).load_data()` |
| Chunk | `SentenceSplitter(...).get_nodes_from_documents(docs)` |
| Build index | `VectorStoreIndex.from_documents(docs)` |
| Q&A | `index.as_query_engine().query("...")` |
| Retriever only | `index.as_retriever(similarity_top_k=k)` |
| Chat | `index.as_chat_engine().chat("...")` |
| Persist | `index.storage_context.persist(persist_dir=...)` |
| Reload | `load_index_from_storage(StorageContext.from_defaults(...))` |
| External store | `StorageContext.from_defaults(vector_store=ChromaVectorStore(...))` |
| Global models | `Settings.llm`, `Settings.embed_model` |

---

## Tags

#python #ai #llamaindex #rag #vectordb #query-engine #indexing #embeddings #llm
