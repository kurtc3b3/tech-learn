## What & When

**FAISS** (Facebook AI Similarity Search) is a **C++ library with Python bindings** for efficient **nearest-neighbor search** on dense vectors. It runs **in-process** in your Python app — no separate vector database server.

Use FAISS when:

- You need **maximum search throughput** on a fixed index in RAM (or GPU)
- The index fits in memory and you manage **IDs, metadata, and persistence** yourself
- Building batch offline indexes (research, eval, one-shot retrieval)
- Pairing with [[AI — LangChain]] `FAISS` vectorstore or custom RAG without ops overhead

```python
import faiss
import numpy as np
```

```bash
pip install faiss-cpu
# GPU builds: pip install faiss-gpu  (CUDA required; index types may differ)
```

For metadata + document storage out of the box, use [[AI — Chroma]] or [[AI — Qdrant]]. For cluster scale, [[AI — Milvus]]. Overview: [[AI]].

---

## FAISS vs Other Vector Stores

| Need | Use | Notes |
| --- | --- | --- |
| In-process, fastest ANN | **FAISS** | You own sidecar metadata (dict/SQLite) |
| Embeddings + docs + filters | [[AI — Chroma]] | Higher-level API |
| Production filters / REST | [[AI — Qdrant]] | Server + payload indexes |
| Distributed billion-scale | [[AI — Milvus]] | Cluster + pymilvus |
| RAG abstractions | [[AI — LlamaIndex]] | Can wrap FAISS indirectly |

---

## Vectors — Shape and dtype

FAISS expects a **2D float32** array: `n_vectors × dimension`.

```python
import numpy as np

d = 384
nb = 10_000
vectors = np.random.random((nb, d)).astype("float32")

# L2 indexes assume unnormalized vectors; cosine often uses normalized + inner product
faiss.normalize_L2(vectors)   # in-place; then use IndexFlatIP for cosine similarity
```

---

## IndexFlatL2 — Exact L2 Search

**IndexFlatL2** brute-forces Euclidean distance — exact, simple, best for small/medium `n` or baselines.

```python
import faiss

d = 384
index = faiss.IndexFlatL2(d)     # dimension d
index.add(vectors)               # nb vectors added; ids are 0 .. nb-1

q = np.random.random((1, d)).astype("float32")
distances, indices = index.search(q, k=5)
# distances[i,j] = squared L2; indices[i,j] = row id in index
```

```python
# Inner product (cosine after normalize_L2)
index_ip = faiss.IndexFlatIP(d)
index_ip.add(vectors)
distances, indices = index_ip.search(q, k=5)   # higher inner product = more similar
```

> [!tip] Flat indexes are exact Use `IndexFlatL2` / `IndexFlatIP` to validate recall before switching to IVF or HNSW approximations.

---

## IndexIVFFlat — Faster Approximate Search

**IVF** partitions vectors into `nlist` clusters; at query time only `nprobe` clusters are searched — faster, approximate.

```python
nlist = 100
quantizer = faiss.IndexFlatL2(d)
index = faiss.IndexIVFFlat(quantizer, d, nlist, faiss.METRIC_L2)

assert not index.is_trained
index.train(vectors)          # requires representative sample (often all training data)
index.add(vectors)

index.nprobe = 10               # tradeoff: higher = slower, better recall
distances, indices = index.search(q, k=5)
```

Train on a subset if `nb` is huge; `nlist` ≈ `sqrt(nb)` is a common starting point.

---

## IDs, Metadata, and Deletion

FAISS stores only vectors and integer positions. Map **external ids** yourself:

```python
id_map: list[str] = []   # faiss_row -> your_id

def add_with_ids(index, batch_vectors, batch_ids):
    start = index.ntotal
    index.add(batch_vectors)
    id_map.extend(batch_ids)

# Resolve search result
faiss_row = int(indices[0][0])
doc_id = id_map[faiss_row]
```

Standard indexes do not support efficient delete; for mutable corpora prefer [[AI — Qdrant]] or rebuild the index.

---

## Save and Load

```python
faiss.write_index(index, "my_index.faiss")
index_loaded = faiss.read_index("my_index.faiss")
```

```python
# IndexIVF — save with IDs map separately (pickle/json your id_map + metadata)
import pickle

with open("id_map.pkl", "wb") as f:
    pickle.dump({"id_map": id_map, "meta": meta_by_id}, f)
```

```python
# GPU index (faiss-gpu): clone to GPU for search, optionally keep CPU copy
res = faiss.StandardGpuResources()
gpu_index = faiss.index_cpu_to_gpu(res, 0, index)
distances, indices = gpu_index.search(q, k=5)
```

---

## LangChain FAISS Wrapper

```python
from langchain_community.vectorstores import FAISS
from langchain_openai import OpenAIEmbeddings

vectorstore = FAISS.from_texts(
    ["alpha", "beta", "gamma"],
    embedding=OpenAIEmbeddings(),
)

docs = vectorstore.similarity_search("alpha", k=2)
vectorstore.save_local("faiss_store")
loaded = FAISS.load_local("faiss_store", OpenAIEmbeddings(), allow_dangerous_deserialization=True)
```

LangChain stores the index file plus docstore pickle — convenient for demos; for production control, use raw `faiss` + your metadata layer. See [[AI — LangChain]].

---

## Patterns

### Build once, serve many queries

```python
def build_flat_index(vectors: np.ndarray) -> faiss.Index:
    d = vectors.shape[1]
    index = faiss.IndexFlatL2(d)
    index.add(vectors)
    return index
```

### Evaluate recall vs IVF

Compare `IndexFlatL2` top-k ids with `IndexIVFFlat` at various `nprobe` on a held-out query set before choosing production settings.

### Batch search

```python
queries = np.random.random((32, d)).astype("float32")
D, I = index.search(queries, k=10)   # shape (32, 10)
```

---

## Quick Reference

| Task | Code |
| --- | --- |
| Exact L2 | `IndexFlatL2(d)` + `add(vectors)` + `search(q, k)` |
| Cosine (normalized) | `normalize_L2(vectors)` + `IndexFlatIP(d)` |
| Approximate IVF | `IndexIVFFlat(quantizer, d, nlist)` + `train()` + `add()` |
| Tune IVF speed/recall | `index.nprobe = N` |
| Persist index | `write_index(index, path)` / `read_index(path)` |
| GPU search | `index_cpu_to_gpu(res, 0, index)` (faiss-gpu) |
| External doc ids | Maintain parallel `id_map` list or dict |

---

## Tags

#python #ai #faiss #vectordb #embeddings #ann #in-process #rag #numpy
