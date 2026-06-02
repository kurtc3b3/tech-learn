## What & When

**DSPy** (Declarative Self-improving Python) treats LLM apps as **programs** composed of **modules** (e.g. `ChainOfThought`, `ReAct`) with **signatures** describing input/output behavior. Instead of hand-tuning prompts, you **compile** programs with **optimizers** like `BootstrapFewShot` that bootstrap demonstrations from a small trainset and a metric.

Use DSPy when:

- You have **10–500 labeled examples** and a clear **metric** (exact match, F1, LLM judge)
- Prompt quality matters more than framework ergonomics
- You want reproducible **compile** steps when data or code changes
- You're building RAG, classification, or multi-step reasoning pipelines

```python
import dspy
from dspy.teleprompt import BootstrapFewShot
```

```bash
pip install dspy
```

Configure an LM once with `dspy.configure(lm=...)`. DSPy 3.x uses `dspy.LM("provider/model")`. See [[AI]] for context; pair with [[Python — Pydantic]] when exporting structured fields from signatures.

---

## DSPy vs Related Frameworks

| Need | Use | Notes |
| --- | --- | --- |
| Optimize prompts from data | **DSPy** | `compile()` + metric |
| Production agent API | [[AI — Pydantic AI]] / [[AI — Agno]] | Runtime agents, tools |
| Role-based teams | [[AI — CrewAI]] | No automatic prompt tuning |
| RAG orchestration | [[AI — LangChain]] / [[AI — LlamaIndex]] | DSPy can wrap retriever modules |
| Tool protocol | [[AI — MCP]] | Orthogonal — DSPy optimizes LM calls |

---

## Configure Language Models

```python
import dspy

lm = dspy.LM("openai/gpt-4o-mini")
dspy.configure(lm=lm)

# Optional: stronger teacher for bootstrapping
teacher = dspy.LM("openai/gpt-4o")
```

DSPy routes all module calls through the configured LM. Use environment variables for API keys (`OPENAI_API_KEY`, etc.).

---

## Signatures — Declare I/O Behavior

A **signature** specifies what fields go in and out. Two styles: **string shorthand** or **class-based** with `dspy.InputField` / `dspy.OutputField`.

```python
import dspy

# Shorthand: "inputs -> outputs"
qa = dspy.ChainOfThought("question -> answer")

response = qa(question="What is MCP?")
print(response.answer)
```

```python
class EmotionSignature(dspy.Signature):
    """Classify customer message sentiment."""

    text: str = dspy.InputField(desc="Customer message")
    sentiment: str = dspy.OutputField(desc="One of: positive, neutral, negative")
    confidence: float = dspy.OutputField(desc="0.0 to 1.0")

classify = dspy.Predict(EmotionSignature)

out = classify(text="Your product is amazing!")
print(out.sentiment, out.confidence)
```

| Module | Role |
| --- | --- |
| `dspy.Predict` | Single LM call for signature |
| `dspy.ChainOfThought` | Adds reasoning before answer |
| `dspy.ReAct` | Tool-using agent loop inside module |

---

## Modules — Composable Programs

Subclass `dspy.Module` and implement `forward` to chain steps:

```python
import dspy

class RAG(dspy.Module):
  def __init__(self, num_passages: int = 3):
    super().__init__()
    self.retrieve = dspy.Retrieve(k=num_passages)
    self.generate = dspy.ChainOfThought("context, question -> answer")

  def forward(self, question: str):
    context = self.retrieve(question).passages
    return self.generate(context=context, question=question)

rag = RAG()
pred = rag(question="How does BootstrapFewShot work?")
print(pred.answer)
```

`dspy.Retrieve` expects a configured retriever (e.g. ColBERTv2, custom embedding index) — wire your vector store in setup, then optimize the generator with DSPy.

---

## Examples & Metrics

Training data uses `dspy.Example` with `.with_inputs()` marking input fields:

```python
trainset = [
    dspy.Example(question="2+2?", answer="4").with_inputs("question"),
    dspy.Example(question="Capital of France?", answer="Paris").with_inputs("question"),
]

def exact_match(example, pred, trace=None) -> bool:
    return example.answer.strip().lower() == pred.answer.strip().lower()

# Numeric metric (0–1) also supported
def f1_metric(example, pred, trace=None) -> float:
    ...
```

The **metric** drives optimizers: return `True`/`False` or a float score; optional `trace` inspects intermediate steps during bootstrapping.

---

## BootstrapFewShot — Compile with Demos

`BootstrapFewShot` runs a **teacher** on the trainset, keeps traces that pass the metric, and injects them as **demos** into each predictor.

```python
from dspy.teleprompt import BootstrapFewShot

teleprompter = BootstrapFewShot(
    metric=exact_match,
    max_bootstrapped_demos=4,    # teacher-generated examples per predictor
    max_labeled_demos=8,         # gold examples copied into prompts
    max_rounds=1,
)

compiled_rag = teleprompter.compile(RAG(), trainset=trainset)

# Use compiled program like the original
pred = compiled_rag(question="What is DSPy?")
print(pred.answer)
```

```python
# Stronger search — more compute, better quality on larger trainsets
from dspy.teleprompt import BootstrapFewShotWithRandomSearch

optimizer = BootstrapFewShotWithRandomSearch(
    metric=exact_match,
    num_candidate_programs=8,
)
compiled = optimizer.compile(RAG(), trainset=trainset, valset=devset)
```

| Optimizer | When |
| --- | --- |
| `BootstrapFewShot` | Small trainset, fast iteration |
| `BootstrapFewShotWithRandomSearch` | Search over demo sets |
| `MIPROv2` / others | See [dspy.ai](https://dspy.ai) for latest teleprompters |

Pass `teacher=other_lm` to `compile()` when the student model should bootstrap from a stronger model.

---

## Evaluate Before / After Compile

```python
from dspy.evaluate import Evaluate

evaluate = Evaluate(devset=devset, metric=exact_match, num_threads=4)
baseline = evaluate(RAG())
optimized = evaluate(compiled_rag)
print(baseline, optimized)
```

Log scores across compile runs to detect regressions when you change signatures or data.

---

## Backend / Production Notes

DSPy **compilation** is an offline step (CI job or admin script). Save **state-only JSON** (demos + optimized prompts), then recreate the class and `.load()` at serve time:

```python
# After compile in training job:
compiled_rag.save("compiled_rag.json")

# Serving — reinstantiate same Module class, then load state:
rag = RAG()
rag.load("compiled_rag.json")
dspy.configure(lm=dspy.LM("openai/gpt-4o-mini"))   # LM config is not stored in the file
answer = rag(question=user_query).answer
```

Expose answers via [[API - FastAPI]]; keep `dspy.configure(lm=...)` in app startup. Compilation is not free — cache compiled artifacts per program version.

---

## Common Mistakes

| Mistake | Fix |
| --- | --- |
| No `.with_inputs()` on examples | Mark input fields for the optimizer |
| Metric ignores `pred` shape | Match signature output field names |
| Compiling without `dspy.configure` | Set `lm=` before `compile()` |
| Expecting compile to fix bad data | Clean trainset first; metric must correlate with quality |
| Huge trainset + `BootstrapFewShot` only | Use `BootstrapFewShotWithRandomSearch` or subset |

---

## Quick Reference

| Task | Code |
| --- | --- |
| Install | `pip install dspy` |
| Configure LM | `dspy.configure(lm=dspy.LM("openai/gpt-4o-mini"))` |
| Shorthand signature | `dspy.ChainOfThought("q -> a")` |
| Class signature | Subclass `dspy.Signature` with `InputField` / `OutputField` |
| Custom program | `class MyProg(dspy.Module): def forward(...)` |
| Trainset | `dspy.Example(...).with_inputs("field")` |
| Optimize | `BootstrapFewShot(metric=fn).compile(program, trainset=...)` |
| Evaluate | `Evaluate(devset=..., metric=fn)(program)` |
| Save / load | `program.save(path)` / `program.load(path)` |

---

## Tags

#ai #dspy #prompt-optimization #rag #signatures #python #llm #patterns
