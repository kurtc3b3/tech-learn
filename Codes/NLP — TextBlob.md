## What & When

**TextBlob** is a beginner-friendly NLP library built on **NLTK** and **Pattern**. It exposes high-level APIs for **sentiment** (polarity, subjectivity), **noun phrases**, **translation**, and light NLP helpers without assembling low-level [[NLP — NLTK]] pipelines by hand.

Use TextBlob when:

- You need a **sentiment baseline** in a notebook or demo in minutes
- Prototyping **noun phrase** extraction or simple text features
- Light **translation** on short English-centric snippets

Avoid TextBlob for high-throughput production, strict multilingual coverage, or custom NER — use [[NLP — spaCy]] or [[AI]] extractors instead. Stack context: [[NLP]].

```bash
pip install textblob
python -m textblob.download_corpora
```

> [!warning] Corpora download Fetches NLTK data (punkt, taggers). Run once per venv, CI image, or Docker layer:

```dockerfile
RUN pip install textblob && python -m textblob.download_corpora
```

If `LookupError: punkt` appears, re-run the download or set `NLTK_DATA`.

---

## TextBlob vs Related Tools

| Need | Use | Notes |
| --- | --- | --- |
| Quick sentiment | **TextBlob** | `blob.sentiment` |
| Corpora, stemmers | [[NLP — NLTK]] | Under the hood |
| Production NER / speed | [[NLP — spaCy]] | Trainable pipelines |
| Topics / embeddings | [[NLP — Gensim]] | LDA, Word2Vec |
| Regex cleanup | [[Python — re (Regular Expressions)]] | Before analysis |

---

## TextBlob Basics

```python
from textblob import TextBlob

blob = TextBlob("TextBlob makes sentiment analysis simple.")

print(blob.words)         # WordList
print(blob.sentences)     # list of Sentence
print(blob.noun_phrases)  # may be empty on very short input
print(blob.tags)          # POS tuples
```

Normalize HTML and encoding before analysis ([[Python — BeautifulSoup4 (bs4)]]).

---

## Sentiment — Polarity & Subjectivity

The default **Pattern** analyzer returns `blob.sentiment`:

- **Polarity** — `-1.0` (negative) to `+1.0` (positive)
- **Subjectivity** — `0.0` (objective) to `1.0` (subjective / opinion)

```python
from textblob import TextBlob

reviews = [
    "I love this product, absolutely fantastic!",
    "Terrible experience, would not recommend.",
    "The device weighs 200 grams and ships in a box.",
]

for text in reviews:
    b = TextBlob(text)
    pol, sub = b.sentiment.polarity, b.sentiment.subjectivity
    label = "pos" if pol > 0.1 else "neg" if pol < -0.1 else "neutral"
    print(f"{label:8} pol={pol:+.2f} sub={sub:.2f} | {text[:50]}")
```

**pandas** batch scoring:

```python
import pandas as pd
from textblob import TextBlob

df = pd.DataFrame({"review": reviews})
df["polarity"] = df["review"].apply(lambda t: TextBlob(t).sentiment.polarity)
df["subjectivity"] = df["review"].apply(lambda t: TextBlob(t).sentiment.subjectivity)
```

**Custom classifier** (Naive Bayes on labeled tuples):

```python
from textblob.classifiers import NaiveBayesClassifier

train = [("amazing great product", "pos"), ("horrible worst ever", "neg")]
clf = NaiveBayesClassifier(train)
blob = TextBlob("not bad at all", classifier=clf)
print(blob.classify())
```

Feed `polarity` / `subjectivity` into [[ML — scikit-learn]] alongside TF-IDF or [[NLP — Gensim]] topic features.

---

## Noun Phrases

Chunked **noun groups** for tags, facets, or quick keywords.

```python
from textblob import TextBlob
from collections import Counter

text = """
Apple Inc. announced a new MacBook in Cupertino.
The laptop features improved battery life and a sharper display.
"""
blob = TextBlob(text)
print(list(blob.noun_phrases))

phrases = Counter()
for doc in corpus_docs:
    phrases.update(TextBlob(doc).noun_phrases)
print(phrases.most_common(10))
```

Quality depends on downloaded corpora — validate on your domain. For production keywords, use [[NLP — spaCy]] `noun_chunks`.

---

## Translation

Wraps **Google Translate** via Pattern; requires network and is rate-limited.

```python
from textblob import TextBlob

en = TextBlob("Hello, how are you?")
es = en.translate(to="es")
print(str(es))

fr = TextBlob("Bonjour le monde").translate(to="en")
print(fr)

print(TextBlob("Guten Tag").detect_language())  # 'de'
```

> [!tip] Production i18n Use a managed translation API with SLAs — not TextBlob — on critical paths.

---

## Spelling & Correction (Optional)

```python
from textblob import Word

print(Word("speling").correct())      # 'spelling'
print(TextBlob("fix teh speling").correct())  # slow on long text
```

Useful for noisy user input before sentiment; cache results in bulk jobs.

---

## FastAPI Demo

```python
from fastapi import FastAPI
from pydantic import BaseModel
from textblob import TextBlob

app = FastAPI()

class TextIn(BaseModel):
    text: str

@app.post("/v1/sentiment")
def sentiment(body: TextIn):
    b = TextBlob(body.text)
    return {
        "polarity": b.sentiment.polarity,
        "subjectivity": b.sentiment.subjectivity,
        "noun_phrases": list(b.noun_phrases),
    }
```

See [[API - FastAPI]] for auth, rate limits, and worker queues — TextBlob is CPU-bound per request.

---

## Pitfalls

- **Missing corpora** — always `python -m textblob.download_corpora` in deploy.
- **English bias** — Pattern sentiment is weak on many languages.
- **Sarcasm & negation** — lexicon methods miss nuance; validate on your data.
- **Performance** — full parse per call; cache or batch offline.
- **Translation** — network failures; handle exceptions.
- **Short text** — `noun_phrases` may be empty; not a substitute for NER.

---

## Quick Reference

| Task | Code |
| --- | --- |
| Create | `TextBlob("text")` |
| Polarity / subjectivity | `blob.sentiment.polarity`, `.subjectivity` |
| Noun phrases | `blob.noun_phrases` |
| Classify (custom) | `TextBlob(text, classifier=clf)` |
| Translate | `blob.translate(to="es")` |
| Install + data | `pip install textblob && python -m textblob.download_corpora` |

---

## Related Notes

- [[NLP]]
- [[NLP — NLTK]]
- [[NLP — spaCy]]
- [[NLP — Gensim]]
- [[ML — scikit-learn]]
- [[ML — pandas]]
- [[API - FastAPI]]

---

## Tags

#textblob #nlp #sentiment #polarity #subjectivity #nltk #python #text-processing #prototype
