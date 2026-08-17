# Agent Orchestration Guide — Advanced RAG Architectures

> **Tier 1 — Project 3** | 10 working days | 5 Herdr agents
> Companion to `IMPLEMENTATION_PLAN.md` — read it before starting any phase.
> Prerequisites: Project 1 (Advanced RAG) and Project 2 (Secure RAG) complete.

---

## 1. Agent Roster

| Agent | Kind | Responsibility | Owns (files / dirs) |
|-------|------|----------------|---------------------|
| **core** | codex | Shared ingestion (loader, chunker, text + CLIP embedder, image extractor, ingest pipeline), retrieval (vector, multimodal, retrieval_result), generation (llm_client, prompt_templates), evaluation (metrics, benchmark_runner, golden_dataset), framework (decision_engine, query_analyzer, comparison_tables) | `src/ingestion/`, `src/retrieval/`, `src/generation/`, `src/evaluation/`, `src/framework/`, `scripts/ingest.py`, `scripts/run_benchmark.py`, `scripts/create_golden.py`, `data/raw/`, `data/golden/`, `tests/test_chunker.py`, `tests/test_text_embedder.py`, `tests/test_clip_embedder.py`, `tests/test_image_extractor.py`, `tests/test_multimodal_retriever.py`, `tests/test_vector_retriever.py`, `tests/test_decision_engine.py`, `tests/test_metrics.py`, `tests/test_benchmark_runner.py` |
| **graph** | codex | GraphRAG subsystem: Neo4j client, LLM entity/relationship extraction, graph builder, Leiden community detection + summarization, graph retriever (entity linking + Cypher traversal), Neo4j Cypher scripts | `src/graph/`, `sql/neo4j/`, `scripts/build_graph.py`, `scripts/run_communities.py`, `tests/test_entity_extractor.py`, `tests/test_graph_builder.py`, `tests/test_community_detector.py`, `tests/test_graph_retriever.py` |
| **agentic** | codex | Agentic RAG subsystem: LLM planner, iterative retrieval loop (max 5 iterations), Self-RAG reflection token parser, retrieval trace dataclass | `src/agentic/`, `tests/test_agentic_loop.py`, `tests/test_self_rag.py` |
| **infra** | codex | Docker Compose (postgres+pgvector, neo4j+GDS, api, dashboard), Dockerfile, pyproject.toml, .env.example, PostgreSQL schema, Pydantic config, FastAPI app (routes, schemas, main), Streamlit dashboard (app + 6 pages + components), init scripts, README, shared test fixtures | `docker-compose.yml`, `Dockerfile`, `pyproject.toml`, `.env.example`, `sql/schema.sql`, `src/config.py`, `src/api/`, `src/dashboard/`, `scripts/init_db.py`, `scripts/init_neo4j.py`, `tests/conftest.py`, `tests/test_api.py`, `README.md` |
| **reviewer** | codex | Read-only quality gate after each phase: verifies deliverables match plan specs, checks file-boundary violations, runs verification commands from §6, approves or requests changes | *(no file ownership — read-only across entire repo)* |

---

## 2. Pane Layout

```
┌─────────────────────────────────────────────────────────┐
│                       Pane 0                             │
│                 Main / reviewer                          │
│            (orchestration + review gate)                 │
├──────────────────────────┬──────────────────────────────┤
│        Pane 1            │         Pane 2               │
│        core              │         graph                │
│  ingestion · retrieval   │  neo4j · entity extract      │
│  generation · eval       │  graph build · communities   │
│  framework               │  graph retrieval             │
├──────────────────────────┼──────────────────────────────┤
│        Pane 3            │         Pane 4               │
│       agentic            │         infra                │
│  planner · loop          │  docker · schema · api       │
│  self-rag · trace        │  dashboard · config · readme │
└──────────────────────────┴──────────────────────────────┘
```

---

## 3. Setup Commands

Run these from the project root (`advanced-rag-architectures/`):

```bash
# --- Split panes (Pane 0 already exists as the main pane) ---

# Pane 1 — core agent (top-left)
herdr pane split --cwd "$PWD" --no-focus

# Pane 2 — graph agent (top-right)
herdr pane split --cwd "$PWD" --no-focus

# Pane 3 — agentic agent (bottom-left)
herdr pane split --cwd "$PWD" --no-focus

# Pane 4 — infra agent (bottom-right)
herdr pane split --cwd "$PWD" --no-focus

# --- Start agents ---

herdr agent start core     --kind codex --pane 1
herdr agent start graph    --kind codex --pane 2
herdr agent start agentic  --kind codex --pane 3
herdr agent start infra    --kind codex --pane 4
herdr agent start reviewer --kind codex --pane 0
```

> **Note**: Pane IDs are assigned in creation order. Verify with `herdr pane list` before starting agents. Adjust `--pane` values if your layout differs.
> **Source of truth**: `IMPLEMENTATION_PLAN.md`. Every phase prompt references its section numbers. Do not improvise schemas, prompts, or Cypher — copy them from the plan.

---

## 4. Phase-by-Phase Orchestration

### Phase 1: Environment Setup — Neo4j + Postgres + Docker Compose (Day 1)

**Goal**: Docker Compose running with Neo4j, Postgres, API, Dashboard; both database schemas deployed.

#### Parallel Work

**Agent infra**:
```
You are the infra agent for the Advanced RAG Architectures project. Read IMPLEMENTATION_PLAN.md Sections 3, 4, 5, and 13 (Deployment) before starting.

Your task for Phase 1 — Environment Setup:

1. Create `docker-compose.yml` from Section 13. It must define FOUR services: `postgres` (pgvector/pgvector:pg16), `neo4j` (neo4j:5.11-community, with GDS plugin env NEO4J_PLUGINS=["apoc","graph-data-science"]), `api` (build from Dockerfile), `dashboard` (Streamlit). Include volumes `postgres_data`, `neo4j_data`, `model_cache` and the `rag-network` bridge network. Mount `./sql/schema.sql` read-only into postgres initdb.d. Expose postgres:5432, neo4j:7474+7687, api:8000, dashboard:8501.

2. Create `Dockerfile` from Section 13 — `python:3.11-slim` base, install build-essential, libpq-dev, libglib2.0, libsm6, libxext6, libxrender1 (for Pillow/PyMuPDF), copy pyproject.toml, pip install `.[api,dashboard,dev]`, download spaCy `en_core_web_sm`, expose 8000, CMD runs uvicorn.

3. Create `pyproject.toml` from Section 13 with all dependencies (fastapi, uvicorn, pydantic, pydantic-settings, asyncpg, pgvector, sentence-transformers, spacy, httpx, numpy, neo4j, transformers, torch, Pillow, PyMuPDF, streamlit, streamlit-agraph, plotly, langchain) and optional-dependencies groups (api, dashboard, dev). Include `[tool.pytest.ini_options]` with `asyncio_mode = "auto"` and `testpaths = ["tests"]`, plus ruff and mypy config blocks.

4. Create `.env.example` from Section 13 with all env vars: DATABASE_URL, TEST_DATABASE_URL, NEO4J_URI, NEO4J_USER, NEO4J_PASSWORD, LLM_PROVIDER, OPENAI_API_KEY, OPENAI_MODEL, OPENAI_BASE_URL, OLLAMA_BASE_URL, OLLAMA_MODEL, EMBEDDING_MODEL, CLIP_MODEL, DEVICE, retrieval defaults (top_k, max_iterations), API_HOST, API_PORT, CORS_ORIGINS.

5. Create `sql/schema.sql` from Section 4.1 — full PostgreSQL DDL: CREATE EXTENSION uuid-ossp, vector, pg_trgm; CREATE TABLE documents, chunks (tsvector GENERATED column, vector(1024) text_embedding, partial HNSW indexes), images (vector(512) clip_embedding, vector(1024) caption_embedding, HNSW indexes), queries, golden_questions, eval_results; all indexes from the plan.

6. Create `sql/neo4j/constraints.cypher` from Section 4.2 — Neo4j constraints + indexes: uniqueness on Entity(name,type), Document(id), Chunk(id), Community(id); fulltext index on Entity.name + Entity.description; range indexes on Entity.type, Entity.mention_count.

7. Create `src/config.py` — Pydantic Settings (pydantic-settings BaseSettings) loading all env vars from `.env.example`. Fields: database_url, test_database_url, neo4j_uri, neo4j_user, neo4j_password, llm_provider, openai_*, ollama_*, embedding_model, clip_model, device, top_k, max_iterations, api_host, api_port, cors_origins.

8. Create `scripts/init_db.py` — async script using asyncpg that reads `sql/schema.sql` and executes it. Print "Schema created successfully".

9. Create `scripts/init_neo4j.py` — script using neo4j driver that reads `sql/neo4j/constraints.cypher` and executes each statement. Print "Neo4j constraints and indexes created successfully".

10. Create `src/__init__.py` and all package `__init__.py` files for: src/ingestion/, src/graph/, src/retrieval/, src/agentic/, src/generation/, src/framework/, src/evaluation/, src/api/, src/dashboard/, src/dashboard/pages/, src/dashboard/components/.

Verification (run and confirm output):
- `docker compose up -d postgres neo4j`
- `docker compose exec postgres psql -U rag -d ragdb -c "SELECT extname FROM pg_extension;"` → vector, uuid-ossp, pg_trgm
- `docker compose exec neo4j cypher-shell -u neo4j -p testpassword "SHOW CONSTRAINTS;"` → entity/document/chunk/community constraints
- `python scripts/init_db.py` → "Schema created successfully"
- `python scripts/init_neo4j.py` → "Neo4j constraints and indexes created successfully"
- `docker compose exec postgres psql -U rag -d ragdb -c "\dt"` → documents, chunks, images, queries, golden_questions, eval_results

Do NOT touch any files outside your ownership. Do NOT create src/graph/*, src/agentic/*, src/ingestion/*, src/retrieval/* — those belong to other agents.
```

#### Sequential (after parallel)

*(none — Phase 1 is infra-only)*

#### Review

**reviewer**:
```
Review Phase 1 deliverables. Verify:
1. `docker compose up -d postgres neo4j` starts both services; `pg_isready` passes; Neo4j Browser reachable on :7474.
2. `python scripts/init_db.py` prints "Schema created successfully".
3. `python scripts/init_neo4j.py` prints "Neo4j constraints and indexes created successfully".
4. All 6 tables exist: documents, chunks, images, queries, golden_questions, eval_results.
5. Extensions enabled: vector, uuid-ossp, pg_trgm.
6. `sql/schema.sql` matches Section 4.1 — chunks has vector(1024) text_embedding, images has vector(512) clip_embedding + vector(1024) caption_embedding, HNSW indexes present.
7. Neo4j constraints: uniqueness on Entity(name,type), fulltext index on Entity.name+description.
8. `src/config.py` loads all env vars via pydantic-settings BaseSettings.
9. `pyproject.toml` has neo4j, transformers, torch, Pillow, PyMuPDF, streamlit, streamlit-agraph, plotly, langchain.
10. No files outside infra's ownership were created.
Report PASS or list specific issues to fix.
```

#### Commit

```bash
git add -A && git commit -m "Phase 1: environment, Docker Compose (postgres+neo4j), schemas, config"
```

---

### Phase 2: GraphRAG — Entity/Relationship Extraction & Graph Construction (Day 2–3)

**Goal**: Documents ingested, entities/relationships extracted via LLM, Neo4j graph populated.

#### Parallel Work

**Agent core** (ingestion — no conflict with graph):
```
You are the core agent. Read IMPLEMENTATION_PLAN.md Sections 7.1, 7.2, 7.8 (CLIP encode_text only — image methods come in Phase 5), and Phase 2 before starting.

Your task for Phase 2 — Shared ingestion:

1. Create `src/ingestion/loader.py` from Section 7.1 — document loaders for .txt, .md, .pdf. Return Document dataclass instances (source_path, title, doc_type, content, content_hash SHA-256, char_count, metadata). Deduplicate by content_hash. For .pdf use PyMuPDF (fitz) to extract text.

2. Create `src/ingestion/chunker.py` — SemanticChunker reused from Project 1: spaCy en_core_web_sm, groups sentences until ~1000 chars, never breaks a sentence. Strategy label: "semantic_spacy". Chunk dataclass: content, chunk_index, chunking_strategy, token_count, metadata.

3. Create `src/ingestion/text_embedder.py` — BGE text embedder reused from P1: BAAI/bge-large-en-v1.5, 1024-dim, embed_passages (no instruction prefix) + embed_query (with instruction prefix), normalize_embeddings=True.

4. Create `src/generation/llm_client.py` from Section 7 — LLM client supporting OpenAI (gpt-4o-mini) and Ollama (llama3.1:8b) via httpx async. Methods: async generate(prompt, temperature), async generate_batch(prompts). Load config from src/config.py.

5. Create `src/generation/prompt_templates.py` — initial templates for vanilla + graph RAG answer generation (multimodal/agentic templates added in later phases).

6. Create `scripts/ingest.py` — CLI: --source (dir), --extract-images (flag, default False for Phase 2). Load → chunk → embed text (BGE) → store documents + chunks in Postgres. Print "Ingested N documents, M chunks".

7. Prepare sample corpus in `data/raw/`: `data/raw/papers/` (3-5 short AI/ML .txt/.pdf with entity-rich content), `data/raw/tech_articles/` (2-3 tech articles), `data/raw/company_profiles/` (2-3 company descriptions). Ensure content mentions entities (people, orgs, technologies) and relationships.

8. Create `tests/test_chunker.py` and `tests/test_text_embedder.py` — test chunking produces semantic_spacy chunks, embedder returns 1024-dim normalized vectors (mock SentenceTransformer).

Verification:
- `python scripts/ingest.py --source data/raw/` → documents + chunks in Postgres
- `docker compose exec postgres psql -U rag -d ragdb -c "SELECT COUNT(*) FROM chunks;"` → >50 chunks
- `pytest tests/test_chunker.py tests/test_text_embedder.py -v` → all pass

Do NOT create src/graph/* (graph agent) or src/agentic/* (agentic agent). Do NOT touch infra files.
```

**Agent graph** (parallel — no file conflict with core):
```
You are the graph agent. Read IMPLEMENTATION_PLAN.md Sections 7.3, 7.4, and Phase 2 before starting.

Your task for Phase 2 — GraphRAG extraction & construction:

1. Create `src/graph/neo4j_client.py` from Section 7.3 — Neo4j async-capable driver wrapper. Neo4jClient(uri, user, password) with connection pooling. Methods: async execute_query(cypher, params), async execute_write(cypher, params), async close(). Load credentials from src/config.py.

2. Create `src/graph/entity_extractor.py` from Section 7.4 — LLM-based entity + relationship extraction. EntityExtraction dataclass: entities (list[Entity]), relationships (list[Relationship]). Entity: name, type, description. Relationship: source, target, type, description. Entity types: PERSON, ORG, TECH, CONCEPT, LOCATION, EVENT, PRODUCT. Relationship types: DEVELOPED_BY, WORKS_FOR, COMPETES_WITH, PART_OF, USES, RELATED_TO, FOUNDED_BY, LOCATED_IN. Prompt template: given a chunk, extract entities + relationships as JSON. Batch extraction across chunks; merge entities with same name+type (concatenate descriptions). Use the LLM client from src/generation/llm_client.py.

3. Create `src/graph/graph_builder.py` from Section 7.3 — build Neo4j graph from extracted entities. Create Document, Chunk, Entity nodes; MENTIONS (Chunk→Entity), RELATED_TO (Entity→Entity), PART_OF (Chunk→Document), EXTRACTED_FROM (Entity→Chunk) relationships. Merge duplicate entities (same name+type), increment mention_count. Cypher MERGE patterns to avoid duplicates.

4. Create `scripts/build_graph.py` — CLI: read chunks from Postgres (asyncpg) → LLM entity extraction (batch) → build Neo4j graph. Print "Extracted N entities, M relationships from K chunks".

5. Create `tests/test_entity_extractor.py` — test LLM extraction with mock LLM returning canned JSON. Verify entity/relationship parsing, type validation, merge of duplicates.
6. Create `tests/test_graph_builder.py` — test graph construction with mock Neo4j driver. Verify MERGE Cypher patterns, mention_count increment, relationship creation.

Verification:
- `python scripts/build_graph.py` → entities + relationships in Neo4j
- `docker compose exec neo4j cypher-shell -u neo4j -p testpassword "MATCH (e:Entity) RETURN count(e) AS c;"` → >50 entities
- `docker compose exec neo4j cypher-shell -u neo4j -p testpassword "MATCH ()-[r:RELATED_TO]->() RETURN count(r) AS c;"` → >30 relationships
- `pytest tests/test_entity_extractor.py tests/test_graph_builder.py -v` → all pass

Do NOT touch src/ingestion/*, src/retrieval/*, src/agentic/*, or infra files.
```

#### Sequential (after parallel)

*(none — core ingestion and graph construction are independent; graph_builder reads chunks from Postgres after core's ingest.py has run)*

> **Dependency**: graph agent's `build_graph.py` reads chunks that core's `ingest.py` writes. Run core's ingest first, then graph's build_graph. Coordinate via the reviewer gate.

#### Review

**reviewer**:
```
Review Phase 2 deliverables. Verify:
1. `python scripts/ingest.py --source data/raw/` ingests documents + chunks (semantic_spacy) into Postgres with 1024-dim text_embedding populated.
2. `python scripts/build_graph.py` extracts entities + relationships and builds the Neo4j graph.
3. Neo4j has >50 Entity nodes and >30 RELATED_TO relationships.
4. Entity types restricted to: PERSON, ORG, TECH, CONCEPT, LOCATION, EVENT, PRODUCT.
5. Duplicate entities (same name+type) merged with mention_count incremented.
6. Relationships MENTIONS (Chunk→Entity), RELATED_TO (Entity→Entity), PART_OF (Chunk→Document) all present.
7. `tests/test_entity_extractor.py` and `tests/test_graph_builder.py` pass.
8. No file boundary violations — core did not touch graph/agentic/infra; graph did not touch ingestion/retrieval/agentic/infra.
Report PASS or list specific issues.
```

#### Commit

```bash
git add -A && git commit -m "Phase 2: document ingestion, LLM entity extraction, Neo4j graph construction"
```

---

### Phase 3: GraphRAG Retrieval & Benchmarking vs Vanilla Vector RAG (Day 4)

**Goal**: Graph retrieval operational, benchmarked against vanilla vector RAG on the same corpus.

#### Parallel Work

**Agent core** (retrieval + eval — no conflict with graph):
```
You are the core agent. Read IMPLEMENTATION_PLAN.md Sections 7.6 (graph retriever interface), Phase 3, and Section 9 (Comparison & Evaluation) before starting.

Your task for Phase 3 — Vanilla retrieval + evaluation harness:

1. Create `src/retrieval/retrieval_result.py` — shared RetrievalResult dataclass: chunk_id, content, score, rank, source ('vector'|'graph'|'multimodal'|'agentic'), metadata. Used by all retrievers.

2. Create `src/retrieval/vector_retriever.py` — vanilla vector RAG reused from P1: pgvector cosine distance (<=>), similarity = 1 - distance, filter by embedding IS NOT NULL, order by distance, return list[RetrievalResult]. VectorRetriever(pool, embedder).

3. Create `src/evaluation/metrics.py` — RetrievalMetrics from P1: recall_at_k, precision_at_k, mrr, ndcg_at_k (DCG = sum((2^grade-1)/log2(i+1))). All @staticmethod.

4. Create `src/evaluation/golden_dataset.py` — load/save golden questions from data/golden/golden_questions.json. GoldenQuestion: id, question, expected_answer, relevant_chunk_ids, query_type (factual|multi_hop|comparative|multimodal), expected_best_architecture.

5. Create `scripts/create_golden.py` — CLI to create/manage golden questions. Seed `data/golden/golden_questions.json` with 15-20 questions: 5 factual (vanilla should win), 5 multi-hop reasoning (graph should win), 5 comparative (graph should win). Relevant_chunk_ids from the ingested corpus.

6. Create `src/evaluation/benchmark_runner.py` — run a given architecture on golden questions, compute recall@5, MRR, nDCG, answer latency. BenchmarkResult dataclass per question + aggregate.

7. Create `scripts/run_benchmark.py` — CLI: --architectures vanilla graph. Run both on golden questions, compute metrics, save to data/benchmarks/benchmark_results.json. Print comparison table.

8. Create `tests/test_vector_retriever.py` (mock pool), `tests/test_metrics.py` (recall/precision/mrr/ndcg cases from P1), `tests/test_benchmark_runner.py` (mock retrievers).

Verification:
- `python scripts/create_golden.py` → 15-20 golden questions in data/golden/
- `pytest tests/test_vector_retriever.py tests/test_metrics.py tests/test_benchmark_runner.py -v` → all pass

Do NOT create src/graph/graph_retriever.py (graph agent). Do NOT touch agentic/infra files.
```

**Agent graph** (parallel — no conflict with core):
```
You are the graph agent. Read IMPLEMENTATION_PLAN.md Section 7.6 and Phase 3 before starting.

Your task for Phase 3 — Graph retrieval:

1. Create `src/graph/graph_retriever.py` from Section 7.6 — graph-based retrieval. GraphRetriever(neo4j_client, pool). async retrieve(query, k=10) → GraphRetrievalResult:
   - Entity linking: Neo4j fulltext search on Entity.name+description to find entities mentioned in the query.
   - Graph traversal: Cypher query to find related entities within N hops (default 2) via RELATED_TO.
   - Community lookup: get Community.summary for linked entities.
   - Subgraph context: traverse MENTIONS to retrieve source chunks (join chunk content from Postgres by chunk_id).
   - Return: entity names, community summaries, related chunks (as RetrievalResult list), subgraph context string.

2. Create `sql/neo4j/example_queries.cypher` — documented example Cypher queries: entity linking fulltext, N-hop traversal, community lookup, subgraph chunk retrieval. These serve as reference + dashboard examples.

3. Create `tests/test_graph_retriever.py` — test with mock Neo4j driver + mock pool. Verify entity linking query, traversal hop count, community summary inclusion, chunk join.

Verification:
- `pytest tests/test_graph_retriever.py -v` → all pass
- Graph retriever returns entity names + community summaries + related chunks for a test query.

Do NOT touch src/retrieval/*, src/evaluation/*, src/agentic/*, or infra files.
```

#### Sequential (after parallel)

Run the benchmark once both retrievers are ready:
```bash
python scripts/run_benchmark.py --architectures vanilla graph
```

#### Review

**reviewer**:
```
Review Phase 3 deliverables. Verify:
1. `src/retrieval/vector_retriever.py` uses pgvector cosine (<=>), returns RetrievalResult list.
2. `src/graph/graph_retriever.py` does entity linking (fulltext) → N-hop traversal → community lookup → chunk join. Returns entity names + community summaries + chunks.
3. `data/golden/golden_questions.json` has 15-20 questions across factual/multi_hop/comparative with relevant_chunk_ids + expected_best_architecture.
4. `src/evaluation/metrics.py` implements recall@k, precision@k, mrr, ndcg@k with the (2^grade-1)/log2(i+1) nDCG formula.
5. `python scripts/run_benchmark.py --architectures vanilla graph` produces results showing graph RAG outperforms vanilla on multi-hop questions (higher recall@5 or MRR) and vanilla wins on simple factual (lower latency).
6. `tests/test_graph_retriever.py`, `tests/test_vector_retriever.py`, `tests/test_metrics.py`, `tests/test_benchmark_runner.py` all pass.
7. No file boundary violations.
Report PASS or list specific issues.
```

#### Commit

```bash
git add -A && git commit -m "Phase 3: graph retrieval, vanilla vector RAG, benchmark comparison"
```

---

### Phase 4: Agentic RAG — Iterative Retrieval Loop (Day 5–6)

**Goal**: LLM-driven iterative retrieval with Self-RAG reflection tokens, deciding what to retrieve and when to stop.

#### Parallel Work

**Agent agentic**:
```
You are the agentic agent. Read IMPLEMENTATION_PLAN.md Sections 7.7 (retrieval_loop.py + self_rag.py) and Phase 4 before starting.

Your task for Phase 4 — Agentic RAG:

1. Create `src/agentic/self_rag.py` from Section 7.7 — Self-RAG reflection token parser. ReflectionTokens dataclass: retrieve (bool), no_retrieve (bool), relevant (bool), irrelevant (bool), generate (bool), no_generate (bool), reasoning (str), raw_tokens (str). parse(llm_output) → ReflectionTokens. Token state machine: RETRIEVE → retrieve → RELEVANT/IRRELEVANT → GENERATE/NO_GENERATE. Fallback: if no tokens, heuristic (always retrieve on first iteration, assess on subsequent).

2. Create `src/agentic/planner.py` — LLM planner that decides retrieval strategy. Input: user query + current context (empty on first iteration). Output: action plan with reflection tokens + sub_query + architecture choice (vector|graph). Uses LLM client from src/generation/llm_client.py. PlannerPrompt produces [RETRIEVE]/[NO RETRIEVE] + sub-query + architecture hint.

3. Create `src/agentic/trace.py` — retrieval trace dataclass. IterationTrace: iteration, sub_query, architecture_used, tokens, retrieved_chunk_ids, retrieved_count, is_relevant, reasoning, latency_ms. RetrievalTrace: iterations (list[IterationTrace]), final_answer, total_iterations, total_chunks_retrieved, total_latency_ms, terminated_reason ('generate'|'no_new_chunks'|'max_iterations').

4. Create `src/agentic/retrieval_loop.py` from Section 7.7 — iterative retrieval loop. AgenticRetriever(llm, vector_retriever, graph_retriever=None, max_iterations=5, top_k=10). async retrieve(query) → (answer, RetrievalTrace). Loop: plan → (GENERATE? → answer) → retrieve (vector or graph based on plan) → dedup → assess → (no new relevant + iteration>1 → answer) → loop. Force-generate on max_iterations. Accumulate chunks across iterations; dedup by chunk_id.

5. Create `tests/test_self_rag.py` — parse [RETRIEVE], [RELEVANT]/[IRRELEVANT], [GENERATE]/[NO GENERATE]; fallback when no tokens.
6. Create `tests/test_agentic_loop.py` — mock LLM: loop terminates on [GENERATE]; continues on [NO GENERATE]; max_iterations enforced; context accumulates; dedup works; terminated_reason correct.

Verification:
- `pytest tests/test_self_rag.py tests/test_agentic_loop.py -v` → all pass
- Agentic loop shows 2-4 iterations for a complex multi-hop query (mock LLM), 1 for simple.

Do NOT touch src/graph/*, src/retrieval/*, src/ingestion/*, src/evaluation/*, or infra files. You MAY add agentic prompt templates to src/generation/prompt_templates.py (coordinate with core — add a clearly delimited "Agentic" section).
```

**Agent core** (parallel — add agentic prompt templates):
```
You are the core agent. Read IMPLEMENTATION_PLAN.md Section 7.7 and Phase 4 before starting.

Your task for Phase 4 — add agentic prompt templates (small, no conflict with agentic agent's src/agentic/* files):

1. Append to `src/generation/prompt_templates.py` a clearly delimited "=== AGENTIC RAG ===" section with three templates:
   - PLANNER_PROMPT: "Given the query and current context, decide whether to retrieve more. Output [RETRIEVE] or [NO RETRIEVE], a sub-query, and architecture hint (vector|graph)."
   - ASSESSOR_PROMPT: "Given retrieved chunks and query, assess if context is sufficient. Output [RELEVANT]/[IRRELEVANT] and [GENERATE]/[NO GENERATE] with reasoning."
   - GENERATOR_PROMPT: "Given the query and all retrieved context, generate a final answer."
   Match the token formats the agentic agent's self_rag.py parser expects.

Do NOT modify any other files. Do NOT touch src/agentic/*.
```

#### Sequential (after parallel)

*(none — agentic agent's loop uses vector_retriever + graph_retriever from Phase 3, already merged)*

#### Review

**reviewer**:
```
Review Phase 4 deliverables. Verify:
1. `src/agentic/self_rag.py` parses all 6 reflection tokens + fallback heuristic.
2. `src/agentic/retrieval_loop.py` implements plan → retrieve → dedup → assess → generate/loop with max_iterations=5, three termination conditions (generate, no_new_chunks, max_iterations).
3. Context accumulates across iterations; chunks deduplicated by chunk_id.
4. RetrievalTrace records per-iteration sub_query, architecture, tokens, retrieved chunks, latency + final terminated_reason.
5. `src/generation/prompt_templates.py` has the Agentic section with PLANNER/ASSESSOR/GENERATOR prompts matching token formats.
6. `tests/test_self_rag.py` and `tests/test_agentic_loop.py` pass — including max_iterations enforcement and dedup.
7. No file boundary violations — agentic agent only added the prompt_templates section via core; no other cross-ownership edits.
Report PASS or list specific issues.
```

#### Commit

```bash
git add -A && git commit -m "Phase 4: agentic RAG iterative loop, Self-RAG reflection tokens, trace"
```

---

### Phase 5: Multimodal RAG — CLIP Embeddings & Cross-Modal Retrieval (Day 7)

**Goal**: Images extracted from PDFs, CLIP embeddings generated, cross-modal retrieval (text→image, image→text) operational.

#### Parallel Work

**Agent core**:
```
You are the core agent. Read IMPLEMENTATION_PLAN.md Sections 7.8 (CLIP embedder), Phase 5, and the images table schema (Section 4.1) before starting.

Your task for Phase 5 — Multimodal RAG:

1. Complete `src/ingestion/clip_embedder.py` from Section 7.8 — CLIPEmbedder(model_name="openai/clip-vit-base-patch32", device). DIMENSION=512. Methods: encode_image(image_path) → 512-dim normalized np.ndarray; encode_images(image_paths, batch_size=32) → (N,512) normalized; encode_text(text) → 512-dim normalized; encode_texts(texts) → (N,512) normalized. Use transformers CLIPModel + CLIPProcessor, torch.no_grad(), L2 normalize.

2. Create `src/ingestion/image_extractor.py` — extract images from PDFs using PyMuPDF (fitz). For each PDF: extract embedded images (JPEG/PNG) via page.get_images(); render full pages as images (page.get_pixmap()) for page-level multimodal; classify image type (image|chart|table|diagram|figure) via simple heuristics (size/aspect ratio) or LLM; extract surrounding text context (text blocks near the image bbox); save images to data/images/ with stable filenames. Return ImageRecord dataclass: image_id, source_doc_id, page_number, image_type, image_path, caption, surrounding_text.

3. Create `src/ingestion/ingest_pipeline.py` — full ingestion orchestrator: load documents → extract text + images → chunk text → embed text (BGE 1024-dim) + images (CLIP 512-dim) → generate image captions via LLM → embed captions (BGE 1024-dim for text-based image search) → store documents, chunks, images in Postgres. Async, batched.

4. Update `scripts/ingest.py` — add --extract-images flag (default True in Phase 5). When set, run image extraction + CLIP embedding + captioning, store in images table (clip_embedding vector(512), caption_embedding vector(1024), image_type, caption, surrounding_text, source_doc_id, page_number, image_path).

5. Create `src/retrieval/multimodal_retriever.py` — cross-modal retrieval. MultimodalRetriever(pool, clip_embedder, text_embedder). async search_by_text(query_text, k=10): CLIP-encode query → search images by clip_embedding (<=>) + search chunks by text_embedding → fuse (weighted). async search_by_image(query_image_path, k=10): CLIP-encode image → search images by clip_embedding + search chunks by caption_embedding → fuse. Return list[RetrievalResult] with source='multimodal' + image refs.

6. Create `tests/test_clip_embedder.py` — encode_image/encode_text return 512-dim normalized (unit norm) vectors; image+text of same concept have high cosine similarity (sanity, mock model).
7. Create `tests/test_image_extractor.py` — extracts images from a sample PDF, classifies types, extracts surrounding text.
8. Create `tests/test_multimodal_retriever.py` — text query returns relevant images; image query returns relevant text; fusion combines results (mock pool).

Verification:
- `python scripts/ingest.py --source data/raw/ --extract-images` → images in Postgres
- `docker compose exec postgres psql -U rag -d ragdb -c "SELECT COUNT(*) FROM images;"` → >10 images
- `docker compose exec postgres psql -U rag -d ragdb -c "SELECT image_type, COUNT(*) FROM images GROUP BY 1;"` → multiple types
- `pytest tests/test_clip_embedder.py tests/test_image_extractor.py tests/test_multimodal_retriever.py -v` → all pass

Do NOT touch src/graph/*, src/agentic/*, or infra files.
```

#### Sequential (after parallel)

*(none)*

#### Review

**reviewer**:
```
Review Phase 5 deliverables. Verify:
1. `src/ingestion/clip_embedder.py` — encode_image/encode_text return 512-dim L2-normalized vectors; encode_images/encode_texts batch correctly.
2. `src/ingestion/image_extractor.py` — extracts embedded images + renders pages, classifies type, extracts surrounding text, saves to data/images/.
3. `images` table populated with clip_embedding (vector(512)), caption_embedding (vector(1024)), image_type, caption, surrounding_text.
4. `src/retrieval/multimodal_retriever.py` — search_by_text returns relevant images; search_by_image returns relevant text; fusion is weighted + configurable.
5. `python scripts/ingest.py --source data/raw/ --extract-images` produces >10 images across multiple types.
6. `tests/test_clip_embedder.py`, `tests/test_image_extractor.py`, `tests/test_multimodal_retriever.py` pass.
7. No file boundary violations.
Report PASS or list specific issues.
```

#### Commit

```bash
git add -A && git commit -m "Phase 5: CLIP embeddings, image extraction, cross-modal retrieval"
```

---

### Phase 6: KAG/LightRAG Overview + Comparison Table + Decision Framework (Day 8)

**Goal**: Comprehensive KAG/LightRAG comparison table and interactive RAG architecture decision framework.

#### Parallel Work

**Agent core**:
```
You are the core agent. Read IMPLEMENTATION_PLAN.md Section 8 (RAG Architecture Decision Framework) and Phase 6 before starting.

Your task for Phase 6 — Decision framework + KAG/LightRAG comparison:

1. Create `src/framework/comparison_tables.py` — structured comparison data for 6 systems: Vanilla RAG, GraphRAG, KAG, LightRAG, Agentic RAG, Multimodal RAG. For each: architecture, knowledge representation (vector/structured KB/graph), retrieval mechanism, strengths, weaknesses, use cases, setup complexity (low/med/high), maintenance, typical latency, accuracy on multi-hop/factual/temporal. Provide as a list of Pydantic models / dataclasses + a function get_comparison_table() returning the full matrix.

2. Create `src/framework/query_analyzer.py` — analyze query characteristics. QueryProfile dataclass: is_multimodal (bool), entity_density (float 0-1), relationship_complexity (float 0-1), query_complexity (float 0-1), temporal_aspect (bool), expected_latency_budget ('realtime'|'batch'). analyze(query, corpus_metadata) → QueryProfile. Use NER/keyword heuristics or LLM for entity density; detect multi-hop cues ("how does X relate to Y", "compare", "because", chains).

3. Create `src/framework/decision_engine.py` — RAG architecture decision engine. decide(query_profile, corpus_metadata) → DecisionResult(recommended_architecture, confidence, reasoning, scores per architecture). Decision matrix from Section 8:
   - is_multimodal → Multimodal RAG
   - entity_density high + relationship_complexity high → GraphRAG
   - query_complexity high + iterative reasoning → Agentic RAG
   - structured KB + logical reasoning → KAG
   - simple factual + low latency → Vanilla RAG
   - lightweight graph+vector hybrid → LightRAG
   Confidence from how clearly criteria point to one architecture; scores for all candidates.

4. Create `tests/test_decision_engine.py` — multimodal query → Multimodal RAG; multi-hop + high entity density → GraphRAG; simple factual → Vanilla RAG; complex analytical → Agentic RAG; structured KB → KAG. Verify confidence scores + reasoning.

Verification:
- `pytest tests/test_decision_engine.py -v` → all pass
- Decision engine routes all golden questions to their expected_best_architecture (cross-check with data/golden/golden_questions.json).

Do NOT touch src/graph/*, src/agentic/*, src/ingestion/*, src/retrieval/*, or infra files.
```

#### Sequential (after parallel)

*(none)*

#### Review

**reviewer**:
```
Review Phase 6 deliverables. Verify:
1. `src/framework/comparison_tables.py` covers all 6 systems with the full attribute set (architecture, retrieval mechanism, strengths, weaknesses, use cases, complexity, latency, accuracy).
2. `src/framework/query_analyzer.py` produces a QueryProfile with is_multimodal, entity_density, relationship_complexity, query_complexity, temporal_aspect, latency_budget.
3. `src/framework/decision_engine.py` implements the Section 8 decision matrix and returns recommended_architecture + confidence + per-architecture scores + reasoning.
4. Decision engine routes all golden questions to their expected_best_architecture.
5. `tests/test_decision_engine.py` passes.
6. No file boundary violations.
Report PASS or list specific issues.
```

#### Commit

```bash
git add -A && git commit -m "Phase 6: KAG/LightRAG comparison, decision framework, query analyzer"
```

---

### Phase 7: FastAPI Endpoints + Streamlit Comparison Dashboard (Day 9)

**Goal**: All 4 architectures exposed via FastAPI; interactive Streamlit dashboard for comparison.

#### Parallel Work

**Agent infra**:
```
You are the infra agent. Read IMPLEMENTATION_PLAN.md Sections 10 (API Specification), 13 (Deployment), and Phase 7 before starting.

Your task for Phase 7 — API + Dashboard:

1. Create `src/api/schemas.py` — Pydantic v2 request/response schemas:
   - QueryRequest: query (str), architecture (Literal['vanilla','graph','agentic','multimodal']), config (top_k, max_iterations, etc.)
   - QueryResponse: answer, retrieved_chunks (list), retrieved_images (list), retrieval_trace (dict), latency_ms, architecture
   - CompareRequest: query, architectures (list, default all 4)
   - CompareResponse: results (dict per architecture)
   - DecisionRequest: query
   - DecisionResponse: recommended_architecture, confidence, reasoning, query_profile
   - GraphSubgraphResponse: nodes, edges (for visualization)

2. Create `src/api/routes.py` — FastAPI routes:
   - POST /query — query with architecture selector; dispatch to vector/graph/agentic/multimodal retriever
   - POST /query/compare — run query across all 4 architectures, return side-by-side
   - GET /graph/entities — list entities in Neo4j
   - GET /graph/subgraph?entity=... — subgraph around an entity (for streamlit-agraph)
   - GET /images/{image_id} — serve extracted image file
   - POST /decision — architecture recommendation via decision_engine
   - GET /benchmark — benchmark results from data/benchmarks/
   - GET /health — Postgres + Neo4j connectivity check

3. Create `src/api/main.py` — FastAPI app with CORS (cors_origins from config), lifespan (init db pool + neo4j driver + embedders on startup, close on shutdown), include routes router. Mount at /docs.

4. Create `src/dashboard/app.py` — Streamlit main app with navigation (st.navigation or sidebar radio) to the 6 pages.

5. Create `src/dashboard/pages/1_Comparison.py` — side-by-side architecture comparison: query input, architecture selector, results display, retrieval trace (esp. agentic iterations), latency comparison.
6. Create `src/dashboard/pages/2_GraphRAG_Explorer.py` — Neo4j graph visualization via streamlit-agraph; entity search, subgraph exploration, community summaries.
7. Create `src/dashboard/pages/3_Agentic_Trace.py` — step-by-step iteration display, Self-RAG reflection tokens, sub-query evolution.
8. Create `src/dashboard/pages/4_Multimodal_Gallery.py` — text query → image results grid; image upload → similar images + related text; image metadata.
9. Create `src/dashboard/pages/5_Decision_Framework.py` — query input → query profile display; interactive criteria sliders; architecture recommendation with confidence bars; decision flowchart.
10. Create `src/dashboard/pages/6_KAG_LightRAG.py` — full comparison table from comparison_tables.py; use case examples.
11. Create `src/dashboard/components/graph_viz.py` — streamlit-agraph wrapper (nodes/edges from /graph/subgraph).
12. Create `src/dashboard/components/benchmark_charts.py` — Plotly benchmark charts (recall@5, MRR, latency per architecture).

13. Create `tests/test_api.py` — API integration tests with httpx AsyncClient: POST /query (each architecture), POST /query/compare, POST /decision, GET /health, GET /graph/entities. Use mock retrievers/LLM where needed.

Verification:
- `docker compose up -d` starts all 4 services
- `curl http://localhost:8000/health` → {"status":"healthy","postgres":"connected","neo4j":"connected"}
- `curl -X POST http://localhost:8000/query -d '{"query":"Who developed the Transformer?","architecture":"graph"}'` → answer with graph retrieval
- `curl -X POST http://localhost:8000/query/compare -d '{"query":"Compare GPT-4 and Claude"}'` → results from all 4 architectures
- Streamlit dashboard at http://localhost:8501 → all 6 pages functional
- `pytest tests/test_api.py -v` → all pass

Do NOT touch src/graph/*, src/agentic/*, src/ingestion/*, src/retrieval/*, src/generation/*, src/framework/*, src/evaluation/* — consume them via imports only.
```

#### Sequential (after parallel)

*(none — infra consumes all other agents' modules via imports)*

#### Review

**reviewer**:
```
Review Phase 7 deliverables. Verify:
1. `src/api/routes.py` implements all 8 endpoints (POST /query, /query/compare, /decision; GET /graph/entities, /graph/subgraph, /images/{id}, /benchmark, /health).
2. `src/api/schemas.py` has Pydantic v2 schemas for all requests/responses.
3. `src/api/main.py` configures CORS, lifespan (pool + neo4j + embedders), includes router, /docs mounted.
4. All 6 dashboard pages exist and are functional: Comparison, GraphRAG Explorer, Agentic Trace, Multimodal Gallery, Decision Framework, KAG/LightRAG.
5. `docker compose up -d` starts all 4 services; /health returns healthy with postgres+neo4j connected.
6. POST /query works for each architecture; POST /query/compare returns all 4; POST /decision returns recommendation.
7. `tests/test_api.py` passes.
8. No file boundary violations — infra only created api/, dashboard/, test_api.py; consumed other modules via imports.
Report PASS or list specific issues.
```

#### Commit

```bash
git add -A && git commit -m "Phase 7: FastAPI comparison API, Streamlit 6-page dashboard"
```

---

### Phase 8: Testing, Documentation, README (Day 10)

**Goal**: Full test suite passing, README complete, final benchmark results documented.

#### Parallel Work

**Agent infra** (conftest + README):
```
You are the infra agent. Read IMPLEMENTATION_PLAN.md Section 12 (Testing Strategy) and Phase 8 before starting.

Your task for Phase 8 — Shared fixtures + README:

1. Create `tests/conftest.py` — shared fixtures:
   - db_pool: asyncpg Pool to TEST_DATABASE_URL (skip if unavailable)
   - neo4j_driver: Neo4j driver to test instance (or mock)
   - mock_llm: mock LLM client returning canned responses (entity JSON, reflection tokens, answers)
   - sample_chunks: sample chunk data
   - sample_entities: sample entity + relationship data
   - sample_images: sample image paths / ImageRecord data
   - mock_clip_embedder: returns fixed 512-dim vectors

2. Create `README.md` — project README:
   - Project overview + goals (4 architectures + decision framework)
   - Architecture summary (ASCII diagram reference to plan §2)
   - Quickstart: docker compose up -d → ingest → query
   - API documentation (link to /docs)
   - Dashboard guide (6 pages)
   - Benchmark results summary (table from data/benchmarks/)
   - Decision framework guide (when to use which architecture)
   - KAG/LightRAG summary
   - Development setup (pyproject.toml extras, pytest, ruff, mypy)

3. Run final benchmark: `python scripts/run_benchmark.py --architectures vanilla graph agentic multimodal` → save to data/benchmarks/benchmark_results.json.

Verification:
- `pytest tests/ -v --tb=short` → all tests pass (30+ tests)
- `pytest tests/ --cov=src --cov-report=term-missing` → coverage report
- `curl http://localhost:8000/health` → healthy
- README is complete and accurate.

Do NOT modify other agents' source files — only conftest.py and README.md.
```

**Agent core**, **graph**, **agentic** (parallel — fix any failing tests in their owned test files):
```
You are the {core|graph|agentic} agent. Phase 8 — run the full test suite and fix any failures in YOUR owned test files only.

1. Run `pytest tests/ -v --tb=short`.
2. For any failing test in files you own (see Agent Roster ownership), fix the underlying source or test. Do NOT modify tests owned by other agents.
3. Ensure all your owned tests pass: `pytest <your_test_files> -v`.

Verification:
- All your owned test files pass.
- No edits to other agents' files.
```

#### Sequential (after parallel)

Full suite confirmation:
```bash
pytest tests/ -v --tb=short
pytest tests/ --cov=src --cov-report=term-missing
```

#### Review

**reviewer**:
```
Review Phase 8 deliverables. Verify:
1. `tests/conftest.py` provides db_pool, neo4j_driver, mock_llm, sample_chunks, sample_entities, sample_images, mock_clip_embedder fixtures.
2. `pytest tests/ -v` → all tests pass (30+ tests across all agents' files).
3. Coverage report runs without error.
4. `data/benchmarks/benchmark_results.json` contains final results for all 4 architectures.
5. `docker compose up -d` → all services healthy; /health returns healthy.
6. `README.md` is complete: overview, architecture, quickstart, API, dashboard, benchmark summary, decision framework, KAG/LightRAG, dev setup.
7. Benchmark results show clear differentiation: graph wins multi-hop, vanilla wins factual low-latency, agentic shows iterative trace, multimodal returns image results.
8. No file boundary violations in any phase.
Report PASS or list specific issues.
```

#### Commit

```bash
git add -A && git commit -m "Phase 8: full test suite, shared fixtures, README, final benchmarks"
```

---

## 5. Cross-Phase Validation Checklist

Run after every phase commit:

```bash
# 1. File boundaries — no agent edited outside its ownership
git diff --name-only HEAD~1 | sort

# 2. Tests for the phase pass
pytest tests/ -v --tb=short

# 3. Services healthy (from Phase 1 onward)
docker compose ps
curl -s http://localhost:8000/health

# 4. Lint + type (from Phase 1)
ruff check src/ tests/
mypy src/
```

---

## 6. Dependency Graph (Phase Ordering)

```
Phase 1 (infra: env + schemas)
   │
   ▼
Phase 2 (core: ingest ──▶ graph: extract + build)   [graph reads core's chunks]
   │
   ▼
Phase 3 (core: vector retriever + eval ──▶ graph: graph retriever)  [benchmark after both]
   │
   ▼
Phase 4 (agentic: loop ──▶ core: agentic prompt templates)   [loop uses Phase 3 retrievers]
   │
   ▼
Phase 5 (core: CLIP + image extraction + multimodal retriever)
   │
   ▼
Phase 6 (core: framework — comparison + decision engine)
   │
   ▼
Phase 7 (infra: API + dashboard — consumes all modules)
   │
   ▼
Phase 8 (infra: conftest + README ──▶ all: fix owned tests)
```

**Hard dependencies** (do not start B before A merges):
- Phase 2 graph → Phase 2 core (chunks must exist in Postgres before build_graph)
- Phase 3 benchmark → both Phase 3 retrievers merged
- Phase 4 agentic loop → Phase 3 retrievers merged
- Phase 7 API/dashboard → Phases 2–6 modules merged

**Safe parallelism** (independent file sets):
- Phase 2: core (ingestion) ∥ graph (extraction) — no shared files
- Phase 3: core (retrieval/eval) ∥ graph (graph retriever) — no shared files
- Phase 4: agentic (src/agentic/*) ∥ core (prompt_templates.py append only)
- Phase 8: infra (conftest/README) ∥ all agents (own test files)

---

## 7. Agent Coordination Rules

1. **Source of truth**: `IMPLEMENTATION_PLAN.md`. Copy schemas, Cypher, prompts, and pseudocode verbatim from the cited sections. Do not improvise.
2. **File ownership**: each agent only writes files in its roster. The only cross-ownership edit permitted is core appending a delimited section to `src/generation/prompt_templates.py` in Phase 4.
3. **Review gate**: no phase advances until the reviewer reports PASS. Fix issues in the same phase before committing.
4. **Dependency handoff**: when a phase depends on another agent's merged work, wait for that phase's commit. Use `git log --oneline` to confirm.
5. **Mock first**: unit tests use mock LLM / mock Neo4j / mock pool / mock CLIP. Integration tests (test_api.py) may use real services via docker compose.
6. **No scope creep**: do not add re-ranking, tenant isolation (P2), or features outside the plan. P1/P2 modules are reused, not rebuilt.
