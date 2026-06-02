## What & When

**RAGAS** (Retrieval Augmented Generation Assessment) is a Python framework for **evaluating RAG and agent pipelines** with LLM- and embedding-based metrics. It scores faithfulness to retrieved context, answer relevance to the question, context precision/recall, and more — without hand-labeling every example.

Use RAGAS when:

- Measuring whether answers stay grounded in retrieved chunks (**faithfulness**)
- Checking if responses actually address the user query (**answer relevancy**)
- Regression-testing prompt, chunking, or retriever changes on a golden dataset
- Comparing models or vector stores before production rollout

Pair with [[AI — LangChain]] / [[AI — LlamaIndex]] RAG pipelines and [[AI — Mem0]] when memory quality matters separately from retrieval.

```bash
pip install ragas
# typical evaluator dependencies
pip install openai datasets
```

Docs: [docs.ragas.io](https://docs.ragas.io/)

---

## RAGAS vs Related Tools

| Tool | Focus | When |
| --- | --- | --- |
| **RAGAS** | RAG quality metrics (faithfulness, relevancy, …) | Grounded Q&A evaluation |
| Manual eval set | Human labels | Gold standard, expensive |
| [[AI — LangChain]] callbacks | Tracing / logging | Debug runs, not aggregate scores |
| Unit tests | Deterministic code paths | Parser/chunker logic, not semantic quality |

| Metric (modern API) | Measures | Needs |
| --- | --- | --- |
| **Faithfulness** | Claims in answer supported by context | LLM |
| **AnswerRelevancy** | Answer addresses `user_input` | LLM + embeddings |
| **ContextPrecision** | Relevant chunks ranked high | LLM |
| **ContextRecall** | Retrieved context covers reference | LLM + reference answer |

---

## Modern API (v0.4+ — recommended)

Import from `ragas.metrics.collections`. Use `llm_factory` and `ascore()` with keyword arguments. Returns `MetricResult` (`.value`, `.reason`).

```python
import asyncio
from openai import AsyncOpenAI
from ragas.llms import llm_factory
from ragas.embeddings.base import embedding_factory
from ragas.metrics.collections import Faithfulness, AnswerRelevancy

client = AsyncOpenAI()


async def score_single_turn() -> None:
    llm = llm_factory("gpt-4o-mini", client=client)
    embeddings = embedding_factory(
        "openai", model="text-embedding-3-small", client=client
    )

    faith = Faithfulness(llm=llm)
    relevancy = AnswerRelevancy(llm=llm, embeddings=embeddings)

    contexts = [
        "The First AFL–NFL World Championship Game was played on January 15, 1967, "
        "at the Los Angeles Memorial Coliseum."
    ]

    f = await faith.ascore(
        user_input="When was the first super bowl?",
        response="The first superbowl was held on Jan 15, 1967",
        retrieved_contexts=contexts,
    )
    r = await relevancy.ascore(
        user_input="When was the first super bowl?",
        response="The first superbowl was held on Jan 15, 1967",
    )

    print(f"Faithfulness: {f.value:.3f}")
    print(f"Answer relevancy: {r.value:.3f}")


asyncio.run(score_single_turn())
```

Scores are typically **0.0–1.0** (higher is better).

---

## Faithfulness

Checks that the **response** does not contradict or hallucinate beyond **retrieved_contexts**.

```python
from ragas.metrics.collections import Faithfulness

scorer = Faithfulness(llm=llm)

result = await scorer.ascore(
    user_input="What is the company's refund policy?",
    response="Refunds are available within 30 days with receipt.",
    retrieved_contexts=[
        "Customers may return items within 30 days with proof of purchase.",
    ],
)
# Low score → answer adds facts not in context
```

Critical for [[AI — Chroma]], [[AI — Qdrant]], and other vector pipelines: bad retrieval + confident LLM = low faithfulness signal.

---

## Answer Relevancy

Measures alignment between **user_input** and **response** (embedding similarity of generated questions vs original query).

```python
from ragas.metrics.collections import AnswerRelevancy

scorer = AnswerRelevancy(llm=llm, embeddings=embeddings)

result = await scorer.ascore(
    user_input="When was the first super bowl?",
    response="The first superbowl was held on Jan 15, 1967",
)
```

> [!tip] NaN scores  
> If the judge LLM fails to generate probe questions (rate limits, empty answer), relevancy can return **NaN**. Check logs; shorten answers; ensure `OPENAI_API_KEY` (or your provider) is set.

---

## Batch Evaluation (dataset)

Build rows from your RAG app, then score many examples.

```python
from datasets import Dataset

rows = [
    {
        "user_input": "What is our SLA?",
        "response": "Enterprise SLA is 99.9% uptime.",
        "retrieved_contexts": ["Enterprise plans include 99.9% uptime SLA."],
        "reference": "99.9% uptime",  # optional, for recall-style metrics
    },
    # ... more examples from production logs or synthetic set
]

dataset = Dataset.from_list(rows)
```

For **legacy** batch runs (still common in tutorials):

```python
from ragas import evaluate
from ragas.metrics import faithfulness, answer_relevancy, context_precision

results = evaluate(
    dataset,
    metrics=[faithfulness, answer_relevancy, context_precision],
)
print(results)
```

Prefer **`metrics.collections`** + `ascore()` loops for new code. Legacy: `SingleTurnSample` + `single_turn_ascore()` — migrate when touching old scripts.

---

## Metric Selection Cheat Sheet

| Symptom | Metric to watch |
| --- | --- |
| Hallucinations despite retrieval | **Faithfulness** |
| On-topic docs but rambling answer | **AnswerRelevancy** |
| Right answer, wrong chunks retrieved | **ContextPrecision** / recall metrics |
| Missing info in context | **ContextRecall** (needs reference) |

---

## Quick Reference

| Task | Code |
| --- | --- |
| Install | `pip install ragas` |
| Modern faithfulness | `Faithfulness(llm).ascore(..., retrieved_contexts=[...])` |
| Modern relevancy | `AnswerRelevancy(llm, embeddings).ascore(user_input=..., response=...)` |
| LLM setup | `llm_factory("gpt-4o-mini", client=AsyncOpenAI())` |
| Embeddings | `embedding_factory("openai", model="text-embedding-3-small", client=client)` |
| Batch (legacy) | `evaluate(Dataset, metrics=[faithfulness, ...])` |
| Result object | `result.value`, `result.reason` |

---

## Related Notes

- [[AI]] — evaluate before shipping
- [[AI — LangChain]] · [[AI — LlamaIndex]] · [[AI — Haystack]]
- [[AI — Chroma]] · [[AI — Qdrant]] · [[AI — FAISS]]
- [[AI — Docling]] · [[AI — MegaParse]] — ingestion quality affects faithfulness
- [[AI — Mem0]] — separate memory layer evaluation
- [[Python — Pydantic]] — sample schemas in older examples

---

## Tags

#ai #ragas #rag #evaluation #faithfulness #metrics #llm #python #testing #obsidian
