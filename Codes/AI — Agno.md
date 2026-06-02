## What & When

**Agno** (formerly **Phidata**) is a Python SDK for building **agents**, **teams**, and **workflows** with memory, knowledge (RAG), tools, and optional **AgentOS** — a FastAPI runtime that exposes agents as production APIs with sessions, tracing, and persistence.

Use Agno when:

- You want one framework from notebook prototype → hosted API
- Agents need **vector knowledge**, chat history, and 100+ tool integrations (including [[AI — MCP]])
- You coordinate multiple agents as a **Team** or deterministic **Workflow**
- You deploy on [[API - FastAPI]] via `AgentOS` without rewriting orchestration

```python
from agno.agent import Agent
from agno.models.openai import OpenAIChat
```

```bash
pip install agno

# Production API + tracing + session DB
pip install "agno[os]"
```

Configure provider keys (`OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, etc.). See [[AI]] for how Agno fits the stack; compare [[AI — CrewAI]] for role-only crews and [[AI — Pydantic AI]] for maximal typing on a single agent.

---

## Agno vs Related Frameworks

| Need | Use | Notes |
| --- | --- | --- |
| Agent + RAG + tools + API | **Agno** | `Knowledge`, `AgentOS` |
| Role-based multi-agent | [[AI — CrewAI]] | `Process.sequential` / hierarchical |
| Typed deps + output | [[AI — Pydantic AI]] | `RunContext`, `output_type` |
| Graph / HITL | [[AI — LangGraph]] | Checkpoints, branches |
| Tool protocol only | [[AI — MCP]] | `MCPTools` attaches to Agno agents |
| Validation layer | [[Python — Pydantic]] | Structured I/O on agents |

---

## Agent — Model, Tools, Memory

```python
from agno.agent import Agent
from agno.models.openai import OpenAIChat
from agno.tools.duckduckgo import DuckDuckGoTools

agent = Agent(
    name="Research Assistant",
    model=OpenAIChat(id="gpt-4o"),
    tools=[DuckDuckGoTools()],
    instructions="You are a concise research assistant. Cite sources when possible.",
    markdown=True,
    add_datetime_to_context=True,
)

agent.print_response("What changed in MCP in 2026?", stream=True)
```

**Session memory** — persist runs and inject history:

```python
from agno.agent import Agent
from agno.db.sqlite import SqliteDb
from agno.models.anthropic import Claude

agent = Agent(
    name="Support Bot",
    model=Claude(id="claude-sonnet-4-5"),
    db=SqliteDb(db_file="agno.db"),
    add_history_to_context=True,
    num_history_runs=5,
    user_id="user-42",              # per-user isolation
    session_id="session-abc",
)

agent.print_response("Remember my name is Ali.")
agent.print_response("What's my name?")   # uses stored history
```

Model strings also support the unified form: `model="openai:gpt-4o"`.

---

## Knowledge — RAG on Agents

Load documents into a vector DB; the agent searches when `search_knowledge=True` (agentic RAG — the model decides when to retrieve).

```python
from agno.agent import Agent
from agno.knowledge.knowledge import Knowledge
from agno.models.openai import OpenAIChat
from agno.vectordb.chroma import ChromaDb

knowledge = Knowledge(
    vector_db=ChromaDb(collection="docs", path="tmp/chromadb"),
)

knowledge.insert(url="https://docs.agno.com/introduction.md")
# knowledge.insert(path="docs/handbook.pdf")

agent = Agent(
    model=OpenAIChat(id="gpt-4o-mini"),
    knowledge=knowledge,
    search_knowledge=True,
    instructions="Answer only from knowledge when possible; say if unsure.",
)

agent.print_response("What is Agno?")
```

Agents can **write** to knowledge via custom tools (learning loop):

```python
def save_insight(title: str, text: str) -> str:
    knowledge.insert(name=title, text_content=text)
    return f"Saved: {title}"

agent = Agent(
    knowledge=knowledge,
    search_knowledge=True,
    tools=[save_insight],
)
```

---

## Tools & MCP

Agno ships DuckDuckGo, YFinance, Slack, and many more. Connect external MCP servers with `MCPTools`:

```python
from agno.agent import Agent
from agno.models.anthropic import Claude
from agno.tools.mcp import MCPTools

agent = Agent(
    name="Docs Assistant",
    model=Claude(id="claude-sonnet-4-5"),
    tools=[MCPTools(url="https://docs.agno.com/mcp")],
    markdown=True,
)
```

Build your own MCP servers with [[AI — MCP]] and attach them the same way.

---

## Team — Multi-Agent Coordination

A **Team** routes a single user message across member agents (researcher, writer, etc.):

```python
from agno.agent import Agent
from agno.models.openai import OpenAIChat
from agno.team import Team

researcher = Agent(
    name="Researcher",
    role="Find accurate, recent facts",
    model=OpenAIChat(id="gpt-4o-mini"),
)

writer = Agent(
    name="Writer",
    role="Turn research into clear prose",
    model=OpenAIChat(id="gpt-4o-mini"),
)

team = Team(
    name="Content Team",
    members=[researcher, writer],
    model=OpenAIChat(id="gpt-4o"),   # team-level coordinator model
    instructions="Research first, then write a short article.",
)

team.print_response("Explain MCP for backend engineers", stream=True)
```

For step-by-step pipelines (not free-form coordination), use **Workflow** primitives in the Agno docs.

---

## AgentOS — FastAPI Production Runtime

Wrap agents (and teams) in **AgentOS** for REST endpoints, OpenAPI docs, tracing, and session storage:

```python
from agno.agent import Agent
from agno.db.sqlite import SqliteDb
from agno.models.anthropic import Claude
from agno.os import AgentOS
from agno.tools.mcp import MCPTools

assistant = Agent(
    name="Agno Assist",
    model=Claude(id="claude-sonnet-4-5"),
    db=SqliteDb(db_file="agno.db"),
    tools=[MCPTools(url="https://docs.agno.com/mcp")],
    add_history_to_context=True,
    num_history_runs=3,
    markdown=True,
)

agent_os = AgentOS(agents=[assistant], tracing=True)
app = agent_os.get_app()   # ASGI app for uvicorn / fastapi dev
```

```bash
export ANTHROPIC_API_KEY="sk-..."
fastapi dev agno_assist.py    # http://localhost:8000 — /docs for OpenAPI
```

Typical endpoints (auto-generated): `POST /agents/{agent_id}/run`, `POST /agents/{agent_id}/stream`. Extend the app with custom [[API - FastAPI]] routes on the same `app` instance.

---

## Backend Patterns

```python
# Structured response with Pydantic (Agno structured I/O)
from pydantic import BaseModel, Field
from agno.agent import Agent

class Summary(BaseModel):
    title: str = Field(..., description="Short headline")
    bullets: list[str] = Field(..., max_length=5)

agent = Agent(
    model="openai:gpt-4o",
    output_schema=Summary,
    instructions="Return a structured summary.",
)

response = agent.run("Summarize vector database tradeoffs")
# response.content is validated Summary
```

```python
# Docker / production: run AgentOS with gunicorn + uvicorn workers
# CMD ["gunicorn", "agno_assist:app", "-k", "uvicorn.workers.UvicornWorker", "-w", "4"]
```

Use Postgres (`PostgresDb`) instead of `SqliteDb` for multi-instance deployments.

---

## Common Mistakes

| Mistake | Fix |
| --- | --- |
| No `db=` but expecting memory | Add `SqliteDb` / `PostgresDb` + `add_history_to_context` |
| Knowledge never searched | `search_knowledge=True` on `Agent` |
| Missing `[os]` extras | `pip install "agno[os]"` for AgentOS |
| Sync `print_response` in async API | Use `agent.arun()` / streaming APIs in async routes |
| MCP tool URL unreachable | Verify server transport (stdio vs HTTP) per [[AI — MCP]] |

---

## Quick Reference

| Task | Code |
| --- | --- |
| Install | `pip install agno` / `pip install "agno[os]"` |
| Basic agent | `Agent(model=OpenAIChat(id="gpt-4o"), tools=[...])` |
| CLI chat | `agent.print_response("...", stream=True)` |
| Session DB | `db=SqliteDb(db_file="agno.db")` |
| Knowledge RAG | `Knowledge(vector_db=ChromaDb(...))` + `search_knowledge=True` |
| MCP tools | `tools=[MCPTools(url="...")]` |
| Team | `Team(members=[a, b], model=...)` |
| Serve API | `AgentOS(agents=[...]).get_app()` + `fastapi dev` |
| Tracing | `AgentOS(..., tracing=True)` |

---

## Tags

#ai #agno #phidata #agents #rag #fastapi #mcp #python #backend
