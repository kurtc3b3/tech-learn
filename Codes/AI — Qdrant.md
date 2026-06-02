## What & When

**Qdrant** is a vector database focused on **production-grade similarity search** with rich **payload filtering**, optional **sparse vectors**, and hybrid retrieval. The **`qdrant-client`** library talks to self-hosted Qdrant or **Qdrant Cloud**.

Use Qdrant when:

- You need **metadata filters** (tenant, date, tags) combined with vector search in one API
- Deploying a dedicated vector service (Docker, K8s, or cloud) between dev and [[AI — Milvus]] cluster scale
- Hybrid / multi-vector experiments (dense + sparse) on a managed stack
- Moving from [[AI — Chroma]] local dev to a production-shaped API without Milvus ops complexity

```python
from qdrant_client import QdrantClient
from qdrant_client.models import Distance, VectorParams, PointStruct, Filter, FieldCondition, MatchValue
```

```bash
pip install qdrant-client
```

Run Qdrant via Docker (`qdrant/qdrant`) or use in-memory / local path mode for tests. Overview: [[AI]].

---

## Qdrant vs Other Vector Stores

| Need | Use | Notes |
| --- | --- | --- |
| Filters + production REST/gRPC | **Qdrant** | Payload indexes on metadata fields |
| Fastest local embedded | [[AI — Chroma]] | Fewer moving parts |
| In-process only | [[AI — FAISS]] | No payload DB |
| Massive distributed scale | [[AI — Milvus]] | Heavier cluster story |
| RAG orchestration | [[AI — LangChain]], [[AI — LlamaIndex]] | Native Qdrant integrations |

---

## Client Modes

```python
from qdrant_client import QdrantClient

# Remote server
client = QdrantClient(url="http://localhost:6333")

# Local persistence (embedded storage path)
client = QdrantClient(path="./qdrant_data")

# In-memory (tests)
client = QdrantClient(":memory:")
```

```python
# API key for Qdrant Cloud
# client = QdrantClient(url="https://xxx.cloud.qdrant.io", api_key="...")
```

---

## Collections

```python
from qdrant_client.models import Distance, VectorParams

client.recreate_collection(
    collection_name="docs",
    vectors_config=VectorParams(size=384, distance=Distance.COSINE),
)
```

```python
# Named vectors (multi-vector per point) — advanced layouts
client.recreate_collection(
    collection_name="hybrid",
    vectors_config={
        "dense": VectorParams(size=384, distance=Distance.COSINE),
        "sparse": VectorParams(size=30000, distance=Distance.DOT),
    },
)
```

```python
client.get_collections()
client.delete_collection("docs")
```

---

## Upsert Points

Each **point** has `id`, `vector`, and optional **payload** (JSON metadata + stored text).

```python
from qdrant_client.models import PointStruct

points = [
    PointStruct(
        id=1,
        vector=[0.01] * 384,
        payload={"text": "Qdrant supports payload filters.", "tenant": "acme", "page": 1},
    ),
    PointStruct(
        id=2,
        vector=[0.02] * 384,
        payload={"text": "Use upsert for idempotent indexing.", "tenant": "acme", "page": 2},
    ),
]

client.upsert(collection_name="docs", points=points)
```

```python
# Upsert with numpy / list batch via upload_collection or parallel helper in client
client.upload_points(collection_name="docs", points=points)
```

Use **integer or UUID** ids consistently; upsert replaces existing ids.

---

## Search — Vector + Filters

```python
from qdrant_client.models import Filter, FieldCondition, MatchValue

query_vector = [0.015] * 384

hits = client.search(
    collection_name="docs",
    query_vector=query_vector,
    limit=5,
    query_filter=Filter(
        must=[
            FieldCondition(key="tenant", match=MatchValue(value="acme")),
        ]
    ),
)

for hit in hits:
    print(hit.id, hit.score, hit.payload.get("text"))
```

```python
# Scroll / retrieve without vector (pagination)
records, next_offset = client.scroll(
    collection_name="docs",
    scroll_filter=Filter(must=[FieldCondition(key="page", match=MatchValue(value=2))]),
    limit=10,
    with_payload=True,
    with_vectors=False,
)
```

---

## Payload Indexes (Faster Filters)

```python
from qdrant_client.models import PayloadSchemaType

client.create_payload_index(
    collection_name="docs",
    field_name="tenant",
    field_schema=PayloadSchemaType.KEYWORD,
)
```

Index fields you filter on often (`tenant`, `category`, `date` as keyword/integer).

---

## LangChain Integration

```python
from langchain_qdrant import QdrantVectorStore
from langchain_openai import OpenAIEmbeddings

vectorstore = QdrantVectorStore.from_documents(
    documents,   # LangChain Document list
    embedding=OpenAIEmbeddings(),
    url="http://localhost:6333",
    collection_name="langchain_docs",
)

results = vectorstore.similarity_search("filters", k=4)
```

Wire retrievers into chains via [[AI — LangChain]].

---

## Patterns

### Tenant isolation

Single collection + `query_filter` on `tenant`, or separate collections per tenant for hard isolation.

### Idempotent reindex

Upsert by stable chunk id (`hash(doc_id + chunk_idx)`) so pipeline reruns do not duplicate points.

### Dev → prod

Same client API; swap `url` / `api_key` and collection name prefix (`dev_docs` vs `prod_docs`).

---

## Quick Reference

| Task | Code |
| --- | --- |
| Connect | `QdrantClient(url=...)` or `path=...` |
| Create collection | `recreate_collection(..., vectors_config=VectorParams(...))` |
| Upsert | `upsert(collection_name, points=[PointStruct(...)])` |
| Vector search | `search(collection_name, query_vector, limit=k, query_filter=...)` |
| Filter | `Filter(must=[FieldCondition(key, match=MatchValue(...))])` |
| Scroll | `scroll(collection_name, scroll_filter=..., limit=...)` |
| Payload index | `create_payload_index(..., field_schema=KEYWORD)` |
| Delete collection | `delete_collection(name)` |

---

## Tags

#python #ai #qdrant #vectordb #rag #embeddings #filters #production #hybrid-search
