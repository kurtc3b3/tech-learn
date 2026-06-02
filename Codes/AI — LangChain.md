## What & When

**LangChain** is a Python framework for composing **LLM applications** — models, prompts, tools, retrievers, and memory — as **runnable pipelines** (LCEL). It is the default orchestration layer in this vault before you graduate to [[AI — LangGraph]] for stateful agents.

Use LangChain when:

- Building **RAG** chains (embed → store → retrieve → generate)
- Wiring **tool-calling** agents with OpenAI / Anthropic / local models
- Composing steps with the **`|` operator** (LangChain Expression Language)
- Integrating vector stores like [[AI — Chroma]] or community loaders
- Validating tool args with [[Python — Pydantic]] models

```bash
pip install langchain langchain-openai langchain-community langchain-core
pip install langchain-chroma          # optional — Chroma vector store
```

For stateful multi-step workflows, human approval, and durable checkpoints, prefer [[AI — LangGraph]]. For document parsing before indexing, see [[AI — Docling]] and [[AI — MegaParse]].

---

## LangChain vs Related Tools

| Need | Use | Notes |
| --- | --- | --- |
| Linear chains & RAG | LangChain + LCEL | `prompt \| llm \| parser` |
| Stateful agents, HITL | [[AI — LangGraph]] | Built on `langchain-core` |
| Pipeline-first RAG framework | [[AI — Haystack]] | Components + `Pipeline` |
| Index-centric apps | [[AI — LlamaIndex]] | Data connectors, query engines |
| Structured tool I/O | [[Python — Pydantic]] | `BaseModel` for tool schemas |

---

## Models — ChatOpenAI

```python
import os
from langchain_openai import ChatOpenAI, OpenAIEmbeddings

llm = ChatOpenAI(
    model="gpt-4o-mini",
    temperature=0,
    api_key=os.environ["OPENAI_API_KEY"],
)

embeddings = OpenAIEmbeddings(model="text-embedding-3-small")

response = llm.invoke("Summarize RAG in one sentence.")
print(response.content)
```

```python
# Streaming
for chunk in llm.stream("Explain vector databases briefly."):
    print(chunk.content, end="", flush=True)
```

```python
# Async (FastAPI handlers)
async def ask(question: str) -> str:
    msg = await llm.ainvoke(question)
    return msg.content
```

---

## Prompts — ChatPromptTemplate

```python
from langchain_core.prompts import ChatPromptTemplate

prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a backend assistant. Answer using only the provided context."),
    ("human", "Context:\n{context}\n\nQuestion: {question}"),
])

chain = prompt | llm
answer = chain.invoke({
    "context": "LangChain uses LCEL to compose runnables.",
    "question": "What is LCEL?",
})
print(answer.content)
```

```python
# Few-shot messages
from langchain_core.prompts import FewShotChatMessagePromptTemplate

examples = [
    {"input": "2+2", "output": "4"},
    {"input": "3*3", "output": "9"},
]
example_prompt = ChatPromptTemplate.from_messages([
    ("human", "{input}"),
    ("ai", "{output}"),
])
few_shot = FewShotChatMessagePromptTemplate(
    example_prompt=example_prompt,
    examples=examples,
)
```

---

## LCEL — Runnable Composition

LCEL chains are **Runnable** objects: invoke, batch, stream, and async share the same interface.

```python
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnablePassthrough

qa_prompt = ChatPromptTemplate.from_template(
    "Answer concisely.\n\nContext:\n{context}\n\nQuestion: {question}"
)

simple_chain = qa_prompt | llm | StrOutputParser()

result = simple_chain.invoke({
    "context": "Chroma stores embeddings locally.",
    "question": "What is Chroma?",
})
```

```python
# Parallel branches with RunnableParallel
from langchain_core.runnables import RunnableParallel

parallel = RunnableParallel(
    summary=qa_prompt | llm | StrOutputParser(),
    question=RunnablePassthrough(),
)
```

> [!tip] Prefer `StrOutputParser()` at the end of user-facing chains Returns plain strings instead of `AIMessage` objects — easier for APIs and tests.

---

## Tools — Structured Tool Calling

```python
from langchain_core.tools import tool
from pydantic import BaseModel, Field

@tool
def search_docs(query: str, limit: int = 5) -> str:
    """Search internal documentation."""
    return f"Top {limit} hits for: {query}"

class WeatherInput(BaseModel):
    city: str = Field(description="City name")

@tool(args_schema=WeatherInput)
def get_weather(city: str) -> str:
    """Return mock weather for a city."""
    return f"Weather in {city}: 18°C, cloudy"

tools = [search_docs, get_weather]
llm_with_tools = llm.bind_tools(tools)

msg = llm_with_tools.invoke("What's the weather in Berlin?")
print(msg.tool_calls)   # model decides which tool to call
```

```python
# Agent loop (tool-calling) — simple pattern; production agents often use LangGraph
from langchain.agents import AgentExecutor, create_tool_calling_agent
from langchain_core.prompts import ChatPromptTemplate

agent_prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a helpful assistant with tools."),
    ("placeholder", "{chat_history}"),
    ("human", "{input}"),
    ("placeholder", "{agent_scratchpad}"),
])

agent = create_tool_calling_agent(llm, tools, agent_prompt)
executor = AgentExecutor(agent=agent, tools=tools, verbose=True)

executor.invoke({"input": "Search docs for pytest fixtures"})
```

---

## Retrievers & Vector Store

```python
from langchain_chroma import Chroma
from langchain_core.documents import Document

docs = [
    Document(page_content="LangChain composes LLM apps with LCEL.", metadata={"source": "notes"}),
    Document(page_content="Chroma is a local vector database.", metadata={"source": "notes"}),
]

vectorstore = Chroma.from_documents(
    documents=docs,
    embedding=embeddings,
    collection_name="backend_kb",
    persist_directory="./chroma_data",
)

retriever = vectorstore.as_retriever(search_kwargs={"k": 3})
hits = retriever.invoke("What is LCEL?")
for doc in hits:
    print(doc.page_content)
```

---

## Simple RAG Chain

```python
from langchain_core.runnables import RunnablePassthrough
from langchain_core.output_parsers import StrOutputParser

def format_docs(documents):
    return "\n\n---\n\n".join(d.page_content for d in documents)

rag_prompt = ChatPromptTemplate.from_template(
    """Use only the context below. If unknown, say you don't know.

Context:
{context}

Question: {question}
"""
)

rag_chain = (
    {
        "context": retriever | format_docs,
        "question": RunnablePassthrough(),
    }
    | rag_prompt
    | llm
    | StrOutputParser()
)

answer = rag_chain.invoke("How does LangChain compose steps?")
print(answer)
```

```python
# Conversational RAG with message history (trim before invoke)
from langchain_core.messages import HumanMessage, AIMessage

history = [
    HumanMessage("What is Chroma?"),
    AIMessage("Chroma is an embedded vector database."),
]
follow_up = rag_chain.invoke({
    "question": "Can I use it locally?",
    "chat_history": history,   # extend prompt template if needed
})
```

---

## Common Backend / RAG Patterns

### FastAPI endpoint wrapping a chain

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class AskRequest(BaseModel):
    question: str

@app.post("/ask")
async def ask(req: AskRequest):
    answer = await rag_chain.ainvoke(req.question)
    return {"answer": answer}
```

### Ingest pipeline: parse → chunk → index

```python
from langchain_text_splitters import RecursiveCharacterTextSplitter

def ingest_markdown(markdown: str, source: str) -> int:
    splitter = RecursiveCharacterTextSplitter(chunk_size=800, chunk_overlap=100)
    chunks = splitter.create_documents([markdown], metadatas=[{"source": source}])
    vectorstore.add_documents(chunks)
    return len(chunks)
```

Wire parsers from [[AI — Docling]] or [[AI — MegaParse]] before `ingest_markdown`.

### Runnable config (tags, callbacks, run name)

```python
rag_chain.invoke(
    "What is RAG?",
    config={"run_name": "support_bot", "tags": ["prod", "v1"]},
)
```

---

## LangChain vs LangGraph (when to switch)

| Scenario | LangChain | LangGraph |
| --- | --- | --- |
| Single-shot Q&A / RAG | ✅ LCEL chain | Overkill |
| Multi-step agent with memory | Possible | ✅ `StateGraph` |
| Human approval gate | Awkward | ✅ `interrupt_before` |
| Durable checkpoints | Limited | ✅ checkpointers |
| Cyclic tool loops | `AgentExecutor` | ✅ explicit graph edges |

---

## Quick Reference

| Task | Code |
| --- | --- |
| Chat model | `ChatOpenAI(model="gpt-4o-mini")` |
| Prompt template | `ChatPromptTemplate.from_messages([...])` |
| LCEL chain | `prompt \| llm \| StrOutputParser()` |
| Bind tools | `llm.bind_tools(tools)` |
| Tool decorator | `@tool` / `@tool(args_schema=Model)` |
| Vector store | `Chroma.from_documents(docs, embedding=...)` |
| Retriever | `vectorstore.as_retriever(search_kwargs={"k": 3})` |
| RAG chain | `{context: retriever \| format_docs, question: RunnablePassthrough()} \| prompt \| llm` |
| Async invoke | `await chain.ainvoke(input)` |
| Chunk documents | `RecursiveCharacterTextSplitter(chunk_size=800)` |

---

## Tags

#python #langchain #lcel #rag #llm #agents #tools #orchestration #backend #openai
