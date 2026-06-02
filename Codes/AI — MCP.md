## What & When

The **Model Context Protocol (MCP)** is an open standard for connecting LLM applications to **tools**, **resources** (read-only context), and **prompts** over transports like **stdio**, **SSE**, and **Streamable HTTP**. The official **`mcp`** Python SDK includes **FastMCP** — decorators that turn plain functions into MCP primitives with JSON Schema generated from type hints and docstrings.

Use MCP when:

- Multiple hosts (Cursor, Claude Desktop, custom agents) should share the same tool surface
- You want tools **outside** the app process (separate repo, separate lifecycle)
- You're wrapping databases, internal APIs, or [[API - FastAPI]] services for LLM use
- Frameworks ([[AI — Agno]], [[AI — Pydantic AI]]) consume MCP via client toolsets

```python
from mcp.server.fastmcp import FastMCP
```

```bash
pip install mcp

# Dev CLI (inspector, dev server helpers)
pip install "mcp[cli]"
```

Spec and docs: [modelcontextprotocol.io](https://modelcontextprotocol.io). See [[AI]] for MCP vs A2A vs ACP; host agents in [[AI — Pydantic AI]] or [[AI — Agno]].

---

## MCP vs Related Approaches

| Need | Use | Notes |
| --- | --- | --- |
| Standard tool/data wire-up | **MCP** | Tools + resources + prompts |
| Agent-to-agent over HTTP | [[AI — A2A]] | Remote agents, not tool catalogs |
| Editor ↔ coding agent | [[AI — ACP]] | IDE integration |
| In-process tools only | Framework tools | Simpler, no separate server |
| REST API for humans | [[API - FastAPI]] | Often wrapped *behind* MCP tools |

---

## Core Primitives

| Primitive | Analogy | Purpose |
| --- | --- | --- |
| **Tool** | POST / action | Run code, side effects, fetch dynamic data |
| **Resource** | GET / read | Load text into context (files, configs, templates) |
| **Prompt** | Template | Reusable prompt patterns for the host |

Hosts discover capabilities at connect time; servers implement handlers.

---

## FastMCP — Minimal Server

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("demo-server")

@mcp.tool()
def add(a: int, b: int) -> int:
    """Add two integers."""
    return a + b

@mcp.resource("greeting://{name}")
def get_greeting(name: str) -> str:
    """Personalized greeting resource."""
    return f"Hello, {name}!"

@mcp.prompt()
def review_code(language: str = "python") -> str:
    """Prompt template for code review."""
    return f"Review this {language} code for bugs and style issues."

if __name__ == "__main__":
    mcp.run()   # stdio — default for Claude Desktop / Cursor
```

**Type hints** → input JSON Schema. **Docstrings** (and `Args:` sections) → descriptions the model sees.

> [!warning] Never print to stdout on stdio servers stdout is the JSON-RPC channel. Use `logging` to **stderr** only.

---

## Tools — Actions & Side Effects

```python
from mcp.server.fastmcp import FastMCP, Context
import httpx

mcp = FastMCP("github-tools")

@mcp.tool()
async def get_repo_stars(owner: str, repo: str, ctx: Context) -> str:
    """Return star count for a GitHub repository.

    Args:
        owner: GitHub user or org (e.g. 'pydantic')
        repo: Repository name (e.g. 'pydantic')
    """
    await ctx.info(f"Fetching {owner}/{repo}")
    async with httpx.AsyncClient() as client:
        r = await client.get(f"https://api.github.com/repos/{owner}/{repo}")
        r.raise_for_status()
        data = r.json()
    return f"{data['full_name']}: {data['stargazers_count']} stars"
```

`Context` provides logging (`ctx.info`, `ctx.error`), progress reporting, and access to server/session state.

---

## Resources — Read-Only Context

```python
@mcp.resource("config://app/{key}")
def read_config(key: str) -> str:
    """Expose configuration values to the host."""
    settings = {"env": "production", "region": "eu-west-1"}
    return settings.get(key, "")

@mcp.resource("file://docs/{name}")
def read_doc(name: str) -> str:
    path = f"./docs/{name}.md"
    with open(path, encoding="utf-8") as f:
        return f.read()
```

Resources should be cheap reads — heavy computation belongs in **tools**.

---

## Lifespan — Startup / Shutdown

Initialize DB pools or HTTP clients once per server instance:

```python
from collections.abc import AsyncIterator
from contextlib import asynccontextmanager
from dataclasses import dataclass

from mcp.server.fastmcp import Context, FastMCP

@dataclass
class AppContext:
    db_url: str

@asynccontextmanager
async def lifespan(server: FastMCP) -> AsyncIterator[AppContext]:
    # startup
    yield AppContext(db_url="postgresql://localhost/app")
    # shutdown cleanup

mcp = FastMCP("db-server", lifespan=lifespan)

@mcp.tool()
def query(sql: str, ctx: Context) -> str:
    app_ctx = ctx.request_context.lifespan_context
    # use app_ctx.db_url to run query (example)
    return f"Executed on {app_ctx.db_url}"
```

---

## Running — Transports

```python
# stdio (local subprocess — Cursor, Claude Code)
mcp.run()

# Streamable HTTP (remote / browser-friendly)
mcp.run(transport="streamable-http")   # default http://127.0.0.1:8000/mcp
```

```bash
# Dev: MCP Inspector
npx -y @modelcontextprotocol/inspector

# Optional CLI
uv run mcp dev server.py
```

**Claude Desktop** config (stdio example):

```json
{
  "mcpServers": {
    "demo": {
      "command": "python",
      "args": ["/absolute/path/to/server.py"]
    }
  }
}
```

Mount into existing ASGI apps (Starlette/FastAPI) for shared deployment — see SDK docs *Mounting to an Existing ASGI Server*.

---

## Python MCP Client (Consume a Server)

```python
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

server_params = StdioServerParameters(
    command="python",
    args=["my_server.py"],
)

async def main():
    async with stdio_client(server_params) as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()
            tools = await session.list_tools()
            result = await session.call_tool("add", arguments={"a": 2, "b": 3})
            print(result)
```

Agents in [[AI — Pydantic AI]] (`MCPServerStdio`) and [[AI — Agno]] (`MCPTools`) wrap this pattern for you.

---

## Backend Patterns

```python
# Wrap internal REST API as MCP tools (secrets stay server-side)
import os
from mcp.server.fastmcp import FastMCP
import httpx

mcp = FastMCP("internal-api")
API_KEY = os.environ["INTERNAL_API_KEY"]

@mcp.tool()
async def search_customers(query: str, limit: int = 10) -> list[dict]:
    """Search CRM customers by name or email."""
    async with httpx.AsyncClient() as client:
        r = await client.get(
            "https://api.example.com/customers",
            params={"q": query, "limit": limit},
            headers={"Authorization": f"Bearer {API_KEY}"},
        )
        r.raise_for_status()
        return r.json()["items"]
```

One REST endpoint ≈ one MCP tool; keep auth in environment variables, not tool arguments.

```python
# FastAPI coexistence: mount Streamable HTTP MCP on /mcp
# (see python-sdk examples for mount_to_app / route helpers)
```

---

## Common Mistakes

| Mistake | Fix |
| --- | --- |
| `print()` on stdio server | Log to stderr only |
| Secrets in tool parameters | Read from `os.environ` server-side |
| Huge resource payloads | Paginate or use tools for heavy fetches |
| Blocking async tools | Use `async def` tools with `httpx` async client |
| Vague docstrings | Write clear `Args:` — hosts expose them to the model |

---

## Quick Reference

| Task | Code |
| --- | --- |
| Install | `pip install mcp` |
| Create server | `mcp = FastMCP("name")` |
| Tool | `@mcp.tool()` on function |
| Resource | `@mcp.resource("uri://{param}")` |
| Prompt | `@mcp.prompt()` |
| Context | `ctx: Context` parameter + `await ctx.info(...)` |
| Run stdio | `mcp.run()` |
| Run HTTP | `mcp.run(transport="streamable-http")` |
| Lifespan | `FastMCP(..., lifespan=async_context_manager)` |
| Client | `ClientSession` + `stdio_client` |

---

## Tags

#ai #mcp #model-context-protocol #tools #fastmcp #python #agents #backend
