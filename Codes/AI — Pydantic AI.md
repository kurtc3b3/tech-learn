## What & When

**Pydantic AI** is a Python agent framework from the Pydantic team: build **type-safe agents** with **tools**, **dependency injection** (`RunContext`), and **structured outputs** validated by [[Python — Pydantic]]. Models are referenced as strings (`"openai:gpt-4o"`) or provider classes; the agent loop handles tool calls and retries on validation errors.

Use Pydantic AI when:

- The final answer must match a **Pydantic model** (or union of types)
- Tools need typed **dependencies** (DB connections, settings) injected per run
- You want first-class **async**, streaming structured output, and MCP toolsets
- You already use Pydantic/FastAPI patterns elsewhere in the app

```python
from pydantic_ai import Agent, RunContext
```

```bash
pip install pydantic-ai

# Provider extras as needed
pip install "pydantic-ai[openai]"
```

Set API keys for your providers. See [[AI]] for stack placement; compare [[AI — LangChain]] for largest ecosystem and [[AI — Agno]] for bundled RAG + AgentOS.

---

## Pydantic AI vs Related Frameworks

| Need | Use | Notes |
| --- | --- | --- |
| Typed agent + output | **Pydantic AI** | `output_type`, `@agent.tool` |
| Multi-role crews | [[AI — CrewAI]] | Sequential/hierarchical tasks |
| RAG + hosted API | [[AI — Agno]] | `Knowledge`, `AgentOS` |
| Prompt optimization | [[AI — DSPy]] | Compile-time demo tuning |
| HTTP API layer | [[API - FastAPI]] | Mount agent endpoints yourself |
| Schema definitions | [[Python — Pydantic]] | Shared models for API + agent |

---

## Agent — Basics

```python
from pydantic_ai import Agent

agent = Agent(
    "openai:gpt-4o",
    instructions="You are a helpful assistant. Be concise.",
)

result = agent.run_sync("What is the capital of France?")
print(result.output)   # str by default
```

```python
# Async
result = await agent.run("Explain asyncio in one paragraph.")
print(result.output)
```

`result.all_messages()` exposes the full conversation including tool calls.

---

## Structured Output — `output_type`

Pass a Pydantic model (or union) so the agent **must** return validated data. Failed validation triggers model retries.

```python
from pydantic import BaseModel, Field
from pydantic_ai import Agent

class SupportOutput(BaseModel):
    advice: str = Field(description="Actionable guidance for the customer")
    block_card: bool = Field(description="Whether to block the payment card")
    risk: int = Field(ge=0, le=10, description="Risk score 0-10")

support_agent = Agent(
    "openai:gpt-4o",
    output_type=SupportOutput,
    instructions="You are a bank support agent. Assess risk honestly.",
)

result = support_agent.run_sync("I just lost my card!")
out: SupportOutput = result.output
print(out.advice, out.block_card, out.risk)
```

```python
# Union outputs — model picks one registered output tool/schema
from typing import Literal
from pydantic import BaseModel

class Success(BaseModel):
    kind: Literal["success"] = "success"
    answer: str

class Refusal(BaseModel):
    kind: Literal["refusal"] = "refusal"
    reason: str

agent = Agent("openai:gpt-4o", output_type=Success | Refusal)
```

Stream partial structured objects with `run_stream()` → `result.stream_output()`.

---

## Tools — Typed with `RunContext`

Register functions with `@agent.tool`. Use `RunContext[Deps]` for dependencies; parameters (except `ctx`) become the JSON schema sent to the model.

```python
from dataclasses import dataclass
from pydantic_ai import Agent, RunContext

@dataclass
class SupportDeps:
    customer_id: int

class Database:
    async def get_balance(self, customer_id: int, include_pending: bool) -> float:
        return 123.45

agent = Agent(
    "openai:gpt-4o",
    deps_type=SupportDeps,
    instructions="Help customers with account questions.",
)

@agent.tool
async def customer_balance(
    ctx: RunContext[SupportDeps],
    include_pending: bool,
) -> float:
    """Return the customer's current balance."""
    db = Database()
    return await db.get_balance(ctx.deps.customer_id, include_pending)

deps = SupportDeps(customer_id=42)
result = agent.run_sync("What's my balance?", deps=deps)
print(result.output)
```

`@agent.tool_plain` registers tools that do **not** receive `RunContext` (no deps access).

---

## Dynamic Instructions & Settings

```python
from pydantic_ai import Agent, RunContext

agent = Agent("openai:gpt-4o")

@agent.instructions
def dynamic_instructions(ctx: RunContext[SupportDeps]) -> str:
    return f"Customer ID is {ctx.deps.customer_id}. Be formal."

# Per-run model settings
result = agent.run_sync(
    "Summarize my account",
    deps=deps,
    model_settings={"temperature": 0.2},
)
```

---

## MCP Toolsets

Connect MCP servers as tool sources (same protocol as [[AI — MCP]]):

```python
from pydantic_ai import Agent
from pydantic_ai.toolsets.mcp import MCPServerStdio

server = MCPServerStdio("python", args=["my_mcp_server.py"])
agent = Agent("openai:gpt-4o", toolsets=[server])

async def main():
    async with agent:
        result = await agent.run("List tables in the database")
        print(result.output)
```

Use HTTP/SSE MCP transports when the server is remote (see Pydantic AI MCP docs).

---

## FastAPI Integration Pattern

```python
from fastapi import FastAPI, Depends
from pydantic import BaseModel
from pydantic_ai import Agent

app = FastAPI()
router_agent = Agent("openai:gpt-4o", output_type=str)

class AskRequest(BaseModel):
    question: str

class AskResponse(BaseModel):
    answer: str

@app.post("/ask", response_model=AskResponse)
async def ask(body: AskRequest):
    result = await router_agent.run(body.question)
    return AskResponse(answer=result.output)
```

Share [[Python — Pydantic]] models between REST bodies and `output_type` for one schema source of truth. Add auth, rate limits, and `deps` wiring per request (DB session from `Depends`).

---

## Testing

Swap the model for a **test model** or mock tool implementations; keep agent code unchanged:

```python
from pydantic_ai.models.test import TestModel

test_agent = Agent(TestModel(), output_type=SupportOutput)
result = test_agent.run_sync("test", deps=deps)
assert isinstance(result.output, SupportOutput)
```

Use `agent.override(model=...)` in pytest fixtures for integration tests without calling real LLMs.

---

## Common Mistakes

| Mistake | Fix |
| --- | --- |
| Using `result.data` | Current API uses `result.output` |
| Forgetting `deps=` on `run` | Pass `deps=...` when tools use `RunContext` |
| Unhashable tool args | Tool parameters must be JSON-serializable types |
| Blocking async app | Use `await agent.run()`, not `run_sync`, in async routes |
| Huge `output_type` unions | Split agents or simplify schema for reliability |

---

## Quick Reference

| Task | Code |
| --- | --- |
| Install | `pip install pydantic-ai` |
| Create agent | `Agent("openai:gpt-4o", instructions="...")` |
| Run sync | `agent.run_sync("prompt", deps=deps)` |
| Run async | `await agent.run("prompt", deps=deps)` |
| Structured output | `Agent(..., output_type=MyModel)` → `result.output` |
| Register tool | `@agent.tool` + `RunContext[Deps]` |
| Dynamic prompt | `@agent.instructions` function |
| MCP tools | `toolsets=[MCPServerStdio(...)]` + `async with agent` |
| Messages | `result.all_messages()` |

---

## Tags

#ai #pydantic-ai #agents #structured-output #tools #python #fastapi #typing
