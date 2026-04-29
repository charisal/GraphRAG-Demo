# GraphRAG Demo — Knowledge Graph + Vector Search over Wikipedia Scientists

> This demo project was created to accompany the blog post **[GraphRAG Explained: The Missing Layer in Modern RAG Systems](https://serkanaytekin.com/graphrag-explained-the-missing-layer-in-modern-rag-systems/)**.

A step-by-step demonstration of **Graph RAG** (Retrieval-Augmented Generation) that combines a Neo4j knowledge graph with Qdrant vector search to answer questions about 51 famous scientists.

The project compares three retrieval strategies side-by-side:

| Mode | Retrieval | Best for |
|------|-----------|---------|
| **Vector only** | Qdrant semantic similarity | "Who worked on quantum mechanics?" |
| **Graph only** | Neo4j multi-hop traversal | "How are Marie Curie and Niels Bohr connected?" |
| **Hybrid** | Vector + Graph combined | Rich, contextually complete answers |

---

## Architecture

```
Wikipedia URLs
      │
      ▼
┌─────────────────────────┐
│  1. Fetch Wikipedia     │  MediaWiki API → plain-text .txt files
└─────────────────────────┘
      │
      ▼
┌─────────────────────────┐
│  2. Extract Entities    │  Azure OpenAI GPT-4o → JSON triples
│     & Relationships     │  (entities + typed relationships)
└─────────────────────────┘
      │
      ├──────────────────────────────────────────────┐
      ▼                                              ▼
┌─────────────────────────┐          ┌──────────────────────────┐
│  3. Neo4j Knowledge     │          │  4. Qdrant Vector Store  │
│     Graph               │          │     (text-embedding-     │
│  (entities + relations) │          │      3-small, 1536-dim)  │
└─────────────────────────┘          └──────────────────────────┘
      │                                              │
      └──────────────┬───────────────────────────────┘
                     ▼
           ┌──────────────────┐
           │  5. GraphRAG     │  Entity extraction → Graph + Vector retrieval
           │     Query        │  → Azure OpenAI GPT-4o generates final answer
           └──────────────────┘
```

---

## Dataset

51 Wikipedia articles covering scientists from antiquity to the present day, including Archimedes, Isaac Newton, Marie Curie, Alan Turing, Richard Feynman, Jennifer Doudna, and many more. The full list is in [`src/data/scientist_wikipedia_urls.txt`](src/data/scientist_wikipedia_urls.txt).

---

## Prerequisites

| Dependency | Version | Notes |
|-----------|---------|-------|
| Python | ≥ 3.11 | |
| [Neo4j Desktop](https://neo4j.com/download/) or Neo4j AuraDB | 5.x | Local: `bolt://localhost:7687` |
| [Qdrant](https://qdrant.tech/documentation/quick-start/) | latest | Local: `http://localhost:6333` |
| Azure OpenAI | — | GPT-4o deployment + text-embedding-3-small deployment |

### Start Qdrant locally (Docker)

```bash
docker run -p 6333:6333 qdrant/qdrant
```

### Start Neo4j locally

Use Neo4j Desktop or run via Docker:

```bash
docker run \
  --name neo4j \
  -p 7474:7474 -p 7687:7687 \
  -e NEO4J_AUTH=neo4j/your_password \
  neo4j:5
```

---

## Setup

```bash
# 1. Clone the repo
git clone https://github.com/<your-username>/GraphRAG_Demo.git
cd GraphRAG_Demo

# 2. Create and activate a virtual environment
python -m venv .venv
# Windows
.venv\Scripts\activate
# macOS / Linux
source .venv/bin/activate

# 3. Install dependencies
pip install -r requirements.txt

# 4. Configure environment variables
cp .env.example .env
# Edit .env and fill in your values
```

### Environment variables (`.env`)

```ini
# ── Neo4j ────────────────────────────────────────────────────────────────────
NEO4J_URI=bolt://localhost:7687
NEO4J_USER=neo4j
NEO4J_PASSWORD=your_neo4j_password
NEO4J_DATABASE=neo4j

# ── Qdrant ───────────────────────────────────────────────────────────────────
QDRANT_URL=http://localhost:6333
COLLECTION_NAME=wikipedia_scientists
VECTOR_SIZE=1536

# ── Azure OpenAI ─────────────────────────────────────────────────────────────
endpoint=https://<your-resource>.openai.azure.com/
subscription_key=<your-api-key>

# Chat model (GPT-4o)
deployment=gpt-4o
api_version=2024-12-01-preview

# Embedding model (text-embedding-3-small)
embedding_deployment=text-embedding-3-small
embedding_api_version=2024-02-01
```

---

## Running the Notebooks

Run the notebooks **in order** from the `src/notebooks/` directory:

| # | Notebook | What it does |
|---|----------|-------------|
| 1 | [`1_get_data_from_wikipedia.ipynb`](src/notebooks/1_get_data_from_wikipedia.ipynb) | Fetches 51 Wikipedia articles as plain text |
| 2 | [`2_extractEntities.ipynb`](src/notebooks/2_extractEntities.ipynb) | Extracts entities & relationships using GPT-4o |
| 3 | [`3_import_triples_to_neo4j.ipynb`](src/notebooks/3_import_triples_to_neo4j.ipynb) | Imports the knowledge graph into Neo4j |
| 4 | [`4_create_vectors.ipynb`](src/notebooks/4_create_vectors.ipynb) | Chunks articles, embeds them, and stores vectors in Qdrant |
| 5 | [`5_execute_rag.ipynb`](src/notebooks/5_execute_rag.ipynb) | Runs GraphRAG queries and compares all three retrieval modes |

> **Tip:** Notebooks 1–4 skip already-processed files, so it is safe to re-run them after interruptions.

---

## Project Structure

```
GraphRAG_Demo/
├── .env                          # Your local environment variables (git-ignored)
├── .env.example                  # Template — copy to .env and fill in values
├── requirements.txt
└── src/
    ├── data/
    │   ├── scientist_wikipedia_urls.txt   # 51 Wikipedia URLs
    │   ├── wikipedia/                     # Raw article text (output of notebook 1)
    │   ├── wikipedia_entities/            # Extracted JSON triples (output of notebook 2)
    │   └── wikipedia_vectors/             # Embedding cache (output of notebook 4)
    ├── notebooks/
    │   ├── 1_get_data_from_wikipedia.ipynb
    │   ├── 2_extractEntities.ipynb
    │   ├── 3_import_triples_to_neo4j.ipynb
    │   ├── 4_create_vectors.ipynb
    │   └── 5_execute_rag.ipynb
    └── prompts/
        └── findentites.md                 # System prompt for entity/relationship extraction
```

---

## How It Works

### Entity & Relationship Extraction (Notebook 2)

GPT-4o reads each Wikipedia article and returns a structured JSON object containing:
- **Entities** — named things with a `name`, `type` (Person, Field, Discovery, …), and `description`
- **Relationships** — typed triples such as `Marie Curie COLLABORATED_WITH Pierre Curie` with an evidence snippet and confidence score

The extraction prompt lives in [`src/prompts/findentites.md`](src/prompts/findentites.md) and enforces a strict set of relationship types to keep the graph consistent.

### Knowledge Graph (Notebook 3)

Entities become `(:Entity)` nodes in Neo4j. Each relationship becomes a typed edge, e.g.:

```cypher
(marie_curie)-[:COLLABORATED_WITH]->(pierre_curie)
(albert_einstein)-[:INFLUENCED]->(niels_bohr)
```

### Vector Store (Notebook 4)

Each article is split into overlapping ~1 500-character chunks. Every chunk is embedded with `text-embedding-3-small` (1 536 dimensions) and stored in Qdrant with metadata (title, chunk index, raw text).

### GraphRAG Query Pipeline (Notebook 5)

For each question the notebook:
1. **Extracts entities** from the question with GPT-4o
2. **Vector search** — embeds the question and retrieves the top-5 most similar chunks from Qdrant
3. **Graph search** — looks up extracted entities (plus article titles from vector results) in Neo4j, fetching 1-hop and 2-hop relationship chains
4. **Answers** — calls GPT-4o three times (vector-only context, graph-only context, hybrid context) so you can compare the results

---

## Example Questions to Try

```python
questions = [
    "Which scientists worked on nuclear fission and what were their contributions?",
    "How are Marie Curie and Niels Bohr connected through their work?",
    "Who influenced Alan Turing's thinking in computer science?",
    "Which female scientists made breakthroughs in genetics?",
    "What discoveries connect the work of Archimedes and Isaac Newton?",
]
```

---

## License

This project is released for educational purposes. Wikipedia content is used under the [Creative Commons Attribution-ShareAlike 4.0 License](https://creativecommons.org/licenses/by-sa/4.0/).
