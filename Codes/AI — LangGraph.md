## What & When

**LangGraph** extends [[AI — LangChain]] with **stateful graphs** — nodes, edges, checkpoints, and **human-in-the-loop** interrupts. Use it when a single LCEL chain is not enough: multi-step agents, branching, retries, or durable conversation state.

Use LangGraph when:

- Building **multi-node agents** (plan → act → reflect)
- Persisting **thread state** across API requests (checkpointers)
- Pausing for **human approval** before tool execution
- Modeling **cycles** (agent ↔ tools) explicitly
- Running **parallel** fan-out / fan-in over state fields

```bash
pip install langgraph langchain-core
pip install langchain-openai    # typical model provider
```

LangGraph builds on `langchain-core` messages and runnables; pair with [[AI — LangChain]] for prompts, tools, and retrievers.

---

## LangGraph vs LangChain

| Concern | LangChain (LCEL) | LangGraph |
| --- | --- | --- |
| Mental model | Pipeline of runnables | Graph of nodes + shared state |
| State across steps | Manual / chat history | Typed `State` + reducers |
| Cycles | Uncommon | First-class |
| Persistence | External DB | Built-in checkpointers |
| Human gates | Custom code | `interrupt_before` / `interrupt_after` |
| Best for | RAG, simple tools | Agents, workflows, HITL |

---

## StateGraph Basics

```python
from typing import Annotated, TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langchain_core.messages import HumanMessage, AIMessage

class AgentState(TypedDict):
    messages: Annotated[list, add_messages]   # reducer appends messages

def echo_node(state: AgentState) -> dict:
    last = state["messages"][-1]
    return {"messages": [AIMessage(content=f"Echo: {last.content}")]}

builder = StateGraph(AgentState)
builder.add_node("echo", echo_node)
builder.add_edge(START, "echo")
builder.add_edge("echo", END)

graph = builder.compile()

result = graph.invoke({"messages": [HumanMessage(content="Hello")]})
print(result["messages"][-1].content)
```

```python
# Conditional routing
def router(state: AgentState) -> str:
    text = state["messages"][-1].content.lower()
    return "tools" if "search" in text else "answer"

builder.add_conditional_edges("agent", router, {"tools": "tools", "answer": "answer"})
```

---

## Tool-Calling Agent Graph

```python
import os
from typing import Annotated, TypedDict
from langchain_openai import ChatOpenAI
from langchain_core.tools import tool
from langchain_core.messages import HumanMessage
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.prebuilt import ToolNode, tools_condition

@tool
def get_ticket_status(ticket_id: str) -> str:
    """Look up support ticket status."""
    return f"Ticket {ticket_id}: in progress"

tools = [get_ticket_status]
tool_node = ToolNode(tools)
llm = ChatOpenAI(model="gpt-4o-mini").bind_tools(tools)

class State(TypedDict):
    messages: Annotated[list, add_messages]

def agent(state: State) -> dict:
    response = llm.invoke(state["messages"])
    return {"messages": [response]}

builder = StateGraph(State)
builder.add_node("agent", agent)
builder.add_node("tools", tool_node)
builder.add_edge(START, "agent")
builder.add_conditional_edges("agent", tools_condition)
builder.add_edge("tools", "agent")

app = builder.compile()

out = app.invoke({"messages": [HumanMessage("Status of ticket T-42")]})
print(out["messages"][-1].content)
```

> [!tip] Prefer `langgraph.prebuilt.ToolNode` + `tools_condition` Standard pattern for OpenAI-style tool loops — less boilerplate than hand-rolled edges.

---

## Checkpointers — Durable Threads

```python
from langgraph.checkpoint.memory import MemorySaver

memory = MemorySaver()
app = builder.compile(checkpointer=memory)

config = {"configurable": {"thread_id": "user-123"}}

app.invoke({"messages": [HumanMessage("Hi")]}, config)
app.invoke({"messages": [HumanMessage("What did I just say?")]}, config)

# List checkpoint history
states = list(app.get_state_history(config))
```

```python
# Production: SQLite or Postgres checkpointer
# pip install langgraph-checkpoint-sqlite
from langgraph.checkpoint.sqlite import SqliteSaver

with SqliteSaver.from_conn_string("checkpoints.db") as checkpointer:
    app = builder.compile(checkpointer=checkpointer)
    app.invoke({"messages": [HumanMessage("Hello")]}, config)
```

---

## Human-in-the-Loop

Pause before dangerous nodes (e.g. `tools` or `send_email`); resume after approval.

```python
app = builder.compile(
    checkpointer=memory,
    interrupt_before=["tools"],   # stop before tool execution
)

config = {"configurable": {"thread_id": "approval-demo"}}
app.invoke({"messages": [HumanMessage("Refund order 99")]}, config)

snapshot = app.get_state(config)
# Human reviews pending tool calls in snapshot.next ...

# Resume after approval
app.invoke(None, config)   # continues from interrupt
```

```python
# Reject / edit tool args before resume
from langgraph.types import Command

app.invoke(
    Command(resume={"action": "approve"}),
    config,
)
```

---

## Subgraphs & Multi-Agent

```python
# Compile a subgraph and mount as a node
research_graph = research_builder.compile()
writer_graph = writer_builder.compile()

main = StateGraph(WorkflowState)
main.add_node("research", research_graph)
main.add_node("write", writer_graph)
main.add_edge(START, "research")
main.add_edge("research", "write")
main.add_edge("write", END)
pipeline = main.compile()
```

Use separate `thread_id` per subgraph only when you need isolated checkpoint namespaces.

---

## Streaming & Observability

```python
for event in app.stream(
    {"messages": [HumanMessage("Summarize our RAG stack")]},
    config,
    stream_mode="updates",
):
    for node_name, update in event.items():
        print(node_name, update)
```

```python
# stream_mode options: "values" | "updates" | "messages" | "debug"
```

Integrate with LangSmith by setting `LANGCHAIN_TRACING_V2=true` — same as LangChain runs.

---

## Common Backend / RAG Patterns

### FastAPI chat with thread_id

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class ChatRequest(BaseModel):
    thread_id: str
    message: str

@app.post("/chat")
def chat(req: ChatRequest):
    config = {"configurable": {"thread_id": req.thread_id}}
    result = app_graph.invoke(
        {"messages": [HumanMessage(req.message)]},
        config,
    )
    return {"reply": result["messages"][-1].content}
```

### RAG node + agent node in one graph

```python
def retrieve(state: State) -> dict:
    query = state["messages"][-1].content
    docs = retriever.invoke(query)
    context = "\n".join(d.page_content for d in docs)
    return {"messages": [HumanMessage(content=f"Context:\n{context}")]}

builder.add_node("retrieve", retrieve)
builder.add_edge(START, "retrieve")
builder.add_edge("retrieve", "agent")
```

Retriever from [[AI — LangChain]]; documents from [[AI — Docling]] / [[AI — MegaParse]].

### Time-travel debugging

```python
history = list(app.get_state_history(config))
parent = history[1].config   # earlier checkpoint
app.invoke({"messages": [HumanMessage("Try again")]}, parent)
```

---

## Quick Reference

| Task | Code |
| --- | --- |
| Define state | `class State(TypedDict): messages: Annotated[list, add_messages]` |
| Build graph | `StateGraph(State)` → `add_node` / `add_edge` |
| Compile | `graph = builder.compile(checkpointer=...)` |
| Invoke | `graph.invoke({"messages": [...]}, config)` |
| Thread config | `{"configurable": {"thread_id": "..."}}` |
| Tool loop | `ToolNode(tools)` + `tools_condition` |
| Memory checkpointer | `MemorySaver()` |
| HITL pause | `interrupt_before=["tools"]` |
| Resume | `graph.invoke(None, config)` |
| Stream | `graph.stream(input, config, stream_mode="updates")` |

---

## Tags

#python #langgraph #agents #state-machine #hitl #checkpoints #orchestration #backend #langchain
