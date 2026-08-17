# Secure Customer RAG System — Detailed Implementation Plan

> **Elevator pitch:** A Retrieval-Augmented Generation system that lets customer-facing teams query sensitive documents via natural language — with strict per-tenant permission filtering, traceable citations on every answer, and a full audit trail of who queried what and what was returned.

---

## Table of Contents

1. [Project Goals & Non-Goals](#1-project-goals--non-goals)
2. [Architecture Overview](#2-architecture-overview)
3. [Tech Stack](#3-tech-stack)
4. [Database Schema](#4-database-schema)
5. [Project Structure](#5-project-structure)
6. [Implementation Phases](#6-implementation-phases)
7. [Component Specifications](#7-component-specifications)
8. [Security Model](#8-security-model)
9. [API Specification](#9-api-specification)
10. [Testing Strategy](#10-testing-strategy)
11. [Deployment](#11-deployment)
12. [Roadmap & Milestones](#12-roadmap--milestones)
13. [Risk Register](#13-risk-register)

---

## 1. Project Goals & Non-Goals

### Goals

| # | Goal | Success Metric |
|---|------|----------------|
| G1 | **Tenant isolation** — no customer can retrieve another customer's documents | Zero cross-tenant leakage in test suite (automated + manual) |
| G2 | **Role-based access control (RBAC)** — within a tenant, not all roles see all docs | All chunks filtered by `allowed_roles` before reaching the LLM |
| G3 | **Citation tracking** — every claim in an answer links back to a source chunk | 100% of answers include structured citations with doc path + chunk ID |
| G4 | **Audit logging** — every query is traceable to user, retrieved chunks, and generated answer | Audit log row per query, queryable by user/tenant/timestamp |
| G5 | **Grounded answers** — LLM uses only retrieved context, not its own training data | Prompt enforces context-only; citations allow human verification |
| G6 | **Production-deployable** — containerized, documented, testable end-to-end | `docker compose up` starts the full system; health check passes |

### Non-Goals (explicitly out of scope)

- Multi-language document support (v1 is English-only)
- Real-time streaming responses (v1 returns complete answers)
- Fine-tuning custom embedding or LLM models
- Federated / cross-tenant search (by design, the opposite of our goal)
- Web UI / chat frontend (v1 is API-only; a UI is a separate project)
- Document versioning / diffing (latest ingest overwrites previous)

---

## 2. Architecture Overview

```
                                    ┌─────────────────────────────────────────────┐
                                    │              Client / API Caller             │
                                    │   (JWT token contains: user_id, customer_id, │
                                    │    roles[])                                  │
                                    └──────────────┬──────────────────────────────┘
                                                   │ HTTPS
                                    ┌──────────────▼──────────────────────────────┐
                                    │              FastAPI Application             │
                                    │  ┌─────────┐  ┌──────────┐  ┌────────────┐  │
                                    │  │  Auth   │  │  Upload  │  │   Query    │  │
                                    │  │ Middlew │  │  Route   │  │   Route   │  │
                                    │  └────┬────┘  └────┬─────┘  └─────┬──────┘  │
                                    │       │            │               │        │
                                    │  ┌────▼────────────▼───────────────▼──────┐ │
                                    │  │            Core Pipeline               │ │
                                    │  │                                         │ │
                                    │  │  ┌──────────┐  ┌────────────────────┐  │ │
                                    │  │  │ Ingest   │  │  Retrieve          │  │ │
                                    │  │  │ Pipeline │  │  (vector search    │  │ │
                                    │  │  │          │  │   + perm filter)   │  │ │
                                    │  │  └────┬─────┘  └────────┬───────────┘  │ │
                                    │  │       │                 │              │ │
                                    │  │       │          ┌──────▼────────┐     │ │
                                    │  │       │          │  Answer Gen   │     │ │
                                    │  │       │          │  (LLM + cite) │     │ │
                                    │  │       │          └──────┬────────┘     │ │
                                    │  │       │                 │              │ │
                                    │  │  ┌────▼─────────────────▼───────────┐  │ │
                                    │  │  │         Audit Logger              │  │ │
                                    │  │  │  (every query → audit_logs row)   │  │ │
                                    │  │  └───────────────────────────────────┘  │ │
                                    │  └─────────────────────────────────────────┘ │
                                    └──────────────────┬──────────────────────────┘
                                                       │
                         ┌─────────────────────────────┼──────────────────────────┐
                         │                             │                          │
                ┌────────▼────────┐           ┌────────▼────────┐       ┌────────▼────────┐
                │  PostgreSQL     │           │  Embedding      │       │  LLM API        │
                │  + pgvector     │           │  Model          │       │  (OpenAI /      │
                │                 │           │  (OpenAI /      │       │   local Llama)  │
                │  Tables:        │           │   local BGE)    │       │                 │
                │  - chunks       │           │                 │       │                 │
                │  - documents    │           │                 │       │                 │
                │  - audit_logs   │           │                 │       │                 │
                │  - users        │           │                 │       │                 │
                └─────────────────┘           └─────────────────┘       └─────────────────┘
```

### Request Flow (Query)

```
1. Client sends POST /query {question: "..."} + JWT
2. Auth middleware decodes JWT → {user_id, customer_id, roles}
3. Query route:
   a. Embed the question → query_vector
   b. Vector search in pgvector WHERE customer_id = ? (pre-filter, tenant isolation)
   c. Retrieve top-k×3 candidates
   d. Post-filter by allowed_roles ∩ user.roles
   e. Take top-k surviving chunks
   f. Build prompt with context + citation instructions
   g. Call LLM → answer text
   h. Extract citations (doc_path, chunk_id, snippet) from retrieved chunks
   i. Audit log: {user_id, customer_id, question, chunk_ids, answer, citations, timestamp}
   j. Return {answer, citations}
```

### Request Flow (Upload)

```
1. Client sends POST /upload (multipart file) + JWT
2. Auth middleware checks roles include "admin" or "editor"
3. Upload route:
   a. Parse file (PDF, TXT, DOCX, MD)
   b. Extract text
   c. Chunk via RecursiveCharacterTextSplitter
   d. Embed each chunk
   e. Store chunks with metadata: {doc_id, customer_id, allowed_roles, doc_path}
   f. Store document metadata in documents table
   g. Audit log: {user_id, customer_id, action: "upload", doc_id, chunk_count}
   h. Return {doc_id, chunk_count}
```

---

## 3. Tech Stack

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| **Language** | Python 3.11+ | Ecosystem maturity for AI/ML, async support |
| **Web Framework** | FastAPI | Async, auto OpenAPI docs, Pydantic validation |
| **Database** | PostgreSQL 16 + pgvector | Single system for relational data + vector search; SQL-based permission filtering is the core security mechanism |
| **Embedding Model** | `text-embedding-3-small` (OpenAI) | 1536-dim, $0.02/1M tokens, good quality. Alternative: `bge-large-en-v1.5` (local, free, 1024-dim) |
| **LLM** | `gpt-4o-mini` (OpenAI) | Fast, cheap, good at following citation instructions. Alternative: Llama 3.1 8B via Ollama for fully on-prem |
| **Document Parsing** | PyPDF2 (PDF), python-docx (DOCX), raw (TXT/MD) | Covers common enterprise document formats |
| **Chunking** | LangChain `RecursiveCharacterTextSplitter` | Industry standard; recursive splitting preserves semantic boundaries |
| **Auth** | JWT (PyJWT) | Stateless, standard, carries customer_id + roles in claims |
| **Containerization** | Docker + Docker Compose | Reproducible dev + prod environment |
| **Testing** | pytest + pytest-asyncio + httpx (API tests) | Async test support, FastAPI-native test client |
| **Linting** | ruff | Fast, replaces flake8 + isort + black |

### Why pgvector over dedicated vector DBs (Pinecone, Qdrant, Weaviate)?

**The security model requires SQL-level filtering.** pgvector lets us write:

```sql
SELECT * FROM chunks
WHERE customer_id = $1          -- tenant isolation (hard boundary)
  AND allowed_roles && $2::text[]  -- role intersection
ORDER BY embedding <=> $3       -- vector similarity
LIMIT $4;
```

Dedicated vector DBs have weaker filtering capabilities or require separate metadata stores, creating a sync problem and a security gap. With pgvector, the permission check and the vector search are **one atomic SQL query** — no race conditions, no stale metadata.

---

## 4. Database Schema

### 4.1 Extensions

```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "vector";
```

### 4.2 Tables

```sql
-- =====================================================
-- USERS: maps JWT subjects to tenant + roles
-- =====================================================
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    external_id     TEXT NOT NULL UNIQUE,          -- sub from JWT / external IdP
    email           TEXT,
    customer_id     TEXT NOT NULL,                 -- tenant identifier
    roles           TEXT[] NOT NULL DEFAULT '{}',  -- e.g. {'admin', 'editor', 'viewer'}
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    -- A user belongs to exactly one tenant
    CONSTRAINT roles_not_empty CHECK (array_length(roles, 1) > 0 OR roles = '{}')
);

CREATE INDEX idx_users_customer ON users(customer_id);
CREATE INDEX idx_users_external ON users(external_id);

-- =====================================================
-- DOCUMENTS: metadata for each ingested document
-- =====================================================
CREATE TABLE documents (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    customer_id     TEXT NOT NULL,                 -- tenant isolation
    file_name       TEXT NOT NULL,
    file_path       TEXT NOT NULL,                 -- storage path (S3 key / local path)
    file_type       TEXT NOT NULL,                 -- 'pdf', 'txt', 'docx', 'md'
    file_hash       TEXT NOT NULL,                 -- SHA-256 for dedup
    chunk_count     INT NOT NULL DEFAULT 0,
    allowed_roles   TEXT[] NOT NULL DEFAULT '{}',  -- which roles can see this doc's chunks
    uploaded_by     UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    -- Prevent duplicate uploads within same tenant
    UNIQUE(customer_id, file_hash)
);

CREATE INDEX idx_documents_customer ON documents(customer_id);
CREATE INDEX idx_documents_roles ON documents USING GIN(allowed_roles);

-- =====================================================
-- CHUNKS: the actual vector-indexed text segments
-- =====================================================
CREATE TABLE chunks (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    doc_id          UUID NOT NULL REFERENCES documents(id) ON DELETE CASCADE,
    customer_id     TEXT NOT NULL,                 -- denormalized for fast filtering
    chunk_idx       INT NOT NULL,                  -- position within document
    content         TEXT NOT NULL,                 -- the actual text
    embedding       vector(1536) NOT NULL,         -- pgvector column
    allowed_roles   TEXT[] NOT NULL DEFAULT '{}',  -- denormalized from document
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    -- Chunk belongs to exactly one document in one tenant
    UNIQUE(doc_id, chunk_idx)
);

-- HNSW index for fast approximate nearest neighbor search
CREATE INDEX idx_chunks_embedding ON chunks
    USING hnsw (embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 64);

-- B-tree for tenant pre-filter (used before/in vector search)
CREATE INDEX idx_chunks_customer ON chunks(customer_id);

-- GIN for role intersection post-filter
CREATE INDEX idx_chunks_roles ON chunks USING GIN(allowed_roles);

-- Composite: tenant + roles for hybrid filter
CREATE INDEX idx_chunks_customer_roles ON chunks(customer_id, allowed_roles);

-- =====================================================
-- AUDIT LOGS: immutable record of every query
-- =====================================================
CREATE TABLE audit_logs (
    id                  UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    timestamp           TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    user_id             UUID REFERENCES users(id),
    customer_id         TEXT NOT NULL,             -- denormalized for tenant-scoped queries
    action              TEXT NOT NULL,             -- 'query', 'upload', 'delete'
    query_text          TEXT,                      -- the user's question (for 'query' action)
    retrieved_chunk_ids UUID[],                    -- which chunks were retrieved
    answer_text         TEXT,                      -- the generated answer
    citations           JSONB,                     -- structured citation array
    latency_ms          INT,                       -- end-to-end query latency
    success             BOOLEAN NOT NULL DEFAULT TRUE,
    error_message       TEXT,

    -- Append-only: no UPDATE or DELETE allowed
    CONSTRAINT no_update CHECK (true)  -- enforced via trigger, see below
);

CREATE INDEX idx_audit_customer_time ON audit_logs(customer_id, timestamp DESC);
CREATE INDEX idx_audit_user_time ON audit_logs(user_id, timestamp DESC);
CREATE INDEX idx_audit_action ON audit_logs(action);

-- Trigger: prevent UPDATE and DELETE on audit_logs
CREATE OR REPLACE FUNCTION prevent_audit_modification()
RETURNS TRIGGER AS $$
BEGIN
    RAISE EXCEPTION 'audit_logs is append-only; UPDATE and DELETE are forbidden';
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER no_audit_update
    BEFORE UPDATE ON audit_logs
    FOR EACH ROW EXECUTE FUNCTION prevent_audit_modification();

CREATE TRIGGER no_audit_delete
    BEFORE DELETE ON audit_logs
    FOR EACH ROW EXECUTE FUNCTION prevent_audit_modification();
```

### 4.3 Row-Level Security (Defense in Depth)

Even if application code has a bug, RLS prevents cross-tenant access at the database level:

```sql
-- Enable RLS on chunks and documents
ALTER TABLE chunks ENABLE ROW LEVEL SECURITY;
ALTER TABLE documents ENABLE ROW LEVEL SECURITY;
ALTER TABLE audit_logs ENABLE ROW LEVEL SECURITY;

-- Policy: users can only see rows from their own customer_id
-- app.current_customer_id is set per-request via SET LOCAL
CREATE POLICY tenant_isolation_chunks ON chunks
    USING (customer_id = current_setting('app.current_customer_id'));

CREATE POLICY tenant_isolation_documents ON documents
    USING (customer_id = current_setting('app.current_customer_id'));

CREATE POLICY tenant_isolation_audit ON audit_logs
    USING (customer_id = current_setting('app.current_customer_id'));
```

In the application, every request sets the tenant context:

```python
# Per-request: set tenant context for RLS
await db.execute(f"SET LOCAL app.current_customer_id = '{customer_id}'")
```

---

## 5. Project Structure

```
secure-rag/
├── docker-compose.yml
├── .env.example
├── pyproject.toml
├── README.md
├── IMPLEMENTATION_PLAN.md          ← this file
├── agent.md                      # Herdr multi-agent orchestration guide
│
├── src/
│   ├── __init__.py
│   ├── main.py                     # FastAPI app factory + lifespan
│   ├── config.py                   # Pydantic Settings (env vars)
│   │
│   ├── auth/
│   │   ├── __init__.py
│   │   ├── jwt_handler.py          # JWT decode + verify
│   │   ├── models.py               # User, TokenPayload pydantic models
│   │   └── middleware.py           # AuthMiddleware: extract user from JWT
│   │
│   ├── db/
│   │   ├── __init__.py
│   │   ├── connection.py           # asyncpg pool + pgvector setup
│   │   ├── schema.sql              # full DDL (tables, indexes, RLS, triggers)
│   │   └── migrations/
│   │       ├── 001_initial.sql
│   │       └── 002_rls.sql
│   │
│   ├── ingest/
│   │   ├── __init__.py
│   │   ├── parser.py               # PDF/DOCX/TXT/MD text extraction
│   │   ├── chunker.py              # RecursiveCharacterTextSplitter wrapper
│   │   ├── embedder.py             # Embedding model interface (OpenAI / local)
│   │   └── service.py              # Orchestrates: parse → chunk → embed → store
│   │
│   ├── retrieve/
│   │   ├── __init__.py
│   │   ├── vector_search.py        # pgvector cosine similarity query
│   │   ├── permission_filter.py    # tenant pre-filter + role post-filter
│   │   └── service.py              # Orchestrates: embed query → search → filter
│   │
│   ├── answer/
│   │   ├── __init__.py
│   │   ├── prompt_builder.py       # Context + citation instruction prompt
│   │   ├── llm_client.py           # LLM interface (OpenAI / Ollama)
│   │   ├── citation_extractor.py   # Parse LLM output → structured citations
│   │   └── service.py              # Orchestrates: build prompt → LLM → citations
│   │
│   ├── audit/
│   │   ├── __init__.py
│   │   └── logger.py               # Append-only audit log writer
│   │
│   └── api/
│       ├── __init__.py
│       ├── routes/
│       │   ├── __init__.py
│       │   ├── upload.py           # POST /upload
│       │   ├── query.py            # POST /query
│       │   ├── documents.py        # GET /documents, DELETE /documents/{id}
│       │   └── audit.py            # GET /audit (admin only)
│       └── dependencies.py         # FastAPI dependency injection (get current user)
│
├── tests/
│   ├── __init__.py
│   ├── conftest.py                 # pytest fixtures: test DB, test client, seed data
│   ├── test_auth.py                # JWT validation, expired tokens, wrong tenant
│   ├── test_ingest.py              # File parsing, chunking, metadata storage
│   ├── test_retrieve.py            # Vector search, tenant isolation, role filtering
│   ├── test_answer.py              # Prompt building, citation extraction
│   ├── test_audit.py               # Audit log immutability, completeness
│   ├── test_api_upload.py          # End-to-end upload via API
│   ├── test_api_query.py           # End-to-end query via API
│   ├── test_security.py            # Cross-tenant leakage tests (critical)
│   └── test_rls.py                 # Row-level security enforcement
│
├── scripts/
│   ├── init_db.py                  # Run schema.sql against fresh database
│   ├── seed_test_data.py           # Insert test users, documents, chunks
│   └── generate_jwt.py             # Helper to create test JWT tokens
│
└── docker/
    ├── Dockerfile                  # App image
    └── postgres/
        └── init.sql                # pgvector extension + schema on first boot
```

---

## 6. Implementation Phases

### Phase 1: Foundation (Days 1-2)

**Goal:** Running Postgres + pgvector + FastAPI skeleton with auth.

| Step | Task | Deliverable |
|------|------|-------------|
| 1.1 | Write `docker-compose.yml` with Postgres+pgvector and app service | `docker compose up` starts both |
| 1.2 | Write `docker/postgres/init.sql` (extensions + schema) | DB initializes on first boot |
| 1.3 | Write `src/config.py` (Pydantic Settings) | Env-driven config |
| 1.4 | Write `src/db/connection.py` (asyncpg pool) | DB connection pool works |
| 1.5 | Write `src/auth/jwt_handler.py` + `middleware.py` | JWT decode, user extraction |
| 1.6 | Write `src/main.py` (FastAPI app factory) | App starts, health check responds |
| 1.7 | Write `scripts/generate_jwt.py` | Can create test tokens |
| 1.8 | Write `tests/test_auth.py` | Auth tests pass |

**Verification:** `docker compose up` → `curl localhost:8000/health` → 200 OK. Auth tests green.

---

### Phase 2: Ingestion Pipeline (Days 3-4)

**Goal:** Upload documents → parse → chunk → embed → store with metadata.

| Step | Task | Deliverable |
|------|------|-------------|
| 2.1 | Write `src/ingest/parser.py` (PDF, TXT, DOCX, MD) | Text extraction from all formats |
| 2.2 | Write `src/ingest/chunker.py` | Chunks with configurable size/overlap |
| 2.3 | Write `src/ingest/embedder.py` (OpenAI + local interface) | Embeddings generated |
| 2.4 | Write `src/ingest/service.py` (orchestrator) | Full pipeline: parse → chunk → embed → DB |
| 2.5 | Write `src/api/routes/upload.py` | `POST /upload` endpoint |
| 2.6 | Write `src/api/dependencies.py` (role check: admin/editor only) | Non-admins get 403 |
| 2.7 | Write `tests/test_ingest.py` + `tests/test_api_upload.py` | Ingestion tests pass |

**Verification:** Upload a PDF via API → check `chunks` table has rows with correct `customer_id` and `allowed_roles`.

---

### Phase 3: Retrieval + Permission Filtering (Days 5-6)

**Goal:** Query → embed → vector search → permission filter → return relevant chunks.

| Step | Task | Deliverable |
|------|------|-------------|
| 3.1 | Write `src/retrieve/vector_search.py` | pgvector cosine similarity query |
| 3.2 | Write `src/retrieve/permission_filter.py` | Tenant pre-filter + role post-filter |
| 3.3 | Write `src/retrieve/service.py` (orchestrator) | Full retrieval pipeline |
| 3.4 | Implement RLS policies (Phase 1 schema → add RLS) | DB-level tenant isolation |
| 3.5 | Write `tests/test_retrieve.py` | Retrieval + filtering tests pass |
| 3.6 | Write `tests/test_security.py` (cross-tenant leakage) | **Critical:** tenant A cannot see tenant B's chunks |
| 3.7 | Write `tests/test_rls.py` | RLS enforcement tests pass |

**Verification:** Seed two tenants with overlapping content. Query as tenant A → only tenant A's chunks returned. Query as viewer role → only public chunks returned.

---

### Phase 4: Answer Generation + Citations (Days 7-8)

**Goal:** Retrieved chunks → LLM prompt → grounded answer with citations.

| Step | Task | Deliverable |
|------|------|-------------|
| 4.1 | Write `src/answer/prompt_builder.py` | Context + citation instruction prompt |
| 4.2 | Write `src/answer/llm_client.py` (OpenAI + Ollama interface) | LLM calls work |
| 4.3 | Write `src/answer/citation_extractor.py` | Parse [1], [2] → structured citations |
| 4.4 | Write `src/answer/service.py` (orchestrator) | Full answer pipeline |
| 4.5 | Write `src/api/routes/query.py` | `POST /query` endpoint |
| 4.6 | Write `tests/test_answer.py` + `tests/test_api_query.py` | Answer + citation tests pass |

**Verification:** Upload a doc → query about its content → answer includes [1] citation linking to the correct doc_path and chunk.

---

### Phase 5: Audit Logging (Day 9)

**Goal:** Every query and upload is logged immutably.

| Step | Task | Deliverable |
|------|------|-------------|
| 5.1 | Write `src/audit/logger.py` | Append-only audit log writer |
| 5.2 | Wire audit logging into query + upload routes | Every request logged |
| 5.3 | Write `src/api/routes/audit.py` (admin-only GET) | Admins can query audit trail |
| 5.4 | Write `tests/test_audit.py` | Immutability + completeness tests pass |

**Verification:** Make 5 queries as different users → `GET /audit?user_id=X` returns only that user's queries. Attempt to UPDATE audit_logs → exception.

---

### Phase 6: Hardening + Documentation (Days 10-11)

**Goal:** Production-ready, documented, edge cases handled.

| Step | Task | Deliverable |
|------|------|-------------|
| 6.1 | Add rate limiting (slowapi or nginx) | Prevents abuse |
| 6.2 | Add request validation + error handling | Clean 4xx/5xx responses |
| 6.3 | Add health check + readiness probe | `/health` and `/ready` endpoints |
| 6.4 | Add structured logging (structlog or loguru) | JSON logs for observability |
| 6.5 | Write `README.md` (setup, usage, API docs) | Someone else can run it |
| 6.6 | Add `.env.example` with all env vars | Configuration is documented |
| 6.7 | Run full test suite end-to-end | All tests green |
| 6.8 | Security review: re-run `test_security.py` + manual penetration | No leaks found |

**Verification:** Fresh `docker compose up` → run `scripts/seed_test_data.py` → upload a doc → query it → check audit log → all works without manual intervention.

---

## 7. Component Specifications

### 7.1 Auth Middleware (`src/auth/middleware.py`)

```python
"""
Extracts user identity from JWT on every request.
Attaches User object to request.state for downstream handlers.
"""

from starlette.middleware.base import BaseHTTPMiddleware
from starlette.responses import JSONResponse

class AuthMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        # Skip auth for health check
        if request.url.path in ("/health", "/ready"):
            return await call_next(request)

        token = request.headers.get("Authorization", "").replace("Bearer ", "")
        if not token:
            return JSONResponse({"detail": "Missing token"}, status_code=401)

        try:
            payload = verify_jwt(token)
            user = await get_user_from_payload(payload)
            request.state.user = user

            # Set tenant context for RLS
            async with db.acquire() as conn:
                await conn.execute(
                    f"SET LOCAL app.current_customer_id = '{user.customer_id}'"
                )

            return await call_next(request)
        except ExpiredTokenError:
            return JSONResponse({"detail": "Token expired"}, status_code=401)
        except InvalidTokenError:
            return JSONResponse({"detail": "Invalid token"}, status_code=401)
```

**JWT Claims Structure:**
```json
{
  "sub": "user-external-id-123",      // maps to users.external_id
  "email": "jane@company.com",
  "customer_id": "acme-corp",          // tenant
  "roles": ["viewer"],                 // RBAC roles
  "exp": 1735689600,                   // expiry
  "iat": 1735603200                    // issued at
}
```

### 7.2 Ingestion Service (`src/ingest/service.py`)

```python
"""
Full ingestion pipeline: file → text → chunks → embeddings → database.

Security: customer_id and allowed_roles are stamped on every chunk.
These fields are NEVER derived from the document content — they come
from the authenticated user's session and the upload request.
"""

async def ingest_document(
    file: UploadFile,
    customer_id: str,
    allowed_roles: list[str],
    uploaded_by: UUID,
) -> IngestResult:
    # 1. Read + hash file (for dedup)
    content = await file.read()
    file_hash = hashlib.sha256(content).hexdigest()

    # 2. Check for duplicate
    existing = await db.fetchrow(
        "SELECT id FROM documents WHERE customer_id=$1 AND file_hash=$2",
        customer_id, file_hash
    )
    if existing:
        raise DuplicateDocumentError(existing["id"])

    # 3. Parse text
    text = parse_file(content, file.filename)

    # 4. Chunk
    chunks = chunk_text(text)  # List[str]

    # 5. Embed all chunks (batch for efficiency)
    embeddings = await embed_batch(chunks)

    # 6. Store document metadata
    doc_id = uuid4()
    await db.execute(
        """INSERT INTO documents (id, customer_id, file_name, file_path,
           file_type, file_hash, chunk_count, allowed_roles, uploaded_by)
           VALUES ($1,$2,$3,$4,$5,$6,$7,$8,$9)""",
        doc_id, customer_id, file.filename, f"{customer_id}/{doc_id}/{file.filename}",
        detect_type(file.filename), file_hash, len(chunks), allowed_roles, uploaded_by
    )

    # 7. Store chunks with embeddings + ACCESS METADATA
    rows = [
        (uuid4(), doc_id, customer_id, i, chunk, embedding, allowed_roles)
        for i, (chunk, embedding) in enumerate(zip(chunks, embeddings))
    ]
    await db.executemany(
        """INSERT INTO chunks (id, doc_id, customer_id, chunk_idx, content,
           embedding, allowed_roles)
           VALUES ($1,$2,$3,$4,$5,$6,$7)""",
        rows
    )

    # 8. Audit log
    await audit_logger.log(
        user_id=uploaded_by, customer_id=customer_id, action="upload",
        extra={"doc_id": str(doc_id), "chunk_count": len(chunks),
               "file_name": file.filename}
    )

    return IngestResult(doc_id=doc_id, chunk_count=len(chunks))
```

### 7.3 Retrieval + Permission Filter (`src/retrieve/service.py`)

```python
"""
Retrieves relevant chunks with strict permission enforcement.

Two-layer filtering:
  1. PRE-FILTER (SQL WHERE): customer_id = ?  → hard tenant boundary
  2. POST-FILTER (Python):    allowed_roles ∩ user.roles ≠ ∅  → RBAC

The pre-filter is in SQL so RLS also enforces it at the DB level.
The post-filter is in Python because array intersection in SQL
combined with vector ORDER BY can produce poor query plans.
"""

async def retrieve(
    query: str,
    user: User,
    top_k: int = 5,
    oversample: int = 3,   # retrieve 3× candidates, filter, return top_k
) -> list[RetrievedChunk]:
    # 1. Embed the query
    query_embedding = await embed_query(query)

    # 2. Vector search with tenant pre-filter (RLS also enforces this)
    candidates = await db.fetch(
        """
        SELECT id, doc_id, chunk_idx, content, allowed_roles,
               embedding <=> $1 AS distance
        FROM chunks
        WHERE customer_id = $2
        ORDER BY embedding <=> $1
        LIMIT $3
        """,
        query_embedding, user.customer_id, top_k * oversample
    )

    # 3. Role-based post-filter
    user_role_set = set(user.roles)
    filtered = [
        c for c in candidates
        if set(c["allowed_roles"]) & user_role_set
    ]

    # 4. Return top_k
    return [
        RetrievedChunk(
            id=c["id"], doc_id=c["doc_id"], chunk_idx=c["chunk_idx"],
            content=c["content"], distance=c["distance"]
        )
        for c in filtered[:top_k]
    ]
```

### 7.4 Answer Generation + Citations (`src/answer/service.py`)

```python
"""
Generates a grounded answer from retrieved chunks with mandatory citations.

The prompt explicitly instructs the LLM to:
  - Use ONLY the provided context
  - Cite sources as [1], [2], etc.
  - Say "I don't have enough information" if context is insufficient
"""

CITATION_PROMPT_TEMPLATE = """\
You are a secure document assistant. Answer the user's question using \
ONLY the context provided below. Do not use any prior knowledge.

Rules:
1. If the context contains the answer, respond with the answer and cite \
sources using [1], [2], etc. matching the source numbers.
2. If the context does not contain enough information, respond with: \
"I don't have enough information to answer this question based on the \
available documents."
3. Do not fabricate information or cite sources that don't exist.
4. Keep the answer concise and directly address the question.

Context:
{context}

Question: {question}

Answer:
"""

async def generate_answer(
    question: str,
    retrieved_chunks: list[RetrievedChunk],
) -> AnswerResult:
    # 1. Build context block with numbered sources
    context_parts = []
    citations_meta = []
    for i, chunk in enumerate(retrieved_chunks):
        context_parts.append(f"[{i+1}] {chunk.content}")
        citations_meta.append({
            "index": i + 1,
            "doc_id": str(chunk.doc_id),
            "chunk_id": str(chunk.id),
            "chunk_idx": chunk.chunk_idx,
            "snippet": chunk.content[:200] + "..." if len(chunk.content) > 200 else chunk.content,
            "distance": float(chunk.distance),
        })

    context = "\n\n".join(context_parts)
    prompt = CITATION_PROMPT_TEMPLATE.format(context=context, question=question)

    # 2. Call LLM
    answer_text = await llm_client.generate(prompt)

    # 3. Return answer + structured citations
    return AnswerResult(
        answer=answer_text,
        citations=citations_meta,
        retrieved_chunk_ids=[c.id for c in retrieved_chunks],
    )
```

### 7.5 Audit Logger (`src/audit/logger.py`)

```python
"""
Append-only audit logger. Every query, upload, and delete is recorded.

The audit_logs table has triggers that prevent UPDATE and DELETE,
making it tamper-evident. This is critical for compliance scenarios
where you need to prove who accessed what data and when.
"""

async def log_query(
    user_id: UUID,
    customer_id: str,
    query_text: str,
    retrieved_chunk_ids: list[UUID],
    answer_text: str,
    citations: list[dict],
    latency_ms: int,
    success: bool = True,
    error_message: str | None = None,
):
    await db.execute(
        """INSERT INTO audit_logs
           (user_id, customer_id, action, query_text, retrieved_chunk_ids,
            answer_text, citations, latency_ms, success, error_message)
           VALUES ($1,$2,'query',$3,$4,$5,$6,$7,$8,$9)""",
        user_id, customer_id, query_text, retrieved_chunk_ids,
        answer_text, json.dumps(citations), latency_ms, success, error_message
    )

async def log_action(
    user_id: UUID,
    customer_id: str,
    action: str,  # 'upload', 'delete', etc.
    extra: dict | None = None,
):
    await db.execute(
        """INSERT INTO audit_logs
           (user_id, customer_id, action, citations)
           VALUES ($1,$2,$3,$4)""",
        user_id, customer_id, action, json.dumps(extra or {})
    )
```

---

## 8. Security Model

### 8.1 Threat Model

| Threat | Mitigation | Layer |
|--------|-----------|-------|
| **Cross-tenant data leakage** (tenant A sees tenant B's docs) | 1. SQL `WHERE customer_id = ?` pre-filter  2. PostgreSQL RLS policies  3. Test suite with explicit leakage tests | App + DB |
| **Unauthorized role access** (viewer sees admin-only docs) | `allowed_roles` array intersection post-filter; upload route checks `admin`/`editor` role | App |
| **Prompt injection** (user crafts query to bypass context) | LLM instructed to use ONLY provided context; system prompt is immutable; user input is in the "Question" field only | App |
| **Audit log tampering** (someone edits/deletes audit records) | DB triggers block UPDATE/DELETE on `audit_logs`; only INSERT allowed | DB |
| **JWT forgery** (fake token to impersonate another tenant) | JWT verified with secret/public key; `customer_id` in token must match `users.customer_id` in DB | App |
| **Vector search bypass** (direct DB access skips permission filter) | RLS policies enforce tenant isolation even with direct DB access; application DB user has limited privileges | DB |
| **Data exfiltration via embeddings** (reverse-engineer doc content from vectors) | Embeddings are stored server-side; never returned to client; API only returns answer text + citations | App |

### 8.2 Permission Hierarchy

```
Roles (within a tenant):
  admin   → upload, delete, query, view audit logs
  editor  → upload, query
  viewer  → query (only chunks where 'viewer' ∈ allowed_roles)

Document-level access:
  Each document has allowed_roles[] (set at upload time)
  Each chunk inherits allowed_roles from its parent document
  At query time: chunk is accessible iff allowed_roles ∩ user.roles ≠ ∅
```

### 8.3 Defense in Depth

```
Layer 1: Application   → SQL WHERE customer_id = ? + role post-filter
Layer 2: Database      → RLS policies (even if app has a bug)
Layer 3: Network       → API behind reverse proxy, DB not exposed externally
Layer 4: Audit         → Immutable log proves who accessed what (compliance)
```

---

## 9. API Specification

### 9.1 `POST /upload`

Upload a document for ingestion. Requires `admin` or `editor` role.

```
Headers:
  Authorization: Bearer <JWT>

Body (multipart/form-data):
  file: <binary file>
  allowed_roles: "admin,editor,viewer"    # comma-separated

Response 201:
{
  "doc_id": "uuid",
  "chunk_count": 42,
  "file_name": "contract.pdf",
  "file_type": "pdf"
}

Response 403: Not admin/editor
Response 409: Duplicate document (same hash already exists)
```

### 9.2 `POST /query`

Ask a question. Returns grounded answer with citations.

```
Headers:
  Authorization: Bearer <JWT>

Body:
{
  "question": "What is the termination clause in the ACME contract?",
  "top_k": 5                          // optional, default 5
}

Response 200:
{
  "answer": "The contract can be terminated with 30 days notice [1]...",
  "citations": [
    {
      "index": 1,
      "doc_id": "uuid",
      "chunk_id": "uuid",
      "chunk_idx": 14,
      "snippet": "Either party may terminate this agreement...",
      "distance": 0.1234
    }
  ],
  "latency_ms": 842
}

Response 200 (no relevant context):
{
  "answer": "I don't have enough information to answer this question...",
  "citations": [],
  "latency_ms": 120
}
```

### 9.3 `GET /documents`

List documents for the current tenant.

```
Headers:
  Authorization: Bearer <JWT>

Query params:
  ?page=1&limit=20

Response 200:
{
  "documents": [
    {
      "id": "uuid",
      "file_name": "contract.pdf",
      "chunk_count": 42,
      "allowed_roles": ["admin", "editor"],
      "created_at": "2025-01-15T10:30:00Z"
    }
  ],
  "total": 15,
  "page": 1
}
```

### 9.4 `DELETE /documents/{doc_id}`

Delete a document and all its chunks. Requires `admin` role.

```
Response 204: Deleted
Response 403: Not admin
Response 404: Document not found (or belongs to another tenant)
```

### 9.5 `GET /audit`

Query the audit trail. Requires `admin` role.

```
Query params:
  ?user_id=uuid          # filter by user
  ?action=query          # filter by action type
  ?from=2025-01-01       # date range start
  ?to=2025-01-31         # date range end
  ?page=1&limit=50

Response 200:
{
  "logs": [
    {
      "id": "uuid",
      "timestamp": "2025-01-15T10:30:00Z",
      "user_id": "uuid",
      "action": "query",
      "query_text": "What is the termination clause?",
      "retrieved_chunk_ids": ["uuid", "uuid"],
      "answer_text": "The contract can be terminated...",
      "citations": [...],
      "latency_ms": 842,
      "success": true
    }
  ],
  "total": 1234,
  "page": 1
}
```

### 9.6 `GET /health`

```
Response 200:
{
  "status": "healthy",
  "database": "connected",
  "version": "1.0.0"
}
```

---

## 10. Testing Strategy

### 10.1 Test Categories

| Category | What It Proves | Priority |
|----------|---------------|----------|
| **Security tests** | No cross-tenant leakage; role enforcement works | P0 — must pass before any merge |
| **Integration tests** | Full pipeline: upload → query → answer → audit | P0 |
| **Unit tests** | Individual components work in isolation | P1 |
| **API tests** | HTTP endpoints return correct status codes + payloads | P1 |
| **RLS tests** | Database-level isolation holds even without app logic | P0 |

### 10.2 Critical Security Tests (`tests/test_security.py`)

```python
"""
These tests are the backbone of the security claim.
If ANY of these fail, the system is not safe to deploy.
"""

async def test_tenant_a_cannot_retrieve_tenant_b_chunks():
    """Tenant A uploads a doc. Tenant B queries for the same content.
    Tenant B must get zero results."""
    # Upload as tenant A
    await upload_doc(tenant_a_token, content="ACME secret merger plans")
    # Query as tenant B with the exact same text
    result = await query(tenant_b_token, "ACME secret merger plans")
    assert len(result["citations"]) == 0
    assert "I don't have enough information" in result["answer"]


async def test_viewer_cannot_access_admin_only_chunks():
    """Upload a doc with allowed_roles=['admin']. Query as viewer.
    Viewer must not see those chunks."""
    await upload_doc(admin_token, content="Confidential salary data",
                     allowed_roles=["admin"])
    result = await query(viewer_token, "Confidential salary data")
    assert len(result["citations"]) == 0


async def test_rls_blocks_cross_tenant_direct_db():
    """Even with direct DB access (bypassing app), RLS prevents
    cross-tenant reads."""
    conn = await get_db_connection(tenant="tenant_a")
    chunks = await conn.fetch("SELECT * FROM chunks")
    assert all(c["customer_id"] == "tenant_a" for c in chunks)


async def test_audit_log_is_immutable():
    """Attempt to UPDATE and DELETE audit log rows → must raise."""
    with pytest.raises(Exception):
        await db.execute("UPDATE audit_logs SET answer_text = 'tampered'")
    with pytest.raises(Exception):
        await db.execute("DELETE FROM audit_logs")


async def test_expired_jwt_rejected():
    token = generate_jwt(customer_id="acme", roles=["viewer"], exp_offset=-3600)
    response = await client.post("/query", json={"question": "test"},
                                  headers={"Authorization": f"Bearer {token}"})
    assert response.status_code == 401


async def test_jwt_with_wrong_customer_id_rejected():
    """JWT claims customer_id=X but user in DB has customer_id=Y.
    Must reject — prevents token tampering."""
    token = generate_jwt(customer_id="fake-tenant", roles=["admin"])
    response = await client.post("/query", json={"question": "test"},
                                  headers={"Authorization": f"Bearer {token}"})
    assert response.status_code == 401
```

### 10.3 Test Fixtures (`tests/conftest.py`)

```python
"""
Test setup:
  - Spin up a test Postgres+pgvector (docker or testcontainers)
  - Run schema.sql
  - Seed two tenants (acme, globex) with users in each
  - Provide JWT tokens for each user role
  - Provide a FastAPI TestClient with overridden DB
"""

@pytest.fixture
async def db():
    """Fresh database for each test module."""
    conn = await asyncpg.connect(TEST_DB_URL)
    await conn.execute(open("src/db/schema.sql").read())
    yield conn
    await conn.close()

@pytest.fixture
def admin_token_acme():
    return generate_jwt(customer_id="acme", roles=["admin", "editor", "viewer"])

@pytest.fixture
def viewer_token_acme():
    return generate_jwt(customer_id="acme", roles=["viewer"])

@pytest.fixture
def admin_token_globex():
    return generate_jwt(customer_id="globex", roles=["admin", "editor", "viewer"])
```

---

## 11. Deployment

### 11.1 `docker-compose.yml`

```yaml
version: "3.9"

services:
  postgres:
    image: pgvector/pgvector:pg16
    environment:
      POSTGRES_DB: secure_rag
      POSTGRES_USER: rag_user
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-changeme}
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./docker/postgres/init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U rag_user -d secure_rag"]
      interval: 5s
      timeout: 5s
      retries: 5

  app:
    build:
      context: .
      dockerfile: docker/Dockerfile
    ports:
      - "8000:8000"
    environment:
      DATABASE_URL: postgres://rag_user:${POSTGRES_PASSWORD:-changeme}@postgres:5432/secure_rag
      JWT_SECRET: ${JWT_SECRET:-dev-secret-change-in-prod}
      OPENAI_API_KEY: ${OPENAI_API_KEY}
      EMBEDDING_MODEL: text-embedding-3-small
      LLM_MODEL: gpt-4o-mini
    depends_on:
      postgres:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 10s
      timeout: 5s
      retries: 3

volumes:
  postgres_data:
```

### 11.2 `docker/Dockerfile`

```dockerfile
FROM python:3.12-slim

WORKDIR /app

# Install system deps for PDF parsing
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential libpq-dev curl \
    && rm -rf /var/lib/apt/lists/*

# Install Python deps
COPY pyproject.toml .
RUN pip install --no-cache-dir -e .

# Copy source
COPY src/ ./src/
COPY scripts/ ./scripts/

EXPOSE 8000

CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### 11.3 `docker/postgres/init.sql`

```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "vector";
-- Schema tables are created by the app on startup (or scripts/init_db.py)
```

### 11.4 `.env.example`

```env
# Database
POSTGRES_PASSWORD=changeme-in-production

# JWT
JWT_SECRET=your-256-bit-secret
JWT_ALGORITHM=HS256
JWT_EXPIRY_HOURS=24

# OpenAI (or set USE_LOCAL_EMBEDDINGS=true for BGE)
OPENAI_API_KEY=sk-...
EMBEDDING_MODEL=text-embedding-3-small
LLM_MODEL=gpt-4o-mini

# Local models (alternative to OpenAI)
# USE_LOCAL_EMBEDDINGS=true
# USE_LOCAL_LLM=true
# OLLAMA_HOST=http://localhost:11434

# App
LOG_LEVEL=info
MAX_UPLOAD_SIZE_MB=50
```

### 11.5 Production Considerations

| Concern | Recommendation |
|---------|---------------|
| **TLS** | Terminate TLS at reverse proxy (nginx/traefik); app listens on HTTP |
| **DB backups** | `pg_dump` cron job; pgvector indexes are included in dump |
| **Secrets** | Use Docker secrets or a vault; never bake secrets into images |
| **DB user privileges** | App uses a limited-privilege user (SELECT, INSERT on chunks/documents; INSERT-only on audit_logs) |
| **Rate limiting** | nginx `limit_req` or slowapi middleware (e.g. 10 queries/min per user) |
| **Monitoring** | Structured JSON logs → Loki/ELK; `/health` endpoint for k8s probes |
| **File storage** | v1 uses local disk; v2 should use S3 with per-tenant bucket prefixes |

---

## 12. Roadmap & Milestones

```
Week 1 (Days 1-6)
├── Phase 1: Foundation           [█░░░░░░░░░] Days 1-2
├── Phase 2: Ingestion            [░░█░░░░░░░] Days 3-4
└── Phase 3: Retrieval + Perms    [░░░░█░░░░░] Days 5-6

Week 2 (Days 7-11)
├── Phase 4: Answer + Citations   [░░░░░░█░░░] Days 7-8
├── Phase 5: Audit Logging        [░░░░░░░░█░] Day 9
└── Phase 6: Hardening + Docs     [░░░░░░░░░█] Days 10-11

Future (Post-v1)
├── Web UI (chat interface)
├── Streaming responses (SSE)
├── Multi-language document support
├── S3 file storage with per-tenant prefixes
├── Document versioning (keep history, query latest)
├── Fine-tuned embeddings for domain-specific vocabulary
├── Kubernetes deployment manifests
└── OAuth2/OIDC integration (replace simple JWT with Keycloak/Auth0)
```

### Milestone Summary

| Milestone | Day | Deliverable |
|-----------|-----|-------------|
| M1: Skeleton running | Day 2 | Docker Compose up, auth works, health check passes |
| M2: Documents ingestible | Day 4 | Upload PDF/TXT → chunks in DB with metadata |
| M3: Secure retrieval | Day 6 | Query returns only permitted chunks; security tests green |
| M4: Full Q&A | Day 8 | Upload → query → grounded answer with citations |
| M5: Audit trail | Day 9 | Every query logged; admin can view audit trail |
| M6: Production-ready | Day 11 | Full test suite green, documented, deployable |

---

## 13. Risk Register

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|-----------|
| **pgvector performance degrades at scale** (>1M chunks) | Medium | Medium | HNSW index helps; partition chunks table by `customer_id` if needed; consider Qdrant for very large tenants |
| **LLM ignores citation instructions** | Medium | High | Use structured output (JSON mode) with explicit citation schema; post-validate that cited indices exist |
| **Prompt injection bypasses context-only rule** | Low | High | User input is only in the "Question" field; system prompt is immutable; add output validation for known injection patterns |
| **JWT secret compromise** | Low | Critical | Rotate secrets; use RS256 (asymmetric) in production so signing key is never on the API server |
| **Large file upload DoS** | Medium | Medium | Enforce `MAX_UPLOAD_SIZE_MB`; stream uploads; rate-limit upload endpoint |
| **Embedding API downtime** (OpenAI) | Low | High | Implement retry with exponential backoff; cache embeddings by content hash; fallback to local BGE model |
| **Audit log grows unbounded** | High | Low | Partition by month; archive old partitions to cold storage; add retention policy (e.g. 2 years) |

---

## Appendix A: Environment Setup (Quick Start)

```bash
# 1. Clone
git clone <repo-url> secure-rag
cd secure-rag

# 2. Configure
cp .env.example .env
# Edit .env: set OPENAI_API_KEY, JWT_SECRET, POSTGRES_PASSWORD

# 3. Start
docker compose up -d

# 4. Wait for health
curl http://localhost:8000/health

# 5. Generate a test JWT
python scripts/generate_jwt.py --customer acme --roles admin,editor,viewer

# 6. Upload a document
curl -X POST http://localhost:8000/upload \
  -H "Authorization: Bearer <TOKEN>" \
  -F "file=@./test-doc.pdf" \
  -F "allowed_roles=admin,editor,viewer"

# 7. Query
curl -X POST http://localhost:8000/query \
  -H "Authorization: Bearer <TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{"question": "What is this document about?"}'

# 8. View audit trail (admin only)
curl http://localhost:8000/audit \
  -H "Authorization: Bearer <ADMIN_TOKEN>"
```

## Appendix B: Key Design Decisions

| Decision | Chosen | Rejected Alternative | Why |
|----------|--------|----------------------|-----|
| Vector DB | pgvector (Postgres) | Pinecone, Qdrant, Weaviate | SQL-level permission filtering is the security mechanism; separate vector DB creates sync gap |
| Filtering strategy | Hybrid (SQL pre-filter + Python post-filter) | Pure pre-filter | Post-filter preserves recall when role filter is restrictive; pre-filter handles hard tenant boundary |
| LLM grounding | Prompt-level instruction + citation validation | Fine-tuned model | Prompt-level is simpler, cheaper, and auditable; fine-tuning is overkill for v1 |
| Audit log | DB table with triggers | External log file / SIEM | DB table is queryable via API; triggers guarantee immutability; SIEM integration is a v2 concern |
| Auth | JWT with customer_id + roles claims | Session-based auth | Stateless, works with API gateways, standard for microservices |
| Chunking | RecursiveCharacterTextSplitter (1000 chars, 200 overlap) | Semantic chunking, sentence-level | Recursive is well-tested, predictable, and good enough for v1; semantic chunking adds complexity |
| Embedding dimension | 1536 (OpenAI text-embedding-3-small) | 768 (BGE), 384 (MiniLM) | OpenAI is reliable and well-documented; BGE is the local fallback for on-prem deployments |
