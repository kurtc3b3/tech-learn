## What & When

**CrewAI** is a Python framework for **role-based multi-agent crews**: you define specialized `Agent`s, assign `Task`s, and run them under a `Crew` with a **sequential** or **hierarchical** process. Each agent gets role, goal, backstory, and optional tools; tasks chain context from prior outputs.

Use CrewAI when:

- Work maps cleanly to job titles (researcher → analyst → writer)
- You want declarative crews without building a full LangGraph state machine
- Handoffs are mostly linear, with optional manager-style coordination
- You need structured task outputs via [[Python — Pydantic]] (`output_pydantic`, `output_json`)

```python
from crewai import Agent, Task, Crew, Process
```

```bash
pip install crewai
# Optional search / scraping tools
pip install crewai-tools
```

Set provider keys (e.g. `OPENAI_API_KEY`). For production orchestration beyond a single crew, CrewAI also offers **Flows** (event-driven workflows); this note focuses on the core **Agent → Task → Crew** model. See [[AI]] for stack context; compare [[AI — LangGraph]] for explicit graphs and [[AI — Agno]] for teams + AgentOS hosting.

---

## CrewAI vs Related Frameworks

| Need | Use | Notes |
| --- | --- | --- |
| Role-based handoffs | **CrewAI** | `role` / `goal` / `backstory` + `Process` |
| Explicit state machine | [[AI — LangGraph]] | Nodes, edges, checkpoints |
| Typed single agent | [[AI — Pydantic AI]] | `output_type`, `RunContext` deps |
| SDK + FastAPI runtime | [[AI — Agno]] | `AgentOS`, sessions, tracing |
| Prompt/program optimization | [[AI — DSPy]] | Signatures + `BootstrapFewShot` |
| Standard tool wire-up | [[AI — MCP]] | Host connects to MCP servers |

---

## Agent — Role, Goal, Tools

An `Agent` is a specialist persona. Tools come from `crewai-tools` or LangChain-compatible tools.

```python
import os
from crewai import Agent
from crewai_tools import SerperDevTool

os.environ["OPENAI_API_KEY"] = "sk-..."
os.environ["SERPER_API_KEY"] = "..."   # serper.dev

researcher = Agent(
    role="Senior Research Analyst",
    goal="Uncover cutting-edge developments in {topic}",
    backstory=(
        "You work at a leading tech think tank. "
        "Your expertise is identifying emerging trends before they go mainstream."
    ),
    tools=[SerperDevTool()],
    verbose=True,
    allow_delegation=False,   # set True in multi-agent crews to delegate sub-tasks
)

writer = Agent(
    role="Tech Content Strategist",
    goal="Craft compelling content about {topic}",
    backstory="You turn complex research into clear, engaging narratives.",
    verbose=True,
)
```

> [!tip] Keep backstories short One paragraph per agent is enough; long backstories burn tokens without improving quality.

---

## Task — Description, Context, Output

A `Task` is work assigned to one agent. Use `context=[other_task]` so later tasks see earlier outputs. Mark the final deliverable with `output_file` or structured output types.

```python
from crewai import Task

research_task = Task(
    description=(
        "Research the latest developments in {topic}. "
        "Identify key players, trends, and risks. Current year: 2026."
    ),
    expected_output="10 bullet points of the most relevant findings",
    agent=researcher,
)

write_task = Task(
    description=(
        "Using the research, write a 4-section markdown report on {topic}. "
        "Each section should be detailed and factual."
    ),
    expected_output="Full markdown report, no code fences around the document",
    agent=writer,
    context=[research_task],          # prior task output injected as context
    output_file="report.md",
    markdown=True,
)
```

```python
from pydantic import BaseModel

class BlogPost(BaseModel):
    title: str
    content: str

structured_task = Task(
    description="Write a short blog post about {topic}.",
    expected_output="Title and body under 300 words",
    agent=writer,
    output_pydantic=BlogPost,         # validated Pydantic on completion
)
```

---

## Crew & Process — Sequential vs Hierarchical

`Crew.kickoff(inputs={...})` runs all tasks. **`Process.sequential`** runs tasks in list order; each task must have an `agent`. **`Process.hierarchical`** adds a manager that plans and delegates — requires `manager_llm` or `manager_agent`.

```python
from crewai import Crew, Process

# Sequential — predictable pipeline
crew = Crew(
    agents=[researcher, writer],
    tasks=[research_task, write_task],
    process=Process.sequential,
    verbose=True,
)

result = crew.kickoff(inputs={"topic": "AI agents"})
print(result.raw)                    # final task output as string
# print(result.pydantic)           # when using output_pydantic
```

```python
# Hierarchical — manager coordinates agents
crew = Crew(
    agents=[researcher, writer],
    tasks=[research_task, write_task],
    process=Process.hierarchical,
    manager_llm="gpt-4o",             # or manager_agent=custom_manager
    verbose=True,
)

result = crew.kickoff(inputs={"topic": "MCP adoption"})
```

| Process | Behavior | Requires |
| --- | --- | --- |
| `Process.sequential` | Tasks run in order; output flows via `context` | Each task has `agent` |
| `Process.hierarchical` | Manager assigns/reviews work | `manager_llm` or `manager_agent` |

> [!warning] Sequential is the reliable default Hierarchical mode depends heavily on the manager model. For production pipelines with branching, prefer [[AI — LangGraph]] or CrewAI Flows.

---

## YAML + CrewBase (Project Layout)

CrewAI projects often split config into `agents.yaml` / `tasks.yaml` and a `CrewBase` class:

```python
from crewai import Agent, Crew, Process, Task
from crewai.project import CrewBase, agent, crew, task
from crewai_tools import SerperDevTool

@CrewBase
class ResearchCrew:
    """Research and report crew."""

    @agent
    def researcher(self) -> Agent:
        return Agent(
            config=self.agents_config["researcher"],  # type: ignore[index]
            tools=[SerperDevTool()],
            verbose=True,
        )

    @agent
    def writer(self) -> Agent:
        return Agent(config=self.agents_config["writer"], verbose=True)

    @task
    def research_task(self) -> Task:
        return Task(config=self.tasks_config["research_task"])

    @task
    def write_task(self) -> Task:
        return Task(config=self.tasks_config["write_task"])

    @crew
    def crew(self) -> Crew:
        return Crew(
            agents=[self.researcher(), self.writer()],
            tasks=[self.research_task(), self.write_task()],
            process=Process.sequential,
        )
```

```bash
crewai run    # CLI when using crewai create template
```

---

## Backend Patterns

```python
# FastAPI endpoint wrapping a crew (sync kickoff — use thread pool in production)
from fastapi import FastAPI
from crewai import Crew

app = FastAPI()

@app.post("/reports")
def generate_report(topic: str):
    crew = build_crew()  # factory returning configured Crew
    result = crew.kickoff(inputs={"topic": topic})
    return {"report": result.raw}
```

Pair with [[API - FastAPI]] for HTTP, [[Python — Pydantic]] for request/response models, and env-based secrets (never hard-code API keys in crew modules).

```python
# Async-friendly: run kickoff in executor
import asyncio
from concurrent.futures import ThreadPoolExecutor

_executor = ThreadPoolExecutor(max_workers=4)

async def kickoff_async(crew: Crew, inputs: dict):
    loop = asyncio.get_running_loop()
    return await loop.run_in_executor(_executor, lambda: crew.kickoff(inputs=inputs))
```

---

## Common Mistakes

| Mistake | Fix |
| --- | --- |
| Hierarchical crew without manager | Add `manager_llm="gpt-4o"` or `manager_agent` |
| Sequential task missing `agent` | Assign `agent=` on every task |
| No context between tasks | `context=[previous_task]` on downstream tasks |
| Huge `description` strings | Move static instructions to agent `backstory` |
| Blocking FastAPI event loop | Run `kickoff()` in a thread pool |

---

## Quick Reference

| Task | Code |
| --- | --- |
| Install | `pip install crewai` |
| Define agent | `Agent(role=..., goal=..., backstory=..., tools=[...])` |
| Define task | `Task(description=..., expected_output=..., agent=...)` |
| Chain context | `context=[task_a]` on `task_b` |
| Sequential crew | `Crew(..., process=Process.sequential)` |
| Hierarchical crew | `Crew(..., process=Process.hierarchical, manager_llm="gpt-4o")` |
| Run | `crew.kickoff(inputs={"topic": "..."})` |
| Structured output | `output_pydantic=MyModel` on `Task` |
| Project scaffold | `@CrewBase` + YAML configs |

---

## Tags

#ai #crewai #agents #multi-agent #python #llm #patterns #backend
