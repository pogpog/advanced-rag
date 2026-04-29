# Blitz 2026 - GraphRAG

Agentic RAG pipeline combining Knowledge Graph (Neo4j) with vector search, orchestrated by LangGraph.

## Setup

```bash
# Install all dependencies (creates .venv automatically)
uv sync

# Add a new package
uv add <package-name>

# Add a dev-only package (e.g. testing, notebooks)
uv add --dev <package-name>

# Remove a package
uv remove <package-name>
```

Packages are tracked in `pyproject.toml` — no `requirements.txt` needed.

## Requirements

- **AWS credentials** with access to Amazon Bedrock (used for Nova models and embeddings)
  - `us.amazon.nova-pro-v1:0` — graph entity extraction
  - `us.amazon.nova-lite-v1:0` — agent routing, grading, and generation
  - `amazon.nova-2-multimodal-embeddings-v1:0` — document embeddings
- **Neo4j** — knowledge graph store (local Docker or cloud instance at [https://console.neo4j.io](https://console.neo4j.io)
  - `NEO4J_URI` (default `bolt://localhost:7687`)
  - `NEO4J_USERNAME` (default `neo4j`)
  - `NEO4J_PASSWORD`

Set these in a `.env` file in the project root:

```text
AWS_REGION=us-east-1
NEO4J_URI=bolt://localhost:7687
NEO4J_USERNAME=neo4j
NEO4J_PASSWORD=your-password
```

## Running the Notebook

1. Start Neo4j (see notebook section 3)
2. Copy `.env.example` to `.env` and fill in your keys
3. Open `agentic_graphrag.ipynb` and select the kernel at `.venv/bin/python`
