## What & When

**Chroma** is an open-source **embedding database** optimized for local development and small-to-medium RAG apps. It stores vectors, metadata, and optional document text in **collections**, with a simple Python API and no cluster setup for the embedded mode.

Use Chroma when:

- Prototyping RAG on a laptop (embedded or persistent client)
- You want **documents + embeddings + metadata** in one store without running a separate DB server
- Integrating with [[AI — LangChain]] or [[AI — LlamaIndex]] via built-in adapters
- Dev/staging workloads where [[AI — Qdrant]] or [[AI — Milvus]] is overkill

```python
import chromadb
from chromadb.config import Settings
```

```bash
pip install chromadb
```

For in-process-only speed with no persistence API, see [[AI — FAISS]]. For production filters and scale, see [[AI — Qdrant]] and [[AI — Milvus]]. Stack context: [[AI]].

---

## Chroma vs Other Vector Stores

| Need | Use | Notes |
| --- | --- | --- |
| Fastest local dev, minimal setup | **Chroma** | Embedded or `./chroma_db` persistence |
| No server, max raw speed | [[AI — FAISS]] | You manage IDs and metadata yourself |
| Payload filters, hybrid, cloud | [[AI — Qdrant]] | Rich filtering, HNSW at scale |
| Billion-scale cluster | [[AI — Milvus]] | Dedicated vector DB cluster |
| High-level RAG indexing | [[AI — LlamaIndex]] | Wraps Chroma and others |

---

## Clients — Ephemeral, Persistent, Server

**Ephemeral** — in-memory only; data lost when the process exits.

```python
import chromadb

client = chromadb.Client()   # or chromadb.EphemeralClient()
```

**Persistent** — writes to disk under a path; survives restarts.

```python
client = chromadb.PersistentClient(path="./chroma_db")
```

**HTTP client** — talk to a running Chroma server (Docker / `chroma run`).

```python
client = chromadb.HttpClient(host="localhost", port=8000)
```

> [!tip] Default for vault RAG tutorials Use `PersistentClient(path="./chroma_db")` so re-indexing is optional during iteration.

---

## Collections

A **collection** is a named bucket of embeddings. Create or get by name; optionally set a distance function (`l2`, `cosine`, `ip`).

```python
collection = client.get_or_create_collection(
    name="docs",
    metadata={"hnsw:space": "cosine"},   # cosine is common for text embeddings
)
```

```python
# List / delete
client.list_collections()
client.delete_collection("docs")
```

Chroma can **embed for you** (default sentence-transformers) or accept **precomputed** vectors from OpenAI, etc.

---

## Add — Documents, IDs, Metadata

```python
collection.add(
    ids=["doc-1", "doc-2"],
    documents=[
        "Chroma stores text and embeddings together.",
        "Use metadata for filtering at query time.",
    ],
    metadatas=[
        {"source": "readme", "topic": "chroma"},
        {"source": "blog", "topic": "rag"},
    ],
)
```

```python
# Precomputed embeddings (must match collection dimension)
collection.add(
    ids=["doc-3"],
    embeddings=[[0.1, 0.2, ...]],          # list of float vectors
    documents=["Optional text for display"],
    metadatas=[{"source": "api"}],
)
```

```python
# Upsert — insert or replace by id
collection.upsert(
    ids=["doc-1"],
    documents=["Updated text for doc-1"],
    metadatas=[{"source": "readme", "version": 2}],
)
```

```python
collection.delete(ids=["doc-2"])
collection.update(ids=["doc-1"], metadatas=[{"version": 3}])
```

---

## Query — Similarity Search

```python
results = collection.query(
    query_texts=["How do I filter by metadata?"],
    n_results=5,
    where={"topic": "rag"},              # metadata filter
    where_document={"$contains": "embedding"},  # optional full-text on stored docs
)

# results keys: ids, distances, metadatas, documents, embeddings (if requested)
for i, doc_id in enumerate(results["ids"][0]):
    print(doc_id, results["distances"][0][i], results["documents"][0][i])
```

```python
# Query by embedding vector instead of text
results = collection.query(
    query_embeddings=[[0.05, 0.12, ...]],
    n_results=10,
    include=["metadatas", "documents", "distances"],
)
```

---

## Embeddings Function (Custom)

```python
from chromadb.utils import embedding_functions

openai_ef = embedding_functions.OpenAIEmbeddingFunction(
    api_key="...",
    model_name="text-embedding-3-small",
)

collection = client.get_or_create_collection(
    name="openai_docs",
    embedding_function=openai_ef,
)
collection.add(ids=["1"], documents=["Uses OpenAI embeddings automatically."])
```

---

## LangChain Integration

```python
from langchain_chroma import Chroma
from langchain_openai import OpenAIEmbeddings

vectorstore = Chroma(
    collection_name="docs",
    embedding_function=OpenAIEmbeddings(),
    persist_directory="./chroma_db",
)

vectorstore.add_texts(
    ["First chunk", "Second chunk"],
    metadatas=[{"page": 1}, {"page": 2}],
)

docs = vectorstore.similarity_search("chunk", k=4)
```

See [[AI — LangChain]] for retrievers and RAG chains.

---

## Patterns

### Rebuild index from chunked files

```python
def index_chunks(client, collection_name: str, chunks: list[dict]):
    col = client.get_or_create_collection(collection_name)
    col.upsert(
        ids=[c["id"] for c in chunks],
        documents=[c["text"] for c in chunks],
        metadatas=[c["meta"] for c in chunks],
    )
```

### Reset during tests

```python
client.delete_collection("test_docs")
client.get_or_create_collection("test_docs")
```

---

## Quick Reference

| Task | Code |
| --- | --- |
| In-memory client | `chromadb.Client()` |
| Persist to disk | `PersistentClient(path="./chroma_db")` |
| Get collection | `get_or_create_collection("name")` |
| Insert / update | `add(...)`, `upsert(...)` |
| Similarity search | `query(query_texts=[...], n_results=k, where={...})` |
| Get by id | `get(ids=[...])` |
| Delete | `delete(ids=[...])` or `delete_collection(name)` |
| Custom embeddings | `embedding_functions.OpenAIEmbeddingFunction(...)` |

---

## Tags

#python #ai #chroma #vectordb #rag #embeddings #local-dev #chromadb
