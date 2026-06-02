## What & When

**Gensim** is a Python library for **unsupervised topic modeling**, **word/document embeddings**, and **similarity search** over large text corpora. It streams data from disk (no need to load everything into RAM) and trains classical models — Word2Vec, Doc2Vec, LDA — with a consistent `corpus → model → index` workflow.

Use Gensim when:

- You need **custom domain embeddings** on your own corpus
- **LDA topic models** or **document similarity** without transformer GPUs
- **Millions of documents** via iterators and `LdaMulticore`
- Bridging text to tabular ML ([[ML — scikit-learn]]) or exporting vectors to [[AI — Chroma]]

```bash
pip install gensim
```

Overview: [[NLP]]. For neural RAG embeddings, prefer [[AI — Chroma]].

---

## Gensim vs Related Tools

| Need | Use | Notes |
| --- | --- | --- |
| Word/doc vectors on your corpus | **Gensim** | Word2Vec, Doc2Vec |
| LDA / topics | **Gensim** | `LdaModel`, `LdaMulticore` |
| Similarity index | **Gensim** | `MatrixSimilarity` |
| Tabular ML on topics | [[ML — scikit-learn]] | LDA doc-topic vectors as `X` |
| Semantic search (modern) | [[AI — Chroma]] | Sentence embeddings |
| Quick sentiment | [[NLP — TextBlob]] | Not Gensim's focus |

---

## Core Objects — Dictionary, Corpus, Model

Gensim represents text as sparse **bags-of-words** `(word_id, count)` tied to a `Dictionary`.

```python
from gensim.corpora import Dictionary
from gensim.utils import simple_preprocess

documents = [
    "machine learning models need clean text",
    "natural language processing uses tokens",
    "topic models discover latent themes",
]

tokenized = [simple_preprocess(doc) for doc in documents]
dictionary = Dictionary(tokenized)
dictionary.filter_extremes(no_below=1, no_above=0.9)
corpus = [dictionary.doc2bow(text) for text in tokenized]

dictionary.save("dict.gensim")
# corpora.MmCorpus.serialize("corpus.mm", corpus)  # disk-backed stream
```

---

## Streaming Corpus (Large Data)

Yield documents from disk so RAM stays bounded; train Word2Vec or build a `Dictionary` in one pass.

```python
from gensim import corpora
from gensim.models import Word2Vec
from gensim.utils import simple_preprocess

class MyCorpus:
    def __iter__(self):
        with open("reviews.txt", encoding="utf-8") as f:
            for line in f:
                yield simple_preprocess(line.strip())

dictionary = corpora.Dictionary(MyCorpus())
dictionary.filter_extremes(no_below=5, no_above=0.5, keep_n=50_000)

def bow_corpus():
    for tokens in MyCorpus():
        yield dictionary.doc2bow(tokens)

w2v = Word2Vec(sentences=MyCorpus(), vector_size=100, window=5, min_count=5, workers=4)
```

`MmCorpus` on disk lets LDA stream without loading the full matrix.

---

## Word2Vec

Train **word embeddings**; neighbors have high cosine similarity.

```python
from gensim.models import Word2Vec, KeyedVectors
from gensim.utils import simple_preprocess

sentences = [simple_preprocess(s) for s in [
    "king rules the kingdom",
    "queen rules the queendom",
    "prince is young royalty",
]]

model = Word2Vec(
    sentences=sentences,
    vector_size=100,
    window=5,
    min_count=1,
    workers=4,
    sg=1,       # 1=Skip-gram, 0=CBOW
    epochs=20,
)

print(model.wv.most_similar("king", topn=5))
model.save("w2v.model")

# Vectors only (smaller artifact)
model.wv.save("w2v.kv")
kv = KeyedVectors.load("w2v.kv")
kv.similarity("king", "queen")
```

Average word vectors per document → features for [[ML — scikit-learn]] `Pipeline`.

---

## Doc2Vec (Paragraph Vectors)

Embed **whole documents** for document-level similarity.

```python
from gensim.models.doc2vec import Doc2Vec, TaggedDocument
from gensim.utils import simple_preprocess

raw = [
    "gensim trains on unlabeled text",
    "lda finds topics in documents",
    "word2vec learns word neighbors",
]
docs = [TaggedDocument(simple_preprocess(t), [i]) for i, t in enumerate(raw)]

d2v = Doc2Vec(docs, vector_size=64, window=5, min_count=1, epochs=40, dm=1)

inferred = d2v.infer_vector(simple_preprocess("topic modeling with gensim"))
print(d2v.dv.most_similar([inferred], topn=3))
```

At scale, export normalized vectors to [[AI — Chroma]] instead of brute-force loops.

---

## LDA (Latent Dirichlet Allocation)

Discover **K topics** as word distributions; each document is a mixture of topics.

```python
from gensim.models import LdaModel, LdaMulticore
from gensim.corpora import Dictionary
from gensim.utils import simple_preprocess

texts = [
    simple_preprocess("bank loan interest rate mortgage"),
    simple_preprocess("river bank fishing water stream"),
    simple_preprocess("loan application credit score approval"),
]
dictionary = Dictionary(texts)
corpus = [dictionary.doc2bow(t) for t in texts]

lda = LdaModel(
    corpus=corpus,
    id2word=dictionary,
    num_topics=2,
    random_state=42,
    passes=15,
    alpha="auto",
)
print(lda.print_topics(num_words=5))

# Large corpora
# lda_mc = LdaMulticore(corpus=bow_corpus(), id2word=dictionary, num_topics=20, workers=4)

doc_topics = lda.get_document_topics(corpus[0], minimum_probability=0)  # → sklearn X
```

---

## Similarity Search

Index a corpus with TF-IDF (or LSI) and rank documents for a query.

```python
from gensim import similarities
from gensim.models import TfidfModel

tfidf = TfidfModel(corpus)
corpus_tfidf = tfidf[corpus]
index = similarities.MatrixSimilarity(corpus_tfidf, num_features=len(dictionary))

query = dictionary.doc2bow(simple_preprocess("latent topics in text"))
sims = index[tfidf[query]]
top = sorted(enumerate(sims), key=lambda x: -x[1])[:3]
```

**Word-level** similarity:

```python
model.wv.most_similar(positive=["king", "woman"], negative=["man"], topn=5)
```

Use [[ML — scikit-learn]] `cosine_similarity` when you already have dense NumPy matrices.

---

## sklearn Bridge

Materialize LDA `get_document_topics` into a dense matrix → `X` for [[ML — scikit-learn]] `LogisticRegression` or `Pipeline` (concatenate with numeric features from [[ML — pandas]]).

---

## Pitfalls

- **Tiny corpora** — LDA/Word2Vec need volume; dozens of docs produce noise.
- **Classical vs neural embeddings** — Gensim vectors ≠ sentence-transformer quality for RAG ([[AI — Chroma]]).
- **Dictionary drift** — version `dict.gensim` with every model release.
- **`workers>1`** — use `if __name__ == "__main__":` on Windows/macOS spawn.
- **`MatrixSimilarity` RAM** — shard or migrate to a vector DB at huge scale.

Tokenize with `simple_preprocess` or [[NLP — NLTK]] lemmas; use `Phrases` for bigrams on compound terms.

---

## Quick Reference

| Task | Code |
| --- | --- |
| Tokenize | `simple_preprocess(text)` |
| Dictionary / BoW | `Dictionary(...)`, `doc2bow(tokens)` |
| Word2Vec | `Word2Vec(sentences=..., vector_size=100)` |
| Doc2Vec | `Doc2Vec(TaggedDocument(...))` |
| LDA | `LdaModel(corpus=..., id2word=dictionary, num_topics=K)` |
| Word sim | `model.wv.most_similar("word")` |
| Doc sim | `similarities.MatrixSimilarity(corpus_tfidf)` |

---

## Related Notes

- [[NLP]]
- [[ML — scikit-learn]]
- [[AI — Chroma]]
- [[NLP — NLTK]]
- [[NLP — TextBlob]]
- [[Machine Learning]]

---

## Tags

#gensim #nlp #word2vec #doc2vec #lda #topic-modeling #embeddings #similarity #python #text-mining
