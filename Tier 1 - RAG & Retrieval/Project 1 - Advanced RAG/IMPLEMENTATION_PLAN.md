# Advanced RAG with Hybrid Search + Re-ranking

## Implementation Plan

> **Tier 1 — Project 1** | Estimated duration: 10 working days (~2 weeks)
> Target audience: Data engineer / analytics engineer transitioning to AI engineering

---

## Table of Contents

1. [Project Goals & Non-Goals](#1-project-goals--non-goals)
2. [Architecture Overview](#2-architecture-overview)
3. [Tech Stack with Rationale](#3-tech-stack-with-rationale)
4. [Database & Schema Design](#4-database--schema-design)
5. [Project Structure](#5-project-structure)
6. [Implementation Phases](#6-implementation-phases)
7. [Component Specifications](#7-component-specifications)
8. [Comparison Tables](#8-comparison-tables)
9. [Retrieval Evaluation](#9-retrieval-evaluation)
10. [API Specification](#10-api-specification)
11. [Security & Safety Considerations](#11-security--safety-considerations)
12. [Testing Strategy](#12-testing-strategy)
13. [Deployment](#13-deployment)
14. [Roadmap & Milestones](#14-roadmap--milestones)
15. [Risk Register](#15-risk-register)
16. [Appendix](#16-appendix)

---

## 1. Project Goals & Non-Goals

### Goals

| # | Goal | Success Criterion |
|---|------|-------------------|
| G1 | Build an end-to-end RAG pipeline that goes **beyond naive vector search** | Hybrid (BM25 + vector) + cross-encoder re-ranking pipeline fully operational |
| G2 | Implement and compare **three chunking strategies** | Fixed-size, semantic (spaCy), and sentence-window all produce indexed corpora |
| G3 | Implement **query transformation** techniques | HyDE and multi-query expansion both functional and benchmarked |
| G4 | Build a **retrieval evaluation dashboard** | Streamlit app displays recall@k, MRR, nDCG, precision@k across all strategy combinations |
| G5 | Expose a **FastAPI query endpoint** with configurable retrieval strategy | `POST /query` accepts `strategy` parameter and returns ranked results + answer |
| G6 | Demonstrate that **retrieval quality, not the LLM call, is the hard part of RAG** | Evaluation dashboard shows >15% nDCG variance between strategies on the same dataset |

### Non-Goals

| # | Non-Goal | Rationale |
|---|----------|-----------|
| NG1 | Production-scale serving (thousands of QPS) | Focus is on retrieval quality, not serving infrastructure |
| NG2 | Fine-tuning embedding models | Out of scope; we use pre-trained models (BGE, OpenAI embeddings) |
| NG3 | Multi-modal RAG (images, tables, PDF layout) | Text-only corpus for this project |
| NG4 | Streaming responses / SSE | Synchronous request-response is sufficient for evaluation |
| NG5 | User authentication & multi-tenancy | Single-user evaluation tool, not a SaaS product |
| NG6 | Automatic re-indexing pipelines / CDC | Manual re-index trigger is sufficient for evaluation |

---

## 2. Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                              ADVANCED RAG SYSTEM                                  │
├──────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌─────────────────────── INGESTION PIPELINE ───────────────────────────┐        │
│  │                                                                       │        │
│  │  ┌──────────┐   ┌──────────────┐   ┌──────────────────────────┐      │        │
│  │  │ Document │──▶│  Chunking    │──▶│  Chunking Strategies:    │      │        │
│  │  │  Loader  │   │  Orchestrator│   │  • Fixed (1000/200)      │      │        │
│  │  │          │   │              │   │  • Semantic (spaCy)      │      │        │
│  │  │ .txt     │   │              │   │  • Sentence-window (3)   │      │        │
│  │  │ .md      │   │              │   └──────────┬───────────────┘      │        │
│  │  │ .pdf     │   │              │              │                      │        │
│  │  └──────────┘   └──────────────┘              │                      │        │
│  │                                               ▼                      │        │
│  │                                    ┌──────────────────────┐           │        │
│  │                                    │  Embedding Generator │           │        │
│  │                                    │  (BGE-large-en-v1.5) │           │        │
│  │                                    └──────────┬───────────┘           │        │
│  │                                               │                      │        │
│  └───────────────────────────────────────────────┼──────────────────────┘        │
│                                                  │                               │
│                    ┌─────────────────────────────┼───────────────────┐           │
│                    │            DUAL INDEXING     │                   │           │
│                    │                               ▼                   │           │
│                    │  ┌─────────────────┐   ┌──────────────────┐      │           │
│                    │  │  BM25 Index     │   │  Vector Index    │      │           │
│                    │  │  (PostgreSQL    │   │  (PostgreSQL     │      │           │
│                    │  │   tsvector)     │   │   pgvector       │      │           │
│                    │  │                 │   │   HNSW)          │      │           │
│                    │  └────────┬────────┘   └────────┬─────────┘      │           │
│                    └───────────┼─────────────────────┼────────────────┘           │
│                                │                     │                            │
│  ┌─────────────────── QUERY PIPELINE ────────────────┼────────────────┐           │
│  │                                                   │                │           │
│  │  ┌──────────┐    ┌──────────────────────┐         │                │           │
│  │  │  User    │───▶│  Query Transform     │         │                │           │
│  │  │  Query   │    │  ┌────────────────┐  │         │                │           │
│  │  │          │    │  │ HyDE           │  │         │                │           │
│  │  │          │    │  │  (generate     │  │         │                │           │
│  │  │          │    │  │   hypothetical │  │         │                │           │
│  │  │          │    │  │   doc, embed)  │  │         │                │           │
│  │  │          │    │  ├────────────────┤  │         │                │           │
│  │  │          │    │  │ Multi-Query    │  │         │                │           │
│  │  │          │    │  │  (3            │  │         │                │           │
│  │  │          │    │  │   reformulations)│ │         │                │           │
│  │  │          │    │  └────────────────┘  │         │                │           │
│  │  └──────────┘    └─────────┬────────────┘         │                │           │
│  │                            │                      │                │           │
│  │                            ▼                      ▼                │           │
│  │                   ┌──────────────────────────────────┐            │           │
│  │                   │     HYBRID RETRIEVAL              │            │           │
│  │                   │  BM25 top-50 + Vector top-50      │            │           │
│  │                   │  → Reciprocal Rank Fusion (RRF)   │            │           │
│  │                   │  → Top-20 candidates              │            │           │
│  │                   └──────────────┬───────────────────┘            │           │
│  │                                  │                               │           │
│  │                                  ▼                               │           │
│  │                   ┌──────────────────────────────────┐            │           │
│  │                   │   CROSS-ENCODER RE-RANKING        │            │           │
│  │                   │   BGE-reranker-large              │            │           │
│  │                   │   Top-20 → Top-5                  │            │           │
│  │                   └──────────────┬───────────────────┘            │           │
│  │                                  │                               │           │
│  │                                  ▼                               │           │
│  │                   ┌──────────────────────────────────┐            │           │
│  │                   │   LLM GENERATION                  │            │           │
│  │                   │   (top-5 context → answer)        │            │           │
│  │                   └──────────────┬───────────────────┘            │           │
│  │                                  │                               │           │
│  └──────────────────────────────────┼───────────────────────────────┘           │
│                                     │                                            │
│                                     ▼                                            │
│  ┌──────────────────── EVALUATION LAYER ─────────────────────────────┐           │
│  │                                                                   │           │
│  │  ┌──────────────────┐    ┌───────────────────────────────────┐   │           │
│  │  │  Golden Dataset  │───▶│  Evaluation Runner                │   │           │
│  │  │  (Q&A pairs +    │    │  (runs all strategy combinations  │   │           │
│  │  │   relevance      │    │   across chunking × retrieval ×   │   │           │
│  │  │   judgments)     │    │   transform × rerank)             │   │           │
│  │  └──────────────────┘    └──────────────┬────────────────────┘   │           │
│  │                                         │                        │           │
│  │                                         ▼                        │           │
│  │                          ┌──────────────────────────────────┐   │           │
│  │                          │  Streamlit Evaluation Dashboard   │   │           │
│  │                          │  • recall@k, MRR, nDCG, P@k      │   │           │
│  │                          │  • Strategy comparison tables    │   │           │
│  │                          │  • Per-query drill-down          │   │           │
│  │                          └──────────────────────────────────┘   │           │
│  └─────────────────────────────────────────────────────────────────┘           │
│                                                                                  │
└──────────────────────────────────────────────────────────────────────────────────┘

External Services:
  ┌────────────────┐     ┌──────────────────┐     ┌─────────────────────┐
  │  PostgreSQL    │     │  LLM API         │     │  HuggingFace Hub    │
  │  + pgvector    │     │  (OpenAI / local │     │  (model download)   │
  │  + tsvector    │     │   Ollama)        │     │                     │
  └────────────────┘     └──────────────────┘     └─────────────────────┘
```

### Data Flow Summary

```
INGEST:  Documents → Chunk (3 strategies) → Embed → Store (BM25 + pgvector)
QUERY:   User query → Transform (HyDE/MultiQuery/none) → Hybrid retrieve → Re-rank → LLM answer
EVAL:    Golden Q&A → Run all combos → Compute metrics → Streamlit dashboard
```

---

## 3. Tech Stack with Rationale

| Layer | Technology | Version | Rationale |
|-------|-----------|---------|-----------|
| **Language** | Python | 3.11+ | Type hints, `TaskGroup`, speed; standard for ML/AI |
| **Web Framework** | FastAPI | 0.110+ | Async, auto-docs (OpenAPI), type-safe with Pydantic v2 |
| **Database** | PostgreSQL | 16 | Already familiar from data engineering; single system for both BM25 (tsvector) and vector (pgvector) |
| **Vector Extension** | pgvector | 0.7+ | Keeps everything in Postgres — no separate vector DB to manage; HNSW index for ANN search |
| **Embedding Model** | BAAI/bge-large-en-v1.5 | — | Open-source, top-tier MTEB scores, runs locally; 1024-dim vectors |
| **Re-ranker** | BAAI/bge-reranker-large | — | Cross-encoder, open-source, strong re-ranking benchmarks; alternative: Cohere Rerank API |
| **NLP Segmentation** | spaCy | 3.7+ | Production-grade sentence boundary detection; `en_core_web_sm` model |
| **LLM Orchestration** | LangChain | 0.2+ | Widely used, good abstractions for query transformation; LlamaIndex is an alternative |
| **LLM** | OpenAI `gpt-4o-mini` or local Ollama (`llama3.1:8b`) | — | `gpt-4o-mini` for quality/latency; Ollama for zero-cost local development |
| **Eval Dashboard** | Streamlit | 1.35+ | Fastest path to interactive data dashboards; familiar to data analysts |
| **Containerization** | Docker + Docker Compose | — | Reproducible environment; Postgres + pgvector + app in one `docker compose up` |
| **Testing** | pytest | 8.0+ | Standard Python testing; `pytest-asyncio` for async endpoints |
| **Data Validation** | Pydantic | 2.6+ | Schema validation for API and internal data models |
| **HTTP Client** | httpx | 0.27+ | Async HTTP for LLM API calls; replaces `requests` in async context |
| **Experiment Tracking** | Simple JSON/CSV logging | — | Lightweight; no MLflow overhead for this project scope |

### Why pgvector over Qdrant?

For a first RAG project, **pgvector** is the recommended choice:
- **Single infrastructure**: Postgres handles both BM25 (`tsvector`) and vector search (`pgvector`) — one container, one connection, one backup strategy.
- **Familiarity**: Data engineers already know Postgres — SQL, indexes, transactions, `EXPLAIN ANALYZE`.
- **Sufficient scale**: For evaluation corpora (1K–10K chunks), pgvector's HNSW index is fast enough.
- **Trade-off**: At >1M vectors with high QPS, a dedicated vector DB (Qdrant, Weaviate) becomes necessary. This is noted as a future migration path.

> **Alternative**: If you prefer a dedicated vector database or plan to scale beyond 100K chunks, swap pgvector for Qdrant. The `VectorStore` interface (Section 7.5) abstracts this — only the implementation class changes.

---

## 4. Database & Schema Design

### PostgreSQL Schema

```sql
-- Enable extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "vector";       -- pgvector
CREATE EXTENSION IF NOT EXISTS "pg_trgm";      -- trigram fuzzy search (optional)

-- ============================================================
-- Documents: source documents before chunking
-- ============================================================
CREATE TABLE documents (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    source_path     TEXT NOT NULL,               -- original file path
    title           TEXT NOT NULL,
    doc_type        TEXT NOT NULL CHECK (doc_type IN ('txt', 'md', 'pdf', 'html')),
    content_hash    TEXT NOT NULL UNIQUE,        -- SHA-256 of raw content (dedup)
    char_count      INTEGER NOT NULL,
    metadata        JSONB NOT NULL DEFAULT '{}', -- author, date, tags, etc.
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_documents_content_hash ON documents(content_hash);
CREATE INDEX idx_documents_metadata_gin ON documents USING GIN(metadata);

-- ============================================================
-- Chunks: the actual indexed units (one row per chunk per strategy)
-- ============================================================
CREATE TABLE chunks (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    document_id     UUID NOT NULL REFERENCES documents(id) ON DELETE CASCADE,
    chunk_index     INTEGER NOT NULL,             -- position within document
    chunking_strategy TEXT NOT NULL CHECK (
        chunking_strategy IN ('fixed_1000_200', 'semantic_spacy', 'sentence_window_3')
    ),
    content         TEXT NOT NULL,                -- the chunk text
    token_count     INTEGER NOT NULL,
    -- BM25 full-text search column
    tsv             TSVECTOR GENERATED ALWAYS AS (
        to_tsvector('english', content)
    ) STORED,
    -- pgvector embedding (1024-dim for bge-large-en-v1.5)
    embedding       vector(1024),
    -- sentence-window: surrounding context for display
    window_context  TEXT,                         -- NULL for non-window strategies
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(document_id, chunking_strategy, chunk_index)
);

-- BM25-ish ranking via tsvector GIN index
CREATE INDEX idx_chunks_tsv ON chunks USING GIN(tsv);
CREATE INDEX idx_chunks_strategy ON chunks(chunking_strategy);
CREATE INDEX idx_chunks_doc_strategy ON chunks(document_id, chunking_strategy);

-- HNSW vector index (one per chunking strategy via partial index)
CREATE INDEX idx_chunks_embedding_fixed
    ON chunks USING hnsw (embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 64)
    WHERE chunking_strategy = 'fixed_1000_200';

CREATE INDEX idx_chunks_embedding_semantic
    ON chunks USING hnsw (embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 64)
    WHERE chunking_strategy = 'semantic_spacy';

CREATE INDEX idx_chunks_embedding_window
    ON chunks USING hnsw (embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 64)
    WHERE chunking_strategy = 'sentence_window_3';

-- ============================================================
-- Queries: logged user queries and their retrieval results
-- ============================================================
CREATE TABLE queries (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    query_text      TEXT NOT NULL,
    strategy_config JSONB NOT NULL,               -- which strategies were used
    retrieved_ids   UUID[] NOT NULL,              -- ordered list of chunk IDs
    final_answer    TEXT,
    latency_ms      INTEGER,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ============================================================
-- Golden Dataset: evaluation ground truth
-- ============================================================
CREATE TABLE golden_questions (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    question        TEXT NOT NULL,
    expected_answer TEXT NOT NULL,
    -- chunks that SHOULD be retrieved for this question
    -- (manually or LLM-judge assigned)
    relevant_chunk_ids UUID[] NOT NULL,
    -- relevance grades for nDCG: {chunk_id: grade (0-3)}
    relevance_grades  JSONB NOT NULL DEFAULT '{}',
    difficulty       TEXT CHECK (difficulty IN ('easy', 'medium', 'hard')),
    category         TEXT,                        -- topic category
    created_at       TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ============================================================
-- Eval Results: metrics per (question × strategy combination)
-- ============================================================
CREATE TABLE eval_results (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    golden_question_id UUID NOT NULL REFERENCES golden_questions(id) ON DELETE CASCADE,
    chunking_strategy   TEXT NOT NULL,
    retrieval_method   TEXT NOT NULL,             -- 'bm25', 'vector', 'hybrid'
    query_transform    TEXT NOT NULL,             -- 'none', 'hyde', 'multi_query'
    reranker           TEXT NOT NULL,             -- 'none', 'bge_reranker'
    retrieved_ids      UUID[] NOT NULL,
    recall_at_5        FLOAT,
    recall_at_10       FLOAT,
    precision_at_5     FLOAT,
    mrr                FLOAT,
    ndcg_at_5          FLOAT,
    ndcg_at_10         FLOAT,
    latency_ms         INTEGER,
    created_at         TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_eval_results_combo ON eval_results(
    chunking_strategy, retrieval_method, query_transform, reranker
);
```

### Entity Relationship

```
documents 1───∞ chunks
                    │
golden_questions ────∞ eval_results
queries (runtime log, no FK)
```

### Key Design Decisions

1. **One row per chunk per strategy**: The same document produces 3 sets of chunks (one per chunking strategy). This allows direct comparison on the same underlying content.
2. **`tsvector` as generated column**: Postgres auto-maintains the BM25 index — no application-side sync needed.
3. **Partial HNSW indexes**: One vector index per chunking strategy, so vector search only scans the relevant subset.
4. **`relevance_grades` as JSONB**: Enables graded relevance (0=irrelevant, 1=marginally, 2=relevant, 3=perfect) for nDCG, not just binary relevance.

---

## 5. Project Structure

```
advanced-rag/
├── docker-compose.yml
├── Dockerfile
├── .env.example
├── pyproject.toml
├── README.md
├── IMPLEMENTATION_PLAN.md          # ← this file
├── agent.md                      # Herdr multi-agent orchestration guide
│
├── data/
│   ├── raw/                         # source documents
│   │   ├── paul_graham_essays/
│   │   ├── arxiv_abstracts/
│   │   └── company_docs/
│   └── golden/
│       └── golden_dataset.jsonl     # evaluation Q&A pairs
│
├── src/
│   ├── __init__.py
│   ├── config.py                    # Pydantic settings (env-driven)
│   │
│   ├── ingestion/
│   │   ├── __init__.py
│   │   ├── loader.py                # document loaders (txt, md, pdf)
│   │   ├── chunkers.py              # 3 chunking strategies
│   │   └── embedder.py              # BGE embedding generation
│   │
│   ├── indexing/
│   │   ├── __init__.py
│   │   ├── bm25_index.py            # tsvector-based BM25 search
│   │   ├── vector_index.py          # pgvector HNSW search
│   │   └── store.py                 # unified VectorStore interface
│   │
│   ├── retrieval/
│   │   ├── __init__.py
│   │   ├── hybrid.py                # BM25 + vector + RRF fusion
│   │   ├── query_transform.py       # HyDE + multi-query expansion
│   │   └── reranker.py              # BGE cross-encoder re-ranking
│   │
│   ├── generation/
│   │   ├── __init__.py
│   │   └── llm.py                   # LLM answer generation from context
│   │
│   ├── evaluation/
│   │   ├── __init__.py
│   │   ├── metrics.py               # recall@k, MRR, nDCG, precision@k
│   │   ├── runner.py                # run all strategy combinations
│   │   └── golden_loader.py         # load/validate golden dataset
│   │
│   ├── api/
│   │   ├── __init__.py
│   │   ├── main.py                  # FastAPI app
│   │   ├── routes.py                # /query, /health, /ingest endpoints
│   │   └── schemas.py               # Pydantic request/response models
│   │
│   └── dashboard/
│       ├── __init__.py
│       └── app.py                   # Streamlit evaluation dashboard
│
├── scripts/
│   ├── ingest.py                    # CLI: ingest documents
│   ├── create_golden.py             # CLI: generate golden dataset
│   ├── run_eval.py                  # CLI: run evaluation suite
│   └── init_db.py                   # CLI: create schema + extensions
│
├── tests/
│   ├── __init__.py
│   ├── conftest.py                  # fixtures: test DB, sample docs
│   ├── test_chunkers.py
│   ├── test_embedder.py
│   ├── test_bm25_index.py
│   ├── test_vector_index.py
│   ├── test_hybrid.py
│   ├── test_query_transform.py
│   ├── test_reranker.py
│   ├── test_metrics.py
│   ├── test_api.py
│   └── test_evaluation_runner.py
│
├── sql/
│   └── schema.sql                   # full DDL (Section 4)
│
└── notebooks/
    └── exploration.ipynb            # ad-hoc analysis (optional)
```

---

## 6. Implementation Phases

### Phase Overview

| Phase | Days | Focus | Deliverable |
|-------|------|-------|-------------|
| 1 | Day 1 | Environment & schema | Docker Compose running, DB schema deployed |
| 2 | Day 2 | Document ingestion & chunking | 3 chunking strategies producing chunks |
| 3 | Day 3 | Embedding & dual indexing | BM25 + pgvector indexes populated |
| 4 | Day 4 | Hybrid retrieval + RRF | Combined search returning fused results |
| 5 | Day 5 | Query transformation (HyDE + multi-query) | Transform module functional |
| 6 | Day 6 | Cross-encoder re-ranking | BGE-reranker integrated, top-20→top-5 |
| 7 | Day 7 | FastAPI query endpoint | `/query` endpoint with configurable strategy |
| 8 | Day 8 | Evaluation metrics + golden dataset | Metrics computed, golden dataset created |
| 9 | Day 9 | Streamlit evaluation dashboard | Interactive dashboard showing all comparisons |
| 10 | Day 10 | Testing, polish, documentation | Full test suite, README, demo run |

---

### Phase 1 — Environment & Schema (Day 1)

**Goal**: Get the infrastructure running and the database schema deployed.

**Tasks**:
1. Create `docker-compose.yml` with PostgreSQL 16 + pgvector service
2. Create `Dockerfile` for the Python application
3. Create `pyproject.toml` with all dependencies
4. Create `.env.example` with all configuration variables
5. Write `sql/schema.sql` (from Section 4)
6. Write `scripts/init_db.py` to apply schema
7. Write `src/config.py` with Pydantic settings

**Deliverables**:
- `docker compose up` starts Postgres with pgvector extension
- `python scripts/init_db.py` creates all tables and indexes
- `src/config.py` loads settings from `.env`

**Verification**:
```bash
docker compose up -d postgres
docker compose exec postgres psql -U rag -d ragdb -c "SELECT extname FROM pg_extension;"
# Expected: vector, uuid-ossp, pg_trgm

python scripts/init_db.py
# Expected: "Schema created successfully"

docker compose exec postgres psql -U rag -d ragdb -c "\dt"
# Expected: documents, chunks, queries, golden_questions, eval_results
```

---

### Phase 2 — Document Ingestion & Chunking (Day 2)

**Goal**: Implement document loading and all three chunking strategies.

**Tasks**:
1. Write `src/ingestion/loader.py` — load `.txt`, `.md`, `.pdf` files
2. Write `src/ingestion/chunkers.py` — implement three chunking strategies
3. Download spaCy `en_core_web_sm` model
4. Prepare sample corpus in `data/raw/` (Paul Graham essays, arXiv abstracts, or company docs)
5. Write `scripts/ingest.py` CLI to chunk and store documents

**Deliverables**:
- Three chunking strategies produce chunks stored in `chunks` table
- Each strategy can be run independently via CLI flag

**Verification**:
```bash
python scripts/ingest.py --source data/raw/ --strategy fixed_1000_200
python scripts/ingest.py --source data/raw/ --strategy semantic_spacy
python scripts/ingest.py --source data/raw/ --strategy sentence_window_3

# Verify chunk counts per strategy
docker compose exec postgres psql -U rag -d ragdb -c \
  "SELECT chunking_strategy, COUNT(*) FROM chunks GROUP BY 1;"
# Expected: 3 rows with different counts (fixed ≈ semantic > window)
```

---

### Phase 3 — Embedding & Dual Indexing (Day 3)

**Goal**: Generate embeddings and populate both BM25 and vector indexes.

**Tasks**:
1. Write `src/ingestion/embedder.py` — load BGE model, batch-embed chunks
2. Update `scripts/ingest.py` to embed chunks after chunking
3. Write `src/indexing/bm25_index.py` — tsvector search with `ts_rank_cd`
4. Write `src/indexing/vector_index.py` — pgvector cosine similarity search
5. Write `src/indexing/store.py` — unified `VectorStore` interface

**Deliverables**:
- All chunks have embeddings (1024-dim) stored in `chunks.embedding`
- BM25 search returns ranked results via `ts_rank_cd`
- Vector search returns ranked results via cosine distance

**Verification**:
```bash
# Verify embeddings populated
docker compose exec postgres psql -U rag -d ragdb -c \
  "SELECT chunking_strategy, COUNT(embedding) FROM chunks GROUP BY 1;"

# Test BM25 search
docker compose exec postgres psql -U rag -d ragdb -c \
  "SELECT id, ts_rank_cd(tsv, query) AS score
   FROM chunks, plainto_tsquery('english', 'startup advice') query
   WHERE tsv @@ query AND chunking_strategy = 'fixed_1000_200'
   ORDER BY score DESC LIMIT 5;"

# Test vector search (using a dummy vector)
python -c "
from src.indexing.vector_index import VectorIndex
vi = VectorIndex()
results = vi.search(query_text='how to start a startup', strategy='fixed_1000_200', k=5)
print(results)
"
```

---

### Phase 4 — Hybrid Retrieval + RRF (Day 4)

**Goal**: Combine BM25 and vector search with Reciprocal Rank Fusion.

**Tasks**:
1. Write `src/retrieval/hybrid.py` — implement RRF fusion
2. Implement configurable retrieval: `bm25_only`, `vector_only`, `hybrid`
3. Add unit tests for RRF fusion logic

**Deliverables**:
- `HybridRetriever` class that combines results from both indexes
- RRF merges top-50 from each into a single ranked list of top-20

**Verification**:
```bash
python -c "
from src.retrieval.hybrid import HybridRetriever
r = HybridRetriever()
results = r.retrieve('how to start a startup', strategy='fixed_1000_200', k=20)
print(f'Hybrid returned {len(results)} results')
for r in results[:5]:
    print(f'  score={r.rrf_score:.4f}  source={r.source}  text={r.content[:80]}...')
"
# Expected: 20 results, each with rrf_score and source ('bm25', 'vector', or 'both')
```

---

### Phase 5 — Query Transformation (Day 5)

**Goal**: Implement HyDE and multi-query expansion.

**Tasks**:
1. Write `src/retrieval/query_transform.py`
2. Implement HyDE: generate hypothetical answer → embed → search
3. Implement multi-query: LLM generates 3 reformulations → search all → merge
4. Write `src/generation/llm.py` — LLM client wrapper for transformations and answers
5. Add unit tests with mocked LLM responses

**Deliverables**:
- `HyDETransformer` and `MultiQueryTransformer` classes
- Both can be used as drop-in replacements for raw query embedding

**Verification**:
```bash
python -c "
from src.retrieval.query_transform import HyDETransformer, MultiQueryTransformer

hyde = HyDETransformer()
embedding = hyde.transform('How do I validate a startup idea?')
print(f'HyDE embedding dim: {len(embedding)}')

mq = MultiQueryTransformer()
queries = mq.transform('How do I validate a startup idea?')
print(f'Multi-query generated: {queries}')
# Expected: original + 3 reformulations
"
```

---

### Phase 6 — Cross-Encoder Re-ranking (Day 6)

**Goal**: Re-rank top-20 candidates down to top-5 using BGE-reranker.

**Tasks**:
1. Write `src/retrieval/reranker.py` — load BGE-reranker-large
2. Implement `rerank(query, candidates, top_k=5)` method
3. Integrate reranker into the retrieval pipeline
4. Add option to disable reranking for comparison

**Deliverables**:
- `CrossEncoderReranker` class that re-scores candidates
- Pipeline: hybrid retrieve top-20 → rerank → top-5

**Verification**:
```bash
python -c "
from src.retrieval.hybrid import HybridRetriever
from src.retrieval.reranker import CrossEncoderReranker

retriever = HybridRetriever()
reranker = CrossEncoderReranker()

candidates = retriever.retrieve('how to start a startup', strategy='fixed_1000_200', k=20)
reranked = reranker.rerank('how to start a startup', candidates, top_k=5)

print('Before rerank (top 5 by RRF):')
for c in candidates[:5]:
    print(f'  {c.rrf_score:.4f}  {c.content[:60]}...')

print('After rerank (top 5 by cross-encoder):')
for c in reranked:
    print(f'  {c.rerank_score:.4f}  {c.content[:60]}...')
"
```

---

### Phase 7 — FastAPI Query Endpoint (Day 7)

**Goal**: Expose the full RAG pipeline via a configurable API.

**Tasks**:
1. Write `src/api/schemas.py` — Pydantic models for request/response
2. Write `src/api/routes.py` — `/query`, `/health`, `/ingest` endpoints
3. Write `src/api/main.py` — FastAPI app with lifespan, CORS
4. Add integration tests for the API

**Deliverables**:
- `POST /query` accepts strategy configuration and returns answer + retrieved chunks
- `GET /health` returns service status
- OpenAPI docs at `/docs`

**Verification**:
```bash
# Start the API
uvicorn src.api.main:app --reload --port 8000

# Test health
curl http://localhost:8000/health
# Expected: {"status": "healthy", "db": "connected"}

# Test query
curl -X POST http://localhost:8000/query \
  -H "Content-Type: application/json" \
  -d '{
    "query": "How do I validate a startup idea?",
    "chunking_strategy": "semantic_spacy",
    "retrieval_method": "hybrid",
    "query_transform": "hyde",
    "reranker": "bge_reranker",
    "top_k": 5
  }' | python -m json.tool

# Expected: JSON with answer, retrieved_chunks[], latency_ms, strategy_config
```

---

### Phase 8 — Evaluation Metrics & Golden Dataset (Day 8)

**Goal**: Build the evaluation framework and create a golden dataset.

**Tasks**:
1. Write `src/evaluation/metrics.py` — implement recall@k, precision@k, MRR, nDCG
2. Write `src/evaluation/golden_loader.py` — load and validate golden Q&A pairs
3. Write `src/evaluation/runner.py` — run all strategy combinations
4. Create `data/golden/golden_dataset.jsonl` with 30–50 Q&A pairs
5. Write `scripts/run_eval.py` CLI to run the full evaluation suite
6. Store results in `eval_results` table

**Deliverables**:
- All 4 metrics implemented and unit-tested
- Golden dataset with relevant chunk IDs and graded relevance
- Evaluation runner produces results for all strategy combinations

**Strategy Combinations Evaluated**:
```
3 chunking strategies × 3 retrieval methods × 3 query transforms × 2 reranker settings
= 54 combinations × 30-50 questions = 1,620–2,700 eval runs
```

**Verification**:
```bash
python scripts/run_eval.py --golden data/golden/golden_dataset.jsonl

# Verify results stored
docker compose exec postgres psql -U rag -d ragdb -c \
  "SELECT chunking_strategy, retrieval_method, query_transform, reranker,
          AVG(ndcg_at_5) as avg_ndcg5, AVG(mrr) as avg_mrr
   FROM eval_results GROUP BY 1,2,3,4 ORDER BY avg_ndcg5 DESC LIMIT 10;"
# Expected: 54 rows, hybrid + hyde + reranker near the top
```

---

### Phase 9 — Streamlit Evaluation Dashboard (Day 9)

**Goal**: Build an interactive dashboard to compare retrieval strategies.

**Tasks**:
1. Write `src/dashboard/app.py` — Streamlit app
2. Add overall metrics comparison table (sortable, filterable)
3. Add per-query drill-down view
4. Add strategy comparison bar charts (Plotly)
5. Add latency vs. quality trade-off scatter plot

**Deliverables**:
- `streamlit run src/dashboard/app.py` launches the dashboard
- Dashboard reads from `eval_results` table
- Shows side-by-side comparison of all strategy combinations

**Dashboard Sections**:
1. **Overview**: Best strategy per metric, summary statistics
2. **Strategy Comparison**: Filterable table + bar charts of all 54 combinations
3. **Chunking Comparison**: Isolate chunking strategy effect (hold others constant)
4. **Query Transform Comparison**: HyDE vs. multi-query vs. none
5. **Reranker Impact**: With vs. without cross-encoder re-ranking
6. **Per-Query Drill-Down**: Select a question, see retrieved chunks vs. relevant chunks
7. **Latency vs. Quality**: Scatter plot of latency_ms vs. nDCG@5

**Verification**:
```bash
streamlit run src/dashboard/app.py --server.port 8501
# Open http://localhost:8501
# Verify: all 7 sections render, tables populate from eval_results, charts display
```

---

### Phase 10 — Testing, Polish & Documentation (Day 10)

**Goal**: Achieve full test coverage, clean code, and usable documentation.

**Tasks**:
1. Write/complete all unit tests in `tests/`
2. Write integration tests for the full pipeline (ingest → query → answer)
3. Write `README.md` with quick start, architecture summary, key findings
4. Add type hints to all public functions
5. Run `pytest` with coverage, fix any failing tests
6. Clean up dead code, TODOs, unused imports
7. Final `docker compose up` end-to-end smoke test

**Deliverables**:
- `pytest tests/` passes with >80% coverage on `src/`
- `README.md` with quick start guide
- Full end-to-end demo: ingest → query → evaluate → dashboard

**Verification**:
```bash
pytest tests/ --cov=src --cov-report=term-missing
# Expected: all tests pass, coverage > 80%

# End-to-end smoke test
docker compose up -d
python scripts/init_db.py
python scripts/ingest.py --source data/raw/ --strategy all
python scripts/run_eval.py --golden data/golden/golden_dataset.jsonl
curl -X POST http://localhost:8000/query -H "Content-Type: application/json" \
  -d '{"query": "What makes a good startup idea?"}' | python -m json.tool
# Expected: valid JSON response with answer and retrieved chunks
```

---

## 7. Component Specifications

### 7.1 Chunking Strategies (`src/ingestion/chunkers.py`)

#### Fixed-Size Chunker

```python
class FixedSizeChunker:
    """Splits text into fixed-size chunks with overlap.

    Args:
        chunk_size: Number of characters per chunk (default 1000)
        overlap: Number of overlapping characters between chunks (default 200)
    """

    def __init__(self, chunk_size: int = 1000, overlap: int = 200):
        self.chunk_size = chunk_size
        self.overlap = overlap
        assert overlap < chunk_size, "Overlap must be less than chunk size"

    def chunk(self, text: str) -> list[Chunk]:
        chunks = []
        start = 0
        idx = 0
        while start < len(text):
            end = min(start + self.chunk_size, len(text))
            # Try to break at a sentence boundary near the end
            end = self._adjust_to_sentence_boundary(text, start, end)
            content = text[start:end].strip()
            if content:
                chunks.append(Chunk(
                    content=content,
                    chunk_index=idx,
                    chunking_strategy="fixed_1000_200",
                    token_count=len(content.split()),
                ))
                idx += 1
            start = end - self.overlap if end < len(text) else len(text)
        return chunks

    def _adjust_to_sentence_boundary(self, text: str, start: int, end: int) -> int:
        """Look for '. ' within the last 20% of the chunk to avoid mid-sentence cuts."""
        search_start = start + int(self.chunk_size * 0.8)
        last_period = text.rfind('. ', search_start, end)
        return last_period + 1 if last_period != -1 else end
```

#### Semantic Chunker (spaCy)

```python
import spacy

class SemanticChunker:
    """Chunks text at sentence boundaries using spaCy.

    Groups sentences into chunks of approximately target_size characters,
    but never breaks a sentence across chunks.

    Args:
        target_size: Target chunk size in characters (default 1000)
        nlp_model: spaCy model name (default 'en_core_web_sm')
    """

    def __init__(self, target_size: int = 1000, nlp_model: str = "en_core_web_sm"):
        self.target_size = target_size
        self.nlp = spacy.load(nlp_model)

    def chunk(self, text: str) -> list[Chunk]:
        doc = self.nlp(text)
        sentences = [sent.text.strip() for sent in doc.sents if sent.text.strip()]

        chunks = []
        current_sentences = []
        current_size = 0

        for sentence in sentences:
            if current_size + len(sentence) > self.target_size and current_sentences:
                # Flush current chunk
                content = ' '.join(current_sentences)
                chunks.append(Chunk(
                    content=content,
                    chunk_index=len(chunks),
                    chunking_strategy="semantic_spacy",
                    token_count=len(content.split()),
                ))
                current_sentences = []
                current_size = 0
            current_sentences.append(sentence)
            current_size += len(sentence)

        # Don't forget the last chunk
        if current_sentences:
            content = ' '.join(current_sentences)
            chunks.append(Chunk(
                content=content,
                chunk_index=len(chunks),
                chunking_strategy="semantic_spacy",
                token_count=len(content.split()),
            ))
        return chunks
```

#### Sentence-Window Chunker

```python
import spacy

class SentenceWindowChunker:
    """Each chunk is a single sentence, but stores surrounding context.

    At retrieval time, when a sentence chunk is matched, the surrounding
    N sentences are included in the context sent to the LLM, providing
    broader context without inflating the embedding granularity.

    Args:
        window_size: Number of sentences before/after to include as context (default 3)
    """

    def __init__(self, window_size: int = 3, nlp_model: str = "en_core_web_sm"):
        self.window_size = window_size
        self.nlp = spacy.load(nlp_model)

    def chunk(self, text: str) -> list[Chunk]:
        doc = self.nlp(text)
        sentences = [sent.text.strip() for sent in doc.sents if sent.text.strip()]
        n = len(sentences)

        chunks = []
        for i, sentence in enumerate(sentences):
            # Window: surrounding sentences for context display
            start = max(0, i - self.window_size)
            end = min(n, i + self.window_size + 1)
            window_context = ' '.join(sentences[start:end])

            chunks.append(Chunk(
                content=sentence,              # only the sentence is embedded
                chunk_index=i,
                chunking_strategy="sentence_window_3",
                token_count=len(sentence.split()),
                window_context=window_context,  # full context for LLM
                metadata={"sentence_index": i, "total_sentences": n},
            ))
        return chunks
```

### 7.2 Embedding Generator (`src/ingestion/embedder.py`)

```python
from sentence_transformers import SentenceTransformer
import numpy as np
from typing import Sequence

class Embedder:
    """Generates embeddings using BGE-large-en-v1.5.

    The BGE model requires a specific query instruction prefix for retrieval
    queries (not for passages). This is handled in embed_query vs. embed_passages.
    """

    MODEL_NAME = "BAAI/bge-large-en-v1.5"
    QUERY_INSTRUCTION = "Represent this sentence for searching relevant passages: "
    DIMENSION = 1024

    def __init__(self, model_name: str | None = None, device: str = "cpu"):
        model = model_name or self.MODEL_NAME
        self.model = SentenceTransformer(model, device=device)
        self.device = device

    def embed_passages(self, texts: Sequence[str], batch_size: int = 32) -> np.ndarray:
        """Embed document chunks (no instruction prefix)."""
        embeddings = self.model.encode(
            texts,
            batch_size=batch_size,
            normalize_embeddings=True,   # BGE recommends L2 normalization
            show_progress_bar=True,
        )
        return embeddings

    def embed_query(self, query: str) -> np.ndarray:
        """Embed a search query (with instruction prefix)."""
        instructed = self.QUERY_INSTRUCTION + query
        embedding = self.model.encode(
            [instructed],
            normalize_embeddings=True,
        )
        return embedding[0]

    def embed_queries(self, queries: Sequence[str]) -> np.ndarray:
        """Embed multiple queries (with instruction prefix)."""
        instructed = [self.QUERY_INSTRUCTION + q for q in queries]
        embeddings = self.model.encode(
            instructed,
            normalize_embeddings=True,
            show_progress_bar=True,
        )
        return embeddings
```

### 7.3 BM25 Index (`src/indexing/bm25_index.py`)

```python
import asyncpg
from dataclasses import dataclass

@dataclass
class BM25Result:
    chunk_id: str
    content: str
    score: float
    rank: int

class BM25Index:
    """BM25-style full-text search using PostgreSQL tsvector + ts_rank_cd.

    Note: PostgreSQL's ts_rank_cd uses cover density ranking, which is a
    proximity-based score similar to but not identical to classic BM25.
    For true BM25, consider the 'rum' extension or application-side BM25
    (e.g., rank_bm25 library). For this project, ts_rank_cd is sufficient
    and keeps everything in Postgres.
    """

    def __init__(self, pool: asyncpg.Pool):
        self.pool = pool

    async def search(
        self,
        query: str,
        chunking_strategy: str,
        k: int = 50,
    ) -> list[BM25Result]:
        sql = """
            SELECT id, content, ts_rank_cd(tsv, plainto_tsquery('english', $1)) AS score
            FROM chunks
            WHERE tsv @@ plainto_tsquery('english', $1)
              AND chunking_strategy = $2
            ORDER BY score DESC
            LIMIT $3
        """
        async with self.pool.acquire() as conn:
            rows = await conn.fetch(sql, query, chunking_strategy, k)

        return [
            BM25Result(
                chunk_id=str(row['id']),
                content=row['content'],
                score=float(row['score']),
                rank=i + 1,
            )
            for i, row in enumerate(rows)
        ]
```

### 7.4 Vector Index (`src/indexing/vector_index.py`)

```python
import asyncpg
import numpy as np
from dataclasses import dataclass

@dataclass
class VectorResult:
    chunk_id: str
    content: str
    score: float       # cosine similarity (higher = better)
    rank: int

class VectorIndex:
    """Vector similarity search using pgvector with HNSW index."""

    def __init__(self, pool: asyncpg.Pool, embedder):
        self.pool = pool
        self.embedder = embedder

    async def search(
        self,
        query_embedding: np.ndarray,
        chunking_strategy: str,
        k: int = 50,
    ) -> list[VectorResult]:
        # pgvector uses <=> for cosine distance; similarity = 1 - distance
        sql = """
            SELECT id, content,
                   1 - (embedding <=> $1::vector) AS similarity
            FROM chunks
            WHERE chunking_strategy = $2 AND embedding IS NOT NULL
            ORDER BY embedding <=> $1::vector
            LIMIT $3
        """
        embedding_str = self._format_vector(query_embedding)

        async with self.pool.acquire() as conn:
            rows = await conn.fetch(sql, embedding_str, chunking_strategy, k)

        return [
            VectorResult(
                chunk_id=str(row['id']),
                content=row['content'],
                score=float(row['similarity']),
                rank=i + 1,
            )
            for i, row in enumerate(rows)
        ]

    @staticmethod
    def _format_vector(vec: np.ndarray) -> str:
        """Format numpy array as pgvector string: '[0.1,0.2,...]'"""
        return '[' + ','.join(f'{x:.8f}' for x in vec) + ']'
```

### 7.5 Unified Store Interface (`src/indexing/store.py`)

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass

@dataclass
class RetrievalResult:
    chunk_id: str
    content: str
    score: float
    rank: int
    source: str          # 'bm25', 'vector', 'both'
    rrf_score: float = 0.0
    rerank_score: float = 0.0

class VectorStore(ABC):
    """Abstract interface for vector + keyword search.

    This abstraction allows swapping pgvector for Qdrant or Weaviate
    without changing the retrieval pipeline.
    """

    @abstractmethod
    async def keyword_search(self, query: str, strategy: str, k: int) -> list[RetrievalResult]:
        ...

    @abstractmethod
    async def vector_search(self, query_embedding: list[float], strategy: str, k: int) -> list[RetrievalResult]:
        ...

    @abstractmethod
    async def store_chunks(self, chunks: list[dict]) -> None:
        ...
```

### 7.6 Hybrid Retrieval with RRF (`src/retrieval/hybrid.py`)

```python
from collections import defaultdict
from src.indexing.bm25_index import BM25Index
from src.indexing.vector_index import VectorIndex
from src.indexing.store import RetrievalResult

class HybridRetriever:
    """Combines BM25 and vector search using Reciprocal Rank Fusion (RRF).

    RRF Formula:
        RRF_score(d) = sum over retrievers: 1 / (k_constant + rank_in_retriever)

    where k_constant is typically 60 (from the original RRF paper).

    This is rank-based fusion — it doesn't need score calibration between
    BM25 (unbounded) and cosine similarity (0-1), which is its key advantage.
    """

    RRF_K = 60  # constant from Cormack et al. 2009

    def __init__(self, bm25_index: BM25Index, vector_index: VectorIndex, embedder):
        self.bm25 = bm25_index
        self.vector = vector_index
        self.embedder = embedder

    async def retrieve(
        self,
        query: str,
        query_embedding,           # pre-computed embedding (may be HyDE-transformed)
        chunking_strategy: str,
        method: str = "hybrid",    # 'bm25', 'vector', 'hybrid'
        k_per_index: int = 50,
        final_k: int = 20,
    ) -> list[RetrievalResult]:
        """Retrieve and fuse results.

        Args:
            query: Original query text (for BM25)
            query_embedding: Query embedding (for vector search; may be HyDE-transformed)
            chunking_strategy: Which chunking strategy's index to search
            method: 'bm25' | 'vector' | 'hybrid'
            k_per_index: How many results to fetch from each index
            final_k: How many fused results to return
        """
        bm25_results = []
        vector_results = []

        if method in ("bm25", "hybrid"):
            bm25_results = await self.bm25.search(query, chunking_strategy, k_per_index)

        if method in ("vector", "hybrid"):
            vector_results = await self.vector.search(query_embedding, chunking_strategy, k_per_index)

        if method == "bm25":
            return self._to_retrieval_results(bm25_results, "bm25")[:final_k]
        if method == "vector":
            return self._to_retrieval_results(vector_results, "vector")[:final_k]

        # Hybrid: apply RRF fusion
        return self._rrf_fuse(bm25_results, vector_results, final_k)

    def _rrf_fuse(self, bm25_results, vector_results, final_k: int) -> list[RetrievalResult]:
        """Reciprocal Rank Fusion of BM25 and vector results."""
        rrf_scores = defaultdict(float)
        chunk_data = {}  # chunk_id -> (content, sources)

        for result in bm25_results:
            rrf_scores[result.chunk_id] += 1.0 / (self.RRF_K + result.rank)
            chunk_data.setdefault(result.chunk_id, (result.content, set()))
            chunk_data[result.chunk_id][1].add("bm25")

        for result in vector_results:
            rrf_scores[result.chunk_id] += 1.0 / (self.RRF_K + result.rank)
            chunk_data.setdefault(result.chunk_id, (result.content, set()))
            chunk_data[result.chunk_id][1].add("vector")

        # Sort by RRF score descending
        sorted_ids = sorted(rrf_scores.keys(), key=lambda cid: rrf_scores[cid], reverse=True)

        results = []
        for rank, chunk_id in enumerate(sorted_ids[:final_k], 1):
            content, sources = chunk_data[chunk_id]
            source = "both" if len(sources) > 1 else sources.pop()
            results.append(RetrievalResult(
                chunk_id=chunk_id,
                content=content,
                score=rrf_scores[chunk_id],
                rank=rank,
                source=source,
                rrf_score=rrf_scores[chunk_id],
            ))
        return results

    @staticmethod
    def _to_retrieval_results(index_results, source: str) -> list[RetrievalResult]:
        return [
            RetrievalResult(
                chunk_id=r.chunk_id,
                content=r.content,
                score=r.score,
                rank=r.rank,
                source=source,
            )
            for r in index_results
        ]
```

### 7.7 Query Transformation (`src/retrieval/query_transform.py`)

#### HyDE — Hypothetical Document Embeddings

```python
from src.generation.llm import LLMClient
from src.ingestion.embedder import Embedder
import numpy as np

class HyDETransformer:
    """Hypothetical Document Embeddings (HyDE).

    Instead of embedding the query directly, HyDE:
    1. Asks the LLM to generate a hypothetical answer to the query
    2. Embeds that hypothetical answer (not the query)
    3. Uses that embedding for vector search

    Intuition: The hypothetical answer is closer in embedding space to
    the real answer passages than the short query is. This bridges the
    semantic gap between query (short, interrogative) and passages
    (longer, declarative).

    Paper: "Precise Zero-Shot Dense Retrieval without Relevance Data"
    (Gao et al., 2023)

    Trade-off: Adds 1 LLM call latency (~200-500ms). Quality gain is
    most visible on complex, multi-hop, or abstract queries.
    """

    HYDE_PROMPT = """You are an expert assistant. Given the following question,
write a short paragraph (3-5 sentences) that would be a good answer.
Do not add preamble like "Here is an answer". Just write the answer content.

Question: {question}

Answer:"""

    def __init__(self, llm: LLMClient, embedder: Embedder):
        self.llm = llm
        self.embedder = embedder

    def transform(self, query: str) -> np.ndarray:
        """Generate hypothetical answer and return its embedding."""
        prompt = self.HYDE_PROMPT.format(question=query)
        hypothetical_doc = self.llm.generate(prompt, max_tokens=200, temperature=0.7)

        # Embed the hypothetical answer as a PASSAGE (not as a query)
        # because it's now a document-like text, not a query
        embedding = self.embedder.embed_passages([hypothetical_doc])[0]
        return embedding

    def transform_batch(self, queries: list[str]) -> np.ndarray:
        """Batch HyDE: generate hypothetical docs for all queries, then embed."""
        prompts = [self.HYDE_PROMPT.format(question=q) for q in queries]
        hypothetical_docs = [self.llm.generate(p, max_tokens=200, temperature=0.7) for p in prompts]
        return self.embedder.embed_passages(hypothical_docs)
```

#### Multi-Query Expansion

```python
class MultiQueryTransformer:
    """Multi-Query Expansion.

    Asks the LLM to generate N reformulations of the query, then searches
    with all of them and merges results.

    Intuition: A single query phrasing may miss relevant passages that
    use different terminology. Multiple reformulations increase recall
    by covering more semantic surface area.

    Paper: "Query Expansion by Prompting Large Language Models"
    (Wang et al., 2023) — part of the RAG-Fusion approach.

    Trade-off: Adds 1 LLM call + N× vector search latency.
    Best for recall-sensitive use cases.
    """

    MULTIQUERY_PROMPT = """You are an AI language model assistant.
Your task is to generate {n} different versions of the given user
question to retrieve relevant documents from a vector database.
By generating multiple perspectives on the user question, your goal
is to help overcome some of the limitations of distance-based
similarity search.

Provide these alternative questions separated by newlines.

Original question: {question}

Alternative questions:"""

    def __init__(self, llm: LLMClient, embedder: Embedder, n_queries: int = 3):
        self.llm = llm
        self.embedder = embedder
        self.n_queries = n_queries

    def transform(self, query: str) -> list[str]:
        """Generate N reformulations of the query. Returns list of query strings."""
        prompt = self.MULTIQUERY_PROMPT.format(n=self.n_queries, question=query)
        response = self.llm.generate(prompt, max_tokens=150, temperature=0.7)

        # Parse the response: one question per line
        reformulations = [
            line.strip().lstrip("0123456789. )")
            for line in response.strip().split('\n')
            if line.strip()
        ]

        # Always include the original query
        all_queries = [query] + reformulations[:self.n_queries]
        return all_queries

    def get_embeddings(self, query: str) -> list[np.ndarray]:
        """Generate reformulations and return embeddings for all of them."""
        queries = self.transform(query)
        return list(self.embedder.embed_queries(queries))

    @staticmethod
    def merge_results(
        result_lists: list[list],
        rrf_k: int = 60,
        final_k: int = 20,
    ) -> list:
        """Merge results from multiple queries using RRF.

        Each result list is already ranked. We apply RRF across the
        multiple query result lists, same as hybrid fusion but across
        queries instead of across retrieval methods.
        """
        from collections import defaultdict
        rrf_scores = defaultdict(float)
        chunk_data = {}

        for results in result_lists:
            for result in results:
                rrf_scores[result.chunk_id] += 1.0 / (rrf_k + result.rank)
                chunk_data.setdefault(result.chunk_id, result)

        sorted_ids = sorted(rrf_scores.keys(), key=lambda cid: rrf_scores[cid], reverse=True)

        merged = []
        for rank, chunk_id in enumerate(sorted_ids[:final_k], 1):
            result = chunk_data[chunk_id]
            result.rrf_score = rrf_scores[chunk_id]
            result.rank = rank
            merged.append(result)
        return merged
```

### 7.8 Cross-Encoder Re-ranking (`src/retrieval/reranker.py`)

```python
from sentence_transformers import CrossEncoder
import numpy as np
from dataclasses import dataclass
from src.indexing.store import RetrievalResult

@dataclass
class RerankedResult:
    chunk_id: str
    content: str
    rerank_score: float
    original_rank: int
    new_rank: int
    source: str

class CrossEncoderReranker:
    """Cross-encoder re-ranking using BGE-reranker-large.

    ARCHITECTURAL DIFFERENCE — Bi-encoder vs. Cross-encoder:

    Bi-encoder (embedding model):
        query → [embedding]    passage → [embedding]
        Score = cosine_similarity(query_emb, passage_emb)
        ✅ Pre-computable, fast at search time (ANN)
        ❌ Query and passage never interact → misses fine-grained matching

    Cross-encoder (reranker):
        [query + passage] → [single score]
        ✅ Full attention between query and passage tokens → much more accurate
        ❌ Cannot pre-compute; must score every (query, passage) pair at query time

    This is why we use a two-stage pipeline:
        Stage 1: Bi-encoder retrieves top-20 (fast, approximate)
        Stage 2: Cross-encoder re-ranks top-20 → top-5 (slow, precise)

    The cross-encoder only scores 20 pairs, not the entire corpus,
    making this computationally feasible.
    """

    MODEL_NAME = "BAAI/bge-reranker-large"

    def __init__(self, model_name: str | None = None, device: str = "cpu"):
        model = model_name or self.MODEL_NAME
        self.model = CrossEncoder(model, device=device)
        self.model.max_length = 512  # truncate long passages

    def rerank(
        self,
        query: str,
        candidates: list[RetrievalResult],
        top_k: int = 5,
    ) -> list[RerankedResult]:
        """Re-rank candidates using cross-encoder.

        Args:
            query: The original user query (NOT the HyDE hypothetical doc)
            candidates: Top-N results from hybrid retrieval (typically 20)
            top_k: Number of results to return after re-ranking

        Returns:
            Re-ranked list of top_k results, sorted by cross-encoder score
        """
        if not candidates:
            return []

        # Build (query, passage) pairs for the cross-encoder
        pairs = [(query, c.content[:512]) for c in candidates]

        # Score all pairs in a single batch
        scores = self.model.predict(pairs, show_progress_bar=False)

        # Sort by cross-encoder score descending
        scored = list(zip(candidates, scores))
        scored.sort(key=lambda x: x[1], reverse=True)

        results = []
        for new_rank, (candidate, score) in enumerate(scored[:top_k], 1):
            results.append(RerankedResult(
                chunk_id=candidate.chunk_id,
                content=candidate.content,
                rerank_score=float(score),
                original_rank=candidate.rank,
                new_rank=new_rank,
                source=candidate.source,
            ))
        return results

    def rerank_with_context(
        self,
        query: str,
        candidates: list[RetrievalResult],
        top_k: int = 5,
    ) -> list[RerankedResult]:
        """Re-rank using window_context for sentence-window chunks.

        For sentence-window chunks, the `content` is just one sentence,
        but the `window_context` field has the surrounding sentences.
        Re-ranking on the fuller context gives the cross-encoder more
        information to judge relevance.
        """
        if not candidates:
            return []

        # Use window_context if available, otherwise use content
        pairs = []
        for c in candidates:
            context = getattr(c, 'window_context', None) or c.content
            pairs.append((query, context[:512]))

        scores = self.model.predict(pairs, show_progress_bar=False)
        scored = list(zip(candidates, scores))
        scored.sort(key=lambda x: x[1], reverse=True)

        results = []
        for new_rank, (candidate, score) in enumerate(scored[:top_k], 1):
            results.append(RerankedResult(
                chunk_id=candidate.chunk_id,
                content=candidate.content,
                rerank_score=float(score),
                original_rank=candidate.rank,
                new_rank=new_rank,
                source=candidate.source,
            ))
        return results
```

### 7.9 Full RAG Pipeline Orchestrator

```python
from dataclasses import dataclass
from typing import Literal
from src.retrieval.hybrid import HybridRetriever
from src.retrieval.query_transform import HyDETransformer, MultiQueryTransformer
from src.retrieval.reranker import CrossEncoderReranker
from src.ingestion.embedder import Embedder
from src.generation.llm import LLMClient

@dataclass
class RAGResponse:
    answer: str
    retrieved_chunks: list[dict]     # [{chunk_id, content, score, rank, source}]
    strategy_config: dict
    latency_ms: int

class RAGPipeline:
    """Orchestrates the full RAG pipeline with configurable strategies.

    This is the main entry point used by the FastAPI endpoint and the
    evaluation runner.
    """

    def __init__(
        self,
        embedder: Embedder,
        retriever: HybridRetriever,
        reranker: CrossEncoderReranker,
        llm: LLMClient,
        hyde: HyDETransformer,
        multi_query: MultiQueryTransformer,
    ):
        self.embedder = embedder
        self.retriever = retriever
        self.reranker = reranker
        self.llm = llm
        self.hyde = hyde
        self.multi_query = multi_query

    async def query(
        self,
        question: str,
        chunking_strategy: str = "semantic_spacy",
        retrieval_method: str = "hybrid",
        query_transform: str = "none",       # 'none', 'hyde', 'multi_query'
        reranker_enabled: bool = True,
        top_k: int = 5,
        retrieve_k: int = 20,
    ) -> RAGResponse:
        import time
        start = time.time()

        # Step 1: Query transformation → get query embedding(s)
        if query_transform == "hyde":
            query_embedding = self.hyde.transform(question)
            candidates = await self.retriever.retrieve(
                query=question,
                query_embedding=query_embedding,
                chunking_strategy=chunking_strategy,
                method=retrieval_method,
                final_k=retrieve_k,
            )
        elif query_transform == "multi_query":
            query_embeddings = self.multi_query.get_embeddings(question)
            result_lists = []
            for qe in query_embeddings:
                results = await self.retriever.retrieve(
                    query=question,
                    query_embedding=qe,
                    chunking_strategy=chunking_strategy,
                    method=retrieval_method,
                    final_k=retrieve_k,
                )
                result_lists.append(results)
            candidates = MultiQueryTransformer.merge_results(result_lists, final_k=retrieve_k)
        else:
            query_embedding = self.embedder.embed_query(question)
            candidates = await self.retriever.retrieve(
                query=question,
                query_embedding=query_embedding,
                chunking_strategy=chunking_strategy,
                method=retrieval_method,
                final_k=retrieve_k,
            )

        # Step 2: Cross-encoder re-ranking
        if reranker_enabled and len(candidates) > 0:
            reranked = self.reranker.rerank(question, candidates, top_k=top_k)
            final_chunks = [
                {
                    "chunk_id": r.chunk_id,
                    "content": r.content,
                    "score": r.rerank_score,
                    "rank": r.new_rank,
                    "source": r.source,
                }
                for r in reranked
            ]
        else:
            final_chunks = [
                {
                    "chunk_id": c.chunk_id,
                    "content": c.content,
                    "score": c.rrf_score or c.score,
                    "rank": c.rank,
                    "source": c.source,
                }
                for c in candidates[:top_k]
            ]

        # Step 3: LLM generation with retrieved context
        context = "\n\n".join(
            f"[{i+1}] {c['content']}" for i, c in enumerate(final_chunks)
        )
        answer = self.llm.generate_answer(question, context)

        latency_ms = int((time.time() - start) * 1000)

        return RAGResponse(
            answer=answer,
            retrieved_chunks=final_chunks,
            strategy_config={
                "chunking_strategy": chunking_strategy,
                "retrieval_method": retrieval_method,
                "query_transform": query_transform,
                "reranker": "bge_reranker" if reranker_enabled else "none",
                "top_k": top_k,
                "retrieve_k": retrieve_k,
            },
            latency_ms=latency_ms,
        )
```

### 7.10 LLM Client (`src/generation/llm.py`)

```python
import httpx
from src.config import Settings

class LLMClient:
    """LLM client supporting OpenAI API or local Ollama.

    Used for:
    - HyDE hypothetical document generation
    - Multi-query expansion
    - Final answer generation from retrieved context
    """

    ANSWER_PROMPT = """You are a helpful assistant. Answer the question based
on the provided context. If the context doesn't contain enough information
to answer, say "I don't have enough information to answer this question."

Context:
{context}

Question: {question}

Answer:"""

    def __init__(self, settings: Settings):
        self.settings = settings
        self.client = httpx.AsyncClient(timeout=30.0)

    async def generate(self, prompt: str, max_tokens: int = 500, temperature: float = 0.0) -> str:
        """Generate text from a prompt (used for HyDE, multi-query)."""
        if self.settings.llm_provider == "openai":
            return await self._call_openai(prompt, max_tokens, temperature)
        else:
            return await self._call_ollama(prompt, max_tokens, temperature)

    async def generate_answer(self, question: str, context: str) -> str:
        """Generate the final RAG answer from question + retrieved context."""
        prompt = self.ANSWER_PROMPT.format(context=context, question=question)
        return await self.generate(prompt, max_tokens=500, temperature=0.0)

    async def _call_openai(self, prompt: str, max_tokens: int, temperature: float) -> str:
        resp = await self.client.post(
            f"{self.settings.openai_base_url}/chat/completions",
            headers={"Authorization": f"Bearer {self.settings.openai_api_key}"},
            json={
                "model": self.settings.openai_model,
                "messages": [{"role": "user", "content": prompt}],
                "max_tokens": max_tokens,
                "temperature": temperature,
            },
        )
        resp.raise_for_status()
        return resp.json()["choices"][0]["message"]["content"]

    async def _call_ollama(self, prompt: str, max_tokens: int, temperature: float) -> str:
        resp = await self.client.post(
            f"{self.settings.ollama_base_url}/api/generate",
            json={
                "model": self.settings.ollama_model,
                "prompt": prompt,
                "stream": False,
                "options": {"num_predict": max_tokens, "temperature": temperature},
            },
        )
        resp.raise_for_status()
        return resp.json()["response"]
```

### 7.11 Evaluation Metrics (`src/evaluation/metrics.py`)

```python
import math
from typing import Sequence

class RetrievalMetrics:
    """Retrieval quality metrics.

    All metrics take:
        retrieved_ids: ordered list of retrieved chunk IDs (rank 1, 2, 3, ...)
        relevant_ids: set of chunk IDs that are relevant (ground truth)

    For graded metrics (nDCG), use the graded variants which take
    relevance_grades: dict[chunk_id -> grade (0-3)].
    """

    @staticmethod
    def recall_at_k(retrieved_ids: Sequence[str], relevant_ids: set[str], k: int) -> float:
        """Recall@k: What fraction of relevant items were retrieved in top-k?

        recall@k = |relevant ∩ retrieved[:k]| / |relevant|

        Interpretation: "Of all the chunks that SHOULD have been retrieved,
        what fraction appeared in the top-k results?"

        High recall = the system finds most of the relevant information.
        Critical for RAG: if the relevant chunk isn't retrieved, the LLM
        can never use it, no matter how good the LLM is.
        """
        if not relevant_ids:
            return 0.0
        retrieved_set = set(retrieved_ids[:k])
        return len(retrieved_set & relevant_ids) / len(relevant_ids)

    @staticmethod
    def precision_at_k(retrieved_ids: Sequence[str], relevant_ids: set[str], k: int) -> float:
        """Precision@k: What fraction of retrieved top-k items are relevant?

        precision@k = |relevant ∩ retrieved[:k]| / k

        Interpretation: "Of the top-k results returned, how many are actually relevant?"

        High precision = the system doesn't waste slots on irrelevant chunks.
        Important for RAG: irrelevant chunks in the context waste the LLM's
        context window and can cause hallucination.
        """
        if k == 0:
            return 0.0
        retrieved_set = set(retrieved_ids[:k])
        return len(retrieved_set & relevant_ids) / k

    @staticmethod
    def mrr(retrieved_ids: Sequence[str], relevant_ids: set[str]) -> float:
        """Mean Reciprocal Rank: 1/rank of the first relevant result.

        MRR = 1 / rank_of_first_relevant

        Interpretation: "How high was the FIRST relevant result?"

        MRR=1.0 → first result was relevant
        MRR=0.5 → second result was relevant
        MRR=0.33 → third result was relevant

        Good for single-answer questions where you just need ONE relevant chunk.
        Less informative for multi-faceted questions needing multiple chunks.
        """
        for i, chunk_id in enumerate(retrieved_ids, 1):
            if chunk_id in relevant_ids:
                return 1.0 / i
        return 0.0

    @staticmethod
    def ndcg_at_k(
        retrieved_ids: Sequence[str],
        relevance_grades: dict[str, int],
        k: int,
    ) -> float:
        """Normalized Discounted Cumulative Gain @k (graded relevance).

        DCG@k = sum_{i=1}^{k} (grade_i / log2(i + 1))

        IDCG@k = DCG@k of the ideal ranking (sorted by grade descending)

        nDCG@k = DCG@k / IDCG@k

        Interpretation: "How well-ordered are the relevant results?"

        nDCG=1.0 → perfect ranking (most relevant first)
        nDCG=0.5 → relevant items are present but poorly ordered

        Key properties:
        - Uses GRADED relevance (0=irrelevant, 1=marginally, 2=relevant, 3=perfect)
        - Discounts by rank position (rank 1 matters more than rank 10)
        - Normalized to [0, 1] for cross-query comparison

        This is the MOST informative single metric for RAG retrieval quality
        because it captures both presence AND ordering of relevant results.
        """
        # DCG: sum of grade / log2(rank+1) for retrieved items
        dcg = 0.0
        for i, chunk_id in enumerate(retrieved_ids[:k], 1):
            grade = relevance_grades.get(chunk_id, 0)
            dcg += (2 ** grade - 1) / math.log2(i + 1)

        # IDCG: ideal DCG (sort all relevant by grade descending, take top-k)
        ideal_grades = sorted(relevance_grades.values(), reverse=True)[:k]
        idcg = sum(
            (2 ** g - 1) / math.log2(i + 1)
            for i, g in enumerate(ideal_grades, 1)
        )

        return dcg / idcg if idcg > 0 else 0.0
```

### 7.12 Evaluation Runner (`src/evaluation/runner.py`)

```python
import itertools
from dataclasses import dataclass
from src.evaluation.metrics import RetrievalMetrics
from src.rag_pipeline import RAGPipeline

@dataclass
class StrategyCombo:
    chunking_strategy: str
    retrieval_method: str
    query_transform: str
    reranker: str

class EvaluationRunner:
    """Runs all strategy combinations against the golden dataset.

    Total combinations:
        3 chunking × 3 retrieval × 3 transform × 2 reranker = 54

    For each combination, runs all golden questions and computes:
    recall@5, recall@10, precision@5, MRR, nDCG@5, nDCG@10, latency_ms
    """

    STRATEGIES = {
        "chunking": ["fixed_1000_200", "semantic_spacy", "sentence_window_3"],
        "retrieval": ["bm25", "vector", "hybrid"],
        "transform": ["none", "hyde", "multi_query"],
        "reranker": ["none", "bge_reranker"],
    }

    def __init__(self, pipeline: RAGPipeline, golden_questions: list[dict]):
        self.pipeline = pipeline
        self.golden = golden_questions
        self.metrics = RetrievalMetrics()

    def get_all_combos(self) -> list[StrategyCombo]:
        combos = []
        for c, r, t, rr in itertools.product(
            self.STRATEGIES["chunking"],
            self.STRATEGIES["retrieval"],
            self.STRATEGIES["transform"],
            self.STRATEGIES["reranker"],
        ):
            combos.append(StrategyCombo(c, r, t, rr))
        return combos

    async def run_combination(self, combo: StrategyCombo) -> list[dict]:
        """Run all golden questions for one strategy combination."""
        results = []
        for q in self.golden:
            response = await self.pipeline.query(
                question=q["question"],
                chunking_strategy=combo.chunking_strategy,
                retrieval_method=combo.retrieval_method,
                query_transform=combo.query_transform,
                reranker_enabled=(combo.reranker == "bge_reranker"),
                top_k=5,
                retrieve_k=20,
            )

            retrieved_ids = [c["chunk_id"] for c in response.retrieved_chunks]
            relevant_ids = set(q["relevant_chunk_ids"])
            grades = q.get("relevance_grades", {rid: 3 for rid in relevant_ids})

            results.append({
                "golden_question_id": q["id"],
                "chunking_strategy": combo.chunking_strategy,
                "retrieval_method": combo.retrieval_method,
                "query_transform": combo.query_transform,
                "reranker": combo.reranker,
                "retrieved_ids": retrieved_ids,
                "recall_at_5": self.metrics.recall_at_k(retrieved_ids, relevant_ids, 5),
                "recall_at_10": self.metrics.recall_at_k(retrieved_ids, relevant_ids, 10),
                "precision_at_5": self.metrics.precision_at_k(retrieved_ids, relevant_ids, 5),
                "mrr": self.metrics.mrr(retrieved_ids, relevant_ids),
                "ndcg_at_5": self.metrics.ndcg_at_k(retrieved_ids, grades, 5),
                "ndcg_at_10": self.metrics.ndcg_at_k(retrieved_ids, grades, 10),
                "latency_ms": response.latency_ms,
            })
        return results

    async def run_all(self) -> list[dict]:
        """Run all 54 combinations × all questions. Returns flat list of results."""
        all_results = []
        combos = self.get_all_combos()
        total = len(combos)
        for i, combo in enumerate(combos, 1):
            print(f"[{i}/{total}] Running {combo.chunking_strategy} | "
                  f"{combo.retrieval_method} | {combo.query_transform} | {combo.reranker}")
            results = await self.run_combination(combo)
            all_results.extend(results)
        return all_results
```

### 7.13 Streamlit Dashboard (`src/dashboard/app.py`)

```python
import streamlit as st
import pandas as pd
import plotly.express as px
import asyncpg

st.set_page_config(page_title="RAG Evaluation Dashboard", layout="wide")

# --- Connection ---
@st.cache_resource
def get_pool():
    return asyncpg.create_pool(dsn=st.secrets["database"]["url"])

# --- Load results ---
@st.cache_data(ttl=60)
def load_results() -> pd.DataFrame:
    """Load all eval_results joined with golden_questions."""
    # In practice, use asyncpg or psycopg2 to query
    df = pd.read_sql("""
        SELECT e.*, g.question, g.category, g.difficulty
        FROM eval_results e
        JOIN golden_questions g ON e.golden_question_id = g.id
    """, con=conn_string)
    return df

df = load_results()

# --- Sidebar: Filters ---
st.sidebar.title("Filters")
chunking_filter = st.sidebar.multiselect("Chunking Strategy", df["chunking_strategy"].unique())
retrieval_filter = st.sidebar.multiselect("Retrieval Method", df["retrieval_method"].unique())
transform_filter = st.sidebar.multiselect("Query Transform", df["query_transform"].unique())
reranker_filter = st.sidebar.multiselect("Reranker", df["reranker"].unique())

if chunking_filter:
    df = df[df["chunking_strategy"].isin(chunking_filter)]
# ... apply other filters

# --- Section 1: Overview ---
st.title("RAG Retrieval Quality Dashboard")
st.header("Overview")

# Aggregate by strategy combination
agg = df.groupby([
    "chunking_strategy", "retrieval_method", "query_transform", "reranker"
]).agg({
    "recall_at_5": "mean",
    "recall_at_10": "mean",
    "precision_at_5": "mean",
    "mrr": "mean",
    "ndcg_at_5": "mean",
    "ndcg_at_10": "mean",
    "latency_ms": "mean",
}).reset_index().sort_values("ndcg_at_5", ascending=False)

st.dataframe(agg, use_container_width=True)

# --- Section 2: Best Strategy per Metric ---
st.subheader("Best Strategy per Metric")
metrics_cols = ["recall_at_5", "precision_at_5", "mrr", "ndcg_at_5"]
best = {}
for m in metrics_cols:
    row = agg.loc[agg[m].idxmax()]
    best[m] = f"{row['chunking_strategy']} | {row['retrieval_method']} | {row['query_transform']} | {row['reranker']}"
st.table(pd.DataFrame(best, index=["Best Strategy"]).T)

# --- Section 3: Chunking Strategy Comparison ---
st.header("Chunking Strategy Comparison (holding retrieval=hybrid, transform=none, reranker=bge_reranker)")
chunk_df = df[
    (df["retrieval_method"] == "hybrid") &
    (df["query_transform"] == "none") &
    (df["reranker"] == "bge_reranker")
]
chunk_agg = chunk_df.groupby("chunking_strategy")[metrics_cols].mean()
st.bar_chart(chunk_agg)

# --- Section 4: Query Transform Impact ---
st.header("Query Transform Impact (holding chunking=semantic_spacy, retrieval=hybrid, reranker=bge_reranker)")
transform_df = df[
    (df["chunking_strategy"] == "semantic_spacy") &
    (df["retrieval_method"] == "hybrid") &
    (df["reranker"] == "bge_reranker")
]
transform_agg = transform_df.groupby("query_transform")[metrics_cols].mean()
st.bar_chart(transform_agg)

# --- Section 5: Reranker Impact ---
st.header("Reranker Impact (holding chunking=semantic_spacy, retrieval=hybrid, transform=none)")
reranker_df = df[
    (df["chunking_strategy"] == "semantic_spacy") &
    (df["retrieval_method"] == "hybrid") &
    (df["query_transform"] == "none")
]
reranker_agg = reranker_df.groupby("reranker")[metrics_cols].mean()
st.bar_chart(reranker_agg)

# --- Section 6: Latency vs Quality ---
st.header("Latency vs. Quality Trade-off")
fig = px.scatter(
    agg, x="latency_ms", y="ndcg_at_5",
    color="chunking_strategy", symbol="query_transform",
    size="recall_at_10",
    hover_data=["retrieval_method", "reranker"],
    title="Latency (ms) vs. nDCG@5 — bigger bubbles = higher recall@10",
)
st.plotly_chart(fig, use_container_width=True)

# --- Section 7: Per-Query Drill-Down ---
st.header("Per-Query Drill-Down")
selected_q = st.selectbox("Select a question", df["question"].unique())
q_df = df[df["question"] == selected_q]
st.dataframe(q_df[[
    "chunking_strategy", "retrieval_method", "query_transform", "reranker",
    "recall_at_5", "ndcg_at_5", "mrr", "latency_ms"
]], use_container_width=True)
```

---

## 8. Comparison Tables

### 8.1 Chunking Strategies Comparison

| Strategy | How It Works | Chunk Size | Pros | Cons | Best For |
|----------|-------------|------------|------|------|----------|
| **Fixed-size (1000/200)** | Slides a window of 1000 chars with 200 char overlap. Adjusts to nearest sentence boundary. | ~1000 chars (~150-200 tokens) | Simple, predictable, consistent chunk sizes; easy to tune; good baseline | Can split related ideas across chunks; overlap wastes context window; no semantic awareness | General-purpose; when you need consistent chunk sizes for batch processing |
| **Semantic (spaCy)** | Segments text into sentences via spaCy, then groups sentences until ~target size. Never breaks a sentence. | Variable (~800-1200 chars) | Respects sentence boundaries; chunks are semantically coherent; no mid-sentence cuts | Chunk sizes vary (some too small, some too large); depends on spaCy model quality; slower to compute | Documents with clear sentence structure (articles, essays, documentation) |
| **Sentence-window (3)** | Each chunk = 1 sentence (embedded). Stores 3 sentences before/after as context. At generation time, uses the full window. | 1 sentence (embedded) + 7 sentences (context) | Finest retrieval granularity — can pinpoint the exact relevant sentence; context window provides broader LLM context; best of both worlds | Many more chunks (1 per sentence) → larger index, slower search; more embeddings to compute; potential redundancy | Precision-critical retrieval; when you need to find specific claims within long documents |

### Chunking Strategy Decision Guide

```
Is your corpus long-form prose (articles, essays)?
  → YES: Semantic chunking (respects narrative flow)
  → NO:  Is it structured (API docs, FAQs, specs)?
           → YES: Fixed-size with sentence boundary adjustment
           → NO:  Mixed content → try both and compare with the eval dashboard

Need pinpoint precision (find the exact sentence)?
  → Sentence-window chunking

Need to minimize index size / embedding cost?
  → Fixed-size (fewest chunks, most predictable)
```

### 8.2 Retrieval Methods Comparison

| Method | How It Works | Strengths | Weaknesses | Best For |
|--------|-------------|-----------|------------|----------|
| **BM25 (keyword)** | TF-IDF-style scoring with term frequency saturation + document length normalization. Matches exact terms. | Exact keyword matching; handles rare terms well; no model needed; interpretable; fast | No semantic matching (synonyms, paraphrases missed); vocabulary mismatch problem; can't match concepts | Technical docs with precise terminology; queries with exact keywords; acronym-heavy content |
| **Vector (semantic)** | Embeds query and passages into same vector space; cosine similarity. Matches meaning. | Semantic matching (synonyms, paraphrases); handles vocabulary mismatch; captures conceptual similarity | Misses exact keyword matches (important for names, IDs, code); embedding model quality matters; can be distracted by semantic similarity without relevance | Conceptual queries; paraphrased questions; when query and document use different words for the same concept |
| **Hybrid (BM25 + Vector + RRF)** | Retrieves top-50 from each, fuses with Reciprocal Rank Fusion (1/(60+rank)). | Best of both worlds; robust across query types; no score calibration needed (RRF is rank-based) | 2× retrieval cost; slightly more complex; RRF constant (k=60) is a hyperparameter to tune | Production RAG systems; when you can't predict query types; default recommendation |

### 8.3 Query Transformation Comparison

| Technique | How It Works | Latency Cost | Quality Impact | Best For |
|-----------|-------------|--------------|----------------|----------|
| **None (raw query)** | Embed the query directly | 0ms (baseline) | Baseline | Simple, direct queries; when latency is critical |
| **HyDE** | LLM generates hypothetical answer → embed that instead of query | +200-500ms (1 LLM call) | +5-15% nDCG on complex/abstract queries; can hurt on simple factual queries | Complex, multi-hop, abstract queries; when query-document semantic gap is large |
| **Multi-query expansion** | LLM generates 3 reformulations → search all → RRF merge | +200-500ms (1 LLM call) + 3× vector search | +5-10% recall; helps when query terminology is narrow | Recall-sensitive use cases; when users may not know the right keywords |

### 8.4 Re-ranking Comparison

| Approach | How It Works | Latency Cost | Quality Impact | Best For |
|----------|-------------|--------------|----------------|----------|
| **No re-ranking** | Return top-k from hybrid retrieval directly | 0ms (baseline) | Baseline | When latency budget is tight; simple queries |
| **BGE cross-encoder** | Score each (query, passage) pair with a cross-encoder; re-sort | +50-200ms (20 pairs) | +10-20% nDCG; fixes ranking errors from bi-encoder | Production RAG; when you can afford ~100ms extra; the single highest-ROI improvement |

### 8.5 Full Strategy Comparison Matrix (Expected Results)

> These are expected relative rankings based on published research. Your actual results will vary based on corpus and golden dataset. The evaluation dashboard will show your real numbers.

| Chunking | Retrieval | Transform | Reranker | Expected nDCG@5 | Expected Latency |
|----------|-----------|-----------|----------|-----------------|------------------|
| fixed_1000_200 | bm25 | none | none | 0.35-0.45 | ~50ms |
| fixed_1000_200 | vector | none | none | 0.40-0.50 | ~60ms |
| fixed_1000_200 | hybrid | none | none | 0.45-0.55 | ~80ms |
| fixed_1000_200 | hybrid | none | bge | 0.55-0.65 | ~180ms |
| semantic_spacy | hybrid | none | none | 0.50-0.60 | ~80ms |
| semantic_spacy | hybrid | none | bge | 0.60-0.70 | ~180ms |
| semantic_spacy | hybrid | hyde | bge | 0.65-0.75 | ~450ms |
| semantic_spacy | hybrid | multi_query | bge | 0.63-0.73 | ~500ms |
| sentence_window_3 | hybrid | none | bge | 0.58-0.68 | ~200ms |
| sentence_window_3 | hybrid | hyde | bge | 0.62-0.72 | ~500ms |

**Key Takeaways (expected)**:
1. **Hybrid > single method**: Consistent ~5-10% improvement over BM25-only or vector-only
2. **Re-ranking is the single highest-ROI improvement**: ~10-20% nDCG gain for ~100ms cost
3. **Semantic chunking > fixed > sentence-window** (for nDCG, but sentence-window may win on precision)
4. **HyDE helps on complex queries, can hurt on simple ones**: Not a universal win
5. **Multi-query helps recall but adds latency**: Use when recall is more important than latency

---

## 9. Retrieval Evaluation

### 9.1 Why Retrieval Quality is the Hard Part of RAG

```
┌─────────────────────────────────────────────────────────────────────┐
│                    THE RAG QUALITY CHAIN                            │
│                                                                     │
│   Document Quality → Chunking → Indexing → Retrieval → Re-ranking   │
│         → Context Assembly → LLM Generation                         │
│                                                                     │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │  RETRIEVAL IS THE BOTTLENECK                                │   │
│   │                                                             │   │
│   │  If the relevant chunk is NOT retrieved:                    │   │
│   │    • The LLM cannot use it — no matter how good the LLM is  │   │
│   │    • The answer will be wrong or hallucinated               │   │
│   │    • Better prompting cannot fix missing context            │   │
│   │                                                             │   │
│   │  If the relevant chunk IS retrieved but poorly ranked:      │   │
│   │    • It may fall outside the top-k context window           │   │
│   │    • The LLM may give it less attention than higher-ranked  │   │
│   │      irrelevant chunks                                      │   │
│   │                                                             │   │
│   │  The LLM call is the EASY part:                             │   │
│   │    • Modern LLMs are very good at synthesizing context      │   │
│   │    • Given correct context, answer quality is high          │   │
│   │    • LLM quality differences are small compared to          │   │
│   │      retrieval quality differences                         │   │
│   └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│   CONCLUSION: Measure retrieval quality FIRST.                      │
│   Optimize retrieval BEFORE swapping LLMs.                          │
│   A GPT-3.5 with great retrieval beats GPT-4 with bad retrieval.    │
└─────────────────────────────────────────────────────────────────────┘
```

### 9.2 Metric Definitions

#### Recall@k

```
recall@k = |relevant ∩ retrieved[:k]| / |relevant|

Where:
  relevant     = set of chunk IDs that should be retrieved (ground truth)
  retrieved[:k] = top-k chunk IDs returned by the system

Range: [0.0, 1.0]  (1.0 = all relevant chunks were in top-k)

Question it answers: "Did we find ALL the relevant information?"
Critical for: RAG (missing context = wrong answers)

Example:
  relevant = {A, B, C}
  retrieved = [X, A, Y, B, Z]  (k=5)
  recall@5 = |{A,B} ∩ {X,A,Y,B,Z}| / |{A,B,C}| = 2/3 = 0.667
```

#### Precision@k

```
precision@k = |relevant ∩ retrieved[:k]| / k

Range: [0.0, 1.0]  (1.0 = every retrieved chunk is relevant)

Question it answers: "Are the results we returned actually relevant?"
Critical for: Not wasting the LLM's context window with irrelevant chunks

Example:
  relevant = {A, B, C}
  retrieved = [X, A, Y, B, Z]  (k=5)
  precision@5 = 2/5 = 0.400
```

#### MRR (Mean Reciprocal Rank)

```
MRR = 1 / rank_of_first_relevant_result

Range: [0.0, 1.0]  (1.0 = first result is relevant)

Question it answers: "How quickly did we find the FIRST relevant result?"
Critical for: Single-answer questions where one relevant chunk suffices

Example:
  relevant = {A, B}
  retrieved = [X, A, Y, B, Z]
  First relevant (A) is at rank 2 → MRR = 1/2 = 0.500

Note: Averaged across all questions to get Mean Reciprocal Rank.
```

#### nDCG@k (Normalized Discounted Cumulative Gain)

```
Step 1: DCG@k (Discounted Cumulative Gain)
  DCG@k = Σ_{i=1}^{k} (2^grade_i - 1) / log2(i + 1)

  Where grade_i is the relevance grade of the item at rank i.
  Grades: 0=irrelevant, 1=marginally, 2=relevant, 3=perfectly relevant

Step 2: IDCG@k (Ideal DCG — best possible ordering)
  IDCG@k = DCG@k computed on the ideal ranking (grades sorted descending)

Step 3: nDCG@k
  nDCG@k = DCG@k / IDCG@k

Range: [0.0, 1.0]  (1.0 = perfect ranking)

Question it answers: "How well-ORDERED are the relevant results?"
Critical for: RAG (the LLM sees top-k in order; higher-ranked items get more attention)

Why nDCG is the best single metric for RAG retrieval:
  1. Uses GRADED relevance (not just binary) — captures "partially relevant"
  2. Discounts by position — rank 1 matters more than rank 5
  3. Normalized — comparable across questions with different # of relevant chunks

Example:
  relevance_grades = {A: 3, B: 2, C: 1}
  retrieved = [X, A, Y, B, Z]  (k=5)

  DCG@5 = (2^0-1)/log2(2) + (2^3-1)/log2(3) + (2^0-1)/log2(4) + (2^2-1)/log2(5) + (2^0-1)/log2(6)
         = 0 + 7/1.585 + 0 + 3/2.322 + 0
         = 0 + 4.416 + 0 + 1.292 + 0
         = 5.708

  IDCG@5 = (2^3-1)/log2(2) + (2^2-1)/log2(3) + (2^1-1)/log2(4)
          = 7/1 + 3/1.585 + 1/2
          = 7 + 1.893 + 0.5
          = 9.393

  nDCG@5 = 5.708 / 9.393 = 0.608
```

### 9.3 Metric Selection Guide

| Metric | What It Captures | When to Optimize |
|--------|-----------------|------------------|
| **recall@10** | Coverage — did we find all relevant chunks? | First priority: if recall is low, nothing else matters |
| **nDCG@5** | Ranking quality — are relevant chunks well-ordered in top-5? | Second priority: ordering matters for LLM attention |
| **precision@5** | Efficiency — are we wasting context window slots? | When context window is limited or expensive |
| **MRR** | Speed to first relevant result | Single-answer questions, quick-lookup use cases |

**Recommended primary metric: nDCG@5** — it captures both presence and ordering, uses graded relevance, and is normalized for cross-query aggregation.

### 9.4 Golden Dataset Construction

The golden dataset is the foundation of evaluation. A poorly constructed golden dataset produces misleading metrics.

**Construction approach**:

1. **Generate questions**: Write 30-50 questions about the corpus. Include:
   - Easy: direct factual questions answerable from a single chunk
   - Medium: questions requiring synthesis across 2-3 chunks
   - Hard: abstract/implicit questions requiring reasoning

2. **Identify relevant chunks**: For each question, manually identify which chunks contain the answer. Use the chunking strategy that produces the most chunks (sentence-window) for finest granularity, then map to other strategies.

3. **Assign relevance grades**:
   - 3 = Perfectly relevant (contains the direct answer)
   - 2 = Relevant (contains supporting information)
   - 1 = Marginally relevant (tangentially related)
   - 0 = Irrelevant

4. **Golden dataset format** (`golden_dataset.jsonl`):
```json
{"id": "q001", "question": "What does Paul Graham say about competition?", "expected_answer": "Paul Graham argues that startups should focus on building something users love rather than worrying about competition...", "relevant_chunk_ids": ["uuid-1", "uuid-2", "uuid-3"], "relevance_grades": {"uuid-1": 3, "uuid-2": 2, "uuid-3": 1}, "difficulty": "easy", "category": "startup_advice"}
{"id": "q002", "question": "How does semantic chunking differ from fixed-size chunking?", "expected_answer": "Semantic chunking splits at sentence boundaries using NLP models like spaCy, while fixed-size chunking uses character count windows...", "relevant_chunk_ids": ["uuid-4", "uuid-5"], "relevance_grades": {"uuid-4": 3, "uuid-5": 2}, "difficulty": "medium", "category": "rag_techniques"}
```

5. **Validation**: After creating the golden dataset, verify:
   - Every `relevant_chunk_id` exists in the `chunks` table
   - Grades are consistent (grade 3 chunks actually contain the answer)
   - Coverage: questions span different categories and difficulties
   - No trivially easy questions where every strategy scores 1.0

---

## 10. API Specification

### Base URL
```
http://localhost:8000
```

### Endpoints

#### `GET /health`

Health check endpoint.

**Response** `200 OK`:
```json
{
  "status": "healthy",
  "db": "connected",
  "embedding_model": "BAAI/bge-large-en-v1.5",
  "reranker_model": "BAAI/bge-reranker-large"
}
```

---

#### `POST /query`

Query the RAG pipeline with configurable retrieval strategy.

**Request**:
```json
{
  "query": "How do I validate a startup idea?",
  "chunking_strategy": "semantic_spacy",
  "retrieval_method": "hybrid",
  "query_transform": "hyde",
  "reranker": "bge_reranker",
  "top_k": 5,
  "retrieve_k": 20
}
```

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `query` | string | *required* | The user's question |
| `chunking_strategy` | enum | `semantic_spacy` | `fixed_1000_200` \| `semantic_spacy` \| `sentence_window_3` |
| `retrieval_method` | enum | `hybrid` | `bm25` \| `vector` \| `hybrid` |
| `query_transform` | enum | `none` | `none` \| `hyde` \| `multi_query` |
| `reranker` | enum | `bge_reranker` | `none` \| `bge_reranker` |
| `top_k` | int | `5` | Number of chunks to return after re-ranking |
| `retrieve_k` | int | `20` | Number of candidates to retrieve before re-ranking |

**Response** `200 OK`:
```json
{
  "answer": "To validate a startup idea, you should talk to potential users...",
  "retrieved_chunks": [
    {
      "chunk_id": "a1b2c3d4-...",
      "content": "The best way to validate a startup idea is to...",
      "score": 0.95,
      "rank": 1,
      "source": "both"
    }
  ],
  "strategy_config": {
    "chunking_strategy": "semantic_spacy",
    "retrieval_method": "hybrid",
    "query_transform": "hyde",
    "reranker": "bge_reranker",
    "top_k": 5,
    "retrieve_k": 20
  },
  "latency_ms": 450
}
```

**Error** `400 Bad Request`:
```json
{
  "detail": "Invalid chunking_strategy: 'foo'. Must be one of: fixed_1000_200, semantic_spacy, sentence_window_3"
}
```

---

#### `POST /ingest`

Trigger document ingestion (async — returns immediately, processes in background).

**Request**:
```json
{
  "source_path": "data/raw/",
  "chunking_strategy": "all",
  "embed": true
}
```

**Response** `202 Accepted`:
```json
{
  "job_id": "ingest-20260810-001",
  "status": "accepted",
  "message": "Ingestion started for 3 chunking strategies"
}
```

---

#### `GET /strategies`

List all available strategy combinations.

**Response** `200 OK`:
```json
{
  "chunking_strategies": ["fixed_1000_200", "semantic_spacy", "sentence_window_3"],
  "retrieval_methods": ["bm25", "vector", "hybrid"],
  "query_transforms": ["none", "hyde", "multi_query"],
  "rerankers": ["none", "bge_reranker"]
}
```

---

#### `GET /eval/results`

Get evaluation results (used by the Streamlit dashboard or external tools).

**Query Parameters**:
| Param | Type | Description |
|-------|------|-------------|
| `chunking_strategy` | string | Filter by chunking strategy |
| `retrieval_method` | string | Filter by retrieval method |
| `query_transform` | string | Filter by query transform |
| `reranker` | string | Filter by reranker |
| `limit` | int | Max results (default 100) |

**Response** `200 OK`:
```json
{
  "results": [
    {
      "golden_question_id": "...",
      "question": "What does Paul Graham say about competition?",
      "chunking_strategy": "semantic_spacy",
      "retrieval_method": "hybrid",
      "query_transform": "hyde",
      "reranker": "bge_reranker",
      "recall_at_5": 0.667,
      "ndcg_at_5": 0.782,
      "mrr": 1.0,
      "latency_ms": 450
    }
  ],
  "summary": {
    "total_results": 1620,
    "best_ndcg_at_5": {
      "combo": "semantic_spacy|hybrid|hyde|bge_reranker",
      "value": 0.782
    }
  }
}
```

---

## 11. Security & Safety Considerations

### 11.1 LLM Safety

| Risk | Mitigation |
|------|-----------|
| **Prompt injection** via user query | Sanitize queries; use system prompt to instruct LLM to only use provided context; log and monitor for injection patterns |
| **Hallucination** (LLM fabricates beyond context) | Instruct LLM to say "I don't have enough information" when context is insufficient; use low temperature (0.0) for answer generation |
| **Sensitive data in context** | Implement content filtering on source documents; consider PII detection before indexing (see Tier 4 Project 10: PII Guardrails) |
| **API key exposure** | Never hardcode keys; use `.env` files (gitignored); use Docker secrets in production |

### 11.2 Data Security

| Risk | Mitigation |
|------|-----------|
| **SQL injection** in search queries | Use parameterized queries exclusively (asyncpg `$1` placeholders); never string-interpolate user input into SQL |
| **Source document poisoning** | Validate document sources; hash and deduplicate; log ingestion provenance |
| **Golden dataset tampering** | Store golden dataset in version control; hash-verify before evaluation runs |

### 11.3 Infrastructure Security

| Risk | Mitigation |
|------|-----------|
| **Postgres exposed externally** | Bind to `127.0.0.1` only in Docker Compose; use strong password; never use default credentials in production |
| **Model download supply chain** | Pin model versions; verify SHA-256 hashes; use HuggingFace `revision` parameter for commit-pinned downloads |
| **Docker image vulnerabilities** | Use slim base images; scan with `trivy` or `grype`; pin dependency versions |

### 11.4 Operational Safety

| Risk | Mitigation |
|------|-----------|
| **LLM API rate limits / costs** | Implement request rate limiting; cache HyDE/multi-query results; monitor token usage |
| **Model loading memory** | BGE-large needs ~1.3GB RAM; BGE-reranker-large needs ~1.3GB RAM; ensure container has ≥4GB RAM |
| **Long-running evaluation** | Evaluation of 54 combos × 50 questions = 2,700 queries; implement checkpointing; allow resumable runs |

---

## 12. Testing Strategy

### 12.1 Test Pyramid

```
                    ┌───────────┐
                    │   E2E     │  2 tests: full pipeline ingest→query→answer
                    │  (slow)   │
                    ├───────────┤
                    │ Integration│  5 tests: DB + API + retrieval pipeline
                    ├───────────┤
                    │   Unit     │  20+ tests: individual components
                    │  (fast)    │
                    └───────────┘
```

### 12.2 Unit Tests

#### `test_chunkers.py`

```python
import pytest
from src.ingestion.chunkers import FixedSizeChunker, SemanticChunker, SentenceWindowChunker

class TestFixedSizeChunker:
    """Tests for fixed-size chunking strategy."""

    def test_basic_chunking(self):
        """A 3000-char text should produce 3-4 chunks with 1000/200 settings."""
        text = "This is a sentence. " * 200  # ~3600 chars
        chunker = FixedSizeChunker(chunk_size=1000, overlap=200)
        chunks = chunker.chunk(text)
        assert len(chunks) >= 3
        assert all(c.chunking_strategy == "fixed_1000_200" for c in chunks)
        assert all(c.token_count > 0 for c in chunks)

    def test_overlap_exists(self):
        """Consecutive chunks should share overlapping text."""
        text = "Sentence one. Sentence two. Sentence three. " * 50
        chunker = FixedSizeChunker(chunk_size=200, overlap=50)
        chunks = chunker.chunk(text)
        if len(chunks) >= 2:
            # The end of chunk 0 should appear at the start of chunk 1
            overlap_text = chunks[0].content[-30:]
            assert overlap_text in chunks[1].content

    def test_empty_text(self):
        """Empty text should produce zero chunks."""
        chunker = FixedSizeChunker()
        assert chunker.chunk("") == []

    def test_short_text(self):
        """Text shorter than chunk_size should produce exactly one chunk."""
        chunker = FixedSizeChunker(chunk_size=1000, overlap=200)
        chunks = chunker.chunk("This is a short text.")
        assert len(chunks) == 1

    def test_sentence_boundary_adjustment(self):
        """Chunks should prefer to end at sentence boundaries."""
        text = "First sentence. Second sentence. Third sentence. " * 30
        chunker = FixedSizeChunker(chunk_size=100, overlap=20)
        chunks = chunker.chunk(text)
        # Most chunks should end with a period (sentence boundary)
        period_endings = sum(1 for c in chunks if c.content.rstrip().endswith('.'))
        assert period_endings > len(chunks) * 0.5  # majority end at sentence boundary


class TestSemanticChunker:
    """Tests for spaCy-based semantic chunking."""

    def test_respects_sentence_boundaries(self):
        """No chunk should contain a partial sentence."""
        text = ("This is the first sentence. "
                "This is the second sentence. "
                "This is the third sentence. " * 20)
        chunker = SemanticChunker(target_size=100)
        chunks = chunker.chunk(text)
        # Each chunk should start with a capital letter (beginning of sentence)
        assert all(c.content[0].isupper() for c in chunks if c.content)

    def test_chunking_strategy_label(self):
        """Chunks should be labeled with 'semantic_spacy'."""
        chunker = SemanticChunker(target_size=500)
        chunks = chunker.chunk("A test sentence. Another one.")
        assert all(c.chunking_strategy == "semantic_spacy" for c in chunks)


class TestSentenceWindowChunker:
    """Tests for sentence-window chunking."""

    def test_each_chunk_is_one_sentence(self):
        """Each chunk content should be a single sentence."""
        text = "First sentence. Second sentence. Third sentence. Fourth sentence."
        chunker = SentenceWindowChunker(window_size=1)
        chunks = chunker.chunk(text)
        assert len(chunks) == 4
        # Each chunk content should not contain multiple sentences
        for c in chunks:
            assert c.content.count('.') <= 1

    def test_window_context_contains_surrounding_sentences(self):
        """Window context should include sentences before and after."""
        text = "Alpha. Beta. Gamma. Delta. Epsilon."
        chunker = SentenceWindowChunker(window_size=1)
        chunks = chunker.chunk(text)
        # Chunk for "Gamma" (index 2) should have "Beta" and "Delta" in context
        gamma_chunk = chunks[2]
        assert "Beta" in gamma_chunk.window_context
        assert "Delta" in gamma_chunk.window_context
        assert "Gamma" in gamma_chunk.window_context
        # Alpha and Epsilon should NOT be in the window (window_size=1)
        assert "Alpha" not in gamma_chunk.window_context
        assert "Epsilon" not in gamma_chunk.window_context

    def test_edge_sentences_have_clipped_windows(self):
        """First and last sentences should have smaller windows."""
        text = "Alpha. Beta. Gamma. Delta. Epsilon."
        chunker = SentenceWindowChunker(window_size=2)
        chunks = chunker.chunk(text)
        # First chunk: window should be Alpha, Beta, Gamma (no sentences before)
        first = chunks[0]
        assert "Alpha" in first.window_context
        assert "Beta" in first.window_context
        assert "Gamma" in first.window_context
```

#### `test_metrics.py`

```python
import pytest
from src.evaluation.metrics import RetrievalMetrics

class TestRecallAtK:
    """Tests for recall@k metric."""

    def test_perfect_recall(self):
        """All relevant items retrieved → recall@k = 1.0."""
        retrieved = ["A", "B", "C", "D", "E"]
        relevant = {"A", "B", "C"}
        assert RetrievalMetrics.recall_at_k(retrieved, relevant, 5) == 1.0

    def test_partial_recall(self):
        """2 of 3 relevant items in top-5 → recall@5 = 0.667."""
        retrieved = ["X", "A", "Y", "B", "Z"]
        relevant = {"A", "B", "C"}
        result = RetrievalMetrics.recall_at_k(retrieved, relevant, 5)
        assert abs(result - 2/3) < 0.001

    def test_zero_recall(self):
        """No relevant items retrieved → recall@k = 0.0."""
        retrieved = ["X", "Y", "Z"]
        relevant = {"A", "B"}
        assert RetrievalMetrics.recall_at_k(retrieved, relevant, 3) == 0.0

    def test_empty_relevant(self):
        """No relevant items exist → recall = 0.0 (edge case)."""
        assert RetrievalMetrics.recall_at_k(["A", "B"], set(), 5) == 0.0

    def test_k_smaller_than_retrieved(self):
        """k=2 should only consider first 2 retrieved items."""
        retrieved = ["A", "X", "B", "Y", "C"]
        relevant = {"A", "B", "C"}
        # Only "A" is in top-2 → recall@2 = 1/3
        result = RetrievalMetrics.recall_at_k(retrieved, relevant, 2)
        assert abs(result - 1/3) < 0.001


class TestPrecisionAtK:
    """Tests for precision@k metric."""

    def test_perfect_precision(self):
        """All top-k are relevant → precision@k = 1.0."""
        retrieved = ["A", "B", "C"]
        relevant = {"A", "B", "C", "D"}
        assert RetrievalMetrics.precision_at_k(retrieved, relevant, 3) == 1.0

    def test_partial_precision(self):
        """2 of 5 are relevant → precision@5 = 0.4."""
        retrieved = ["X", "A", "Y", "B", "Z"]
        relevant = {"A", "B"}
        assert abs(RetrievalMetrics.precision_at_k(retrieved, relevant, 5) - 0.4) < 0.001


class TestMRR:
    """Tests for Mean Reciprocal Rank."""

    def test_first_result_relevant(self):
        """First result is relevant → MRR = 1.0."""
        retrieved = ["A", "B", "C"]
        relevant = {"A"}
        assert RetrievalMetrics.mrr(retrieved, relevant) == 1.0

    def test_second_result_relevant(self):
        """Second result is relevant → MRR = 0.5."""
        retrieved = ["X", "A", "B"]
        relevant = {"A"}
        assert RetrievalMetrics.mrr(retrieved, relevant) == 0.5

    def test_no_relevant_found(self):
        """No relevant result → MRR = 0.0."""
        retrieved = ["X", "Y", "Z"]
        relevant = {"A"}
        assert RetrievalMetrics.mrr(retrieved, relevant) == 0.0


class TestNDCG:
    """Tests for nDCG@k with graded relevance."""

    def test_perfect_ranking(self):
        """Ideal ranking (highest grade first) → nDCG = 1.0."""
        retrieved = ["A", "B", "C"]
        grades = {"A": 3, "B": 2, "C": 1}
        assert RetrievalMetrics.ndcg_at_k(retrieved, grades, 3) == 1.0

    def test_worst_ranking(self):
        """All irrelevant → nDCG = 0.0."""
        retrieved = ["X", "Y", "Z"]
        grades = {"A": 3, "B": 2}
        assert RetrievalMetrics.ndcg_at_k(retrieved, grades, 3) == 0.0

    def test_partial_ranking(self):
        """Relevant items present but poorly ordered → 0 < nDCG < 1."""
        retrieved = ["X", "A", "Y", "B", "Z"]
        grades = {"A": 3, "B": 2}
        result = RetrievalMetrics.ndcg_at_k(retrieved, grades, 5)
        assert 0.0 < result < 1.0

    def test_reversed_order_is_worse_than_ideal(self):
        """Lower grade first should score lower than higher grade first."""
        grades = {"A": 3, "B": 1}
        ideal = RetrievalMetrics.ndcg_at_k(["A", "B"], grades, 2)
        reversed_score = RetrievalMetrics.ndcg_at_k(["B", "A"], grades, 2)
        assert ideal > reversed_score
```

#### `test_hybrid.py`

```python
import pytest
from src.retrieval.hybrid import HybridRetriever
from src.indexing.store import RetrievalResult

class TestRRFFusion:
    """Tests for Reciprocal Rank Fusion logic."""

    def test_rrf_item_in_both_indexes_ranks_higher(self):
        """An item appearing in both BM25 and vector results should rank higher."""
        retriever = HybridRetriever(None, None, None)
        bm25_results = [
            type("R", (), {"chunk_id": "A", "content": "a", "score": 1.0, "rank": 1})(),
            type("R", (), {"chunk_id": "B", "content": "b", "score": 0.8, "rank": 2})(),
        ]
        vector_results = [
            type("R", (), {"chunk_id": "B", "content": "b", "score": 0.9, "rank": 1})(),
            type("R", (), {"chunk_id": "C", "content": "c", "score": 0.7, "rank": 2})(),
        ]
        fused = retriever._rrf_fuse(bm25_results, vector_results, final_k=3)
        # B appears in both → higher RRF score → should be rank 1
        assert fused[0].chunk_id == "B"
        assert fused[0].source == "both"

    def test_rrf_score_formula(self):
        """Verify RRF score = 1/(60+rank_bm25) + 1/(60+rank_vector)."""
        retriever = HybridRetriever(None, None, None)
        bm25_results = [
            type("R", (), {"chunk_id": "A", "content": "a", "score": 1.0, "rank": 1})(),
        ]
        vector_results = [
            type("R", (), {"chunk_id": "A", "content": "a", "score": 0.9, "rank": 3})(),
        ]
        fused = retriever._rrf_fuse(bm25_results, vector_results, final_k=1)
        expected = 1/(60+1) + 1/(60+3)
        assert abs(fused[0].rrf_score - expected) < 0.0001

    def test_rrf_empty_results(self):
        """Empty input lists should produce empty output."""
        retriever = HybridRetriever(None, None, None)
        fused = retriever._rrf_fuse([], [], final_k=5)
        assert fused == []
```

#### `test_query_transform.py`

```python
import pytest
from unittest.mock import Mock, patch
import numpy as np

class TestHyDETransformer:
    """Tests for HyDE query transformation."""

    @patch.object(LLMClient, 'generate')
    def test_hyde_generates_hypothetical_doc(self, mock_generate):
        """HyDE should call LLM to generate a hypothetical answer."""
        mock_generate.return_value = "A startup idea can be validated by talking to users..."
        hyde = HyDETransformer(llm=mock_llm, embedder=mock_embedder)
        embedding = hyde.transform("How to validate a startup idea?")
        mock_generate.assert_called_once()
        assert embedding.shape == (1024,)  # BGE-large dimension

    @patch.object(LLMClient, 'generate')
    def test_hyde_uses_passage_embedding(self, mock_generate):
        """HyDE should embed the hypothetical doc as a passage, not a query."""
        mock_generate.return_value = "Hypothetical answer text."
        mock_embedder.embed_passages = Mock(return_value=np.array([[0.1]*1024]))
        mock_embedder.embed_query = Mock(return_value=np.array([0.2]*1024))
        hyde = HyDETransformer(llm=mock_llm, embedder=mock_embedder)
        hyde.transform("test query")
        # Should call embed_passages, NOT embed_query
        mock_embedder.embed_passages.assert_called_once()
        mock_embedder.embed_query.assert_not_called()


class TestMultiQueryTransformer:
    """Tests for multi-query expansion."""

    @patch.object(LLMClient, 'generate')
    def test_generates_n_reformulations(self, mock_generate):
        """Should generate N reformulations plus the original query."""
        mock_generate.return_value = "How can I test a business idea?\nWhat are ways to validate startup concepts?\nHow do founders verify their ideas work?"
        mq = MultiQueryTransformer(llm=mock_llm, embedder=mock_embedder, n_queries=3)
        queries = mq.transform("How to validate a startup idea?")
        assert len(queries) == 4  # original + 3 reformulations
        assert "How to validate a startup idea?" in queries

    def test_merge_results_with_rrf(self):
        """Merging multiple query results should use RRF."""
        list1 = [type("R", (), {"chunk_id": "A", "content": "a", "rank": 1, "rrf_score": 0})()]
        list2 = [type("R", (), {"chunk_id": "A", "content": "a", "rank": 2, "rrf_score": 0})(),
                 type("R", (), {"chunk_id": "B", "content": "b", "rank": 1, "rrf_score": 0})()]
        merged = MultiQueryTransformer.merge_results([list1, list2], final_k=2)
        # A appears in both → higher RRF score → rank 1
        assert merged[0].chunk_id == "A"
```

#### `test_reranker.py`

```python
import pytest
from unittest.mock import Mock, patch
import numpy as np

class TestCrossEncoderReranker:
    """Tests for cross-encoder re-ranking."""

    def test_rerank_reorders_by_cross_encoder_score(self):
        """Re-ranking should reorder candidates by cross-encoder scores."""
        reranker = CrossEncoderReranker.__new__(CrossEncoderReranker)
        reranker.model = Mock()
        # Cross-encoder scores: candidate 0 gets low score, candidate 2 gets high
        reranker.model.predict = Mock(return_value=np.array([0.1, 0.5, 0.9]))
        candidates = [
            RetrievalResult(chunk_id="A", content="aaa", score=0.9, rank=1, source="vector"),
            RetrievalResult(chunk_id="B", content="bbb", score=0.8, rank=2, source="bm25"),
            RetrievalResult(chunk_id="C", content="ccc", score=0.7, rank=3, source="both"),
        ]
        result = reranker.rerank("test query", candidates, top_k=2)
        # C had highest cross-encoder score (0.9) → should be rank 1
        assert result[0].chunk_id == "C"
        assert result[0].rerank_score == 0.9
        assert result[0].new_rank == 1
        assert result[0].original_rank == 3  # was rank 3 before re-ranking
        assert len(result) == 2  # top_k=2

    def test_rerank_empty_candidates(self):
        """Empty candidate list should return empty list."""
        reranker = CrossEncoderReranker.__new__(CrossEncoderReranker)
        reranker.model = Mock()
        result = reranker.rerank("query", [], top_k=5)
        assert result == []

    def test_rerank_top_k_larger_than_candidates(self):
        """top_k > len(candidates) should return all candidates re-ranked."""
        reranker = CrossEncoderReranker.__new__(CrossEncoderReranker)
        reranker.model = Mock()
        reranker.model.predict = Mock(return_value=np.array([0.5, 0.9]))
        candidates = [
            RetrievalResult(chunk_id="A", content="aaa", score=0.9, rank=1, source="vector"),
            RetrievalResult(chunk_id="B", content="bbb", score=0.8, rank=2, source="bm25"),
        ]
        result = reranker.rerank("query", candidates, top_k=10)
        assert len(result) == 2  # only 2 candidates available
```

### 12.3 Integration Tests

#### `test_api.py`

```python
import pytest
from httpx import AsyncClient
from src.api.main import app

@pytest.mark.asyncio
async def test_health_endpoint(test_db):
    """GET /health should return 200 with service status."""
    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.get("/health")
    assert response.status_code == 200
    data = response.json()
    assert data["status"] == "healthy"
    assert "db" in data


@pytest.mark.asyncio
async def test_query_endpoint(test_db, test_corpus):
    """POST /query should return answer with retrieved chunks."""
    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.post("/query", json={
            "query": "What is a startup?",
            "chunking_strategy": "semantic_spacy",
            "retrieval_method": "hybrid",
            "query_transform": "none",
            "reranker": "bge_reranker",
            "top_k": 5,
        })
    assert response.status_code == 200
    data = response.json()
    assert "answer" in data
    assert "retrieved_chunks" in data
    assert len(data["retrieved_chunks"]) <= 5
    assert "latency_ms" in data
    assert "strategy_config" in data


@pytest.mark.asyncio
async def test_query_invalid_strategy(test_db):
    """POST /query with invalid strategy should return 422."""
    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.post("/query", json={
            "query": "test",
            "chunking_strategy": "invalid_strategy",
        })
    assert response.status_code == 422
```

### 12.4 End-to-End Tests

```python
@pytest.mark.asyncio
async def test_full_pipeline_ingest_query_answer(test_db, sample_docs):
    """E2E: ingest documents → query → verify answer references ingested content."""
    # 1. Ingest
    await ingest_documents(sample_docs, strategy="semantic_spacy")

    # 2. Query
    response = await pipeline.query(
        question="What is the main topic of the ingested documents?",
        chunking_strategy="semantic_spacy",
        retrieval_method="hybrid",
    )

    # 3. Verify
    assert response.answer  # non-empty answer
    assert len(response.retrieved_chunks) > 0
    assert response.latency_ms > 0


@pytest.mark.asyncio
async def test_evaluation_runner_produces_metrics(test_db, test_golden):
    """E2E: run evaluation → verify all 54 combos produce metrics."""
    runner = EvaluationRunner(pipeline, test_golden)
    results = await runner.run_all()
    assert len(results) == 54 * len(test_golden)
    # Every result should have computed metrics
    for r in results:
        assert 0.0 <= r["ndcg_at_5"] <= 1.0
        assert 0.0 <= r["recall_at_5"] <= 1.0
        assert r["latency_ms"] > 0
```

### 12.5 Test Fixtures (`conftest.py`)

```python
import pytest
import pytest_asyncio
import asyncpg
import os

@pytest_asyncio.fixture
async def test_db():
    """Provide a clean test database for each test."""
    pool = await asyncpg.create_pool(
        dsn=os.environ["TEST_DATABASE_URL"],
        min_size=1, max_size=5,
    )
    # Clean tables before each test
    async with pool.acquire() as conn:
        await conn.execute("TRUNCATE chunks, documents, eval_results, queries CASCADE")
    yield pool
    await pool.close()


@pytest.fixture
def sample_docs():
    """Small sample documents for testing."""
    return [
        {"path": "test1.txt", "content": "Startups should focus on building something users love."},
        {"path": "test2.txt", "content": "Competition is less important than understanding user needs."},
    ]


@pytest.fixture
def test_golden():
    """Small golden dataset for testing."""
    return [
        {
            "id": "test-q1",
            "question": "What should startups focus on?",
            "expected_answer": "Building something users love.",
            "relevant_chunk_ids": [],  # populated after ingestion
            "relevance_grades": {},
            "difficulty": "easy",
            "category": "startup",
        }
    ]
```

### 12.6 Test Execution

```bash
# Run all tests
pytest tests/ -v

# Run with coverage
pytest tests/ --cov=src --cov-report=term-missing --cov-report=html

# Run only unit tests (fast)
pytest tests/ -v -k "not integration and not e2e"

# Run only metrics tests
pytest tests/test_metrics.py -v
```

---

## 13. Deployment

### 13.1 Docker Compose (`docker-compose.yml`)

```yaml
version: "3.9"

services:
  postgres:
    image: pgvector/pgvector:pg16
    container_name: rag-postgres
    environment:
      POSTGRES_USER: rag
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-ragpass_dev}
      POSTGRES_DB: ragdb
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./sql/schema.sql:/docker-entrypoint-initdb.d/01-schema.sql:ro
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U rag -d ragdb"]
      interval: 5s
      timeout: 5s
      retries: 5
    networks:
      - rag-network

  api:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: rag-api
    environment:
      - DATABASE_URL=postgresql://rag:ragpass_dev@postgres:5432/ragdb
      - LLM_PROVIDER=${LLM_PROVIDER:-ollama}
      - OPENAI_API_KEY=${OPENAI_API_KEY:-}
      - OPENAI_MODEL=${OPENAI_MODEL:-gpt-4o-mini}
      - OLLAMA_BASE_URL=${OLLAMA_BASE_URL:-http://host.docker.internal:11434}
      - OLLAMA_MODEL=${OLLAMA_MODEL:-llama3.1:8b}
      - EMBEDDING_MODEL=BAAI/bge-large-en-v1.5
      - RERANKER_MODEL=BAAI/bge-reranker-large
      - DEVICE=${DEVICE:-cpu}
    ports:
      - "8000:8000"
    depends_on:
      postgres:
        condition: service_healthy
    volumes:
      - ./data:/app/data:ro
      - model_cache:/root/.cache/huggingface
    networks:
      - rag-network

  dashboard:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: rag-dashboard
    command: streamlit run src/dashboard/app.py --server.port 8501 --server.address 0.0.0.0
    environment:
      - DATABASE_URL=postgresql://rag:ragpass_dev@postgres:5432/ragdb
    ports:
      - "8501:8501"
    depends_on:
      postgres:
        condition: service_healthy
    volumes:
      - model_cache:/root/.cache/huggingface:ro
    networks:
      - rag-network

volumes:
  postgres_data:
  model_cache:

networks:
  rag-network:
    driver: bridge
```

### 13.2 Dockerfile

```dockerfile
FROM python:3.11-slim AS base

# System dependencies for spaCy and sentence-transformers
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Install Python dependencies
COPY pyproject.toml .
RUN pip install --no-cache-dir -e ".[api,dashboard]"

# Download spaCy model
RUN python -m spacy download en_core_web_sm

# Pre-download models (optional — can also download on first run)
# RUN python -c "from sentence_transformers import SentenceTransformer; SentenceTransformer('BAAI/bge-large-en-v1.5')"
# RUN python -c "from sentence_transformers import CrossEncoder; CrossEncoder('BAAI/bge-reranker-large')"

COPY . .

EXPOSE 8000

CMD ["uvicorn", "src.api.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### 13.3 Environment Variables (`.env.example`)

```bash
# ============================================================
# Database
# ============================================================
POSTGRES_PASSWORD=ragpass_dev
DATABASE_URL=postgresql://rag:ragpass_dev@localhost:5432/ragdb
TEST_DATABASE_URL=postgresql://rag:ragpass_dev@localhost:5432/ragdb_test

# ============================================================
# LLM Configuration
# ============================================================
# Choose provider: 'openai' or 'ollama'
LLM_PROVIDER=ollama

# OpenAI settings (if LLM_PROVIDER=openai)
OPENAI_API_KEY=sk-your-key-here
OPENAI_MODEL=gpt-4o-mini
OPENAI_BASE_URL=https://api.openai.com/v1

# Ollama settings (if LLM_PROVIDER=ollama)
OLLAMA_BASE_URL=http://localhost:11434
OLLAMA_MODEL=llama3.1:8b

# ============================================================
# Embedding & Reranker Models
# ============================================================
EMBEDDING_MODEL=BAAI/bge-large-en-v1.5
RERANKER_MODEL=BAAI/bge-reranker-large
DEVICE=cpu  # or 'cuda' if GPU available

# ============================================================
# Retrieval Defaults
# ============================================================
DEFAULT_CHUNKING_STRATEGY=semantic_spacy
DEFAULT_RETRIEVAL_METHOD=hybrid
DEFAULT_QUERY_TRANSFORM=none
DEFAULT_RERANKER=bge_reranker
DEFAULT_TOP_K=5
DEFAULT_RETRIEVE_K=20
RRF_K=60

# ============================================================
# API
# ============================================================
API_HOST=0.0.0.0
API_PORT=8000
CORS_ORIGINS=http://localhost:3000,http://localhost:8501
```

### 13.4 `pyproject.toml`

```toml
[project]
name = "advanced-rag"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
    "fastapi>=0.110",
    "uvicorn[standard]>=0.27",
    "pydantic>=2.6",
    "pydantic-settings>=2.1",
    "asyncpg>=0.29",
    "pgvector>=0.3",
    "sentence-transformers>=2.5",
    "spacy>=3.7",
    "httpx>=0.27",
    "numpy>=1.26",
    "python-dotenv>=1.0",
]

[project.optional-dependencies]
api = [
    "python-multipart>=0.0.9",
]
dashboard = [
    "streamlit>=1.35",
    "plotly>=5.20",
    "pandas>=2.2",
]
dev = [
    "pytest>=8.0",
    "pytest-asyncio>=0.23",
    "pytest-cov>=4.1",
    "httpx>=0.27",
    "ruff>=0.3",
    "mypy>=1.8",
]

[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]

[tool.ruff]
line-length = 100
target-version = "py311"

[tool.mypy]
python_version = "3.11"
strict = true
```

### 13.5 Deployment Steps

```bash
# 1. Clone and configure
git clone <repo-url> advanced-rag
cd advanced-rag
cp .env.example .env
# Edit .env with your settings

# 2. Start infrastructure
docker compose up -d postgres

# 3. Initialize database schema
docker compose exec api python scripts/init_db.py

# 4. Ingest documents
docker compose exec api python scripts/ingest.py --source data/raw/ --strategy all

# 5. Create golden dataset (manual or semi-automated)
# Edit data/golden/golden_dataset.jsonl

# 6. Run evaluation
docker compose exec api python scripts/run_eval.py --golden data/golden/golden_dataset.jsonl

# 7. Start API and dashboard
docker compose up -d

# 8. Access services
# API:        http://localhost:8000/docs
# Dashboard:  http://localhost:8501
```

---

## 14. Roadmap & Milestones

### Milestone Timeline

```
Week 1
  Day 1  ██████  Environment & Schema           → M1: Infrastructure ready
  Day 2  ██████  Ingestion & Chunking            → M2: 3 chunking strategies working
  Day 3  ██████  Embedding & Dual Indexing       → M3: BM25 + Vector search operational
  Day 4  ██████  Hybrid Retrieval + RRF          → M4: Hybrid search returning fused results
  Day 5  ██████  Query Transformation            → M5: HyDE + Multi-query functional

Week 2
  Day 6  ██████  Cross-Encoder Re-ranking        → M6: Re-ranker integrated
  Day 7  ██████  FastAPI Endpoint                → M7: API serving queries
  Day 8  ██████  Evaluation Metrics & Golden     → M8: Metrics computed, golden dataset ready
  Day 9  ██████  Streamlit Dashboard             → M9: Dashboard showing all comparisons
  Day 10 ██████  Testing & Polish                → M10: Full test suite, README, demo
```

### Milestone Deliverables

| Milestone | Day | Deliverable | Verification |
-----------|-----|-------------|--------------|
| M1 | 1 | Docker Compose + DB schema | `docker compose up` + schema applied |
| M2 | 2 | 3 chunking strategies | Chunk counts per strategy in DB |
| M3 | 3 | BM25 + Vector indexes | Both search methods return results |
| M4 | 4 | Hybrid retrieval with RRF | Fused results with RRF scores |
| M5 | 5 | HyDE + Multi-query | Transform modules produce embeddings |
| M6 | 6 | Cross-encoder re-ranking | Top-20 → top-5 with rerank scores |
| M7 | 7 | FastAPI `/query` endpoint | `curl POST /query` returns answer |
| M8 | 8 | Evaluation suite | 54 combos × N questions in eval_results |
| M9 | 9 | Streamlit dashboard | Dashboard shows comparison tables + charts |
| M10 | 10 | Tests + docs | `pytest` passes, README complete, demo works |

### Future Enhancements (Post-Project)

| Enhancement | Priority | Effort | Description |
-------------|----------|--------|-------------|
| Qdrant migration | Medium | 2 days | Swap pgvector for Qdrant via `VectorStore` interface for >100K chunks |
| Cohere Rerank API | Low | 1 day | Alternative to BGE-reranker; API-based, no local model needed |
| LLM-as-judge evaluation | Medium | 2 days | Use LLM to grade answer quality (faithfulness, relevance) beyond retrieval metrics |
| Query routing | Medium | 2 days | Route simple queries to BM25-only, complex to hybrid+HyDE based on query classification |
| Incremental indexing | Low | 3 days | Support adding/removing documents without full re-index |
| Multi-language support | Low | 3 days | Add multilingual embedding model + language detection |
| Graph RAG | Medium | 5 days | Extract entity relationships, augment retrieval with graph traversal |

---

## 15. Risk Register

| # | Risk | Likelihood | Impact | Mitigation | Contingency |
|---|------|-----------|--------|------------|-------------|
| R1 | BGE-large model too slow on CPU | Medium | Medium | Use `bge-base-en-v1.5` (512-dim) as fallback; reduce batch size | Switch to OpenAI embeddings API (no local model) |
| R2 | pgvector HNSW index build time excessive | Low | Low | Build indexes in parallel; use IVFFlat as alternative | Use exact search (no ANN index) for small corpora |
| R3 | spaCy sentence segmentation poor on domain text | Medium | Medium | Test on actual corpus; consider custom sentence boundary rules | Fall back to fixed-size chunking |
| R4 | Golden dataset too small for meaningful comparison | Medium | High | Aim for 30-50 questions; use stratified sampling across categories | Use LLM-generated questions + human review |
| R5 | LLM API costs during evaluation (54 combos × 50 Qs) | Medium | Medium | Use Ollama (local, free) for development; cache HyDE/multi-query results | Reduce combo count; skip HyDE/multi-query for initial run |
| R6 | Evaluation takes too long (2700 queries) | Medium | Low | Implement checkpointing; run combos in parallel; skip redundant combos | Reduce to key combos (e.g., 12 instead of 54) |
| R7 | Cross-encoder re-ranking degrades results | Low | Medium | Compare with/without reranker in evaluation; check for score calibration issues | Disable reranker; use hybrid-only |
| R8 | HyDE generates poor hypothetical documents | Medium | Medium | Use low temperature for generation; inspect HyDE outputs manually | Fall back to multi-query or raw query |
| R9 | Postgres connection pool exhaustion | Low | Medium | Configure pool size appropriately; use connection per request pattern | Increase pool size; add retry logic |
| R10 | Docker resource constraints (RAM for models) | Medium | Medium | Ensure ≥4GB RAM for container; use smaller models if needed | Use API-based embeddings/reranking (no local models) |
| R11 | Inconsistent results between dev and Docker environments | Low | Low | Pin all dependency versions in `pyproject.toml`; use Docker for all runs | Debug with `pip freeze` comparison |
| R12 | Relevance grades in golden dataset are subjective | High | Medium | Have 2 people grade independently; compute inter-annotator agreement | Use binary relevance (relevant/not) instead of graded |

---

## 16. Appendix

### A. Quick Start

```bash
# 1. Prerequisites: Docker, Docker Compose, Python 3.11+

# 2. Clone and setup
git clone <repo-url> advanced-rag
cd advanced-rag
cp .env.example .env

# 3. Start everything
docker compose up -d

# 4. Initialize database (auto-runs schema.sql on first start)
docker compose exec api python scripts/init_db.py

# 5. Ingest sample documents
docker compose exec api python scripts/ingest.py --source data/raw/ --strategy all

# 6. Query the API
curl -X POST http://localhost:8000/query \
  -H "Content-Type: application/json" \
  -d '{"query": "What makes a good startup idea?"}' | python -m json.tool

# 7. View evaluation dashboard
open http://localhost:8501

# 8. Run full evaluation suite
docker compose exec api python scripts/run_eval.py --golden data/golden/golden_dataset.jsonl
```

### B. Key Design Decisions

| Decision | Choice | Alternatives Considered | Rationale |
----------|--------|------------------------|-----------|
| Vector DB | pgvector | Qdrant, Weaviate, Milvus | Single infrastructure with BM25; data engineer familiarity; sufficient for eval scale |
| Embedding model | BGE-large-en-v1.5 | OpenAI text-embedding-3-large, MiniLM | Open-source, top MTEB scores, runs locally, no API cost |
| Re-ranker | BGE-reranker-large | Cohere Rerank API, ms-marco-MiniLM | Open-source, strong benchmarks, no API cost; Cohere as alternative |
| Fusion method | RRF (k=60) | Weighted score fusion, ConvexERA | Rank-based (no score calibration needed); proven in research; simple |
| Chunking comparison | One table, strategy column | Separate tables per strategy | Enables direct SQL comparison; single query point for all strategies |
| LLM provider | Dual (OpenAI + Ollama) | OpenAI only, Ollama only | Flexibility: Ollama for free dev, OpenAI for quality; abstracted via `LLMClient` |
| Eval metrics | recall@k, precision@k, MRR, nDCG | MAP, Hit Rate | Covers coverage (recall), efficiency (precision), speed (MRR), and ordering (nDCG) |
| Dashboard | Streamlit | Gradio, Dash, custom React | Fastest to build; data analysts familiar with it; sufficient for eval display |
| Framework | LangChain | LlamaIndex, custom | Widely used; good abstractions for query transform; not locked in (can swap) |
| Test DB | Separate test database | SQLite mock, in-memory | Tests against real Postgres + pgvector; catches extension-specific issues |

### C. Glossary

| Term | Definition |
------|-----------|
| **RAG** | Retrieval-Augmented Generation — combining retrieved context with LLM generation |
| **BM25** | Best Matching 25 — a ranking function for full-text search based on term frequency and document length |
| **pgvector** | PostgreSQL extension for vector similarity search |
| **HNSW** | Hierarchical Navigable Small World — an approximate nearest neighbor (ANN) index algorithm |
| **RRF** | Reciprocal Rank Fusion — a rank-based method for combining multiple result lists |
| **HyDE** | Hypothetical Document Embeddings — generating a hypothetical answer and using its embedding for search |
| **Cross-encoder** | A model that takes (query, passage) pairs as input and outputs a relevance score; more accurate than bi-encoders but slower |
| **Bi-encoder** | A model that encodes query and passage separately into embeddings; fast but less accurate than cross-encoders |
| **nDCG** | Normalized Discounted Cumulative Gain — a metric that evaluates ranking quality with graded relevance |
| **MRR** | Mean Reciprocal Rank — average of 1/rank of the first relevant result |
| **Recall@k** | Fraction of relevant items retrieved in the top-k results |
| **Precision@k** | Fraction of top-k retrieved items that are relevant |
| **Golden dataset** | A manually curated set of questions with known relevant documents, used as ground truth for evaluation |
| **Chunking** | Splitting documents into smaller units for indexing and retrieval |
| **Sentence-window** | A chunking strategy where each chunk is one sentence, with surrounding sentences stored as context |

### D. References

| Topic | Reference |
-------|-----------|
| HyDE | Gao et al. (2023) "Precise Zero-Shot Dense Retrieval without Relevance Data" — arXiv:2212.10496 |
| Multi-Query / RAG-Fusion | Wang et al. (2023) "Query Expansion by Prompting Large Language Models" |
| RRF | Cormack et al. (2009) "Reciprocal Rank Fusion outperforms Condorcet and individual Rank Learning Methods" — SIGIR |
| BGE Embeddings | Xiao et al. (2023) "C-Pack: Packaged Resources To Advance General Chinese Embedding" — BAAI |
| BGE Reranker | BAAI/bge-reranker-large — HuggingFace model card |
| pgvector | https://github.com/pgvector/pgvector |
| Cross-encoders | Nogueira & Cho (2019) "Passage Re-ranking with BERT" |
| RAG | Lewis et al. (2020) "Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks" — NeurIPS |
| Evaluation Metrics | Manning et al. (2008) "Introduction to Information Retrieval" — Cambridge University Press |

### E. Expected Evaluation Outcomes

When you run the evaluation suite, you should observe patterns like these (your actual numbers will vary):

1. **Hybrid > BM25-only > Vector-only** (on most queries)
   - Hybrid leverages both exact matching and semantic matching
   - BM25 wins on queries with exact terminology; Vector wins on paraphrased queries

2. **Re-ranking provides the largest single improvement**
   - Expect +10-20% nDCG@5 from cross-encoder re-ranking
   - This is because the bi-encoder's ranking is approximate; the cross-encoder fixes ordering errors

3. **Semantic chunking > Fixed-size > Sentence-window** (for nDCG, generally)
   - Semantic chunks are more coherent → better embedding quality
   - Sentence-window may win on precision@1 (pinpoint accuracy) but lose on recall (smaller chunks = more chunks to match)

4. **HyDE helps complex queries, hurts simple ones**
   - Complex/abstract queries benefit from the semantic bridge
   - Simple factual queries may be degraded because the hypothetical document introduces noise

5. **Multi-query improves recall at the cost of latency**
   - More queries = more chances to find relevant chunks
   - But 3× vector search + LLM call adds significant latency

6. **Latency ranking**: `none < hybrid < reranker < HyDE < multi_query < HyDE+reranker`
   - The best quality (HyDE + hybrid + reranker) costs ~400-600ms per query
   - The cheapest (BM25 only, no transform, no reranker) costs ~50ms
   - The dashboard's latency-vs-quality scatter plot visualizes this trade-off

### F. Key Takeaway for the AI Engineering Transition

``+```
┌────────────────────────────────────────────────────────────────────────────┐
│                                                                            │
│  THE MOST IMPORTANT LESSON FROM THIS PROJECT:                              │
│                                                                            │
│  Retrieval quality is the hard part of RAG, not the LLM call.              │
│                                                                            │
│  As a data engineer, you already understand:                               │
│    • Data quality determines downstream quality (garbage in, garbage out)  │
│    • You need to measure before you optimize                               │
│    • Different strategies have different trade-offs                        │
│                                                                            │
│  RAG retrieval is the same discipline applied to unstructured text:        │
│    • Chunking = data partitioning strategy                                 │
│    • Indexing = choosing the right access path (B-tree vs hash vs HNSW)    │
│    • Hybrid search = combining access paths (index merge)                  │
│    • Re-ranking = post-processing / sorting after fetch                    │
│    • Evaluation = data quality monitoring                                  │
│                                                                            │
│  The evaluation dashboard IS the deliverable.                              │
│  The ability to measure and compare IS the skill.                          │
│  The specific strategy that wins IS context-dependent.                     │
│                                                                            │
│  Build the measurement system first. Then optimize.                       │
│  This is the data engineering mindset applied to AI engineering.           │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```