## What & When

**ADK (Agent Development Kit)** is Google's **code-first** Python framework for building, debugging, evaluating, and deploying AI agents. You define a `root_agent` in Python, attach **tools** and **sub-agents**, optionally compose **Workflow** graphs (ADK 2.0), then run locally with `adk run` / `adk web` or deploy to Google Cloud (Cloud Run, GKE, Vertex).

Use ADK when:

- You want **Gemini-first** agents with first-class Google Cloud deployment paths
- You need **multi-agent** routing (`sub_agents`, `transfer_to_agent`) without hand-rolling orchestration
- You prefer **deterministic pipelines** via `Workflow` edges plus **Task API** delegation (2.0+)
- You expose agents to other systems via [[AI — A2A]] or tools via [[AI — MCP]]

```bash
pip install google-adk
pip install "google-adk[extensions]"   # optional integrations
```

**Requirements:** Python 3.11+ (docs also mention 3.10+ for some releases).

Docs: [google.github.io/adk-docs](https://google.github.io/adk-docs/) · Repo: [google/adk-python](https://github.com/google/adk-python)

Overview: [[AI]]. Compare [[AI — LangGraph]] for generic graph state machines, [[AI — Agno]] for AgentOS + multi-provider hosting, [[AI — CrewAI]] for role-based crews.

---

## ADK vs Related Frameworks

| Need | Use | Notes |
| --- | --- | --- |
| Google-native agents + deploy | **ADK** | Gemini, `adk` CLI, Cloud Run |
| Provider-agnostic graph / HITL | [[AI — LangGraph]] | Checkpoints, LangChain ecosystem |
| Role-based multi-agent crew | [[AI — CrewAI]] | `Agent` / `Task` / `Crew` |
| FastAPI agent runtime (multi-model) | [[AI — Agno]] | `AgentOS`, Chroma knowledge |
| Remote agent HTTP standard | [[AI — A2A]] | ADK agents can implement A2A servers |
| LLM ↔ tools wire protocol | [[AI — MCP]] | Complementary to ADK function tools |
| Typed single agent | [[AI — Pydantic AI]] | `output_type`, deps injection |

---

## Project Layout

ADK discovers agents from a package folder with `root_agent` exported:

```text
my_project/
    weather_agent/
        __init__.py      # from . import agent
        agent.py         # root_agent = Agent(...)
        .env             # GOOGLE_API_KEY or GEMINI_API_KEY
```

```python
# weather_agent/__init__.py
from . import agent
```

Scaffold with:

```bash
adk create weather_agent
```

---

## Minimal Agent

```python
# weather_agent/agent.py
from google.adk.agents import Agent

root_agent = Agent(
    name="greeting_agent",
    model="gemini-2.5-flash",
    instruction="You are a helpful assistant. Greet the user warmly.",
)
```

### Environment

```bash
# weather_agent/.env
GOOGLE_API_KEY="your-gemini-api-key"
# or GEMINI_API_KEY — see ADK docs for your model provider
```

Get a key from [Google AI Studio](https://aistudio.google.com/app/apikey) for Gemini API, or use Vertex AI credentials for enterprise deploy.

### Run locally

```bash
# Interactive terminal chat
adk run weather_agent

# Dev UI (debug only — not for production)
adk web weather_agent
adk web --port 8000
```

> ADK Web is for **development and debugging** only. Production: Cloud Run, GKE, or Vertex Agent Engine — see [[GCP]].

---

## Tools — Python Functions

Plain functions with docstrings become tools; return structured `dict` for errors vs success.

```python
import datetime
from zoneinfo import ZoneInfo
from google.adk.agents import Agent

def get_weather(city: str) -> dict:
    """Retrieves the current weather report for a specified city."""
    if city.lower() == "new york":
        return {
            "status": "success",
            "report": "Sunny, 25°C in New York.",
        }
    return {
        "status": "error",
        "error_message": f"Weather for '{city}' is not available.",
    }

def get_current_time(city: str) -> dict:
    """Returns the current time in a specified city."""
    if city.lower() != "new york":
        return {
            "status": "error",
            "error_message": f"No timezone for {city}.",
        }
    tz = ZoneInfo("America/New_York")
    now = datetime.datetime.now(tz)
    return {
        "status": "success",
        "report": now.strftime("%Y-%m-%d %H:%M:%S %Z"),
    }

root_agent = Agent(
    name="weather_time_agent",
    model="gemini-2.5-flash",
    description="Answers questions about time and weather in a city.",
    instruction="Use tools for weather and time. Be concise.",
    tools=[get_weather, get_current_time],
)
```

---

## Multi-Agent — `sub_agents` and Transfer

Parent agents delegate via LLM `transfer_to_agent` when `sub_agents` are set. Give each sub-agent a clear `description`.

```python
from google.adk.agents import LlmAgent

booking_agent = LlmAgent(
    name="Booker",
    model="gemini-2.5-flash",
    description="Handles flight and hotel bookings.",
    instruction="Complete booking requests. Ask for missing details.",
)

info_agent = LlmAgent(
    name="Info",
    model="gemini-2.5-flash",
    description="General information and FAQs.",
    instruction="Answer informational questions; do not book travel.",
)

root_agent = LlmAgent(
    name="Coordinator",
    model="gemini-2.5-flash",
    description="Main assistant that routes user requests.",
    instruction=(
        "Delegate booking tasks to Booker and general questions to Info. "
        "Handle simple greetings yourself."
    ),
    sub_agents=[booking_agent, info_agent],
)
```

### Programmatic transfer / escalate in custom tools

```python
from google.adk.tools.tool_context import ToolContext

def route_urgent(query: str, tool_context: ToolContext) -> dict:
    """Routes urgent queries to the support sub-agent."""
    if "urgent" in query.lower():
        tool_context.actions.transfer_to_agent = "Support"
        return {"status": "success", "message": "Transferring to Support."}
    return {"status": "success", "message": "Handled by coordinator."}

def cannot_handle(_: str, tool_context: ToolContext) -> dict:
    """Escalate back to parent when sub-agent is out of scope."""
    tool_context.actions.escalate = True
    return {"status": "escalated", "message": "Returning to parent agent."}
```

Sub-agents without their own `sub_agents` use **`escalate = True`** to return control to the parent — not `transfer_to_agent` upward.

---

## Workflow (ADK 2.0) — Deterministic Graph

For predictable pipelines (no LLM routing between steps), use `Workflow` with explicit edges:

```python
from google.adk import Agent, Workflow

generate_fruit_agent = Agent(
    name="generate_fruit_agent",
    model="gemini-2.5-flash",
    instruction="Return the name of one random fruit. Only the fruit name.",
)

generate_benefit_agent = Agent(
    name="generate_benefit_agent",
    model="gemini-2.5-flash",
    instruction="Given a fruit name, state one health benefit. Be brief.",
)

root_agent = Workflow(
    name="fruit_workflow",
    edges=[("START", generate_fruit_agent, generate_benefit_agent)],
)
```

ADK 2.0 also adds **Task API** for structured agent-to-agent delegation (multi-turn tasks, HITL) — see `contributing/task_samples/` in the ADK repo.

---

## Sessions and State (concept)

ADK manages **sessions** and **events** for multi-turn runs. When upgrading from 1.x → 2.0, note **breaking changes** to session schema and event model (2.0 sessions need ADK ≥ 1.28 to read).

Use session APIs when embedding agents in [[API - FastAPI]] services instead of only `adk run`.

---

## Evaluation and Observability

ADK ships **evaluation** hooks and integrates with Cloud Logging / tracing when deployed on GCP. Pair qualitative evals with [[AI — RAGAS]] when agents use retrieval.

```bash
# Dev UI shows traces per turn — use for debugging tool calls and transfers
adk web my_agent_dir
```

---

## Deploy on Google Cloud (outline)

| Target | When |
| --- | --- |
| **Cloud Run** | HTTP agent API, scale to zero — [[GCP]] |
| **GKE** | Kubernetes control, sidecars, existing [[K8S]] stack |
| **Vertex AI** | Enterprise Gemini, IAM, VPC — [[Machine Learning]] |

Typical path: containerize agent app → Artifact Registry → Cloud Run service with Secret Manager for API keys — [[GCP]], [[Commands/CLI — Docker & Compose]].

Expose externally to other agents with [[AI — A2A]] (Agent Card + JSON-RPC).

---

## ADK + FastAPI (pattern)

Wrap ADK session runner behind your API when you need custom auth, rate limits, or multi-tenant routing:

```python
# Conceptual — exact runner import varies by ADK version; check adk-docs
from fastapi import FastAPI
from google.adk.agents import Agent

app = FastAPI()
agent = Agent(
    name="api_agent",
    model="gemini-2.5-flash",
    instruction="Answer support questions briefly.",
)

@app.post("/chat")
async def chat(message: str, session_id: str):
    # Use ADK Runner / session APIs to invoke agent with session_id
    # return {"reply": ...}
    ...
```

For a batteries-included ASGI stack across providers, compare [[AI — Agno]] `AgentOS`.

---

## Common Mistakes

| Mistake | Fix |
| --- | --- |
| Missing `root_agent` in package | Export from `agent.py`; `__init__.py` imports module |
| No API key in `.env` | Set `GOOGLE_API_KEY` / Vertex credentials per docs |
| Vague sub-agent `description` | LLM cannot route `transfer_to_agent` correctly |
| Sub-agent calls parent via `transfer_to_agent` | Use `tool_context.actions.escalate = True` |
| Using `adk web` in production | Deploy to Cloud Run / GKE |
| Mixing ADK 1.x and 2.0 sessions | Align package versions across services |

---

## Quick Reference

| Task | Command / code |
| --- | --- |
| Install | `pip install google-adk` |
| Scaffold | `adk create my_agent` |
| Define agent | `root_agent = Agent(..., tools=[fn])` |
| CLI chat | `adk run my_agent` |
| Dev UI | `adk web my_agent` |
| Multi-agent | `LlmAgent(..., sub_agents=[...])` |
| Deterministic flow | `Workflow(edges=[("START", a, b)])` |
| Tool transfer | `tool_context.actions.transfer_to_agent = "Name"` |
| Escalate to parent | `tool_context.actions.escalate = True` |
| Remote agents | [[AI — A2A]] Agent Card + executor |

---

## Tags

#ai #adk #google #gemini #agents #workflow #multi-agent #gcp #python #backend
