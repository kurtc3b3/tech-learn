## What & When

**Microsoft Agent Framework (MAF)** is an open-source, **production-grade** Python (and .NET) SDK for building **single agents**, **graph workflows**, and **multi-agent orchestrations** — with middleware, sessions, checkpointing, human-in-the-loop tool approval, and OpenTelemetry observability. It is the successor path for **Semantic Kernel** and **AutoGen** users.

Use Agent Framework when:

- You need **stable 1.x APIs** for agents you expect to run in production
- You want **graph workflows** (`WorkflowBuilder`) plus high-level patterns (`SequentialBuilder`, concurrent, handoff, group chat)
- You deploy on **Azure AI Foundry**, Azure OpenAI, or OpenAI — and want one client abstraction
- You need **tool approval** (`@tool(approval_mode="always_require")`) or **MCP** integration built into agents
- You host agents on **Foundry**, Azure Functions, or expose them via [[AI — A2A]]

```bash
pip install agent-framework          # all integrations (simplest for learning)
# or selective:
pip install agent-framework-core       # OpenAI + Azure OpenAI + workflows
pip install agent-framework-foundry    # + Azure AI Foundry
```

**Python 3.10+** · Docs: [learn.microsoft.com/agent-framework](https://learn.microsoft.com/en-us/agent-framework/) · Repo: [microsoft/agent-framework](https://github.com/microsoft/agent-framework)

Overview: [[AI]]. Compare [[AI — LangGraph]] for LangChain-native graphs, [[AI — ADK]] for Google/Gemini, [[AI — Agno]] for AgentOS hosting, [[AI — CrewAI]] for role crews.

---

## Agent Framework vs Related Tools

| Need | Use | Notes |
| --- | --- | --- |
| LangChain ecosystem RAG + graphs | [[AI — LangGraph]] | Checkpoints, LangSmith |
| Google Gemini + Cloud Run deploy | [[AI — ADK]] | `adk run`, [[GCP]] |
| Multi-provider agent API + RAG | [[AI — Agno]] | `AgentOS`, Chroma knowledge |
| Role-based handoffs | [[AI — CrewAI]] | Researcher → writer |
| Typed deps + output | [[AI — Pydantic AI]] | `RunContext`, `output_type` |
| Azure / Foundry production agents | **Agent Framework** | Workflows, middleware, OTel |
| Tool wire protocol | [[AI — MCP]] | MAF agents support local/hosted MCP |
| Remote agents | [[AI — A2A]] | MAF hosting samples |

---

## Environment

### OpenAI (simplest local path)

```bash
# .env
OPENAI_API_KEY=sk-...
OPENAI_MODEL=gpt-4o-mini
```

### Azure AI Foundry

```bash
export FOUNDRY_PROJECT_ENDPOINT="https://your-project.services.ai.azure.com"
export FOUNDRY_MODEL="gpt-4o"
az login   # AzureCliCredential
```

---

## Hello Agent — OpenAI

```python
import asyncio
import os

from agent_framework import Agent
from agent_framework.openai import OpenAIChatCompletionClient
from dotenv import load_dotenv

load_dotenv()


async def main() -> None:
    agent = Agent(
        client=OpenAIChatCompletionClient(
            model=os.getenv("OPENAI_MODEL", "gpt-4o-mini"),
            api_key=os.getenv("OPENAI_API_KEY"),
        ),
        name="HelloAgent",
        instructions="You are a friendly assistant. Keep answers brief.",
    )

    result = await agent.run("What is the capital of France?")
    print(result)

    print("Streaming: ", end="", flush=True)
    async for chunk in agent.run("One fun fact about Paris.", stream=True):
        if chunk.text:
            print(chunk.text, end="", flush=True)
    print()


asyncio.run(main())
```

---

## Hello Agent — Azure AI Foundry

```python
import asyncio
import os

from agent_framework import Agent
from agent_framework.foundry import FoundryChatClient
from azure.identity import AzureCliCredential


async def main() -> None:
    agent = Agent(
        client=FoundryChatClient(
            project_endpoint=os.environ["FOUNDRY_PROJECT_ENDPOINT"],
            model=os.environ.get("FOUNDRY_MODEL", "gpt-4o"),
            credential=AzureCliCredential(),
        ),
        name="HaikuAgent",
        instructions="You are an upbeat assistant that writes beautifully.",
    )

    print(await agent.run("Write a haiku about Microsoft Agent Framework."))


asyncio.run(main())
```

Swap `FoundryChatClient` for `OpenAIChatClient` (Responses API) or provider-specific clients — see `python/samples/02-agents/providers/` in the repo.

---

## Tools — `@tool` Decorator

Function tools use [[Python — Pydantic]]-style annotations; the model decides when to call them.

```python
import asyncio
import os
from random import randint
from typing import Annotated

from agent_framework import Agent, tool
from agent_framework.openai import OpenAIChatCompletionClient
from dotenv import load_dotenv
from pydantic import Field

load_dotenv()


@tool(approval_mode="never_require")  # use "always_require" in production
def get_weather(
    location: Annotated[str, Field(description="City or region to look up.")],
) -> str:
    """Get the weather for a given location."""
    conditions = ["sunny", "cloudy", "rainy", "stormy"]
    return (
        f"Weather in {location}: {conditions[randint(0, 3)]}, "
        f"high {randint(10, 30)}°C."
    )


async def main() -> None:
    agent = Agent(
        client=OpenAIChatCompletionClient(
            model=os.getenv("OPENAI_MODEL", "gpt-4o-mini"),
            api_key=os.getenv("OPENAI_API_KEY"),
        ),
        name="WeatherAgent",
        instructions="Use get_weather to answer weather questions.",
        tools=[get_weather],
    )

    result = await agent.run("What's the weather like in Seattle?")
    print(result)


asyncio.run(main())
```

**Human-in-the-loop:** `@tool(approval_mode="always_require")` pauses the workflow until a human approves the tool call — useful for delete/send/payment operations.

---

## Multi-Turn — `AgentSession`

Reuse one session object to keep conversation history across `run()` calls:

```python
import asyncio
import os

from agent_framework import Agent
from agent_framework.openai import OpenAIChatCompletionClient
from dotenv import load_dotenv

load_dotenv()


async def main() -> None:
    agent = Agent(
        client=OpenAIChatCompletionClient(
            model=os.getenv("OPENAI_MODEL", "gpt-4o-mini"),
            api_key=os.getenv("OPENAI_API_KEY"),
        ),
        name="ConversationAgent",
        instructions="You are a friendly assistant.",
    )

    session = agent.create_session()

    print(await agent.run("My name is Alice and I love hiking.", session=session))
    print(await agent.run("What do you remember about me?", session=session))


asyncio.run(main())
```

---

## Structured Output — Pydantic

```python
import asyncio

from agent_framework import Agent, AgentResponse
from agent_framework.openai import OpenAIChatClient
from dotenv import load_dotenv
from pydantic import BaseModel

load_dotenv()


class CityInfo(BaseModel):
    city: str
    description: str


async def main() -> None:
    agent = Agent(
        client=OpenAIChatClient(),
        name="CityAgent",
        instructions="Describe cities in a structured format.",
    )

    result = await agent.run(
        "Tell me about Paris, France",
        options={"response_format": CityInfo},
    )

    if data := result.value:
        print(f"{data.city}: {data.description}")
    else:
        print(result.text)


asyncio.run(main())
```

For streaming + structured output, collect chunks with `AgentResponse.from_update_generator(...)`.

---

## Graph Workflow — `WorkflowBuilder`

Low-level graph API: chain **executors** with edges — no LLM required for the topology itself.

```python
import asyncio

from agent_framework import Executor, WorkflowBuilder, WorkflowContext, executor, handler
from typing_extensions import Never


class UpperCase(Executor):
    def __init__(self, id: str):
        super().__init__(id=id)

    @handler
    async def to_upper(self, text: str, ctx: WorkflowContext[str]) -> None:
        await ctx.send_message(text.upper())


@executor(id="reverse_text")
async def reverse_text(text: str, ctx: WorkflowContext[Never, str]) -> None:
    await ctx.yield_output(text[::-1])


async def main() -> None:
    upper = UpperCase(id="upper_case")
    workflow = WorkflowBuilder(start_executor=upper).add_edge(upper, reverse_text).build()

    events = await workflow.run("hello world")
    print(events.get_outputs())   # ['DLROW OLLEH']


asyncio.run(main())
```

Add LLM agents as executors, fan-out/fan-in, switch/case, and checkpointing for long-running flows.

---

## Sequential Multi-Agent — `SequentialBuilder`

High-level pattern: chain agents that share a **conversation context** (`list[Message]`).

```python
import asyncio
import os
from typing import cast

from agent_framework import Agent, AgentResponse, Message
from agent_framework.foundry import FoundryChatClient
from agent_framework.orchestrations import SequentialBuilder
from azure.identity import AzureCliCredential
from dotenv import load_dotenv

load_dotenv()


async def main() -> None:
    client = FoundryChatClient(
        project_endpoint=os.environ["FOUNDRY_PROJECT_ENDPOINT"],
        model=os.environ["FOUNDRY_MODEL"],
        credential=AzureCliCredential(),
    )

    writer = Agent(
        client=client,
        name="writer",
        instructions="You are a concise copywriter. One punchy sentence only.",
    )
    reviewer = Agent(
        client=client,
        name="reviewer",
        instructions="You are a reviewer. Give brief feedback on the previous message.",
    )

    workflow = SequentialBuilder(
        participants=[writer, reviewer],
        output_from="all",
    ).build()

    prompt = "Write a tagline for a budget-friendly eBike."
    result = await workflow.run(prompt)

    conversation = [Message(role="user", contents=[prompt])]
    for output in result.get_outputs():
        response = cast(AgentResponse, output)
        conversation.extend(response.messages)

    for msg in conversation:
        author = msg.author_name or msg.role
        print(f"[{author}] {msg.text}\n")


asyncio.run(main())
```

Other orchestration builders: **ConcurrentBuilder**, **HandoffBuilder**, **GroupChatBuilder**, **MagenticBuilder** — see `agent_framework.orchestrations`.

---

## MCP & Hosting

| Feature | Notes |
| --- | --- |
| Local MCP | OpenAI/Foundry samples under `providers/*/client_with_local_mcp.py` |
| Hosted MCP | Foundry hosted MCP + approval workflows |
| Foundry hosted agents | Deploy with ~2 extra lines — `python/samples/04-hosting/foundry-hosted-agents/` |
| Azure Functions | `python/samples/04-hosting/` |
| A2A | Agent-to-agent hosting samples — pairs with [[AI — A2A]] |
| OpenTelemetry | Built-in tracing — `python/samples/02-agents/observability/` |

Expose existing [[AI — MCP]] servers as agent tools instead of rewriting business logic.

---

## Patterns

**Provider swap:** Keep `Agent(...)` stable; change only the `client=` (`OpenAIChatCompletionClient`, `FoundryChatClient`, `OllamaChatClient`, etc.).

**Production tools:** Default `approval_mode="always_require"` for side effects; log approvals in your UI.

**Hybrid with LangChain:** Use MAF for production orchestration; keep [[AI — LangChain]] RAG chains as tools or pre-processing steps.

**Vault pipeline:**

```text
[[AI — Docling]] → chunk → embed → agent with file_search / RAG tool
SequentialBuilder: researcher agent → writer agent → [[AI — RAGAS]] eval
```

**Migration:** Official guides for Semantic Kernel and AutoGen → [migration docs](https://learn.microsoft.com/en-us/agent-framework/migration-guide/overview).

---

## Quick Reference

| Task | Code |
| --- | --- |
| Install (full) | `pip install agent-framework` |
| Install (core) | `pip install agent-framework-core` |
| Agent | `Agent(client=..., name=..., instructions=...)` |
| Run | `await agent.run("prompt")` |
| Stream | `async for chunk in agent.run(..., stream=True):` |
| Tool | `@tool` + `tools=[fn]` on `Agent` |
| Session | `session = agent.create_session()` |
| Graph | `WorkflowBuilder(...).add_edge(a, b).build()` |
| Sequential | `SequentialBuilder(participants=[...]).build()` |
| Structured | `options={"response_format": MyModel}` |

---

## Related Notes

- [[AI]]
- [[AI — LangGraph]]
- [[AI — LangChain]]
- [[AI — ADK]]
- [[AI — Agno]]
- [[AI — CrewAI]]
- [[AI — MCP]]
- [[AI — A2A]]
- [[Python — Pydantic]]
- [[Python — asyncio]]
- [[GCP]] — compare with Azure/Foundry deploy paths

---

## Tags

#ai #agent-framework #microsoft #agents #workflows #azure #openai #orchestration #python #mcp #multi-agent
