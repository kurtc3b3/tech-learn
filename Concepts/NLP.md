**Key Points:**

- **spaCy is the production default** — fast tokenization, NER, lemmatization, and trainable pipelines for backend text processing.
- **NLTK for learning & linguistics** — corpora, stemmers, parsers; great for prototypes and education, slower in production.
- **Gensim for topics & embeddings** — Word2Vec, Doc2Vec, LDA, similarity search on large corpora.
- **TextBlob for quick sentiment** — simple API on top of NLTK; fine for demos, not heavy production loads.
- **LLM era** — classical NLP still matters for preprocessing, feature extraction, and hybrid RAG (chunk metadata, entity tags).

# NLP — Overview & Text Processing Stack

## What is NLP (in this vault)?

**Natural language processing** here means turning **raw text** into structured signals — tokens, lemmas, entities, topics, sentiment, vectors — before ML models or LLMs consume it. It sits between document ingestion ([[AI — Docling]], [[Browser Automation — crawl4ai]]) and downstream [[AI]] / [[Machine Learning]] systems.

Typical outcomes:

- **Clean & tokenize** text for search indexes or feature pipelines
- **Extract entities** (people, orgs, dates) for metadata and filters
- **Topic modeling & document similarity** with Gensim
- **Sentiment / polarity** baselines with TextBlob
- **Hybrid RAG** — spaCy NER on chunks + vector retrieval

---

## NLP Pipeline

```mermaid
flowchart LR
    RAW[Raw text / HTML / PDF text] --> CLEAN[Normalize unicode, strip]
    CLEAN --> TOK{Tool choice}
    TOK --> SPACY[spaCy pipeline]
    TOK --> NLTK[NLTK tokenize]
    SPACY --> FEAT[Entities, lemmas, vectors]
    NLTK --> FEAT
    FEAT --> DOWN{Downstream}
    DOWN --> ML[[ML — scikit-learn]]
    DOWN --> GENSIM[[NLP — Gensim]] topics
    DOWN --> RAG[[AI — Chroma]] + LLM
    DOWN --> API[[API - FastAPI]] enrich
```

---

## Tool Roles

| Tool | Best for | Avoid when |
| --- | --- | --- |
| [[NLP — spaCy]] | Production NLP, NER, custom pipelines, large volumes | You only need a one-liner sentiment score |
| [[NLP — NLTK]] | Corpora, stemmers, academic NLP, teaching | Latency-sensitive API hot paths |
| [[NLP — Gensim]] | Word2Vec, LDA, doc similarity, streaming corpora | Need pretrained transformers (use [[AI]] stack) |
| [[NLP — TextBlob]] | Quick sentiment, noun phrases, prototyping | High-throughput or multilingual production |

---

## Classical NLP vs LLM Stack

| Task | Classical ([[NLP]]) | LLM ([[AI]]) |
| --- | --- | --- |
| Sentiment (simple) | TextBlob, spaCy | Prompt + structured output |
| NER (domain-specific) | spaCy train | LLM extract (costlier) |
| Summarization | Extractive (Gensim) | abstractive LLM |
| Embeddings | Gensim Word2Vec, spaCy vectors | OpenAI / sentence-transformers |
| Pre-RAG cleanup | spaCy sentences, entities | markdownify + chunk |

Use **both**: spaCy for cheap structure, LLMs for generation and complex extraction.

---

## When to Use What

| Question | Choose |
| --- | --- |
| Production entity extraction? | [[NLP — spaCy]] |
| Download WordNet / Penn Treebank? | [[NLP — NLTK]] |
| Topic modeling on 1M docs? | [[NLP — Gensim]] |
| Demo sentiment in 5 lines? | [[NLP — TextBlob]] |
| Tabular features from text? | spaCy → lemmas/counts → [[ML — scikit-learn]] |
| Semantic doc search (classic)? | [[NLP — Gensim]] Doc2Vec or migrate to [[AI — Chroma]] |

---

## Recommended Learning Path

1. **TextBlob** — sentiment and noun phrases in minutes — [[NLP — TextBlob]]
2. **NLTK** — tokenization, stemming, corpora — [[NLP — NLTK]]
3. **spaCy** — industrial pipelines and NER — [[NLP — spaCy]]
4. **Gensim** — embeddings and topics at scale — [[NLP — Gensim]]
5. **Integrate** — FastAPI endpoint + [[ML — MLflow]] or RAG metadata — [[AI]]

Foundation: [[ML — pandas]] for text columns, [[Python — re (Regular Expressions)]] for regex cleanup.

---

## NLP in the Broader Landscape

| Concern | Tool |
| --- | --- |
| Regex cleanup | [[Python — re (Regular Expressions)]] |
| HTML → text | [[Python — markdownify]], [[Python — BeautifulSoup4 (bs4)]] |
| Classical NLP | This series |
| Tabular ML on text features | [[Machine Learning]] |
| LLM / RAG | [[AI]] |
| API surface | [[API - FastAPI]] |

---

## Related Notes

- [[NLP — spaCy]]
- [[NLP — NLTK]]
- [[NLP — Gensim]]
- [[NLP — TextBlob]]
- [[AI]]
- [[Machine Learning]]
- [[Python — re (Regular Expressions)]]
- [[Python Development]]

---

## Tags

#nlp #spacy #nltk #gensim #textblob #text-processing #python #backend
