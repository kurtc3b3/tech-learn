## What & When

**Neo4j** is a **property graph database** — **nodes** (entities), **relationships** (typed edges with properties), **Cypher** query language. Use when the problem is **connectivity** — paths, recommendations, fraud rings, org hierarchies — not flat tables ([[ORM - SQLAlchemy]]).

Use Neo4j when:

- **Traversals** dominate ("friends of friends within 3 hops")
- **Relationship metadata** matters (since, weight, role)
- **Pattern matching** in graphs (cycles, shortest path)
- Knowledge graphs feeding [[AI]] / RAG entity links

```bash
pip install neo4j
# Local: docker run -d -p 7474:7474 -p 7687:7687 neo4j:5
# Browser UI: http://localhost:7474
```

Overview: [[DB]].

---

## Neo4j vs Related Tools

| Need | Use | Notes |
| --- | --- | --- |
| Path / recommendation | **Neo4j** | Cypher `MATCH` |
| Tabular CRUD | [[ORM - SQLAlchemy]] | Joins get ugly deep |
| Document blobs | [[DB — MongoDB]] | Weak traversal |
| Graph analytics batch | [[ML — NetworkX]] | In-memory Python |
| RDF / triple stores | Specialized (not here) | Different model |

---

## Graph Model

```text
(Alice:Person)-[:KNOWS {since: 2020}]->(Bob:Person)
(Alice)-[:BOUGHT]->(Product {sku: "X1"})
```

Labels = node types; relationship types = edge semantics.

---

## Cypher Basics

```cypher
// Create
CREATE (a:Person {name: "Alice"})
CREATE (b:Person {name: "Bob"})
CREATE (a)-[:KNOWS {since: 2020}]->(b);

// Match
MATCH (p:Person {name: "Alice"})-[:KNOWS*1..2]-(friend)
RETURN DISTINCT friend.name;

// Shortest path
MATCH path = shortestPath(
  (a:Person {name: "Alice"})-[:KNOWS*]-(b:Person {name: "Carol"})
)
RETURN path;
```

---

## Python Driver

```python
from neo4j import GraphDatabase

driver = GraphDatabase.driver("bolt://localhost:7687", auth=("neo4j", "password"))

def create_person(tx, name):
    tx.run("CREATE (p:Person {name: $name})", name=name)

def find_friends(tx, name):
    result = tx.run(
        """
        MATCH (p:Person {name: $name})-[:KNOWS]-(f)
        RETURN f.name AS friend
        """,
        name=name,
    )
    return [r["friend"] for r in result]

with driver.session() as session:
    session.execute_write(create_person, "Ada")
    friends = session.execute_read(find_friends, "Ada")
```

Always parameterize — never f-string Cypher with user input.

---

## FastAPI Pattern

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI, Depends
from neo4j import GraphDatabase

driver = GraphDatabase.driver("bolt://localhost:7687", auth=("neo4j", "password"))

@asynccontextmanager
async def lifespan(app: FastAPI):
    yield
    driver.close()

app = FastAPI(lifespan=lifespan)

def get_session():
    with driver.session() as session:
        yield session
```

Use sync driver in threadpool or neo4j async driver for heavy async apps.

---

## When SQL Is Enough

| Query | Prefer |
| --- | --- |
| 1-hop FK join | PostgreSQL |
| Aggregates on facts | SQL |
| 4+ hop variable depth | Neo4j |
| Pattern detection | Neo4j |

---

## Docker Compose

```yaml
services:
  neo4j:
    image: neo4j:5
    ports:
      - "7474:7474"
      - "7687:7687"
    environment:
      NEO4J_AUTH: neo4j/password
    volumes:
      - neo4jdata:/data
volumes:
  neo4jdata:
```

---

## ML / AI Link

Export subgraph features → [[ML — scikit-learn]] or embed nodes for [[AI — Chroma]]. [[ML — NetworkX]] for offline analysis; Neo4j for online serving queries.

---

## Quick Reference

| Task | Cypher / API |
| --- | --- |
| Create node | `CREATE (n:Label {prop: val})` |
| Match | `MATCH (n:Label) WHERE ... RETURN n` |
| Variable hops | `-[:REL*1..3]-` |
| Merge upsert | `MERGE (n:Label {id: $id}) SET n += $props` |
| Python | `GraphDatabase.driver`, `session.run` |
| Browser | Neo4j Browser on :7474 |

---

## Related Notes

- [[DB]]
- [[ORM - SQLAlchemy]]
- [[ML — NetworkX]]
- [[AI]]
- [[API - FastAPI — Lifespan]]

---

## Tags

#database #neo4j #graph #cypher #python #relationships #knowledge-graph
