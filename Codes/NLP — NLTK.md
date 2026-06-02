## What & When

**NLTK** (Natural Language Toolkit) is the classic Python library for **linguistic NLP** — tokenizers, stemmers, lemmatizers, POS taggers, n-grams, and downloadable **corpora** (WordNet, stopwords, treebanks). Best for learning, research, and prototypes; for production APIs use [[NLP — spaCy]].

Use NLTK when:

- Exploring **tokenization**, **stemming**, and **lemmatization**
- Downloading **corpora** for batch or academic work
- Building **n-gram** features or collocation analysis
- Teaching NLP before porting to spaCy / [[ML — scikit-learn]]

```bash
pip install nltk
```

Sentiment shortcut: [[NLP — TextBlob]]. Stack map: [[NLP]].

---

## NLTK vs Related Tools

| Need | Use | Notes |
| --- | --- | --- |
| Corpora, stem, POS teaching | **NLTK** | `nltk.download(...)` |
| Production pipeline speed | [[NLP — spaCy]] | Single `nlp()` call |
| One-line sentiment | [[NLP — TextBlob]] | NLTK + Pattern underneath |
| Tabular ML features | NLTK → [[ML — scikit-learn]] | `CountVectorizer` / TF-IDF |
| Regex cleanup | [[Python — re (Regular Expressions)]] | Before tokenize |
| API hot path | [[NLP — spaCy]] + [[API - FastAPI]] | NLTK per-request is slow |

---

## Corpora Download

Data is not bundled — fetch once (or bake into Docker build).

```python
import nltk

nltk.download("punkt_tab")      # tokenizers (NLTK 3.8+)
nltk.download("stopwords")
nltk.download("wordnet")
nltk.download("averaged_perceptron_tagger_eng")
nltk.download("omw-1.4")
```

Containers: set `nltk.data.path` and bake downloads into the Docker image build.

---

## Tokenization

```python
from nltk.tokenize import word_tokenize, sent_tokenize, RegexpTokenizer, TweetTokenizer

text = "NLTK is great! It costs $0. Dr. Smith runs fast."

sent_tokenize(text)
# ['NLTK is great!', 'It costs $0.', 'Dr. Smith runs fast.']

word_tokenize(text)
# ['NLTK', 'is', 'great', '!', 'It', 'costs', '$', '0', '.', ...]

RegexpTokenizer(r"\w+").tokenize("Email alice@example.com")
TweetTokenizer().tokenize("NLTK + spaCy = :) #nlp")
```

## Stemming

Stemmers chop suffixes heuristically — fast but not dictionary-aware.

```python
from nltk.stem import PorterStemmer, SnowballStemmer

porter = PorterStemmer()
snowball = SnowballStemmer("english")
words = ["running", "runs", "easily", "generousness"]

[porter.stem(w) for w in words]    # ['run', 'run', 'easili', 'gener']
[snowball.stem(w) for w in words]  # often cleaner for English
```

## Lemmatization

Uses WordNet POS tags for dictionary base forms.

```python
from nltk.stem import WordNetLemmatizer
from nltk.corpus import wordnet
from nltk import pos_tag
from nltk.tokenize import word_tokenize

lemmatizer = WordNetLemmatizer()

def nltk_pos_to_wordnet(tag: str) -> str:
    if tag.startswith("J"): return wordnet.ADJ
    if tag.startswith("V"): return wordnet.VERB
    if tag.startswith("N"): return wordnet.NOUN
    if tag.startswith("R"): return wordnet.ADV
    return wordnet.NOUN

text = "The mice are running faster"
tagged = pos_tag(word_tokenize(text))
lemmas = [lemmatizer.lemmatize(w, nltk_pos_to_wordnet(t)) for w, t in tagged]
# ['The', 'mouse', 'be', 'run', 'fast']
```

## Part-of-Speech Tagging

```python
from nltk import pos_tag
from nltk.tokenize import word_tokenize

tagged = pos_tag(word_tokenize("NLTK tokenizes and tags parts of speech."))
# [('NLTK', 'NNP'), ('tokenizes', 'VBZ'), ('and', 'CC'), ...]

nouns = [w for w, tag in tagged if tag.startswith("NN")]
```

## N-grams

```python
from nltk import ngrams, bigrams
from nltk.tokenize import word_tokenize
from collections import Counter

tokens = word_tokenize("natural language processing with python")

list(bigrams(tokens))
list(ngrams(tokens, 4))

texts = ["new york city", "new york times", "york city limits"]
all_bigrams = [b for t in texts for b in bigrams(word_tokenize(t))]
Counter(all_bigrams).most_common(5)
```

Feed into [[ML — scikit-learn]] `CountVectorizer(ngram_range=(1, 2))`.

---

## Stopwords & Normalization

```python
from nltk.corpus import stopwords
from nltk.tokenize import word_tokenize

stops = set(stopwords.words("english"))
filtered = [
    t.lower() for t in word_tokenize("This is a simple example")
    if t.isalpha() and t.lower() not in stops
]
```

Chain with stem/lemmatize for cleaner vocabularies.

---

## sklearn Feature Pipeline

```python
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.pipeline import Pipeline
from sklearn.linear_model import LogisticRegression
from nltk.corpus import stopwords
from nltk.stem import WordNetLemmatizer
from nltk.tokenize import word_tokenize

stops = set(stopwords.words("english"))
lemmatizer = WordNetLemmatizer()

def normalize(text: str) -> str:
    tokens = [t for t in word_tokenize(text.lower()) if t.isalpha() and t not in stops]
    return " ".join(lemmatizer.lemmatize(t) for t in tokens)

corpus = ["good product", "bad product", "excellent service"]
clf = Pipeline([
    ("tfidf", TfidfVectorizer()),
    ("lr", LogisticRegression(max_iter=1000)),
])
# clf.fit([normalize(t) for t in corpus], y)
```

Swap NLTK normalize for spaCy lemmas in production.

---

## FastAPI — Internal / Dev Endpoint

Acceptable for **low-traffic** tools if corpora are pre-downloaded.

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI
from pydantic import BaseModel
import nltk
from nltk.tokenize import word_tokenize, sent_tokenize
from nltk import pos_tag

@asynccontextmanager
async def lifespan(app: FastAPI):
    for pkg in ("punkt_tab", "averaged_perceptron_tagger_eng", "stopwords"):
        nltk.download(pkg, quiet=True)
    yield

app = FastAPI(lifespan=lifespan)

class TextIn(BaseModel):
    text: str

@app.post("/v1/nltk/analyze")
def analyze(body: TextIn):
    tokens = word_tokenize(body.text)
    return {
        "sentences": sent_tokenize(body.text),
        "tokens": tokens,
        "pos": [{"token": w, "tag": t} for w, t in pos_tag(tokens)],
    }
```

Public APIs: [[NLP — spaCy]] + [[API - FastAPI]].

---

## TextBlob Bridge

[[NLP — TextBlob]] wraps NLTK for simpler sentiment and tagging:

```python
from textblob import TextBlob
blob = TextBlob("NLTK is powerful but spaCy is faster.")
blob.words; blob.tags; blob.sentiment  # polarity, subjectivity
```

---

## Pitfalls

| Issue | Fix |
| --- | --- |
| `LookupError: punkt` | `nltk.download("punkt_tab")` |
| Lemma unchanged | Pass WordNet POS to `lemmatize` |
| Slow large corpora | Batch offline; migrate hot path to spaCy |
| Docker without network | Pre-download at image build |

---

## Quick Reference

| Task | Code |
| --- | --- |
| Install | `pip install nltk` |
| Download | `nltk.download("punkt_tab")` |
| Sentences / words | `sent_tokenize(text)` / `word_tokenize(text)` |
| Stem / lemma | `PorterStemmer().stem(w)` / `WordNetLemmatizer().lemmatize(w, pos)` |
| POS | `pos_tag(tokens)` |
| N-grams | `bigrams(tokens)` / `ngrams(tokens, n)` |
| Stopwords | `stopwords.words("english")` |

---

## Related Notes

- [[NLP]] — overview and tool selection
- [[NLP — spaCy]] — production pipelines and NER
- [[NLP — TextBlob]] — sentiment and simple API
- [[ML — scikit-learn]] — vectorizers and classifiers
- [[API - FastAPI]] — expose analysis endpoints

---

## Tags

#python #nlp #nltk #tokenization #stemming #lemmatization #pos-tagging #ngrams #corpora #wordnet
