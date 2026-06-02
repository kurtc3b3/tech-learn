## What & When

**Mem0** is a **memory layer for AI agents** — it extracts, stores, updates, and retrieves user-specific facts across conversations. Instead of stuffing full chat history into the context window, you **search relevant memories** and inject only what matters.

Use Mem0 when:

- Personal assistants that remember preferences, names, and past decisions
- Multi-session agents (support bots, copilots, tutoring apps)
- Reducing token cost vs sending entire transcript every turn
- Scoping memory by `user_id`, `agent_id`, or `run_id`

For **document grounding**, use RAG ([[AI — LangChain]], [[AI — Chroma]]). For **RAG quality metrics**, see [[AI — RAGAS]]. Mem0 complements retrieval with **long-term conversational memory**.

```bash
pip install mem0ai
# hybrid BM25 + entity extraction
pip install "mem0ai[nlp]"
python -m spacy download en_core_web_sm
```

Requires **Python 3.10+** and an API key for the default OpenAI-backed stack (`OPENAI_API_KEY`).

Docs: [docs.mem0.ai](https://docs.mem0.ai/) · [Python quickstart](https://docs.mem0.ai/open-source/python-quickstart)

---

## Mem0 vs Related Approaches

| Approach | Stores | Best for |
| --- | --- | --- |
| **Mem0** | Extracted facts (semantic memory) | User prefs, identity, episodic facts |
| Vector RAG | Document chunks | Manuals, PDFs, knowledge bases |
| LangGraph checkpointer | Raw thread state | Exact replay of graph runs ([[AI — LangGraph]]) |
| Chat history in prompt | Full messages | Short threads only |

| Deployment | Class | Setup |
| --- | --- | --- |
| **Self-hosted library** | `Memory()` | `pip install mem0ai`, local Qdrant + SQLite |
| **Mem0 Cloud** | `MemoryClient()` | API key from [app.mem0.ai](https://app.mem0.ai) |
| **Docker server** | HTTP API | `docker compose` (team infra) |

---

## Defaults (open-source `Memory()`)

Out of the box (configurable):

| Component | Default |
| --- | --- |
| LLM (extraction / updates) | OpenAI `gpt-4o-mini` (check docs for current default) |
| Embeddings | `text-embedding-3-small` |
| Vector store | Qdrant on disk under `/tmp/qdrant` |
| History | SQLite at `~/.mem0/history.db` |

Override LLM, embedder, and vector store in a `config` dict for production ([[AI — Qdrant]], local Ollama, etc.).

---

## Quick Start — add & search

```python
from mem0 import Memory

m = Memory()

messages = [
    {"role": "user", "content": "Hi, I'm Alex. I love basketball and gaming."},
    {"role": "assistant", "content": "Hey Alex! I'll remember your interests."},
]
m.add(messages, user_id="alex")

results = m.search("What do you know about me?", filters={"user_id": "alex"})
for item in results["results"]:
    print(item["memory"])
```

`add()` runs **inference** by default: the LLM extracts salient facts and merges them into existing memories (update / delete / insert).

---

## Chat Loop with Memory

```python
from openai import OpenAI
from mem0 import Memory

openai_client = OpenAI()
memory = Memory()


def chat_with_memories(message: str, user_id: str = "default_user") -> str:
    relevant = memory.search(
        query=message,
        filters={"user_id": user_id},
        limit=3,
    )
    memories_str = "\n".join(f"- {e['memory']}" for e in relevant["results"])

    system_prompt = (
        "You are a helpful AI. Use the memories below when answering.\n"
        f"User Memories:\n{memories_str}"
    )
    messages = [
        {"role": "system", "content": system_prompt},
        {"role": "user", "content": message},
    ]
    response = openai_client.chat.completions.create(
        model="gpt-4o-mini",
        messages=messages,
    )
    assistant_response = response.choices[0].message.content

    messages.append({"role": "assistant", "content": assistant_response})
    memory.add(messages, user_id=user_id)

    return assistant_response


print(chat_with_memories("Suggest a weekend activity", user_id="alex"))
```

Pattern: **search → augment system prompt → reply → add** turn to memory.

---

## Scoping & Metadata

| Parameter | Use |
| --- | --- |
| `user_id` | Per-human or per-tenant memory |
| `agent_id` | Per-agent persona or tool surface |
| `run_id` | Ephemeral session / task scope |
| `metadata` | Custom tags (plan tier, locale, …) |

```python
m.add(
    [{"role": "user", "content": "I'm vegetarian"}],
    user_id="alex",
    agent_id="meal_planner",
    metadata={"source": "onboarding"},
)

m.search(
    "dietary restrictions",
    filters={"user_id": "alex", "agent_id": "meal_planner"},
)
```

At least one of `user_id`, `agent_id`, or `run_id` is required for search in most configurations.

---

## CRUD & Cloud

```python
m.get_all(filters={"user_id": "alex"})
m.update(memory_id="mem_abc123", data="Alex prefers plant-based meals")
m.delete(memory_id="mem_abc123")
```

**Cloud:** `MemoryClient(api_key="m0-...")` — same `add` / `search` / dashboard at [app.mem0.ai](https://app.mem0.ai).

---

## `infer=False` — raw storage

Skip LLM fact extraction when you already have structured facts:

```python
m.add(
    "User timezone is America/New_York",
    user_id="alex",
    infer=False,
)
```

Useful for admin overrides or synced CRM fields.

---

## Mem0 + RAG

Use **Mem0** for user-specific facts and **vector RAG** for docs ([[AI — LlamaIndex]], [[AI — Haystack]]). Evaluate grounding with [[AI — RAGAS]].

---

Production: `Memory.from_config({...})` with [[AI — Qdrant]] or your vector host. LangGraph: search in a node, `add` after each turn ([[AI — LangGraph]]). CLI: `mem0 add` / `mem0 search --user-id alice`.

---

## Quick Reference

| Task | API |
| --- | --- |
| Install | `pip install mem0ai` |
| Init | `m = Memory()` |
| Add turn | `m.add(messages, user_id="...")` |
| Search | `m.search(query, filters={"user_id": "..."})` |
| List | `m.get_all(filters={...})` |
| Update / delete | `m.update(id, data)` / `m.delete(id)` |
| Cloud | `MemoryClient(api_key=...)` |
| Hybrid search | `pip install "mem0ai[nlp]"` |

---

## Related Notes

- [[AI]] — memory in the stack map
- [[AI — RAGAS]] — evaluate answers; Mem0 for retention
- [[AI — LangChain]] · [[AI — LangGraph]] · [[AI — Agno]]
- [[AI — Qdrant]] · [[AI — Chroma]] — vector backends
- [[API - FastAPI]] — expose memory in APIs
- [[Python — Pydantic]] — config schemas

---

## Tags

#ai #mem0 #memory #agents #llm #vector-db #personalization #python #rag #obsidian
