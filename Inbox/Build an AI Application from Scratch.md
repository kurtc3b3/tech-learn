> **Expanded roadmap:** See [[AI]] and [[Python Development]] (scraping + API delivery). This note is a short checklist.

- **Project foundation:** Typed settings, logging, tests [[Python — Pydantic]] [[Python — logging]] [[Unit Testing - pytest]]
- **Ingest content:** HTTP fetch, HTML cleanup, PDF parse [[Python — httpx Package]] [[Python — markdownify]] [[AI — Docling]] [[AI — MegaParse]]
- **Orchestration:** Chains and graphs for multi-step flows [[AI — LangChain]] [[AI — LangGraph]]
- **RAG stack:** Chunk, embed, retrieve, generate [[AI — LlamaIndex]] [[AI — Haystack]] vector stores in [[AI]] hub
- **Agents & crews:** Role-based and tool-using agents [[AI — CrewAI]] [[AI — Agno]] [[AI — Pydantic AI]]
- **Google-native agents:** Code-first agents and workflows on [[GCP]] [[AI — ADK]]
- **Protocols:** Tools (MCP), agent-to-agent (A2A), editor agents (ACP) [[AI — MCP]] [[AI — A2A]] [[AI — ACP]]
- **Memory & evaluation:** Long-term context and quality metrics [[AI — Mem0]] [[AI — RAGAS]]
- **Expose via API:** Production HTTP surface [[API - FastAPI]] [[API - FastAPI — Rate Limiting (SlowAPI)]]
- **Background jobs:** Long-running ingestion or indexing [[Processing — Celery]] [[DB — Redis]]
- **Optional ML bridge:** Classical models alongside LLM features [[Machine Learning]] [[ML — MLflow]]
