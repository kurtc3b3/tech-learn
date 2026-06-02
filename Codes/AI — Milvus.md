## What & When

**Milvus** is an open-source **vector database** built for large-scale similarity search, filtering, and hybrid retrieval. The Python client **pymilvus** talks to Milvus standalone, cluster, or **Zilliz Cloud**.

Use Milvus when:

- Index size or QPS exceeds comfortable **embedded** limits ([[AI — Chroma]], [[AI — FAISS]])
- You need **production HA**, replication, and collection management at scale
- Running on Kubernetes or managed Zilliz with **pymilvus** as the app interface
- Combining dense vectors with scalar filters and (in newer versions) sparse / hybrid search

```python
from pymilvus import connections, Collection, FieldSchema, CollectionSchema, DataType, utility
```

```bash
pip install pymilvus
```

Milvus server runs separately (Docker Compose, Helm, or Zilliz). For lighter self-hosted ops with strong filters, compare [[AI — Qdrant]]. Stack map: [[AI]].

---

## Milvus vs Other Vector Stores

| Need | Use | Notes |
| --- | --- | --- |
| Cluster / very large corpora | **Milvus** | pymilvus + Milvus or Zilliz |
| Rich payload filters, simpler ops | [[AI — Qdrant]] | Often faster to adopt for mid-size prod |
| Local dev, embedded | [[AI — Chroma]] | No cluster |
| In-process library only | [[AI — FAISS]] | No server |
| RAG framework layer | [[AI — LlamaIndex]], [[AI — LangChain]] | Milvus integrations available |

---

## Connect

```python
from pymilvus import connections

connections.connect(
    alias="default",
    host="localhost",
    port="19530",
)

# Zilliz Cloud example
# connections.connect(
#     alias="default",
#     uri="https://xxx.api.gcp-us-west1.zillizcloud.com",
#     token="user:password_or_api_key",
# )
```

```python
connections.disconnect("default")
```

---

## Schema and Collection

Define **fields**: primary key, vector field, optional scalars for filtering.

```python
from pymilvus import FieldSchema, CollectionSchema, DataType, Collection

dim = 384

fields = [
    FieldSchema(name="id", dtype=DataType.INT64, is_primary=True, auto_id=False),
    FieldSchema(name="embedding", dtype=DataType.FLOAT_VECTOR, dim=dim),
    FieldSchema(name="source", dtype=DataType.VARCHAR, max_length=256),
    FieldSchema(name="chunk_index", dtype=DataType.INT64),
]

schema = CollectionSchema(fields, description="RAG chunks")
collection = Collection(name="rag_chunks", schema=schema)
```

```python
# Drop if re-creating in dev
from pymilvus import utility
if utility.has_collection("rag_chunks"):
    utility.drop_collection("rag_chunks")
```

---

## Index and Load

Create an **ANN index** on the vector field, then **load** into memory for search.

```python
index_params = {
    "index_type": "IVF_FLAT",      # also HNSW, IVF_SQ8, etc.
    "metric_type": "L2",           # or IP, COSINE
    "params": {"nlist": 128},
}

collection.create_index(field_name="embedding", index_params=index_params)
collection.load()
```

```python
# HNSW example (common for latency-sensitive search)
index_params = {
    "index_type": "HNSW",
    "metric_type": "COSINE",
    "params": {"M": 16, "efConstruction": 200},
}
```

---

## Insert

Rows must match schema column order (or use dict-style insert in ORM-like helpers).

```python
import random

nb = 1000
ids = list(range(nb))
vectors = [[random.random() for _ in range(dim)] for _ in range(nb)]
sources = [f"doc-{i % 50}.pdf" for i in range(nb)]
chunk_idxs = [i % 20 for i in range(nb)]

collection.insert([ids, vectors, sources, chunk_idxs])
collection.flush()   # persist segments before search
```

```python
print(collection.num_entities)
```

---

## Search

```python
search_params = {"metric_type": "L2", "params": {"nprobe": 16}}

query_vectors = [[random.random() for _ in range(dim)]]

results = collection.search(
    data=query_vectors,
    anns_field="embedding",
    param=search_params,
    limit=5,
    expr='source == "doc-3.pdf"',     # boolean filter on scalars
    output_fields=["source", "chunk_index"],
)

for hits in results:
    for hit in hits:
        print(hit.id, hit.distance, hit.entity.get("source"))
```

```python
# Query by primary key
collection.query(expr="id in [1, 2, 3]", output_fields=["source", "embedding"])
```

---

## Upsert and Delete (Milvus 2.3+)

```python
# Upsert replaces rows with same primary key
collection.upsert([ids, vectors, sources, chunk_idxs])
collection.flush()

collection.delete(expr='source == "doc-3.pdf"')
collection.flush()
```

Check your server version for exact upsert/delete semantics.

---

## LangChain / LlamaIndex

```python
# LangChain community vectorstore
from langchain_community.vectorstores import Milvus
from langchain_openai import OpenAIEmbeddings

vectorstore = Milvus(
    embedding_function=OpenAIEmbeddings(),
    collection_name="langchain_docs",
    connection_args={"host": "localhost", "port": "19530"},
)
vectorstore.add_texts(["hello milvus"])
```

See [[AI — LangChain]] and [[AI — LlamaIndex]] for higher-level RAG pipelines.

---

## Patterns

### One collection per environment

`rag_dev`, `rag_staging`, `rag_prod` — same schema, different connection aliases or URIs.

### Reindex workflow

Create new collection → bulk insert → build index → alias swap (application config) → drop old collection.

---

## Quick Reference

| Task | Code |
| --- | --- |
| Connect | `connections.connect(host=..., port=...)` |
| Define schema | `FieldSchema` + `CollectionSchema` + `Collection(...)` |
| Create ANN index | `create_index("embedding", index_params)` |
| Load for search | `collection.load()` |
| Insert rows | `insert([ids, vectors, ...])` + `flush()` |
| Vector search | `search(data=..., anns_field="embedding", limit=k, expr=...)` |
| Scalar filter | `expr='source == "x.pdf"'` |
| Query by id | `query(expr="id in [...]", output_fields=[...])` |
| Delete | `delete(expr=...)` |

---

## Tags

#python #ai #milvus #pymilvus #vectordb #rag #embeddings #scale #zilliz
