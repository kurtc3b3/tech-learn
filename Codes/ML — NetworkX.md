## What & When

**NetworkX** is a Python library for **creating, analyzing, and visualizing graphs** — nodes, edges, weights, and attributes. In ML workflows it supports **graph features**, community detection, centrality for tabular enrichment, and pipeline graphs (related to but distinct from [[AI — LangGraph]] agent graphs).

Use NetworkX when:

- Building **graph-based features** (degree, PageRank) for [[ML — scikit-learn]]
- Analyzing **relationship data** (social, fraud rings, supply chain)
- Prototyping **GNN** inputs before PyTorch Geometric / DGL
- Visualizing small graphs in EDA with [[ML — matplotlib]]

```bash
pip install networkx
# Layout / viz often needs matplotlib: pip install matplotlib
```

For production graph databases at scale, pair with Neo4j or similar; NetworkX is in-memory. Overview: [[Machine Learning]].

---

## NetworkX vs Related Tools

| Need | Use | Notes |
| --- | --- | --- |
| In-memory graph algorithms | **NetworkX** | Pure Python, easy API |
| Billion-edge analytics | Spark GraphFrames, Neo4j | Out-of-core / server |
| Neural graphs | PyTorch Geometric | Training on `edge_index` |
| Agent state machines | [[AI — LangGraph]] | LLM orchestration, not NetworkX |
| Tabular ML | [[ML — scikit-learn]] | Consume NetworkX-derived features |
| Plotting | [[ML — matplotlib]] | `nx.draw`, `spring_layout` |

---

## Create Graphs

```python
import networkx as nx

# Undirected
G = nx.Graph()
G.add_edge("A", "B", weight=0.5)
G.add_edge("B", "C", weight=1.2)

# Directed
DG = nx.DiGraph()
DG.add_edges_from([("user1", "user2"), ("user2", "user3")])

# From pandas (edge list)
import pandas as pd

edges = pd.DataFrame({"source": ["a", "b"], "target": ["b", "c"], "weight": [1.0, 2.0]})
G = nx.from_pandas_edgelist(edges, "source", "target", edge_attr=["weight"])

# Classic generators
karate = nx.karate_club_graph()
erdos = nx.erdos_renyi_graph(n=100, p=0.05, seed=42)
```

---

## Node & Edge Attributes

```python
G = nx.Graph()
G.add_node("u1", role="buyer", country="TR")
G.add_node("u2", role="seller", country="DE")
G.add_edge("u1", "u2", amount=1500.0, currency="EUR")

# Access
G.nodes["u1"]["role"]
G.edges["u1", "u2"]["amount"]

# Bulk attributes
nx.set_node_attributes(G, {n: n for n in G.nodes}, name="label")
degrees = dict(G.degree())
nx.set_node_attributes(G, degrees, name="degree")
```

---

## Centrality & ML Features

Export centralities as columns for [[ML — scikit-learn]].

```python
import pandas as pd

def graph_features(G: nx.Graph) -> pd.DataFrame:
    nodes = list(G.nodes())
    return pd.DataFrame({
        "node": nodes,
        "degree": [G.degree(n) for n in nodes],
        "pagerank": list(nx.pagerank(G).values()),
        "betweenness": list(nx.betweenness_centrality(G).values()),
        "clustering": list(nx.clustering(G).values()),
    })

feat = graph_features(karate)
# Merge feat into tabular training frame on node id
```

```python
# Weighted PageRank
pr = nx.pagerank(G, weight="weight")

# k-core for fraud — dense interconnect
core = nx.core_number(G)
```

---

## Community Detection

```python
from networkx.algorithms import community

G = nx.karate_club_graph()
communities = community.greedy_modularity_communities(G)
partition = {node: i for i, com in enumerate(communities) for node in com}

# Louvain (needs python-louvain package) — alternative:
# import community as community_louvain
# partition = community_louvain.best_partition(G)
```

Use **community id** as categorical feature or for stratified sampling.

---

## Paths, Connectivity & Subgraphs

```python
# Shortest path
path = nx.shortest_path(DG, "user1", "user3")

# Weakly connected components (directed)
components = list(nx.weakly_connected_components(DG))

# Ego network around a node (1-hop neighborhood)
ego = nx.ego_graph(G, "u1", radius=2)

# Subgraph induced by high-degree nodes
hub_nodes = [n for n, d in G.degree() if d >= 5]
H = G.subgraph(hub_nodes).copy()
```

---

## IO — Read & Write

```python
# Adjacency / edgelist
nx.write_edgelist(G, "data/graph.edgelist")
G2 = nx.read_edgelist("data/graph.edgelist", nodetype=int)

# GraphML (preserves attributes)
nx.write_graphml(G, "data/graph.graphml")
G3 = nx.read_graphml("data/graph.graphml")

# GEXF for Gephi
nx.write_gexf(G, "data/graph.gexf")
```

---

## Visualization (Small Graphs)

```python
import matplotlib.pyplot as plt

pos = nx.spring_layout(G, seed=42)
nx.draw_networkx_nodes(G, pos, node_size=80, node_color="lightblue")
nx.draw_networkx_edges(G, pos, alpha=0.4)
nx.draw_networkx_labels(G, pos, font_size=8)
plt.axis("off")
plt.tight_layout()
plt.savefig("figures/graph.png", dpi=150)
plt.close()
```

For large graphs, sample nodes or use **Gephi** / **yFiles** exports.

---

## Bipartite Graphs

```python
B = nx.Graph()
B.add_nodes_from(["user1", "user2"], bipartite=0)
B.add_nodes_from(["itemA", "itemB"], bipartite=1)
B.add_edges_from([("user1", "itemA"), ("user2", "itemA")])

# Project to user-user co-occurrence
users = {n for n, d in B.nodes(data=True) if d.get("bipartite") == 0}
user_graph = nx.bipartite.weighted_projected_graph(B, users)
```

Common for **recommendation** and **collaborative filtering** feature engineering.

---

## FastAPI — Graph Stats Endpoint

```python
from fastapi import FastAPI
import networkx as nx

app = FastAPI()
GRAPH: nx.Graph | None = None

@app.on_event("startup")
def load_graph():
    global GRAPH
    GRAPH = nx.read_graphml("data/graph.graphml")

@app.get("/nodes/{node_id}/metrics")
def node_metrics(node_id: str):
    if node_id not in GRAPH:
        return {"error": "not found"}
    return {
        "degree": GRAPH.degree(node_id),
        "pagerank": nx.pagerank(GRAPH)[node_id],
        "clustering": nx.clustering(GRAPH, node_id),
    }
```

Combine with [[API - FastAPI]] patterns from the vault; batch features offline, serve precomputed scores in production.

---

## Pitfalls

- **Memory** — `O(n + m)` in RAM; millions of nodes need another stack.
- **Directed vs undirected** — algorithms differ (`DiGraph` vs `Graph`).
- **Isolates** — filter or impute before ML joins.
- **Leaking future edges** — in temporal graphs, only use edges available at prediction time.

---

## Quick Reference

| Task | Code |
| --- | --- |
| Empty graph | `G = nx.Graph()` / `nx.DiGraph()` |
| Add edge | `G.add_edge(u, v, weight=w)` |
| From edgelist DF | `nx.from_pandas_edgelist(df, "src", "dst")` |
| Degree | `G.degree(n)` / `dict(G.degree())` |
| PageRank | `nx.pagerank(G, weight="weight")` |
| Communities | `community.greedy_modularity_communities(G)` |
| Shortest path | `nx.shortest_path(G, source, target)` |
| Draw | `nx.spring_layout(G)` + `nx.draw_networkx_*` |

---

## Related Notes

- [[Machine Learning]]
- [[ML — scikit-learn]]
- [[ML — pandas]]
- [[ML — matplotlib]]
- [[API - FastAPI]]
- [[AI — LangGraph]]

---

## Tags

#python #machine-learning #networkx #graphs #graph-features #centrality #eda
