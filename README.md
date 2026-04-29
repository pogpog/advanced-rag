# Advanced RAG EXamples

## Agentic Knowledge Graph RAG

Agentic RAG pipeline combining Knowledge Graph (Neo4j) with vector search, orchestrated by LangGraph.

### Setup

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

### Requirements

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

### How Ingestion Works (Two-Pass Schema Discovery)

The pipeline uses a **schema-free two-pass** approach so no domain-specific node/relationship configuration is needed. It works with any document topic automatically:

**Pass 1 — Schema Discovery** runs `LLMGraphTransformer` on a small sample of chunks (`DISCOVERY_SAMPLE`, default 8) with no constraints, letting the LLM freely infer entity and relationship types. All raw types are collected.

**Schema Normalization** feeds the raw types into an LLM which clusters synonyms and near-duplicates (e.g. "Researcher"/"Scientist" → "Person") into a clean canonical schema of 5–15 node types and 5–15 relationship types.

**Pass 2 — Full Ingestion** runs `LLMGraphTransformer` on all chunks (`INGEST_LIMIT`, default 30) using the auto-discovered schema. This ensures consistent labels so Cypher-based graph queries work reliably.

This means you can swap in any documents (PDFs, wikis, CSVs, etc.) and the graph schema adapts automatically — no manual configuration.

**DISCOVERY_SAMPLE** — Number of chunks used in Pass 1 to discover the schema.

- Too low (e.g. 2-3): The sample may not represent the full range of entity/relationship types in your corpus, so the discovered schema will be incomplete. Unusual entity types won't get a canonical label and will be missed or mislabeled during full ingestion.
- Too high (e.g. 30+): Diminishing returns — you'll discover a stable schema after ~8-10 chunks for most corpora. Going higher just adds unnecessary LLM cost since Pass 1 re-processes chunks that Pass 2 will process again.
- Sweet spot: 5-12 chunks typically captures the vocabulary of a diverse corpus. Increase if your documents span many distinct domains.

**INGEST_LIMIT** — Number of chunks processed in Pass 2 (the actual graph build).

- Lower (e.g. 10): Faster, cheaper ingestion but a sparser graph — fewer entities and relationships, so graph-based retrieval returns less context. Good for testing.
- Higher (e.g. len(doc_splits)): Fuller graph with more entities and cross-references, improving graph retrieval quality. Costs more (one LLM call per chunk) and takes longer.
- Trade-off: This is purely a budget/time vs. completeness knob. For production you'd set it to len(doc_splits) to ingest everything.

In short: **DISCOVERY_SAMPLE** controls schema quality, **INGEST_LIMIT** controls graph completeness.

### Running the Notebook

1. Start Neo4j (see notebook section 3)
2. Copy `.env.example` to `.env` and fill in your keys
3. Open `agentic_graphrag.ipynb` and select the kernel at `.venv/bin/python`

## Future Work

### Adding New Documents

#### The Core Problem

The schema discovered in Pass 1 is frozen at that point. When new documents arrive with novel entity types (e.g. original corpus was AI/ML papers, new docs are legal contracts), LLMGraphTransformer in Pass 2 will either:

- Force novel entities into the closest existing label (e.g. "Judge" → "Person") — loss of specificity
- Drop entities that don't match any allowed_nodes entry at all

#### Does Everything Need Regeneration?

**Vectors** — No. The vector index is independent of the graph schema. You can incrementally add new documents via vector_index.add_documents(new_chunks) without touching existing embeddings.

**Graph** — Partially. Existing nodes/relationships are fine, but you have three options for the new documents:

| Strategy                                                        | Pros                                     | Cons                                                                                     |
| :-------------------------------------------------------------- | :--------------------------------------- | :--------------------------------------------------------------------------------------- |
| Re-run both passes on everything                                | Clean, consistent schema                 | Expensive — reprocesses all old chunks                                                   |
| Re-run Pass 1 on mixed sample (old+new), ingest only new chunks | Cheaper — avoids reprocessing old chunks | Old nodes keep original labels; same entity may get different labels in old vs. new data |
| Ingest new docs without constraints                             | Cheapest                                 | Inconsistent schema; Cypher QA degrades                                                  |

#### The Subtle Problem: Entity Matching

Even if the schema is fine, the graph doesn't auto-merge entities across ingestion runs. If chunk 1 creates a node "Chain-of-Thought" and a new document creates "Chain of thought prompting", you get duplicate nodes. This already happens within a single run but gets worse across incremental runs — the entity disambiguation problem mentioned in the notebook's "next steps" section.

#### Practical Recommendation

For incremental updates, the most practical approach is:

1. Re-run Pass 1 on a sample that mixes old + new documents → produces an expanded schema
2. Ingest only the new chunks with the expanded allowed_nodes/allowed_relationships
3. Run entity resolution afterward to merge duplicates (e.g. fuzzy-match node names)

This avoids reprocessing old chunks while keeping the schema reasonably consistent. For a full production system, you'd want a persistent entity registry that normalizes names before writing to Neo4j — but that's a separate enhancement beyond the two-pass schema discovery.
