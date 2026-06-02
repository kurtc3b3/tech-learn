## What & When

**spaCy** is a production-oriented NLP library — fast tokenization, POS, dependency parsing, NER, and trainable **pipelines** in one `nlp()` call. Default in this vault for backend text enrichment, custom entity rules, and structured features for [[ML — scikit-learn]] or hybrid [[AI]] / RAG metadata.

Use spaCy when:

- **Low-latency** tokenization, lemmas, and entities on API hot paths
- **Custom NER** or rule extractors with `Matcher` / `PhraseMatcher`
- Exporting **Doc** objects for downstream ML
- Visualizing parses with **displaCy**
- **Updating** a pretrained pipeline with domain labels

```bash
pip install spacy
python -m spacy download en_core_web_sm
# Larger: en_core_web_md (vectors), en_core_web_lg
```

Overview: [[NLP]]. Serve with [[API - FastAPI]].

---

## spaCy vs Related Tools

| Need | Use | Notes |
| --- | --- | --- |
| Production NLP pipeline | **spaCy** | Single `nlp(text)` → Doc |
| Corpora, stemmers, teaching | [[NLP — NLTK]] | Slower; great for learning |
| Quick sentiment demo | [[NLP — TextBlob]] | Built on NLTK |
| Tabular text features | spaCy → [[ML — scikit-learn]] | Lemmas / counts → TF-IDF |
| LLM extraction | [[AI]] | Costlier; spaCy for cheap structure |
| Regex cleanup | [[Python — re (Regular Expressions)]] | Pre-normalization |

---

## Pipeline & Doc API

```python
import spacy

nlp = spacy.load("en_core_web_sm")
doc = nlp("Apple is looking at buying U.K. startup for $1 billion.")

for token in doc:
    print(token.text, token.lemma_, token.pos_, token.dep_, token.is_stop)

for ent in doc.ents:
    print(ent.text, ent.label_, ent.start_char, ent.end_char)
```

Key attrs: `lemma_`, `pos_`, `dep_`, `is_stop`; entities in `doc.ents`.

Disable unused pipes for speed:

```python
nlp = spacy.load("en_core_web_sm", disable=["parser", "ner"])
with nlp.select_pipes(enable=["tok2vec", "tagger"]):
    doc = nlp("Fast tokenize and tag only.")
```

---

## Tokenization & Sentences

```python
doc = nlp("Dr. Smith arrived. He met Jane Doe at 3 p.m.")
tokens = [t.text for t in doc]
sents = [sent.text for sent in doc.sents]   # needs parser or sentencizer

span = doc[0:2]
print(span.start_char, span.end_char, span.text)
```

Without parser: `nlp.add_pipe("sentencizer")`. Split `doc.sents` for RAG chunking before [[AI]] embedding.

---

## NER

```python
doc = nlp("Elon Musk founded SpaceX in California in 2002.")
[(ent.text, ent.label_) for ent in doc.ents]
```

Custom span (runtime only — does not retrain):

```python
from spacy.tokens import Span

doc = nlp("Acme Corp released Product X.")
doc.ents = list(doc.ents) + [Span(doc, 0, 2, label="ORG")]
```

Train domain NER — see **Train / Update Pipeline**.

---

## Matcher — Token Patterns

```python
from spacy.matcher import Matcher

matcher = Matcher(nlp.vocab)
pattern = [
    {"LOWER": "email"},
    {"IS_PUNCT": True, "OP": "?"},
    {"TEXT": {"REGEX": r"[\w\.-]+@[\w\.-]+"}},
]
matcher.add("EMAIL_PATTERN", [pattern])

doc = nlp("Contact email: alice@example.com today.")
for match_id, start, end in matcher(doc):
    print(nlp.vocab.strings[match_id], doc[start:end].text)
```

## PhraseMatcher — Lexicon Lookup

Faster than Matcher for long phrase lists.

```python
from spacy.matcher import PhraseMatcher

skills = ["python", "machine learning", "fastapi"]
phrase_matcher = PhraseMatcher(nlp.vocab, attr="LOWER")
phrase_matcher.add("SKILLS", [nlp.make_doc(s) for s in skills])

doc = nlp("She knows Python and machine learning.")
for _, start, end in phrase_matcher(doc):
    print(doc[start:end].text)
```

## Batch — `nlp.pipe`

```python
texts = ["First doc.", "Second doc.", "Third doc."]
for doc in nlp.pipe(texts, batch_size=64, n_process=2):
    lemmas = [t.lemma_ for t in doc if not t.is_stop and t.is_alpha]
```

Prefer `nlp.pipe` over repeated `nlp(text)` in loops.

---

## Train / Update Pipeline

**1. Build training file** with `DocBin`:

```python
import spacy
from spacy.tokens import DocBin

nlp = spacy.blank("en")
data = [
    ("Acme Pay launched in Berlin.", {"entities": [(0, 8, "PRODUCT"), (20, 26, "GPE")]}),
]
db = DocBin()
for text, annot in data:
    doc = nlp.make_doc(text)
    doc.ents = [s for s in (doc.char_span(a, b, label=l) for a, b, l in annot["entities"]) if s]
    db.add(doc)
db.to_disk("train.spacy")
```

**2. Train** (spaCy 3.x):

```bash
python -m spacy init fill-config base_config.cfg config.cfg
python -m spacy train config.cfg --output ./models \
  --paths.train ./train.spacy --paths.dev ./dev.spacy --training.max_steps 500
```

**3. Load:** `nlp = spacy.load("./models/model-best")`

---

## displacy

```python
from spacy import displacy

doc = nlp("Google acquired DeepMind in London.")
displacy.serve(doc, style="ent", port=5000)
html = displacy.render(doc, style="ent", page=True)
```

Styles: `ent`, `dep`, `span`.

---

## sklearn Features

```python
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.pipeline import Pipeline
from sklearn.linear_model import LogisticRegression

def spacy_lemmas(texts):
    for doc in nlp.pipe(texts, batch_size=128):
        yield " ".join(t.lemma_ for t in doc if t.is_alpha and not t.is_stop)

X = list(spacy_lemmas(["great product", "terrible service"]))
clf = Pipeline([("tfidf", TfidfVectorizer()), ("lr", LogisticRegression(max_iter=1000))])
# clf.fit(X, y)
```

---

## FastAPI Enrichment Endpoint

Load `nlp` once at startup ([[API - FastAPI]] lifespan).

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI
from pydantic import BaseModel
import spacy

nlp = None

@asynccontextmanager
async def lifespan(app: FastAPI):
    global nlp
    nlp = spacy.load("en_core_web_sm")
    yield

app = FastAPI(lifespan=lifespan)

class TextIn(BaseModel):
    text: str

@app.post("/v1/analyze")
def analyze(body: TextIn):
    doc = nlp(body.text)
    return {
        "tokens": [t.lemma_ for t in doc if t.is_alpha and not t.is_stop],
        "entities": [{"text": e.text, "label": e.label_} for e in doc.ents],
        "sentences": [s.text for s in doc.sents],
    }
```

Offload heavy batch jobs to workers; `disable` unused pipes in the API process.

---

## Pitfalls

| Issue | Fix |
| --- | --- |
| Model not found | `python -m spacy download en_core_web_sm` |
| Slow cold start | Load once; `nlp.pipe`; disable unused components |
| `char_span` is `None` | Align entity offsets to token boundaries |
| Matcher false positives | Tighten patterns; combine with NER |

---

## Quick Reference

| Task | Code |
| --- | --- |
| Install + model | `pip install spacy` → `python -m spacy download en_core_web_sm` |
| Load / process | `nlp = spacy.load(...)` → `doc = nlp(text)` |
| Batch | `nlp.pipe(texts, batch_size=64)` |
| Entities | `doc.ents` |
| Token rules | `Matcher(nlp.vocab).add(...)` |
| Phrase list | `PhraseMatcher(nlp.vocab, attr="LOWER")` |
| Visualize | `displacy.render(doc, style="ent")` |
| Train NER | `python -m spacy train config.cfg --output ./models` |

---

## Related Notes

- [[NLP]] — stack overview
- [[NLP — NLTK]] — corpora and classical algorithms
- [[ML — scikit-learn]] — features on spaCy output
- [[AI]] — LLM vs cheap spaCy structure
- [[API - FastAPI]] — serve endpoints

---

## Tags

#python #nlp #spacy #ner #tokenization #matcher #lemmatization #pipeline #fastapi #production
