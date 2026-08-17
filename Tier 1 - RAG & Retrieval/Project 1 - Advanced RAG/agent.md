# Agent Orchestration Guide — Advanced RAG with Hybrid Search + Re-ranking

> **Tier 1 — Project 1** | 10 working days | 4 Herdr agents
> Companion to `IMPLEMENTATION_PLAN.md` — read it before starting any phase.

---

## 1. Agent Roster

| Agent | Kind | Responsibility | Owns (files / dirs) |
|-------|------|----------------|---------------------|
| **core** | codex | Chunking strategies, embedder, dual indexing (BM25 + pgvector), hybrid retrieval + RRF, query transformation (HyDE + multi-query), cross-encoder re-ranking, LLM client, RAG pipeline orchestrator | `src/ingestion/`, `src/indexing/`, `src/retrieval/`, `src/generation/`, `scripts/ingest.py`, `scripts/init_db.py`, `tests/test_chunkers.py`, `tests/test_embedder.py`, `tests/test_bm25_index.py`, `tests/test_vector_index.py`, `tests/test_hybrid.py`, `tests/test_query_transform.py`, `tests/test_reranker.py` |
| **eval** | codex | Retrieval metrics (recall@k, precision@k, MRR, nDCG), golden dataset construction, evaluation runner (54 strategy combos), Streamlit evaluation dashboard | `src/evaluation/`, `src/dashboard/`, `scripts/run_eval.py`, `scripts/create_golden.py`, `data/golden/`, `tests/test_metrics.py`, `tests/test_evaluation_runner.py` |
| **infra** | codex | Docker Compose, Dockerfile, pyproject.toml, .env.example, SQL schema, Pydantic config, FastAPI app (routes, schemas, main), README, test fixtures, API integration tests | `docker-compose.yml`, `Dockerfile`, `pyproject.toml`, `.env.example`, `sql/schema.sql`, `src/config.py`, `src/api/`, `tests/conftest.py`, `tests/test_api.py`, `README.md` |
| **reviewer** | codex | Read-only quality gate after each phase: verifies deliverables match plan specs, checks for file-boundary violations, runs verification commands from the plan, approves or requests changes | *(no file ownership — read-only across entire repo)* |

---

## 2. Pane Layout

```
┌─────────────────────────────────────────────────────────┐
│                       Pane 0                             │
│                 Main / reviewer                          │
│            (orchestration + review gate)                 │
├──────────────────────────┬──────────────────────────────┤
│        Pane 1            │         Pane 2               │
│        core              │         eval                 │
│  ingestion · indexing    │  metrics · golden · runner   │
│  retrieval · generation  │  dashboard                   │
├──────────────────────────┴──────────────────────────────┤
│                       Pane 3                             │
│                       infra                              │
│         docker · schema · api · config                   │
└─────────────────────────────────────────────────────────┘
```

---

## 3. Setup Commands

Run these from the project root (`advanced-rag/`):

```bash
# --- Split panes (Pane 0 already exists as the main pane) ---

# Pane 1 — core agent (top-left)
herdr pane split --cwd "$PWD" --no-focus

# Pane 2 — eval agent (top-right)
herdr pane split --cwd "$PWD" --no-focus

# Pane 3 — infra agent (bottom, full width)
herdr pane split --cwd "$PWD" --no-focus

# --- Start agents ---

herdr agent start core    --kind codex --pane 1
herdr agent start eval    --kind codex --pane 2
herdr agent start infra   --kind codex --pane 3
herdr agent start reviewer --kind codex --pane 0
```

> **Note**: Pane IDs are assigned in creation order. Verify with `herdr pane list` before starting agents. Adjust `--pane` values if your layout differs.

---

## 4. Phase-by-Phase Orchestration

### Phase 1: Environment & Schema (Day 1)

**Goal**: Docker Compose running, DB schema deployed, config loaded.

#### Parallel Work

**Agent infra**:
```
You are the infra agent for the Advanced RAG project. Read IMPLEMENTATION_PLAN.md Sections 3, 4, 5, and 13 before starting.

Your task for Phase 1 — Environment & Schema:

1. Create `docker-compose.yml` using the exact YAML from IMPLEMENTATION_PLAN.md Section 13.1. It must define three services: `postgres` (pgvector/pgvector:pg16), `api` (build from Dockerfile), and `dashboard` (Streamlit). Include the `postgres_data` and `model_cache` volumes and the `rag-network` bridge network. Mount `./sql/schema.sql` as a read-only init script in the postgres container.

2. Create `Dockerfile` from Section 13.2 — `python:3.11-slim` base, install `build-essential` and `libpq-dev`, copy `pyproject.toml`, pip install with `.[api,dashboard]` extras, download spaCy `en_core_web_sm`, expose port 8000, CMD runs uvicorn.

3. Create `pyproject.toml` from Section 13.4 with all dependencies (fastapi, uvicorn, pydantic, pydantic-settings, asyncpg, pgvector, sentence-transformers, spacy, httpx, numpy, python-dotenv) and optional-dependencies groups (api, dashboard, dev). Include `[tool.pytest.ini_options]` with `asyncio_mode = "auto"` and `testpaths = ["tests"]`. Include ruff and mypy config blocks.

4. Create `.env.example` from Section 13.3 with all environment variables: DATABASE_URL, TEST_DATABASE_URL, LLM_PROVIDER, OPENAI_*, OLLAMA_*, EMBEDDING_MODEL, RERANKER_MODEL, DEVICE, retrieval defaults, RRF_K, API_HOST, API_PORT, CORS_ORIGINS.

5. Create `sql/schema.sql` from Section 4 — the full DDL. This includes: CREATE EXTENSION for uuid-ossp, vector, pg_trgm; CREATE TABLE documents, chunks (with tsvector generated column, vector(1024) embedding, partial HNSW indexes per chunking strategy), queries, golden_questions, eval_results; all indexes from the plan.

6. Create `src/config.py` — Pydantic Settings class that loads all env vars from `.env.example`. Use `pydantic-settings` BaseSettings with env prefix (no prefix, field names match env var names). Include fields for database_url, llm_provider, openai_api_key, openai_model, openai_base_url, ollama_base_url, ollama_model, embedding_model, reranker_model, device, and retrieval defaults (default_chunking_strategy, default_retrieval_method, default_query_transform, default_reranker, default_top_k, default_retrieve_k, rrf_k, api_host, api_port, cors_origins).

7. Create `scripts/init_db.py` — async script that connects to Postgres using asyncpg, reads `sql/schema.sql`, and executes it. Print "Schema created successfully" on completion. Use the DATABASE_URL from config.

8. Create `src/__init__.py` and all package `__init__.py` files for: src/ingestion/, src/indexing/, src/retrieval/, src/generation/, src/evaluation/, src/api/, src/dashboard/.

Verification (run these and confirm output):
- `docker compose up -d postgres`
- `docker compose exec postgres psql -U rag -d ragdb -c "SELECT extname FROM pg_extension;"` → expect vector, uuid-ossp, pg_trgm
- `python scripts/init_db.py` → expect "Schema created successfully"
- `docker compose exec postgres psql -U rag -d ragdb -c "\dt"` → expect documents, chunks, queries, golden_questions, eval_results

Do NOT touch any files outside your ownership. Do NOT create src/ingestion/chunkers.py, src/ingestion/embedder.py, etc. — those belong to the core agent.
```

#### Sequential (after parallel)

*(none — Phase 1 is infra-only)*

#### Review

**reviewer**:
```
Review Phase 1 deliverables for the Advanced RAG project. Verify:

1. `docker compose up -d postgres` starts successfully and `pg_isready` passes.
2. `python scripts/init_db.py` prints "Schema created successfully".
3. All 5 tables exist: `docker compose exec postgres psql -U rag -d ragdb -c "\dt"` shows documents, chunks, queries, golden_questions, eval_results.
4. Extensions are enabled: vector, uuid-ossp, pg_trgm.
5. `sql/schema.sql` matches Section 4 of IMPLEMENTATION_PLAN.md exactly — check the tsvector GENERATED ALWAYS AS column, the vector(1024) embedding column, the 3 partial HNSW indexes (one per chunking_strategy), the GIN index on tsv, and the UNIQUE constraint on (document_id, chunking_strategy, chunk_index).
6. `src/config.py` loads all env vars listed in `.env.example` and uses pydantic-settings BaseSettings.
7. `pyproject.toml` has all dependencies from Section 13.4 and the dev extras include pytest, pytest-asyncio, pytest-cov, ruff, mypy.
8. No files outside infra's ownership were created.

Report PASS or list specific issues to fix.
```

#### Commit

```bash
git add -A && git commit -m "Phase 1: environment, Docker Compose, DB schema, config"
```

---

### Phase 2: Document Ingestion & Chunking (Day 2)

**Goal**: 3 chunking strategies producing chunks stored in the `chunks` table.

#### Parallel Work

**Agent core**:
```
You are the core agent for the Advanced RAG project. Read IMPLEMENTATION_PLAN.md Sections 5, 7.1, and Phase 2 before starting.

Your task for Phase 2 — Document Ingestion & Chunking:

1. Create `src/ingestion/loader.py` — document loaders for .txt, .md, and .pdf files. For .txt: read text directly. For .md: strip markdown formatting (use markdown library or simple regex). For .pdf: use PyPDF2 or pdfplumber. Return a list of Document dataclass instances with fields: source_path, title, doc_type, content, content_hash (SHA-256), char_count, metadata. Deduplicate by content_hash.

2. Create `src/ingestion/chunkers.py` — implement all three chunking strategies from IMPLEMENTATION_PLAN.md Section 7.1. Define a `Chunk` dataclass with fields: content, chunk_index, chunking_strategy, token_count, window_context (optional, default None), metadata (default empty dict).

   a) `FixedSizeChunker(chunk_size=1000, overlap=200)` — slides a window with overlap, adjusts to nearest sentence boundary in the last 20% of the chunk. Strategy label: "fixed_1000_200". Follow the pseudocode in Section 7.1 exactly, including `_adjust_to_sentence_boundary`.

   b) `SemanticChunker(target_size=1000, nlp_model="en_core_web_sm")` — uses spaCy sentence segmentation, groups sentences until ~target_size, never breaks a sentence. Strategy label: "semantic_spacy". Follow the pseudocode in Section 7.1.

   c) `SentenceWindowChunker(window_size=3, nlp_model="en_core_web_sm")` — each chunk is one sentence (embedded), stores window_context with surrounding N sentences. Strategy label: "sentence_window_3". Follow the pseudocode in Section 7.1, including the metadata with sentence_index and total_sentences.

3. Download spaCy model: `python -m spacy download en_core_web_sm`

4. Prepare a small sample corpus in `data/raw/` — create `data/raw/paul_graham_essays/` with 2-3 short .txt files (you can use public domain text or write synthetic startup-themed essays of ~3000 chars each). Also create `data/raw/arxiv_abstracts/` with 2-3 short .txt files containing AI/ML abstracts.

5. Create `scripts/ingest.py` — CLI that:
   - Takes `--source` (directory path) and `--strategy` (fixed_1000_200 | semantic_spacy | sentence_window_3 | all) flags
   - Loads documents via the loader
   - Chunks them using the selected strategy (or all three if --strategy all)
   - Stores documents and chunks in the database using asyncpg
   - For each document: INSERT into documents table (with content_hash dedup)
   - For each chunk: INSERT into chunks table with document_id, chunk_index, chunking_strategy, content, token_count, window_context, metadata
   - Print progress: "Ingested N documents, M chunks with strategy X"

6. Create `tests/test_chunkers.py` — follow the test cases from IMPLEMENTATION_PLAN.md Section 12.2 (test_chunkers.py). Include TestFixedSizeChunker (test_basic_chunking, test_overlap_exists, test_empty_text, test_short_text, test_sentence_boundary_adjustment), TestSemanticChunker (test_respects_sentence_boundaries, test_chunking_strategy_label), TestSentenceWindowChunker (test_each_chunk_is_one_sentence, test_window_context_contains_surrounding_sentences, test_edge_sentences_have_clipped_windows).

Verification:
- `python scripts/ingest.py --source data/raw/ --strategy fixed_1000_200`
- `python scripts/ingest.py --source data/raw/ --strategy semantic_spacy`
- `python scripts/ingest.py --source data/raw/ --strategy sentence_window_3`
- `docker compose exec postgres psql -U rag -d ragdb -c "SELECT chunking_strategy, COUNT(*) FROM chunks GROUP BY 1;"` → 3 rows with different counts
- `pytest tests/test_chunkers.py -v` → all tests pass

Do NOT create embedding or indexing code — that's Phase 3. Do NOT touch docker-compose.yml, sql/schema.sql, src/config.py, or src/api/ — those belong to infra.
```

**Agent eval** (parallel — no file conflicts with core):
```
You are the eval agent for the Advanced RAG project. Read IMPLEMENTATION_PLAN.md Sections 7.11, 9.2, and 12.2 (test_metrics.py) before starting.

Your task for Phase 2 — start evaluation metrics early (no dependency on chunking):

1. Create `src/evaluation/metrics.py` — implement the `RetrievalMetrics` class from IMPLEMENTATION_PLAN.md Section 7.11 with all four metrics as @staticmethod:

   a) `recall_at_k(retrieved_ids, relevant_ids, k)` — |relevant ∩ retrieved[:k]| / |relevant|. Return 0.0 if relevant_ids is empty.
   b) `precision_at_k(retrieved_ids, relevant_ids, k)` — |relevant ∩ retrieved[:k]| / k. Return 0.0 if k == 0.
   c) `mrr(retrieved_ids, relevant_ids)` — 1/rank of first relevant result. Return 0.0 if none found.
   d) `ndcg_at_k(retrieved_ids, relevance_grades, k)` — graded nDCG with formula: DCG = sum((2^grade - 1) / log2(i+1)), IDCG = ideal DCG sorted by grade descending, nDCG = DCG/IDCG. Return 0.0 if IDCG == 0.

   Follow the exact pseudocode in Section 7.11. Use `import math` for log2.

2. Create `tests/test_metrics.py` — follow the test cases from IMPLEMENTATION_PLAN.md Section 12.2 (test_metrics.py) exactly:
   - TestRecallAtK: test_perfect_recall, test_partial_recall (2/3 = 0.667), test_zero_recall, test_empty_relevant, test_k_smaller_than_retrieved
   - TestPrecisionAtK: test_perfect_precision, test_partial_precision (2/5 = 0.4)
   - TestMRR: test_first_result_relevant (1.0), test_second_result_relevant (0.5), test_no_relevant_found (0.0)
   - TestNDCG: test_perfect_ranking (1.0), test_worst_ranking (0.0), test_partial_ranking (0 < nDCG < 1), test_reversed_order_is_worse_than_ideal

Verification:
- `pytest tests/test_metrics.py -v` → all tests pass

Do NOT touch any files in src/ingestion/, src/indexing/, src/retrieval/, src/api/, or infra-owned files.
```

#### Sequential (after parallel)

*(none)*

#### Review

**reviewer**:
```
Review Phase 2 deliverables. Verify:

1. All three chunking strategies produce chunks in the database:
   `docker compose exec postgres psql -U rag -d ragdb -c "SELECT chunking_strategy, COUNT(*) FROM chunks GROUP BY 1;"` → 3 rows.

2. `FixedSizeChunker` uses 1000/200 params, adjusts to sentence boundaries, labels chunks "fixed_1000_200".

3. `SemanticChunker` uses spaCy `en_core_web_sm`, never breaks sentences, labels chunks "semantic_spacy".

4. `SentenceWindowChunker` embeds single sentences, stores window_context with 3 sentences before/after, labels chunks "sentence_window_3".

5. `scripts/ingest.py` accepts --source and --strategy flags, stores documents with content_hash dedup, stores chunks with correct schema fields.

6. `tests/test_chunkers.py` passes: `pytest tests/test_chunkers.py -v`

7. `src/evaluation/metrics.py` implements all 4 metrics correctly — verify the nDCG formula uses (2^grade - 1) gain and log2(i+1) discount.

8. `tests/test_metrics.py` passes: `pytest tests/test_metrics.py -v`

9. No file boundary violations — core did not touch eval or infra files, eval did not touch core or infra files.

Report PASS or list specific issues.
```

#### Commit

```bash
git add -A && git commit -m "Phase 2: document ingestion, 3 chunking strategies, retrieval metrics"
```

---

### Phase 3: Embedding & Dual Indexing (Day 3)

**Goal**: Embeddings generated, BM25 + pgvector indexes populated and searchable.

#### Parallel Work

**Agent core**:
```
You are the core agent. Read IMPLEMENTATION_PLAN.md Sections 7.2, 7.3, 7.4, 7.5, and Phase 3 before starting.

Your task for Phase 3 — Embedding & Dual Indexing:

1. Create `src/ingestion/embedder.py` — implement the `Embedder` class from Section 7.2:
   - MODEL_NAME = "BAAI/bge-large-en-v1.5", QUERY_INSTRUCTION = "Represent this sentence for searching relevant passages: ", DIMENSION = 1024
   - `__init__(model_name=None, device="cpu")` — loads SentenceTransformer
   - `embed_passages(texts, batch_size=32)` — embeds document chunks (NO instruction prefix), normalize_embeddings=True
   - `embed_query(query)` — embeds a single query WITH instruction prefix, returns 1D array
   - `embed_queries(queries)` — embeds multiple queries WITH instruction prefix
   Follow the exact pseudocode in Section 7.2.

2. Update `scripts/ingest.py` — after chunking, embed all chunks using `embedder.embed_passages()` and store the embedding in the `chunks.embedding` column. Format the numpy array as a pgvector string: '[0.1,0.2,...]'. Add `--embed` flag (default True) to skip embedding if needed. Batch the embedding (32 at a time) and show progress.

3. Create `src/indexing/bm25_index.py` — implement `BM25Index` from Section 7.3:
   - `BM25Result` dataclass: chunk_id, content, score, rank
   - `BM25Index(pool)` — takes an asyncpg Pool
   - `async search(query, chunking_strategy, k=50)` — uses ts_rank_cd with plainto_tsquery, filters by chunking_strategy, orders by score DESC, returns list[BM25Result]
   Follow the exact SQL and pseudocode in Section 7.3.

4. Create `src/indexing/vector_index.py` — implement `VectorIndex` from Section 7.4:
   - `VectorResult` dataclass: chunk_id, content, score, rank
   - `VectorIndex(pool, embedder)` — takes asyncpg Pool and an Embedder instance
   - `async search(query_embedding, chunking_strategy, k=50)` — uses pgvector cosine distance (`<=>`), similarity = 1 - distance, filters by chunking_strategy AND embedding IS NOT NULL, orders by distance, returns list[VectorResult]
   - `_format_vector(vec)` — static method converting numpy array to pgvector string '[0.1,0.2,...]'
   Follow the exact SQL and pseudocode in Section 7.4.

5. Create `src/indexing/store.py` — implement the unified interface from Section 7.5:
   - `RetrievalResult` dataclass: chunk_id, content, score, rank, source ('bm25'|'vector'|'both'), rrf_score=0.0, rerank_score=0.0
   - `VectorStore` ABC with abstract methods: keyword_search, vector_search, store_chunks
   Follow the exact pseudocode in Section 7.5.

6. Create `tests/test_embedder.py` — test that:
   - embed_passages returns array with correct dimension (1024)
   - embed_query returns 1D array with correct dimension
   - embed_query prepends the instruction prefix (mock the model to verify input)
   Use mocking for the SentenceTransformer to avoid loading the actual model in unit tests.

7. Create `tests/test_bm25_index.py` and `tests/test_vector_index.py` — test the search methods with a mock asyncpg pool (return canned rows, verify SQL parameters and result mapping).

Verification:
- Re-run ingestion with embedding: `python scripts/ingest.py --source data/raw/ --strategy all --embed`
- `docker compose exec postgres psql -U rag -d ragdb -c "SELECT chunking_strategy, COUNT(embedding) FROM chunks GROUP BY 1;"` → all chunks have embeddings
- Test BM25: run the SQL query from Phase 3 verification in the plan
- Test vector search: `python -c "from src.indexing.vector_index import VectorIndex; ..."`
- `pytest tests/test_embedder.py tests/test_bm25_index.py tests/test_vector_index.py -v` → all pass

Do NOT create retrieval/hybrid.py or retrieval/query_transform.py — those are later phases. Do NOT touch infra or eval files.
```

**Agent eval** (parallel — no file conflicts):
```
You are the eval agent. Read IMPLEMENTATION_PLAN.md Sections 9.4 and Phase 8 (golden dataset section) before starting.

Your task for Phase 3 — start golden dataset loader (no dependency on indexing):

1. Create `src/evaluation/golden_loader.py` — implement a `GoldenLoader` class that:
   - `load(path: str) -> list[dict]` — reads a JSONL file (one JSON object per line), each with fields: id, question, expected_answer, relevant_chunk_ids (list of UUID strings), relevance_grades (dict of chunk_id -> int 0-3), difficulty (easy|medium|hard), category (string)
   - `validate(questions: list[dict]) -> bool` — checks: every question has all required fields, relevant_chunk_ids is non-empty, relevance_grades keys are a subset of relevant_chunk_ids, grades are 0-3, difficulty is valid enum. Raises ValueError with descriptive message on failure.
   - `validate_against_db(questions, pool)` — async method that checks every relevant_chunk_id exists in the chunks table. Returns list of invalid IDs.

2. Create `data/golden/golden_dataset.jsonl` — start with 5 placeholder questions following the format from Section 9.4. These will be expanded in Phase 8 after chunks are ingested. Use the format:
   {"id": "q001", "question": "...", "expected_answer": "...", "relevant_chunk_ids": [], "relevance_grades": {}, "difficulty": "easy", "category": "startup_advice"}
   Leave relevant_chunk_ids empty for now — they'll be populated in Phase 8 after you can query the chunks table.

Do NOT touch core or infra files.
```

#### Sequential (after parallel)

*(none)*

#### Review

**reviewer**:
```
Review Phase 3 deliverables. Verify:

1. All chunks have embeddings: `docker compose exec postgres psql -U rag -d ragdb -c "SELECT chunking_strategy, COUNT(embedding) FROM chunks GROUP BY 1;"` → non-null counts for all 3 strategies.

2. `Embedder` class: uses BAAI/bge-large-en-v1.5, embed_passages does NOT add instruction prefix, embed_query DOES add the prefix, normalize_embeddings=True. Dimension is 1024.

3. `BM25Index.search` uses ts_rank_cd with plainto_tsquery, parameterized queries ($1, $2, $3), filters by chunking_strategy.

4. `VectorIndex.search` uses pgvector `<=>` cosine distance, similarity = 1 - distance, filters by chunking_strategy AND embedding IS NOT NULL, parameterized queries.

5. `store.py` defines RetrievalResult dataclass with all fields (chunk_id, content, score, rank, source, rrf_score, rerank_score) and VectorStore ABC with 3 abstract methods.

6. `golden_loader.py` loads JSONL, validates required fields, validates relevance_grades keys ⊆ relevant_chunk_ids.

7. All new tests pass: `pytest tests/test_embedder.py tests/test_bm25_index.py tests/test_vector_index.py -v`

8. No file boundary violations.

Report PASS or list specific issues.
```

#### Commit

```bash
git add -A && git commit -m "Phase 3: BGE embedder, BM25 + pgvector dual indexing, golden loader"
```

---

### Phase 4: Hybrid Retrieval + RRF (Day 4)

**Goal**: Combined BM25 + vector search with Reciprocal Rank Fusion returning fused results.

#### Parallel Work

**Agent core**:
```
You are the core agent. Read IMPLEMENTATION_PLAN.md Sections 7.6 and Phase 4 before starting.

Your task for Phase 4 — Hybrid Retrieval + RRF:

1. Create `src/retrieval/hybrid.py` — implement `HybridRetriever` from Section 7.6:
   - RRF_K = 60 (constant from Cormack et al. 2009)
   - `__init__(bm25_index, vector_index, embedder)` — takes BM25Index, VectorIndex, Embedder
   - `async retrieve(query, query_embedding, chunking_strategy, method="hybrid", k_per_index=50, final_k=20)` — retrieves from BM25 and/or vector based on method ('bm25'|'vector'|'hybrid'), then fuses with RRF if hybrid
   - `_rrf_fuse(bm25_results, vector_results, final_k)` — RRF formula: score = sum(1/(60+rank)) for each retriever. Items in both indexes get higher scores. Returns RetrievalResult list sorted by RRF score descending, with source = 'both' if in both, else 'bm25' or 'vector'.
   - `_to_retrieval_results(index_results, source)` — converts BM25Result/VectorResult to RetrievalResult
   Follow the exact pseudocode in Section 7.6. Use `collections.defaultdict` for rrf_scores.

2. Create `tests/test_hybrid.py` — follow the test cases from IMPLEMENTATION_PLAN.md Section 12.2 (test_hybrid.py) exactly:
   - TestRRFFusion: test_rrf_item_in_both_indexes_ranks_higher (item B in both → rank 1, source='both'), test_rrf_score_formula (verify 1/(60+1) + 1/(60+3)), test_rrf_empty_results (empty input → empty output)
   Use the `type("R", (), {...})()` pattern from the plan to create mock result objects. Instantiate HybridRetriever with None args since tests only call _rrf_fuse directly.

Verification:
- `python -c "from src.retrieval.hybrid import HybridRetriever; ..."` — test hybrid retrieval returns 20 results with rrf_score and source fields
- `pytest tests/test_hybrid.py -v` → all tests pass

Do NOT create query_transform.py or reranker.py yet. Do NOT touch infra or eval files.
```

#### Sequential (after parallel)

*(none)*

#### Review

**reviewer**:
```
Review Phase 4 deliverables. Verify:

1. `HybridRetriever` implements RRF with k=60 constant. The formula is 1/(60+rank) summed across retrievers.

2. `retrieve()` method supports all three modes: 'bm25' (BM25 only), 'vector' (vector only), 'hybrid' (both + RRF fusion). Default is 'hybrid', k_per_index=50, final_k=20.

3. Items appearing in both BM25 and vector results get source='both' and higher RRF scores.

4. `_to_retrieval_results` correctly maps BM25Result/VectorResult fields to RetrievalResult.

5. `tests/test_hybrid.py` passes: `pytest tests/test_hybrid.py -v`
   - Verify test_rrf_score_formula checks the exact formula value
   - Verify test_rrf_item_in_both_indexes_ranks_higher checks source='both'

6. No file boundary violations.

Report PASS or list specific issues.
```

#### Commit

```bash
git add -A && git commit -m "Phase 4: hybrid retrieval with Reciprocal Rank Fusion"
```

---

### Phase 5: Query Transformation (Day 5)

**Goal**: HyDE and multi-query expansion functional as drop-in replacements for raw query embedding.

#### Parallel Work

**Agent core**:
```
You are the core agent. Read IMPLEMENTATION_PLAN.md Sections 7.7, 7.10, and Phase 5 before starting.

Your task for Phase 5 — Query Transformation (HyDE + Multi-Query):

1. Create `src/generation/llm.py` — implement `LLMClient` from Section 7.10:
   - `__init__(settings)` — stores settings, creates httpx.AsyncClient(timeout=30.0)
   - `async generate(prompt, max_tokens=500, temperature=0.0)` — routes to _call_openai or _call_ollama based on settings.llm_provider
   - `async generate_answer(question, context)` — formats ANSWER_PROMPT from Section 7.10 and calls generate
   - `_call_openai(prompt, max_tokens, temperature)` — POST to {openai_base_url}/chat/completions with Bearer auth, returns choices[0].message.content
   - `_call_ollama(prompt, max_tokens, temperature)` — POST to {ollama_base_url}/api/generate with stream=False, returns response field
   Follow the exact pseudocode and prompts in Section 7.10. The ANSWER_PROMPT must include the "I don't have enough information" instruction.

2. Create `src/retrieval/query_transform.py` — implement both transformers from Section 7.7:

   a) `HyDETransformer(llm, embedder)`:
      - HYDE_PROMPT from Section 7.7 (expert assistant, 3-5 sentence answer, no preamble)
      - `transform(query) -> np.ndarray` — generates hypothetical answer via LLM, embeds it as a PASSAGE (not query) using embedder.embed_passages, returns the embedding
      - `transform_batch(queries) -> np.ndarray` — batch version
      IMPORTANT: HyDE embeds the hypothetical doc as a passage (embed_passages), NOT as a query (embed_query). This is a critical distinction.

   b) `MultiQueryTransformer(llm, embedder, n_queries=3)`:
      - MULTIQUERY_PROMPT from Section 7.7 (generate N alternative questions)
      - `transform(query) -> list[str]` — generates N reformulations, parses one per line (strip leading numbers/periods/parens), always includes original query. Returns [original] + reformulations[:n_queries]
      - `get_embeddings(query) -> list[np.ndarray]` — calls transform then embeds all queries with embed_query
      - `@staticmethod merge_results(result_lists, rrf_k=60, final_k=20)` — merges results from multiple query searches using RRF (same formula as hybrid, but across queries instead of across retrieval methods)

   Follow the exact pseudocode and prompts in Section 7.7.

3. Create `tests/test_query_transform.py` — follow the test cases from Section 12.2 (test_query_transform.py):
   - TestHyDETransformer: test_hyde_generates_hypothetical_doc (mock LLM, verify generate called once, embedding shape (1024,)), test_hyde_uses_passage_embedding (verify embed_passages called, embed_query NOT called)
   - TestMultiQueryTransformer: test_generates_n_reformulations (mock LLM returns 3 lines, verify 4 queries returned including original), test_merge_results_with_rrf (item A in both lists → rank 1)
   Use unittest.mock.Mock and patch for LLM and embedder. Do NOT load actual models.

Verification:
- `python -c "from src.retrieval.query_transform import HyDETransformer, MultiQueryTransformer; ..."` — test both transformers produce output
- `pytest tests/test_query_transform.py -v` → all tests pass

Do NOT create reranker.py yet. Do NOT touch infra or eval files.
```

**Agent eval** (parallel — no file conflicts):
```
You are the eval agent. Read IMPLEMENTATION_PLAN.md Sections 9.4 and Phase 8 before starting.

Your task for Phase 5 — expand golden dataset (now that chunks are ingested from Phase 2-3):

1. Update `data/golden/golden_dataset.jsonl` — expand to 30-50 questions. The corpus is in data/raw/ (Paul Graham essays + arXiv abstracts). Write questions that span:
   - Easy (direct factual, answerable from one chunk): ~15 questions
   - Medium (synthesis across 2-3 chunks): ~15 questions
   - Hard (abstract/implicit reasoning): ~10 questions
   
   Categories: startup_advice, ai_ml_concepts, rag_techniques, etc.

2. For each question, query the chunks table to identify relevant chunks:
   `docker compose exec postgres psql -U rag -d ragdb -c "SELECT id, chunking_strategy, LEFT(content, 100) FROM chunks WHERE content ILIKE '%keyword%' LIMIT 10;"`
   
   Assign relevance_grades: 3=perfectly relevant (direct answer), 2=relevant (supporting info), 1=marginally relevant, 0=irrelevant.
   
   Use sentence_window_3 strategy chunks for finest granularity when identifying relevant chunks, then include the same content's chunks from other strategies.

3. Validate with golden_loader: `python -c "from src.evaluation.golden_loader import GoldenLoader; GoldenLoader().validate(GoldenLoader().load('data/golden/golden_dataset.jsonl'))"`

Do NOT touch core or infra files.
```

#### Sequential (after parallel)

*(none)*

#### Review

**reviewer**:
```
Review Phase 5 deliverables. Verify:

1. `LLMClient` supports both OpenAI and Ollama providers, uses httpx.AsyncClient, ANSWER_PROMPT includes "I don't have enough information" instruction.

2. `HyDETransformer`: uses HYDE_PROMPT from Section 7.7, generates hypothetical answer, embeds it as a PASSAGE (embed_passages, NOT embed_query). This is the critical HyDE distinction — verify it.

3. `MultiQueryTransformer`: uses MULTIQUERY_PROMPT, generates N reformulations, always includes original query, parses one per line stripping leading numbers/periods/parens. merge_results uses RRF across query result lists.

4. `tests/test_query_transform.py` passes: `pytest tests/test_query_transform.py -v`
   - Verify test_hyde_uses_passage_embedding checks embed_passages is called and embed_query is NOT called
   - Verify test_generates_n_reformulations returns original + 3 = 4 queries

5. Golden dataset has 30-50 questions with valid relevant_chunk_ids (all exist in chunks table) and relevance_grades. Difficulty distribution covers easy/medium/hard.

6. No file boundary violations.

Report PASS or list specific issues.
```

#### Commit

```bash
git add -A && git commit -m "Phase 5: HyDE + multi-query transformation, LLM client, golden dataset"
```

---

### Phase 6: Cross-Encoder Re-ranking (Day 6)

**Goal**: BGE-reranker integrated, top-20 candidates re-ranked to top-5.

#### Parallel Work

**Agent core**:
```
You are the core agent. Read IMPLEMENTATION_PLAN.md Sections 7.8, 7.9, and Phase 6 before starting.

Your task for Phase 6 — Cross-Encoder Re-ranking + Full Pipeline Orchestrator:

1. Create `src/retrieval/reranker.py` — implement `CrossEncoderReranker` from Section 7.8:
   - `RerankedResult` dataclass: chunk_id, content, rerank_score, original_rank, new_rank, source
   - MODEL_NAME = "BAAI/bge-reranker-large"
   - `__init__(model_name=None, device="cpu")` — loads CrossEncoder, sets max_length=512
   - `rerank(query, candidates, top_k=5)` — builds (query, passage[:512]) pairs, scores with model.predict in single batch, sorts by score descending, returns top_k RerankedResult list. Returns [] for empty candidates.
   - `rerank_with_context(query, candidates, top_k=5)` — same but uses window_context if available (for sentence-window chunks), falls back to content
   Follow the exact pseudocode in Section 7.8. The docstring explaining bi-encoder vs cross-encoder should be preserved.

2. Create the full RAG pipeline orchestrator. Place it at `src/rag_pipeline.py` (referenced as RAGPipeline in Section 7.9 and by the evaluation runner in Section 7.12):
   - `RAGResponse` dataclass: answer, retrieved_chunks (list[dict]), strategy_config (dict), latency_ms (int)
   - `RAGPipeline(embedder, retriever, reranker, llm, hyde, multi_query)` — takes all components
   - `async query(question, chunking_strategy="semantic_spacy", retrieval_method="hybrid", query_transform="none", reranker_enabled=True, top_k=5, retrieve_k=20)` — orchestrates the full pipeline:
     Step 1: Query transformation → get embedding(s)
       - "hyde": hyde.transform(question) → single embedding → retriever.retrieve
       - "multi_query": multi_query.get_embeddings(question) → multiple embeddings → retriever.retrieve for each → merge_results
       - "none": embedder.embed_query(question) → single embedding → retriever.retrieve
     Step 2: Cross-encoder re-ranking (if reranker_enabled and candidates non-empty)
       - reranker.rerank(question, candidates, top_k=top_k)
       - If disabled, take candidates[:top_k] directly
     Step 3: LLM generation — join context as "[1] chunk1\n\n[2] chunk2...", call llm.generate_answer
     Return RAGResponse with answer, retrieved_chunks, strategy_config, latency_ms
   Follow the exact pseudocode in Section 7.9. Use `import time` for latency measurement.

3. Create `tests/test_reranker.py` — follow the test cases from Section 12.2 (test_reranker.py):
   - test_rerank_reorders_by_cross_encoder_score (candidates A,B,C with cross-encoder scores 0.1,0.5,0.9 → C is rank 1, verify rerank_score, new_rank, original_rank, top_k=2 returns 2)
   - test_rerank_empty_candidates (empty → empty)
   - test_rerank_top_k_larger_than_candidates (top_k=10 with 2 candidates → returns 2)
   Use CrossEncoderReranker.__new__ to avoid loading the model, then set reranker.model = Mock() with predict returning canned scores.

Verification:
- `python -c "from src.retrieval.reranker import CrossEncoderReranker; ..."` — test reranking
- `python -c "from src.rag_pipeline import RAGPipeline; ..."` — test full pipeline (may need DB running)
- `pytest tests/test_reranker.py -v` → all tests pass

Do NOT touch infra or eval files.
```

#### Sequential (after parallel)

*(none)*

#### Review

**reviewer**:
```
Review Phase 6 deliverables. Verify:

1. `CrossEncoderReranker`: loads BAAI/bge-reranker-large, sets max_length=512, rerank() builds (query, passage[:512]) pairs, scores in single batch via model.predict, sorts descending, returns RerankedResult with original_rank and new_rank. Empty candidates → empty list.

2. `rerank_with_context` uses window_context when available (for sentence-window chunks), falls back to content.

3. `RAGPipeline.query()` orchestrates all 3 steps correctly:
   - HyDE path: uses hyde.transform for embedding, single retrieve call
   - Multi-query path: uses multi_query.get_embeddings, multiple retrieve calls, merge_results
   - None path: uses embedder.embed_query, single retrieve call
   - Re-ranking: applied when reranker_enabled=True and candidates non-empty
   - LLM generation: context formatted as [1] chunk1\n\n[2] chunk2
   - latency_ms measured with time.time()

4. `tests/test_reranker.py` passes: `pytest tests/test_reranker.py -v`
   - Verify test_rerank_reorders_by_cross_encoder_score checks C is rank 1 with rerank_score=0.9, original_rank=3

5. No file boundary violations.

Report PASS or list specific issues.
```

#### Commit

```bash
git add -A && git commit -m "Phase 6: cross-encoder re-ranking, full RAG pipeline orchestrator"
```

---

### Phase 7: FastAPI Query Endpoint (Day 7)

**Goal**: `POST /query` accepts strategy configuration and returns ranked results + answer.

#### Parallel Work

**Agent infra**:
```
You are the infra agent. Read IMPLEMENTATION_PLAN.md Sections 7.9, 10, and Phase 7 before starting.

Your task for Phase 7 — FastAPI Query Endpoint:

1. Create `src/api/schemas.py` — Pydantic v2 models for API request/response:
   - `QueryRequest`: query (str, required), chunking_strategy (enum: fixed_1000_200|semantic_spacy|sentence_window_3, default semantic_spacy), retrieval_method (enum: bm25|vector|hybrid, default hybrid), query_transform (enum: none|hyde|multi_query, default none), reranker (enum: none|bge_reranker, default bge_reranker), top_k (int, default 5), retrieve_k (int, default 20)
   - `ChunkResult`: chunk_id (str), content (str), score (float), rank (int), source (str)
   - `QueryResponse`: answer (str), retrieved_chunks (list[ChunkResult]), strategy_config (dict), latency_ms (int)
   - `HealthResponse`: status (str), db (str), embedding_model (str), reranker_model (str)
   - `IngestRequest`: source_path (str), chunking_strategy (str, default "all"), embed (bool, default True)
   - `IngestResponse`: job_id (str), status (str), message (str)
   - `StrategiesResponse`: chunking_strategies (list[str]), retrieval_methods (list[str]), query_transforms (list[str]), rerankers (list[str])
   Use Pydantic v2 Field with constraints and descriptions matching Section 10 API spec.

2. Create `src/api/routes.py` — FastAPI router with endpoints from Section 10:
   - `GET /health` → HealthResponse (status="healthy", db="connected", embedding_model, reranker_model)
   - `POST /query` → accepts QueryRequest, instantiates RAGPipeline (or uses a cached singleton), calls pipeline.query(), returns QueryResponse. Handle 400 for invalid strategy combos.
   - `POST /ingest` → accepts IngestRequest, returns 202 Accepted with IngestResponse (async background task — just return accepted status, actual ingestion runs in background)
   - `GET /strategies` → StrategiesResponse with all available strategy options
   - `GET /eval/results` → query eval_results table with optional filters (chunking_strategy, retrieval_method, query_transform, reranker, limit), return results + summary with best_ndcg_at_5
   Use dependency injection for DB pool and pipeline. All DB queries must use parameterized queries.

3. Create `src/api/main.py` — FastAPI app:
   - Create app with title="Advanced RAG API", version="0.1.0"
   - Include router from routes.py
   - Add CORS middleware with origins from config (cors_origins)
   - Lifespan context: create asyncpg pool on startup, initialize Embedder/Retriever/Reranker/LLM/HyDE/MultiQuery/RAGPipeline singletons; close pool on shutdown
   - Mount OpenAPI docs at /docs (default FastAPI behavior)
   - Handle startup errors gracefully (log and exit if DB unavailable)

4. Create `tests/test_api.py` — follow the test cases from Section 12.3:
   - test_health_endpoint: GET /health returns 200, status="healthy", "db" in response
   - test_query_endpoint: POST /query with valid params returns 200, has "answer", "retrieved_chunks" (≤5), "latency_ms", "strategy_config"
   - test_query_invalid_strategy: POST /query with chunking_strategy="invalid_strategy" returns 422
   Use httpx.AsyncClient(app=app, base_url="http://test"). Mark all tests with @pytest.mark.asyncio. Use the test_db fixture from conftest.py.

5. Update `tests/conftest.py` — add fixtures from Section 12.5:
   - `test_db` (async): creates asyncpg pool from TEST_DATABASE_URL, truncates tables before each test, yields pool, closes pool
   - `sample_docs`: returns small sample documents
   - `test_golden`: returns small golden dataset for testing
   - `test_corpus` (async): ingests sample_docs into test DB before query tests

Verification:
- `uvicorn src.api.main:app --reload --port 8000`
- `curl http://localhost:8000/health` → {"status": "healthy", "db": "connected", ...}
- `curl -X POST http://localhost:8000/query -H "Content-Type: application/json" -d '{"query": "How do I validate a startup idea?"}' | python -m json.tool` → valid JSON with answer and retrieved_chunks
- `curl http://localhost:8000/strategies` → all strategy options
- `pytest tests/test_api.py -v` → all tests pass

Do NOT modify src/rag_pipeline.py or any core/eval files. You may import and use them.
```

#### Sequential (after parallel)

*(none)*

#### Review

**reviewer**:
```
Review Phase 7 deliverables. Verify:

1. `GET /health` returns 200 with status, db, embedding_model, reranker_model fields.

2. `POST /query` accepts all strategy parameters from Section 10 (chunking_strategy, retrieval_method, query_transform, reranker, top_k, retrieve_k), returns QueryResponse with answer, retrieved_chunks, strategy_config, latency_ms.

3. Invalid strategy values return 422 (Pydantic validation error).

4. `POST /ingest` returns 202 Accepted.

5. `GET /strategies` returns all available options.

6. `GET /eval/results` supports filter query params and returns results + summary.

7. `main.py` has CORS middleware, lifespan for pool/pipeline initialization, OpenAPI docs at /docs.

8. `conftest.py` has test_db, sample_docs, test_golden fixtures matching Section 12.5.

9. `tests/test_api.py` passes: `pytest tests/test_api.py -v`

10. No file boundary violations — infra only created src/api/ files, conftest.py, test_api.py. Did not modify core or eval source files.

Report PASS or list specific issues.
```

#### Commit

```bash
git add -A && git commit -m "Phase 7: FastAPI query endpoint with configurable retrieval strategy"
```

---

### Phase 8: Evaluation Metrics & Golden Dataset (Day 8)

**Goal**: Evaluation runner produces results for all 54 strategy combinations.

#### Parallel Work

**Agent eval**:
```
You are the eval agent. Read IMPLEMENTATION_PLAN.md Sections 7.11, 7.12, 9.4, and Phase 8 before starting.

Your task for Phase 8 — Evaluation Runner (metrics already done in Phase 2, golden dataset done in Phase 5):

1. Create `src/evaluation/runner.py` — implement `EvaluationRunner` from Section 7.12:
   - `StrategyCombo` dataclass: chunking_strategy, retrieval_method, query_transform, reranker
   - `STRATEGIES` dict with 4 keys: chunking (3 options), retrieval (3 options), transform (3 options), reranker (2 options) = 54 total combos
   - `__init__(pipeline, golden_questions)` — takes RAGPipeline and list of golden question dicts
   - `get_all_combos() -> list[StrategyCombo]` — uses itertools.product to generate all 54 combos
   - `async run_combination(combo) -> list[dict]` — runs all golden questions for one combo, computes recall@5, recall@10, precision@5, MRR, nDCG@5, nDCG@10, latency_ms for each
   - `async run_all() -> list[dict]` — runs all 54 combos × all questions, prints progress "[i/54] Running X | Y | Z | W", returns flat list of results
   Follow the exact pseudocode in Section 7.12. Use RetrievalMetrics from src/evaluation/metrics.py.

2. Create `scripts/run_eval.py` — CLI that:
   - Takes `--golden` (path to golden_dataset.jsonl) flag
   - Loads golden questions via GoldenLoader
   - Validates golden questions (validate + validate_against_db)
   - Initializes RAGPipeline with all components (embedder, retriever, reranker, llm, hyde, multi_query)
   - Creates EvaluationRunner and calls run_all()
   - Stores results in eval_results table (INSERT with all metric columns)
   - Prints summary: best combo per metric, total runs, total time
   - Supports `--checkpoint` flag to save progress every N combos (for resumable runs, addressing Risk R6)

3. Create `scripts/create_golden.py` — CLI that:
   - Helps semi-automate golden dataset creation
   - Takes a question and searches the chunks table for candidate relevant chunks
   - Outputs candidate chunks with content preview for manual relevance grading
   - Appends validated Q&A pairs to golden_dataset.jsonl

4. Create `tests/test_evaluation_runner.py` — test:
   - test_get_all_combos_returns_54: verify itertools.product produces 3×3×3×2=54 combos
   - test_run_combination_produces_metrics: mock pipeline.query, verify all 6 metrics computed and in [0,1] range (except latency_ms > 0)
   - test_run_all_count: verify total results = 54 × len(golden)
   Use mock pipeline to avoid actual LLM/model calls.

Verification:
- `python scripts/run_eval.py --golden data/golden/golden_dataset.jsonl` → runs all 54 combos
- `docker compose exec postgres psql -U rag -d ragdb -c "SELECT chunking_strategy, retrieval_method, query_transform, reranker, AVG(ndcg_at_5) as avg_ndcg5, AVG(mrr) as avg_mrr FROM eval_results GROUP BY 1,2,3,4 ORDER BY avg_ndcg5 DESC LIMIT 10;"` → 54 rows, hybrid+hyde+reranker near top
- `pytest tests/test_evaluation_runner.py -v` → all tests pass

Do NOT touch core or infra files. You may import RAGPipeline and RetrievalMetrics.
```

#### Sequential (after parallel)

*(none)*

#### Review

**reviewer**:
```
Review Phase 8 deliverables. Verify:

1. `EvaluationRunner` generates exactly 54 strategy combinations (3×3×3×2) via itertools.product.

2. `run_combination` computes all 6 metrics: recall@5, recall@10, precision@5, MRR, nDCG@5, nDCG@10, plus latency_ms.

3. `run_all` prints progress and returns flat list of all results.

4. `scripts/run_eval.py` loads golden dataset, validates it, runs evaluation, stores results in eval_results table.

5. `eval_results` table is populated: `docker compose exec postgres psql -U rag -d ragdb -c "SELECT COUNT(*) FROM eval_results;"` → 54 × N questions.

6. Best combo by nDCG@5 is hybrid + hyde + bge_reranker (or close to it, per expected results in Section 8.5).

7. `tests/test_evaluation_runner.py` passes: `pytest tests/test_evaluation_runner.py -v`

8. No file boundary violations.

Report PASS or list specific issues.
```

#### Commit

```bash
git add -A && git commit -m "Phase 8: evaluation runner, 54 strategy combinations, golden dataset validation"
```

---

### Phase 9: Streamlit Evaluation Dashboard (Day 9)

**Goal**: Interactive dashboard showing all strategy comparisons.

#### Parallel Work

**Agent eval**:
```
You are the eval agent. Read IMPLEMENTATION_PLAN.md Sections 7.13 and Phase 9 before starting.

Your task for Phase 9 — Streamlit Evaluation Dashboard:

1. Create `src/dashboard/app.py` — implement the Streamlit dashboard from Section 7.13 with all 7 sections:

   a) Connection: `@st.cache_resource` for asyncpg pool, `@st.cache_data(ttl=60)` for loading eval_results joined with golden_questions into a pandas DataFrame.

   b) Sidebar filters: multiselect for chunking_strategy, retrieval_method, query_transform, reranker. Apply filters to the DataFrame.

   c) Section 1 — Overview: aggregate by (chunking_strategy, retrieval_method, query_transform, reranker) with mean of all metrics + latency_ms. Sort by nDCG@5 descending. Show as st.dataframe.

   d) Section 2 — Best Strategy per Metric: for each metric (recall@5, precision@5, MRR, nDCG@5), find the combo with the highest value and display as st.table.

   e) Section 3 — Chunking Strategy Comparison: hold retrieval=hybrid, transform=none, reranker=bge_reranker constant. Group by chunking_strategy, show bar chart of metrics.

   f) Section 4 — Query Transform Impact: hold chunking=semantic_spacy, retrieval=hybrid, reranker=bge_reranker. Group by query_transform, bar chart.

   g) Section 5 — Reranker Impact: hold chunking=semantic_spacy, retrieval=hybrid, transform=none. Group by reranker, bar chart.

   h) Section 6 — Latency vs Quality: Plotly scatter plot, x=latency_ms, y=ndcg_at_5, color=chunking_strategy, symbol=query_transform, size=recall_at_10, hover_data=[retrieval_method, reranker].

   i) Section 7 — Per-Query Drill-Down: selectbox to pick a question, show dataframe of all strategy combos for that question with metrics.

   Follow the exact code structure in Section 7.13. Use `st.set_page_config(page_title="RAG Evaluation Dashboard", layout="wide")`. Use plotly.express for the scatter plot. Use pandas.read_sql or asyncpg to load data.

2. Ensure the dashboard reads from the `eval_results` table (populated in Phase 8). The connection string should come from environment variables or st.secrets.

Verification:
- `streamlit run src/dashboard/app.py --server.port 8501`
- Open http://localhost:8501 — all 7 sections render
- Tables populate from eval_results
- Charts display (bar charts + Plotly scatter)
- Sidebar filters work
- Per-query drill-down shows data for selected question

Do NOT touch core or infra files.
```

#### Sequential (after parallel)

*(none)*

#### Review

**reviewer**:
```
Review Phase 9 deliverables. Verify:

1. `streamlit run src/dashboard/app.py` launches without errors.

2. All 7 sections from Section 7.13 are present:
   - Overview (aggregate table sorted by nDCG@5)
   - Best Strategy per Metric
   - Chunking Strategy Comparison (bar chart, others held constant)
   - Query Transform Impact (bar chart)
   - Reranker Impact (bar chart)
   - Latency vs Quality (Plotly scatter with color/symbol/size/hover_data)
   - Per-Query Drill-Down (selectbox + dataframe)

3. Sidebar has multiselect filters for all 4 strategy dimensions.

4. Dashboard reads from eval_results table (data is populated from Phase 8).

5. `@st.cache_data(ttl=60)` is used for data loading to avoid re-querying on every interaction.

6. No file boundary violations.

Report PASS or list specific issues.
```

#### Commit

```bash
git add -A && git commit -m "Phase 9: Streamlit evaluation dashboard with 7 comparison sections"
```

---

### Phase 10: Testing, Polish & Documentation (Day 10)

**Goal**: Full test suite passing with >80% coverage, README, end-to-end demo.

#### Parallel Work

**Agent core**:
```
You are the core agent. Read IMPLEMENTATION_PLAN.md Section 12 and Phase 10 before starting.

Your task for Phase 10 — Complete unit tests for core components:

1. Review and complete all test files in your ownership:
   - `tests/test_chunkers.py` — ensure all tests from Section 12.2 are implemented and pass
   - `tests/test_embedder.py` — ensure dimension checks, instruction prefix verification, batch handling
   - `tests/test_bm25_index.py` — ensure SQL parameterization, result mapping, empty results
   - `tests/test_vector_index.py` — ensure vector formatting, cosine distance, result mapping
   - `tests/test_hybrid.py` — ensure RRF formula, both-index ranking, empty results
   - `tests/test_query_transform.py` — ensure HyDE uses passage embedding, multi-query generates N+1 queries, merge_results RRF
   - `tests/test_reranker.py` — ensure reordering by cross-encoder score, empty candidates, top_k > candidates

2. Add any missing edge case tests:
   - FixedSizeChunker: overlap < chunk_size assertion, very large text
   - SemanticChunker: single sentence text, text with no sentence boundaries
   - SentenceWindowChunker: single sentence (window clipped), window at document boundaries
   - Embedder: batch embedding returns correct shape, empty input
   - Reranker: rerank_with_context uses window_context when available

3. Add type hints to all public functions in src/ingestion/, src/indexing/, src/retrieval/, src/generation/, src/rag_pipeline.py.

4. Clean up dead code, TODOs, unused imports in your owned files.

5. Run: `pytest tests/test_chunkers.py tests/test_embedder.py tests/test_bm25_index.py tests/test_vector_index.py tests/test_hybrid.py tests/test_query_transform.py tests/test_reranker.py -v --cov=src/ingestion --cov=src/indexing --cov=src/retrieval --cov=src/generation --cov-report=term-missing`

All tests must pass. Fix any failures. Do NOT touch infra or eval test files.
```

**Agent eval** (parallel — no file conflicts with core):
```
You are the eval agent. Read IMPLEMENTATION_PLAN.md Section 12 and Phase 10 before starting.

Your task for Phase 10 — Complete evaluation tests + end-to-end eval test:

1. Review and complete:
   - `tests/test_metrics.py` — ensure all test cases from Section 12.2 pass, add edge cases (k=0, single element, all relevant, no relevant)
   - `tests/test_evaluation_runner.py` — ensure 54 combos, metric computation, run_all count

2. Add end-to-end evaluation test from Section 12.4:
   - `test_evaluation_runner_produces_metrics`: run evaluation with test_golden, verify all 54×N results have metrics in valid ranges (0.0 ≤ nDCG ≤ 1.0, 0.0 ≤ recall ≤ 1.0, latency_ms > 0)

3. Add type hints to all public functions in src/evaluation/ and src/dashboard/.

4. Clean up dead code, TODOs, unused imports in your owned files.

5. Run: `pytest tests/test_metrics.py tests/test_evaluation_runner.py -v`

All tests must pass. Do NOT touch core or infra test files.
```

**Agent infra** (parallel — no file conflicts with core or eval):
```
You are the infra agent. Read IMPLEMENTATION_PLAN.md Sections 12, 13, and Phase 10 before starting.

Your task for Phase 10 — Integration tests, README, end-to-end smoke test:

1. Complete `tests/test_api.py` — ensure all tests from Section 12.3 pass. Add:
   - test_strategies_endpoint: GET /strategies returns all options
   - test_ingest_endpoint: POST /ingest returns 202
   - test_eval_results_endpoint: GET /eval/results returns results with summary

2. Add end-to-end integration tests from Section 12.4:
   - `test_full_pipeline_ingest_query_answer`: ingest sample docs → query → verify non-empty answer, retrieved_chunks > 0, latency_ms > 0
   Place these in `tests/test_api.py` or a new `tests/test_e2e.py`.

3. Write `README.md` with:
   - Project title and one-paragraph description
   - Quick Start (from Section 16.A): clone, cp .env.example .env, docker compose up, init_db, ingest, query, dashboard
   - Architecture summary (brief, reference IMPLEMENTATION_PLAN.md for details)
   - API endpoints summary (from Section 10)
   - Key findings section (to be filled after evaluation runs — leave placeholder)
   - Tech stack table (from Section 3)

4. Run full test suite: `pytest tests/ -v --cov=src --cov-report=term-missing`
   - Target: >80% coverage on src/
   - Fix any failing tests in your ownership (test_api.py, conftest.py)
   - If tests in other agents' files fail, report the failure but do NOT fix their code

5. Run end-to-end smoke test from Phase 10 verification:
   - `docker compose up -d`
   - `python scripts/init_db.py`
   - `python scripts/ingest.py --source data/raw/ --strategy all`
   - `python scripts/run_eval.py --golden data/golden/golden_dataset.jsonl`
   - `curl -X POST http://localhost:8000/query -H "Content-Type: application/json" -d '{"query": "What makes a good startup idea?"}' | python -m json.tool`
   - Verify valid JSON response with answer and retrieved chunks

Do NOT modify core or eval source files or their test files.
```

#### Sequential (after parallel)

*(none — all three agents work in parallel on non-overlapping test files)*

#### Review

**reviewer**:
```
Review Phase 10 deliverables — final quality gate. Verify:

1. Full test suite passes: `pytest tests/ -v --cov=src --cov-report=term-missing`
   - All tests pass (unit + integration + e2e)
   - Coverage > 80% on src/

2. Test count matches expectations from Section 12.1:
   - Unit tests: 20+ (chunkers, embedder, bm25, vector, hybrid, query_transform, reranker, metrics)
   - Integration tests: 5+ (API endpoints, DB + retrieval)
   - E2E tests: 2 (full pipeline, evaluation runner)

3. `README.md` exists with: project description, quick start, architecture summary, API endpoints, tech stack.

4. Type hints on all public functions across src/.

5. No dead code, TODOs, or unused imports (run `ruff check src/` and `mypy src/` if configured).

6. End-to-end smoke test passes:
   - docker compose up → all services healthy
   - init_db → schema created
   - ingest → chunks stored with embeddings
   - run_eval → 54 combos × N questions in eval_results
   - curl /query → valid JSON with answer + retrieved_chunks
   - streamlit dashboard → all 7 sections render

7. No file boundary violations across all 3 parallel agents.

8. Final check: all 6 project goals (G1-G6 from Section 1) are met:
   - G1: Hybrid + re-ranking pipeline operational
   - G2: 3 chunking strategies produce indexed corpora
   - G3: HyDE and multi-query functional and benchmarked
   - G4: Streamlit dashboard shows recall@k, MRR, nDCG, precision@k
   - G5: POST /query accepts strategy parameter
   - G6: >15% nDCG variance between strategies (check eval_results)

Report PASS or list specific issues. This is the final gate before project completion.
```

#### Commit

```bash
git add -A && git commit -m "Phase 10: full test suite, README, end-to-end demo, type hints, cleanup"
```

---

## 5. Coordination Rules

### File Ownership Boundaries

| Directory / File | Owner | Notes |
|-----------------|-------|-------|
| `src/ingestion/` | core | loader, chunkers, embedder |
| `src/indexing/` | core | bm25_index, vector_index, store |
| `src/retrieval/` | core | hybrid, query_transform, reranker |
| `src/generation/` | core | llm |
| `src/rag_pipeline.py` | core | full pipeline orchestrator |
| `src/evaluation/` | eval | metrics, runner, golden_loader |
| `src/dashboard/` | eval | Streamlit app |
| `src/api/` | infra | main, routes, schemas |
| `src/config.py` | infra | Pydantic settings |
| `src/__init__.py` (root) | infra | package init |
| `docker-compose.yml` | infra | |
| `Dockerfile` | infra | |
| `pyproject.toml` | infra | |
| `.env.example` | infra | |
| `sql/schema.sql` | infra | |
| `scripts/init_db.py` | infra | |
| `scripts/ingest.py` | core | |
| `scripts/run_eval.py` | eval | |
| `scripts/create_golden.py` | eval | |
| `data/raw/` | core | sample corpus |
| `data/golden/` | eval | golden dataset |
| `tests/conftest.py` | infra | shared fixtures |
| `tests/test_api.py` | infra | |
| `tests/test_chunkers.py` | core | |
| `tests/test_embedder.py` | core | |
| `tests/test_bm25_index.py` | core | |
| `tests/test_vector_index.py` | core | |
| `tests/test_hybrid.py` | core | |
| `tests/test_query_transform.py` | core | |
| `tests/test_reranker.py` | core | |
| `tests/test_metrics.py` | eval | |
| `tests/test_evaluation_runner.py` | eval | |
| `README.md` | infra | |

### Conflict Avoidance Rules

1. **Never edit another agent's files.** If you need a change in another agent's module, message them via `herdr agent send`.
2. **Package `__init__.py` files** are owned by infra. If core or eval needs to export something from their package, ask infra to update the init or create a local one — infra created them in Phase 1.
3. **`src/rag_pipeline.py`** is owned by core but imported by eval (runner) and infra (API). If the interface changes, core must notify eval and infra via hub messaging before committing.
4. **`scripts/ingest.py`** is owned by core but infra's Dockerfile runs it. If CLI flags change, core must notify infra.
5. **`tests/conftest.py`** is owned by infra but used by all agents' tests. If fixtures change, infra must notify all agents.
6. **Database schema (`sql/schema.sql`)** is owned by infra. If core or eval needs schema changes (e.g., new column), request via hub messaging — infra applies the change.

### Parallel vs Sequential Rules

- **Parallel**: Phases 2, 3, 5, and 10 have parallel work (core + eval, or core + eval + infra) because the tasks touch non-overlapping files.
- **Sequential**: Phases 1, 4, 6, 7, 8, 9 are single-agent phases. The review gate must pass before the next phase begins.
- **Dependency chain**: Phase N depends on Phase N-1's review passing. Do NOT start Phase N+1 until the reviewer approves Phase N.
- **Cross-agent imports**: eval's runner imports core's RAGPipeline (Phase 8 depends on Phase 6). infra's API imports core's RAGPipeline (Phase 7 depends on Phase 6). These are read-only imports — no file conflicts.

### Blocked Agent Protocol

1. If an agent is blocked (e.g., needs a schema change from infra, or needs RAGPipeline interface from core), send a hub message to the owning agent:
   `herdr agent send <agent_name> "Blocked: need <specific change> in <file>. Details: <description>"`
2. While waiting, the blocked agent should work on any non-blocked tasks within their phase.
3. If the blocker is the reviewer rejecting a phase, the owning agent fixes the issues and re-requests review.
4. If an agent cannot proceed at all, they should yield with a status report rather than stalling silently.

---

## 6. Quick Reference

### Herdr Commands

```bash
# Pane management
herdr pane list                          # list all panes
herdr pane split --cwd "$PWD" --no-focus # split a new pane
herdr pane focus <id>                    # focus a pane

# Agent management
herdr agent start <name> --kind codex --pane <id>  # start an agent
herdr agent list                                   # list running agents
herdr agent stop <name>                            # stop an agent
herdr agent send <name> "<message>"               # send a message to an agent

# Monitoring
herdr agent logs <name>                  # view agent output
herdr agent status <name>                # check agent status
```

### Project Verification Cheat Sheet

```bash
# Infrastructure
docker compose up -d postgres
docker compose exec postgres psql -U rag -d ragdb -c "SELECT extname FROM pg_extension;"
python scripts/init_db.py

# Ingestion
python scripts/ingest.py --source data/raw/ --strategy all
docker compose exec postgres psql -U rag -d ragdb -c "SELECT chunking_strategy, COUNT(*) FROM chunks GROUP BY 1;"

# Evaluation
python scripts/run_eval.py --golden data/golden/golden_dataset.jsonl
docker compose exec postgres psql -U rag -d ragdb -c "SELECT COUNT(*) FROM eval_results;"

# API
uvicorn src.api.main:app --reload --port 8000
curl http://localhost:8000/health
curl -X POST http://localhost:8000/query -H "Content-Type: application/json" -d '{"query": "What makes a good startup idea?"}' | python -m json.tool

# Dashboard
streamlit run src/dashboard/app.py --server.port 8501

# Tests
pytest tests/ -v                                    # all tests
pytest tests/ -v -k "not integration and not e2e"   # unit only
pytest tests/ --cov=src --cov-report=term-missing   # with coverage
```

### Phase Dependency Graph

```
Phase 1 (infra)
    │
    ▼
Phase 2 (core + eval ∥)
    │
    ▼
Phase 3 (core + eval ∥)
    │
    ▼
Phase 4 (core)
    │
    ▼
Phase 5 (core + eval ∥)
    │
    ▼
Phase 6 (core)  ← RAGPipeline complete, eval & infra can now import it
    │
    ▼
Phase 7 (infra)  ← depends on Phase 6 (RAGPipeline)
    │
    ▼
Phase 8 (eval)   ← depends on Phase 6 (RAGPipeline) + Phase 7 (API)
    │
    ▼
Phase 9 (eval)   ← depends on Phase 8 (eval_results)
    │
    ▼
Phase 10 (core + eval + infra ∥)  ← final polish
```

### Key Interfaces (Cross-Agent Contracts)

| Interface | Owner | Consumers | Defined In |
|-----------|-------|-----------|------------|
| `RAGPipeline.query()` | core | eval (runner), infra (API) | Section 7.9 |
| `RetrievalResult` dataclass | core | eval (runner) | Section 7.5 |
| `RetrievalMetrics` | eval | core (not directly, but metrics feed back) | Section 7.11 |
| `Embedder` | core | eval (runner via pipeline), infra (API via pipeline) | Section 7.2 |
| `LLMClient` | core | eval (runner via pipeline), infra (API via pipeline) | Section 7.10 |
| DB schema (tables, columns) | infra | core (ingest), eval (eval_results) | Section 4 |
| `Settings` (config) | infra | all agents | Section 13.3 |
| `conftest.py` fixtures | infra | all test files | Section 12.5 |
